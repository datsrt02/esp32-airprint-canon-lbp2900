# Add AirPrint / AirScan support to your old printer, using just a cheap ESP32-S3

> [!IMPORTANT]
> This project is actively looking for contributors to add support for more printers. Most importantly, you need to really own the printer you add: every driver must be tested on the real hardware, not only in emulation. See [Adding a printer](#adding-a-printer) to get started.


Turns any USB-only printers into a wireless printer and scanner. An ESP32-S3 board plugs into the printer's USB port and joins your WiFi:

- **Print** from iPhone, iPad and Mac (AirPrint), Android (Mopria) and Linux, with no drivers or apps.
- **Scan** from any browser, from Image Capture and Preview on a Mac, or with the button on the board.
- **A web app** on your phone or computer: print PDFs and pictures, scan, keep the files in the browser, and put pages together (rotate, crop, ID card copies) into a PDF.

![](./docs/IMG_3577.jpeg)

| | |
|---|---|
| ![](./docs/screenshot-printer.png) | ![](./docs/screenshot-scanned.png) |
| ![](./docs/screenshot-files.png) | ![](./docs/screenshot-process.png) |

## Supported printers

| Series | Models | Scan |
|---|---|---|
| 🟠 LBP2900 | LBP2900 / LBP2900B (`04a9:2676`) | no scanner |
| 🟢 MG2500 | MG2550S | yes |
| 🟡 MG2400 | MG2410 to MG2490 | yes |
| 🟡 MG2500 | MG2510 to MG2590 (other than the MG2500S) | yes |
| 🟡 P200 | P200 | no scanner |
| 🟡 MG2900 | MG2910 to MG2990 | yes |
| 🟡 MG3000 | MG3010 to MG3090 | yes |
| 🟡 E470 | E470 to E478 | yes |
| 🟡 E460 | E460 to E468 | yes |
| 🟡 iP2800 | iP2810 to iP2899 | no scanner |
| 🟡 E3100 | E3170 to E3195 | yes |
| 🟡 TS3100 | TS3110 to TS3195 | yes |
| 🟡 MP230 | MP230 to MP237 | yes |
| 🟡 iP2700 | iP2700 to iP2788 | no scanner |
| 🟡 MP250, MP270, MP280 | MP250 to MP288 | yes |
| 🟡 MP490, MP493, MP495 | MP490 to MP499 | yes |
| 🟡 MX340, MX350 | MX340 to MX358 | yes (glass only) |
| 🟡 MX360, MX410, MX420 | MX360 to MX428 | yes (glass only) |

- 🟢 Validated on real printer
- 🟡 Expected to work, not tested on real hardware yet.
- 🟠 Experimental CAPT implementation: builds and passes the core host self-tests, but still
  needs validation on a real LBP2900 before it can be called supported.

The firmware recognises the printer when it is plugged in. Other printers? Please open an issue or a PR.

## What you need

Just one thing: An ESP32-S3 dev board with two USB-C ports:

![](./docs/amazon.png)

Optionally: a SSD1315 OLED screen and a 3D-printed case, see later in this tutorial

## Setup

**Step 1: Hardware.** Plug the ESP32-S3 into your computer via the COM port on the ESP32 board (NOT the port labelled "USB")

![](./docs/IMG_3538.jpeg)

**Step 2: Settings.** Build once (`pio run`): this creates `.env`. Edit it:
   ```
   WIFI_SSID=your-network
   WIFI_PASSWORD=your-password
   PRINTER_NAME=Canon LBP2900 (ESP32)
   HOSTNAME=canon-lbp2900
   MODE=debug
   ```
   `MODE=debug` lets you update the board over WiFi later (see below); `MODE=normal` does without. `.env` is git-ignored, so your WiFi password stays local.

**Step 3: Flash.** Connect the board's **COM** port to your computer and run `pio run -t upload`. PlatformIO finds the board's serial port; if several are connected, add `--upload-port /dev/cu.usbmodem…`.

## Using it

Plug the printer to the port labelled "USB"

NOTE: You may also need to bridge the "USB-OTG" solder join on the board, so that it supplies 5V to the printer. While the printer doesn't actively use 5V, it sense the USB plug using this signal.

- **The web app:** open `http://canon-lbp2900.local/` (your `HOSTNAME`) on your phone or computer.
- **The button on the board** (BOOT, or one on GPIO4): press to scan, press twice for a test page, hold 2 s to cancel.
- **Print from a phone or a Mac:** on an iPhone, Share → Print and pick the printer. On a Mac, add it in System Settings → Printers & Scanners.
- **Scan on a Mac:** open Image Capture (or Preview → File → Import from Scanner) and pick the printer.

For the LBP2900, printing is monochrome at 600 dpi on A4 or Letter. Scanning
and the button test page are unavailable. CAPT is bidirectional: use the
printer's normal USB printer-class interface and do not put a USB/IP or CUPS
backend between the ESP32 and the printer.

## Adding OLED screen and buttons

This step is **optional**, you can connect a SSD1315 OLED screen with the pinout below:

| OLED | ESP32-S3 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| SCL | GPIO41 |
| SDA | GPIO42 |

Connect it before powering on: the board only looks for it at startup.

![](./docs/IMG_3568.jpeg)

For buttons, wire each one between a pin and GND:

| Button | ESP32-S3 | Action |
|---|---|---|
| 1 | GPIO4 | Same as BOOT (scan, test page, cancel) |
| 2 | GPIO5 | Not assigned: add your own in `on_user_button()` (`src/main.c`) |

While the screen is off, a press only wakes it up.

## Development

Set `MODE=dev` and flash: print jobs are processed but not printed, and scans show a built-in test page.

On the computer, without the board: `make -C host && cd host && ./airprint_host`, then open http://localhost:8631/ (an emulated printer and scanner). To work on the web app with live reloading: `cd web && npm run dev` (it talks to that server; `DEVICE=http://<board IP> npm run dev` talks to the board).

Technical details are in [AGENTS.md](AGENTS.md).
The LBP2900 port and its hardware-validation checklist are documented in
[docs/lbp2900.md](docs/lbp2900.md).

## Adding a printer

Each printer model is a driver in `components/printers/`, listed in `printers.c`; the firmware picks the one of the printer that is plugged in. A Canon PIXMA whose stream is close to the MG2500's is one line in `canon_pixma.c`; another printer is a new file in the same folder. See [AGENTS.md](AGENTS.md) for details.

The LBP2900 driver is in `canon_lbp2900.c`. It implements the CAPT command
sequence and streams 70-line Hi-SCoA bands, so a page does not need to fit in
RAM. The protocol implementation follows the public reverse-engineering notes
from [captdriver](https://github.com/agalakhov/captdriver); Canon's proprietary
driver or binary blobs are not included.

## License

The original project and all non-CAPT modules remain MIT, see
[LICENSE](LICENSE). `components/printers/canon_lbp2900.c` is
GPL-3.0-or-later because its Hi-SCoA implementation is based on GPLv3
`captdriver`; a firmware image linked with that module is therefore
distributed under [GPL-3.0-or-later](LICENSES/GPL-3.0-or-later.txt). Canon
binary blobs are not included.

The printer and scanner protocols follow [Gutenprint](https://gimp-print.sourceforge.io/) and [SANE](http://www.sane-project.org/)'s pixma backend. The JPEG encoder's DCT is a translation of libjpeg's `jfdctint.c`: this software is based in part on the work of the Independent JPEG Group. The web UI bundles [Svelte](https://svelte.dev/) (MIT) and [pdf.js](https://mozilla.github.io/pdf.js/) (Apache-2.0).
