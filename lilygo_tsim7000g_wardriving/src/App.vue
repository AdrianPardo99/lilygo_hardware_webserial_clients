<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue';
import { buildCsv, makeCsvFilename } from './csv';
import { flushSerialCarry, parseSerialChunk, parseSerialLine } from './parser';
import type { BleRecord, LteRecord, ParsedSerialEvent, RawLogLine, ScanType, WifiRecord } from './types';

const baudRates = [9600, 19200, 38400, 57600, 115200, 230400, 460800, 921600];
const maxRawLines = 1000;
const isSerialSupported = computed(() => 'serial' in navigator && Boolean(navigator.serial));

const baudRate = ref(115200);
const isConnected = ref(false);
const isConnecting = ref(false);
const statusMessage = ref('Disconnected');
const errorMessage = ref('');
const ignoredCount = ref(0);
const rawLogs = ref<RawLogLine[]>([]);
const lteRows = ref<LteRecord[]>([]);
const wifiRows = ref<WifiRecord[]>([]);
const bleRows = ref<BleRecord[]>([]);
const terminalRef = ref<HTMLElement | null>(null);
const colorMode = ref<'white' | 'black'>(getInitialColorMode());

let port: SerialPort | null = null;
let reader: ReadableStreamDefaultReader<string> | null = null;
let readCarry = '';
let rawLogId = 0;
let isDisconnecting = false;

const lteFilename = computed(() => makeCsvFilename('lte'));
const wifiFilename = computed(() => makeCsvFilename('wifi'));
const bleFilename = computed(() => makeCsvFilename('ble'));
const colorModeLabel = computed(() => (colorMode.value === 'black' ? 'White mode' : 'Black mode'));

watch(
  colorMode,
  (mode) => {
    document.documentElement.dataset.theme = mode;
    localStorage.setItem('lilygo-color-mode', mode);
  },
  { immediate: true }
);

function getInitialColorMode(): 'white' | 'black' {
  const saved = localStorage.getItem('lilygo-color-mode');

  if (saved === 'white' || saved === 'black') {
    return saved;
  }

  return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'black' : 'white';
}

function toggleColorMode(): void {
  colorMode.value = colorMode.value === 'black' ? 'white' : 'black';
}

async function connectSerial(): Promise<void> {
  if (!navigator.serial) {
    errorMessage.value = 'Web Serial is not available in this browser. Use Chrome or Edge over HTTPS or localhost.';
    return;
  }

  isConnecting.value = true;
  errorMessage.value = '';
  statusMessage.value = 'Requesting serial port...';

  try {
    port = await navigator.serial.requestPort();
    await port.open({ baudRate: baudRate.value });

    if (!port.readable) {
      throw new Error('The selected serial port is not readable.');
    }

    isConnected.value = true;
    statusMessage.value = `Connected at ${baudRate.value} baud`;
    startReadLoop(port);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    errorMessage.value = message;
    statusMessage.value = 'Disconnected';
    await closePort();
  } finally {
    isConnecting.value = false;
  }
}

async function disconnectSerial(): Promise<void> {
  isDisconnecting = true;
  statusMessage.value = 'Disconnecting...';
  errorMessage.value = '';

  try {
    await reader?.cancel();
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : String(error);
  } finally {
    await closePort();
    isDisconnecting = false;
    isConnected.value = false;
    statusMessage.value = 'Disconnected';
  }
}

function startReadLoop(activePort: SerialPort): void {
  const decoder = new TextDecoderStream();
  const decoderWritable = decoder.writable as WritableStream<Uint8Array>;
  const inputClosed = activePort.readable?.pipeTo(decoderWritable).catch((error: unknown) => {
    if (!isDisconnecting) {
      errorMessage.value = error instanceof Error ? error.message : String(error);
    }
  });

  reader = decoder.readable.getReader();
  void readLoop(inputClosed);
}

async function readLoop(inputClosed?: Promise<void>): Promise<void> {
  try {
    while (reader) {
      const { value, done } = await reader.read();

      if (done) {
        break;
      }

      if (value) {
        consumeChunk(value);
      }
    }
  } catch (error) {
    if (!isDisconnecting) {
      errorMessage.value = error instanceof Error ? error.message : String(error);
    }
  } finally {
    consumeEvents(flushSerialCarry(readCarry));
    readCarry = '';
    reader?.releaseLock();
    reader = null;
    await inputClosed?.catch(() => undefined);

    if (!isDisconnecting) {
      await closePort();
      isConnected.value = false;
      statusMessage.value = 'Disconnected';
    }
  }
}

function consumeChunk(chunk: string): void {
  const parsed = parseSerialChunk(chunk, readCarry);
  readCarry = parsed.carry;
  consumeEvents(parsed.events);
}

function consumeEvents(events: ParsedSerialEvent[]): void {
  for (const event of events) {
    if (event.type === 'lte') {
      lteRows.value.unshift(event.record);
    } else if (event.type === 'wifi') {
      wifiRows.value.unshift(event.record);
    } else if (event.type === 'ble') {
      bleRows.value.unshift(event.record);
    } else if (event.type === 'ignoredInvalidCoordinates') {
      ignoredCount.value += 1;
    }

    appendRawLog(lineForEvent(event));
  }
}

function lineForEvent(event: ParsedSerialEvent): string {
  if (event.type === 'log' || event.type === 'header' || event.type === 'ignoredInvalidCoordinates') {
    return event.line;
  }
  return event.line;
}

function appendRawLog(text: string): void {
  rawLogs.value.push({
    id: rawLogId,
    text,
    receivedAt: new Date().toLocaleTimeString()
  });
  rawLogId += 1;

  if (rawLogs.value.length > maxRawLines) {
    rawLogs.value.splice(0, rawLogs.value.length - maxRawLines);
  }

  void nextTick(() => {
    if (terminalRef.value) {
      terminalRef.value.scrollTop = terminalRef.value.scrollHeight;
    }
  });
}

async function closePort(): Promise<void> {
  try {
    await port?.close();
  } catch {
    // The read loop may already have closed or detached the port.
  } finally {
    port = null;
  }
}

function clearRows(type: ScanType): void {
  if (type === 'lte') {
    lteRows.value = [];
  } else if (type === 'wifi') {
    wifiRows.value = [];
  } else {
    bleRows.value = [];
  }
}

function clearTerminal(): void {
  rawLogs.value = [];
  ignoredCount.value = 0;
}

function downloadRows(type: 'lte'): void;
function downloadRows(type: 'wifi'): void;
function downloadRows(type: 'ble'): void;
function downloadRows(type: ScanType): void {
  const rows = type === 'lte' ? lteRows.value : type === 'wifi' ? wifiRows.value : bleRows.value;
  const csv = buildCsv(type, rows);
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const anchor = document.createElement('a');

  anchor.href = url;
  anchor.download = makeCsvFilename(type);
  anchor.click();
  URL.revokeObjectURL(url);
}

function loadSample(): void {
  const sample = [
    '[modem] AT sync OK',
    '[gps] GPS power enabled',
    'Source,Timestamp,Tecnología,Estado,MCC,MNC,LAC,CellID,Banda,RSSI,RSRP,RSRQ,SINR,Operador,Longitud,Latitud',
    'Source,Timestamp,Lat,Long,SSID,BSSID,Canal,Señal,Seguridad',
    'Source,Timestamp,Lat,Long,Dirección,RSSI,Nombre',
    '[ble] logged 1 devices'
  ];

  consumeEvents(sample.map((line) => parseSerialLine(line)));
}
</script>

<template>
  <main class="app-shell">
    <section class="topbar">
      <div>
        <p class="eyebrow">LilyGO Web Serial</p>
        <h1>Wardriving capture client</h1>
        <p class="intro">
          Read serial output, classify LTE, WiFi, and BLE rows, and export only records with usable coordinates.
        </p>
      </div>

      <div class="connection-panel">
        <label>
          Baud rate
          <select v-model.number="baudRate" :disabled="isConnected || isConnecting">
            <option v-for="rate in baudRates" :key="rate" :value="rate">{{ rate }}</option>
          </select>
        </label>

        <button v-if="!isConnected" type="button" :disabled="isConnecting || !isSerialSupported" @click="connectSerial">
          {{ isConnecting ? 'Connecting...' : 'Connect serial' }}
        </button>
        <button v-else type="button" class="secondary" @click="disconnectSerial">Disconnect</button>
        <button type="button" class="secondary" :aria-pressed="colorMode === 'black'" @click="toggleColorMode">
          {{ colorModeLabel }}
        </button>
      </div>
    </section>

    <section v-if="!isSerialSupported" class="notice">
      Web Serial is available in Chromium browsers such as Chrome and Edge on HTTPS or localhost.
    </section>

    <section class="status-strip">
      <span :class="['status-dot', { online: isConnected }]"></span>
      <span>{{ statusMessage }}</span>
      <span>{{ ignoredCount }} zero-coordinate rows ignored</span>
      <button type="button" class="ghost" @click="loadSample">Load sample</button>
      <button type="button" class="ghost" @click="clearTerminal">Clear terminal</button>
    </section>

    <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    <section class="terminal-section">
      <header>
        <div>
          <p class="eyebrow">Raw serial</p>
          <h2>{{ rawLogs.length }} lines</h2>
        </div>
      </header>
      <div ref="terminalRef" class="terminal" aria-live="polite">
        <p v-if="rawLogs.length === 0" class="empty">Connect a device or load the sample data.</p>
        <p v-for="line in rawLogs" :key="line.id">
          <span>{{ line.receivedAt }}</span>
          {{ line.text }}
        </p>
      </div>
    </section>
    <section class="tables-grid">
      <article class="scan-section">
        <header>
          <div>
            <p class="eyebrow">LTE</p>
            <h2>{{ lteRows.length }} records</h2>
          </div>
          <div class="section-actions">
            <button type="button" class="secondary" :disabled="lteRows.length === 0" @click="downloadRows('lte')">
              Download {{ lteFilename }}
            </button>
            <button type="button" class="ghost" :disabled="lteRows.length === 0" @click="clearRows('lte')">Clear</button>
          </div>
        </header>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Lat</th>
                <th>Long</th>
                <th>Tech</th>
                <th>Operator</th>
                <th>RSSI</th>
                <th>RSRP</th>
                <th>SINR</th>
                <th>Captured</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in lteRows" :key="`${index}-${row.capturedAt}-${row.longitude}-${row.latitude}`">
                <td>{{ row.latitude }}</td>
                <td>{{ row.longitude }}</td>
                <td>{{ row.technology }}</td>
                <td>{{ row.operator || 'Unknown' }}</td>
                <td>{{ row.rssi }}</td>
                <td>{{ row.rsrp }}</td>
                <td>{{ row.sinr }}</td>
                <td>{{ new Date(row.capturedAt).toLocaleTimeString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </article>

      <article class="scan-section">
        <header>
          <div>
            <p class="eyebrow">WiFi</p>
            <h2>{{ wifiRows.length }} access points</h2>
          </div>
          <div class="section-actions">
            <button type="button" class="secondary" :disabled="wifiRows.length === 0" @click="downloadRows('wifi')">
              Download {{ wifiFilename }}
            </button>
            <button type="button" class="ghost" :disabled="wifiRows.length === 0" @click="clearRows('wifi')">Clear</button>
          </div>
        </header>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Lat</th>
                <th>Long</th>
                <th>SSID</th>
                <th>BSSID</th>
                <th>Channel</th>
                <th>Signal</th>
                <th>Security</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in wifiRows" :key="`${index}-${row.capturedAt}-${row.bssid}`">
                <td>{{ row.latitude }}</td>
                <td>{{ row.longitude }}</td>
                <td>{{ row.ssid || '(hidden)' }}</td>
                <td>{{ row.bssid }}</td>
                <td>{{ row.channel }}</td>
                <td>{{ row.signal }}</td>
                <td>{{ row.security }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </article>

      <article class="scan-section">
        <header>
          <div>
            <p class="eyebrow">BLE</p>
            <h2>{{ bleRows.length }} devices</h2>
          </div>
          <div class="section-actions">
            <button type="button" class="secondary" :disabled="bleRows.length === 0" @click="downloadRows('ble')">
              Download {{ bleFilename }}
            </button>
            <button type="button" class="ghost" :disabled="bleRows.length === 0" @click="clearRows('ble')">Clear</button>
          </div>
        </header>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Lat</th>
                <th>Long</th>
                <th>Address</th>
                <th>RSSI</th>
                <th>Name</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in bleRows" :key="`${index}-${row.capturedAt}-${row.address}`">
                <td>{{ row.latitude }}</td>
                <td>{{ row.longitude }}</td>
                <td>{{ row.address }}</td>
                <td>{{ row.rssi }}</td>
                <td>{{ row.name || 'Unknown' }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </article>
    </section>
  </main>
</template>
