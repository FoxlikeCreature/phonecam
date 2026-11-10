# phonecam

Two cameras, one OBS scene, zero virtual-camera drivers.

`phonecam` is a small command-line tool with two jobs:

1. **Phone as a camera.** It creates a [VDO.Ninja](https://vdo.ninja) room,
   adds it to OBS as a browser source and prints a QR code for the phone.
2. **Lovense Cam overlay.** It runs a local relay that speaks the Lovense OBS
   Toolset protocol, serves the Lovense Cam widget to OBS and answers the
   Lovense Chrome extension. No Splitcam, no Windows toolset, no virtual
   camera.

Everything is one Python file plus the Lovense web widget.

---

## How it works

```
phone ──WebRTC──> vdo.ninja ──> OBS browser source "PhoneCam"
                                        │
Lovense toy ──Bluetooth──> Lovense Cam extension (Chrome)
                                        │ ws://localhost:20996
                                        v
                              phonecam relay (20996 -> 11000)
                                        │ ws://127.0.0.1:11000/smartcam/ws
                                        v
                     OBS browser source "Lovense Cam"
                     http://127.0.0.1:8888/?from=smart&port=11000
```

The Lovense extension is hard-wired to `ws://localhost:20996`. The widget is
hard-wired to `ws://127.0.0.1:<port>/smartcam/ws?from=smart` (the port comes
from `?port=` in its URL, default `11000`). The relay stitches both sides
together and translates the OBS state into the answers the extension expects
(`GetSceneList`, `GetStreamingStatus`, `GetVirtualCamStatus`, ...).

The extension encrypts every frame it receives with AES-128-CBC using the
key/IV that ship inside the extension. The relay uses the same constants so
the panel accepts its `hello` and its answers. Those constants are protocol
details, not secrets.

The widget is served by the relay's HTTP server on port `8888`. The relay
also exposes `GET /health` with a small JSON status (relay, OBS, widget).

## Requirements

- Linux with systemd (the relay runs as a user service)
- OBS Studio with **obs-websocket** enabled (Tools -> WebSocket Server
  Settings), default port `4455`
- Python 3.10+ and:
  - `python-websockets`
  - `python-aiohttp`
  - `python-cryptography`
- `qrencode` (terminal QR code)
- `p7zip` (only to unpack the official Lovense installer)
- Chrome/Chromium with the Lovense Cam extension

On Arch:

```sh
sudo pacman -S python-websockets python-aiohttp python-cryptography qrencode p7zip
```

## Install

```sh
git clone https://github.com/FoxlikeCreature/phonecam.git ~/projects/phonecam
install -Dm755 ~/projects/phonecam/phonecam ~/.local/bin/phonecam
```

The repository contains the Lovense widget in `widget/`, so no download is
needed. If you prefer a package, use the included `PKGBUILD`.

## First run

```sh
phonecam doctor              # checks every dependency and the OBS setup
phonecam lovense extension   # installs / patches the Lovense extension
phonecam lovense setup       # shows the bundled widget version
phonecam start               # relay + both OBS sources + QR code
```

`phonecam lovense extension` does two things:

- downloads the official `lovense_cam.zip` (if no unpacked copy is found) and
  patches `pages/chrome_background.html` so the extension's iframe is allowed
  to talk to `localhost`;
- if Chrome already has the extension loaded, it patches that folder in place
  and asks you to click **Reload** in `chrome://extensions`.

Then load it once: `chrome://extensions` -> Developer mode -> **Load
unpacked** -> the folder printed by the command. Keep the folder where it is;
Chrome loads it from disk.

### Chrome Local Network Access

Since Chrome 142-ish, pages on the public internet cannot open WebSocket
connections to `localhost` without permission. The Lovense panel lives on
`extension.lovense.com`, so the connection would be blocked. `phonecam` uses
the same workaround as the official toolset: disable the check for this
Chrome profile.

Create `~/.config/chrome-flags.conf`:

```
--disable-features=LocalNetworkAccessChecks
```

The Arch `google-chrome-stable` wrapper reads that file. Restart Chrome
afterwards. `phonecam doctor` checks it.

> This is a global switch for that Chrome profile. It is the price of running
> the Lovense panel against a local relay. If you do not want it, the phone
> part of `phonecam` works without Chrome at all.

## Daily use

```sh
phonecam start
```

That is the whole routine. It:

1. starts (or reuses) the relay user service,
2. creates or updates the `PhoneCam` and `Lovense Cam` browser sources,
3. puts `Lovense Cam` on top as a transparent overlay,
4. switches OBS to the scene,
5. prints the phone URL and the QR code.

Scan the QR with the phone, open the Lovense panel in Chrome, and stream.

`phonecam start --no-obs` starts only the relay; `--no-qr` skips the QR;
`--scene NAME` uses another OBS scene.

## Commands

```
phonecam start                     relay + OBS sources + QR
phonecam doctor                    environment diagnostics

phonecam phone new [--room ID] [--back] [--bitrate 2500] [--scene NAME]
phonecam phone url                 print the room URLs
phonecam phone qr [--open]         regenerate / open the QR image
phonecam phone obs [--scene NAME] [--activate]
phonecam phone refresh             force-reload the browser source
phonecam phone layout [--scene NAME]
phonecam phone status
phonecam phone enable-ws           enable obs-websocket in the OBS config
phonecam phone all                 new + obs

phonecam lovense setup [--download] [--installer FILE] [--to DIR]
phonecam lovense extension [--dir DIR] [--force]
phonecam lovense obs [--scene NAME] [--activate]
phonecam lovense relay             run the relay in the foreground
phonecam lovense start | stop | restart | status
phonecam lovense service install | uninstall | status
```

Old commands (`new`, `url`, `qr`, `obs`, `all`, `refresh`, `layout`, `status`,
`enable-ws`) still work and map to `phonecam phone ...`.

## Files

| Path | Purpose |
| --- | --- |
| `~/.config/phonecam/config.json` | room, bitrate, scene, codec |
| `~/.config/phonecam/qr.png` | last QR code |
| `~/.local/share/phonecam/widget` | widget downloaded with `setup --download` |
| `~/.local/share/phonecam/extension` | extension downloaded with `extension` |
| `~/.config/systemd/user/phonecam-lovense.service` | relay user service |

Environment overrides: `PHONECAM_CONFIG_DIR`, `PHONECAM_DATA_DIR`,
`PHONECAM_WIDGET_DIR`, `PHONECAM_RELAY_PORT`, `PHONECAM_HTTP_PORT`.
The proxy port `20996` cannot be changed: the extension hard-codes it.

## Troubleshooting

**The Lovense panel keeps spinning / says "connecting".**
Check `phonecam lovense status`: it must show `panels=1` when the panel is
open. If the panel is connected but OBS is not, the relay falls back to safe
default answers. Make sure obs-websocket is enabled and the password in
`~/.config/obs-studio/plugin_config/obs-websocket/config.json` is readable.
Also check that Chrome was restarted after writing `chrome-flags.conf`.

**The extension lost the patch after an update.**
Run `phonecam lovense extension` again. It is idempotent.

**Port already in use.**
The relay needs `11000`, `20996` and `8888`. `phonecam lovense status` shows
whether the relay is up. A leftover `lovense-obs-relay.service` is stopped and
disabled automatically when the new service is installed.

**OBS shows a blank Lovense source.**
Run `phonecam doctor`. The widget must exist and the relay must answer
`http://127.0.0.1:8888/health`. Reload the source with
`phonecam lovense obs --scene NAME`.

**Phone has no picture.**
The phone must keep the VDO.Ninja page open. `phonecam phone status` shows
the URL OBS is using. `phonecam phone refresh` forces a page reload.

## License

The tool itself is MIT (see `LICENSE`).

The bundled `widget/` directory is the Lovense Cam OBS widget, version
`2.5.6`. It is proprietary software by Lovense and is redistributed here
unchanged, only so that the tool works offline. It is **not** covered by the
MIT license. See `NOTICE` and `widget/2.5.6.txt`. If you prefer not to ship
proprietary files, remove `widget/` and let
`phonecam lovense setup --download` fetch it from the official installer.

The AES key and IV are constants from the Lovense extension, used for
interoperability with the local panel protocol.
