# LilyGO TSIM 7600H-G 16 MB Web Serial Client

Frontend client for reading LilyGO TSIM 7600H-G 16 MB serial output from the browser, classifying wardriving data into LTE, WiFi, and BLE sections, and exporting filtered CSV files.

The app is read-only over Web Serial. It connects to the device, streams logs, parses known CSV rows, ignores records with invalid or zero coordinates, and downloads one CSV per data type.

## Requirements

- Node.js 22 or newer.
- npm.
- Chrome, Edge, or another Chromium browser with Web Serial support.
- The app must run from `localhost` or HTTPS for Web Serial access.

## Setup

```bash
npm install
```

## Development

```bash
npm run dev
```

Open the local Vite URL in a Chromium browser, connect the LilyGO serial device, select the baud rate, and start reading incoming output.

Default baud rate:

```text
115200
```

## Build And Preview

```bash
npm run build
npm run preview
```

## Tests

```bash
npm test
```

The tests cover serial parsing, zero-coordinate filtering, chunk buffering, CSV escaping, and export filenames.

## CSV Exports

Rows where both latitude and longitude are zero-like values, such as `0`, `0.0`, or `0.0000000`, are ignored and are not saved.

WiFi export:

```csv
Timestamp,Lat,Long,SSID,BSSID,Canal,Señal,Seguridad
```

BLE export:

```csv
Timestamp,Lat,Long,Dirección,RSSI,Nombre
```

LTE export:

```csv
Timestamp,Tecnología,Estado,MCC,MNC,LAC,CellID,Banda,RSSI,RSRP,RSRQ,SINR,Operador,Longitud,Latitud
```

Downloaded filenames use this format:

```text
lilygo_wifi_YYYYMMDD_HHmmss.csv
lilygo_ble_YYYYMMDD_HHmmss.csv
lilygo_lte_YYYYMMDD_HHmmss.csv
```

## Browser Use

1. Run `npm run dev`.
2. Open the Vite local URL in Chrome or Edge.
3. Click `Connect LilyGO USB`.
4. Choose the LilyGO serial port.
5. Watch raw serial logs and parsed LTE, WiFi, and BLE tables.
6. Download the CSV file for each section when records are available.

## Serial Port Selection

The main connection button asks Chrome for common USB serial bridges used by LilyGO and ESP32 boards:

- Silicon Labs CP210x: `0x10c4`
- WCH CH34x/CH9102: `0x1a86`
- FTDI: `0x0403`

If the TSIM 7600H-G does not appear, use `Show all ports` to open the unfiltered Web Serial chooser.

Chrome may log messages like this in DevTools:

```text
Chooser dialog is not displaying a port blocked by the Serial blocklist...
```

Those messages mean Chrome hid nearby Bluetooth/HID services from the Web Serial chooser. They are browser diagnostics, not parser errors and not a failed connection by themselves.
