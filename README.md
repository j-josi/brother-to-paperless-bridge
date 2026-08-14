# Brother to Paperless-ngx Bridge

This Docker container turns the hardware **scan button** on a Brother network
scanner into a one-press upload into **Paperless-ngx**. Pressing "Scan to PC" on
the printer's own touchscreen (any of the "Image", "Text/OCR", or "File" sub-types)
scans the document via the scanner's native SANE driver (`brscan4`) and uploads it
directly to your Paperless-ngx instance — no browser, no manual step.

This is a fork of [Moepchi/brother-to-paperless-bridge](https://github.com/moepchi/brother-to-paperless-bridge),
adjusted for the Brother MFC-J480DW and to drop the AirSane/web-UI part of the
original project (see "Why no AirSane" below).

---

## How it works

    Hardware scan button (Image / Text / File)
        ↓
    brscan-skey-exe (background listener)
        ↓
    scantoimage.sh
        ↓
    brscan4 / native SANE (brother4:net1;dev0)
        ↓
    Paperless-ngx (direct upload via curl to /api/documents/post_document/)

That's the entire job of this container. There is no HTTP server, no exposed
port, and nothing else running inside it — the container's foreground process is
just `tail -f /dev/null`, keeping it alive while `brscan-skey-exe` does the real
work in the background.

All three "Scan to PC" sub-types on the printer's touchscreen ("Image",
"Text"/OCR, "File") are routed to the same upload script, so it doesn't matter
which one you pick.

---

## Why no AirSane

The original project also ran **AirSane** to republish the scanner as a
driverless eSCL/AirPrint web server, for browser-based and phone-based scanning.
That path had a confirmed bug: batch-scanning the ADF through AirSane's own
republished eSCL device never detected "feeder empty" and kept trying to feed
pages indefinitely (root-caused to AirSane's own job-continuation logic in
`scanjob.cpp`, not a general `sane-airscan` problem — the scanner's own native
eSCL endpoint, bypassing AirSane entirely, does not have this bug).

Since the scanner already exposes its own native eSCL endpoint directly (no
AirSane needed), the AirSane part of this project was dropped rather than
patched. For driverless web-UI scanning, point a separate
[scanservjs](https://github.com/sbs20/scanservjs) instance at the scanner's own
eSCL device instead — that instance is not part of this repository.

---

## Prerequisites

1. A network-connected Brother scanner/printer with a hardware scan button and
   `brscan4` SANE driver support.
2. A running Paperless-ngx instance with an API token.
3. The `brscan-skey` "Scan Key Tool" `.deb` package for your architecture,
   downloaded separately (see below) — **not included in this repo**, since it's
   Brother's own proprietary binary and redistributing it isn't appropriate here.

### Downloading brscan-skey

`brscan-skey` is a generic tool, not tied to a specific printer/scanner model.
Get the current version from Brother's own download server, e.g.:

```bash
wget https://download.brother.com/welcome/dlf006652/brscan-skey-0.3.5-0.amd64.deb
```

(Check [support.brother.com](https://support.brother.com) under your model's
Linux downloads → "Scan Key Tool" for the current version if this link is
outdated — Brother updates it occasionally.)

---

## Quick Start

### 1. Configuration (`.env`)

Copy `example.env` to `.env` and fill in your values:

```env
BRSCAN_IP=192.168.1.X
SCAN_PC_NAME=ScanToPaperless
PAPERLESS_URL=https://your-paperless-instance
PAPERLESS_TOKEN=your_secret_api_token
```

Note the printer's own model is currently hard-coded in `compose.yaml`'s
`command:` block (`model=MFC-J480DW`) — change that line if you're using a
different Brother model.

### 2. Place the driver

Put the downloaded `brscan-skey-0.3.5-0.amd64.deb` in the same directory as
`compose.yaml` (matches the filename referenced in the `volumes:` section).

### 3. Start

```bash
docker compose up -d
```

Press "Scan to PC" → your registered PC name → "Image"/"Text"/"File" on the
printer's touchscreen. The scan is picked up automatically and uploaded to
Paperless-ngx.

---

## Contributing & License

Feel free to open issues or submit pull requests if you want to add support for
other brscan driver versions or printer models.

Original project maintained by [Moepchi](https://github.com/moepchi).
