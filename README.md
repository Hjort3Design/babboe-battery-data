# Battery Companion: Bluetooth + GitHub scan database

`index.html` is a single-file web page. It connects to `GWA_Battery_Diag` over Bluetooth LE,
runs scans, and commits each scan as a JSON file to a GitHub repo, which acts as the scan database.

```
ESP32 (BLE, read-only) ──► phone browser (Chrome) ──► GitHub API ──► data repo
                              ▲ keeps its own internet (mobile/home WiFi)
```

The ESP32 never holds a GitHub token. The token stays in the browser that uploads.

## Browser support

| Browser | Works? |
|---|---|
| Chrome / Edge on Android, Windows, macOS, ChromeOS | Yes, full BLE |
| iPhone / iPad | Not in Safari/Chrome (Apple's WebKit has no Web Bluetooth). Use the free **Bluefy** browser from the App Store, or **Import file** (see below) |
| Firefox | No Web Bluetooth. Use **Import file** instead (see below) |

Web Bluetooth only works on **HTTPS** pages. That is why this page can't live inside the
firmware: the firmware serves plain `http://192.168.4.1`.

## One-time setup

1. **Create the data repo**, e.g. `Hjort3Design/babboe-battery-data`:
   ```sh
   gh repo create Hjort3Design/babboe-battery-data --public --add-readme
   ```
   Public repos get free GitHub Pages. Pack serial numbers and EEPROM dumps will then be public.
   To keep the data private, use a private data repo and host `index.html` in a separate
   small public repo. The page can upload to any repo the token can write to.
2. **Put `index.html` at the repo root**, commit it, then enable Pages:
   *Settings → Pages → Deploy from branch → `main` / root*. The page appears at
   `https://hjort3design.github.io/babboe-battery-data/`.
3. **Create a fine-grained token**: *GitHub → Settings → Developer settings →
   Fine-grained tokens*. Set **Repository access** to *only* the data repo, and
   **Permissions** to *Contents: Read and write*. Nothing else.
4. Open the page, expand **GitHub settings**, enter owner / repo / branch / token, then
   **Save & test**.

## Use

1. Power the diagnostic tool and plug in the pack.
2. **Connect**, then pick `GWA-BattDiag`. The tool's clock is synced from the phone
   automatically.
3. **Scan pack**, add an optional note, then **Upload to GitHub**.
4. Offline? A failed upload goes into the **Upload queue**. It retries when the browser comes
   back online, or when you press *Upload pending*.

### Without Web Bluetooth (iPhone)

1. Join the tool's WiFi (`GWA-BattDiag`) and scan as usual.
2. Open `http://192.168.4.1/data` (the latest scan) or `/snapshots` (up to 50 saved
   snapshots), and save or share the JSON file.
3. Back on normal internet, open the companion page, tap **Import file**, and pick that
   JSON. Several snapshots go straight into the upload queue.

**Import file** also accepts the `/export` scan-log CSV (`gwa_battery_scans.csv`). Older
snapshots and CSV rows have no serial field, so the page decodes the serial from the
dump. CSV rows have no real date (only ms since boot), so they're saved as
`undated-<dump hash>.json`. Identical dumps collapse into one file, and anything already
in the repo is skipped, so importing the same backup twice is harmless.

## Database layout

```
scans/<serial>/<UTC timestamp>.json      e.g. scans/BBAPTN5203039/2026-09-29T15-04-11-512Z.json
```

Packs with an unreadable serial block go under `scans/unknown-<battery_id>/`. Each file
contains:

```json
{
  "schema": 1,
  "uploaded_at": "2026-09-29T15:04:11.512Z",
  "source": "ble",
  "note": "after cell replacement",
  "scan": { "...": "same fields as the firmware's /scan JSON, incl. raw_dump (512 hex chars)" }
}
```

The firmware's device-local fields (`fleet`, `fs_scans`, `auto_save`) are stripped. Every
upload is its own commit, so git history doubles as an audit log. The full `raw_dump` is
kept, so older scans can be re-decoded whenever the register map changes.

## BLE protocol (for other clients)

| | UUID | |
|---|---|---|
| Service | `4f1a0001-6c2e-4b8e-9f3a-7d2c5e8b1a90` | |
| CMD | `4f1a0002-…` | write: `scan`, `data`, `time <unix epoch>` |
| DATA | `4f1a0003-…` | notify: JSON reply in MTU-sized chunks, ended by one `0x04` byte |

The BLE service is **read-only by design**. Recode and restore are available only in the
WiFi UI, behind its ID-confirmation gate, because BLE has no pairing and is open to anyone
in range.
