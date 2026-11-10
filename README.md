# phonecam
**Warning! Vibecoded as fuck, use at your own risk, or don't, I don't care**


Two cameras, one OBS scene, zero virtual-camera drivers.

Unofficial project, not affiliated with Lovense or VDO.Ninja. "Lovense" is a trademark of its owner.

phonecam is a small command-line tool with two jobs:

1. **Phone as a camera.** Creates a [VDO.Ninja](https://vdo.ninja) room, adds it to OBS as a browser source and prints a QR code for the phone.
2. **Lovense Cam overlay.** Runs a local relay that speaks the Lovense OBS Toolset protocol, serves the Lovense Cam widget (the on-stream toy-activity overlay) to OBS and answers the Lovense Chrome extension. No Splitcam, no Windows toolset, no virtual camera.

Everything is one Python file plus the Lovense web widget.

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

The extension is hard-wired to `ws://localhost:20996`. The widget is hard-wired to `ws://127.0.0.1:<port>/smartcam/ws?from=smart` (the port comes from `?port=` in its URL, default 11000). The relay connects both sides and translates the OBS state into the answers the extension expects (`GetSceneList`, `GetStreamingStatus`, `GetVirtualCamStatus`, ...).

Every frame is AES-128-CBC encrypted with a key and IV taken from the extension. The relay uses the same constants for compatibility with the local panel protocol.

The widget is served by the relay's HTTP server on port 8888. `GET /health` returns a small JSON status (relay, OBS, widget).

## Requirements

- Linux with systemd (the relay runs as a user service)
- OBS Studio with obs-websocket enabled (Tools -> WebSocket Server Settings), default port 4455
- Python 3.10+ with `websockets`, `aiohttp`, `cryptography`
- `qrencode` (terminal QR code)
- `p7zip` (only to unpack the official Lovense installer)
- Chrome with the Lovense Cam extension (tested with `google-chrome-stable` on Arch)

Arch:

```sh
sudo pacman -S python-websockets python-aiohttp python-cryptography qrencode p7zip
```

Debian/Ubuntu (package names, not tested):

```sh
sudo apt install python3-websockets python3-aiohttp python3-cryptography qrencode p7zip-full
```

## Install

```sh
git clone https://github.com/FoxlikeCreature/phonecam.git ~/projects/phonecam
mkdir -p ~/.local/bin
ln -sf ~/projects/phonecam/phonecam ~/.local/bin/phonecam
```

Use a **symlink**, not a copy. The script looks for the widget next to its real path (`widget/` in the repo), so a plain copy into `~/.local/bin` would not find it. If you do copy the file, run `phonecam lovense setup --download` afterwards to fetch the widget from the official installer.

On Arch you can instead build the included `PKGBUILD` (installs the widget to `/usr/share/phonecam/widget`).

## First run

```sh
phonecam doctor              # checks every dependency and the OBS setup
phonecam lovense extension   # installs / patches the Lovense extension
phonecam lovense setup       # shows the widget version in use
phonecam start               # relay + both OBS sources + QR code
```

`phonecam lovense extension` does two things:

- downloads the official `lovense_cam.zip` (if no unpacked copy is found) and patches `pages/chrome_background.html`: it adds `local-network-access` and `loopback-network` to the `allow=` attribute of the extension's iframe, so the iframe may talk to localhost. Nothing else is changed.
- if Chrome already has the extension loaded, it patches that folder in place and asks you to click Reload in `chrome://extensions`.

Then load it once: `chrome://extensions` -> Developer mode -> Load unpacked -> the folder printed by the command. Keep the folder where it is, Chrome loads it from disk.

### Chrome Local Network Access

Recent Chrome versions block pages on the public internet from opening WebSocket connections to localhost without permission. The Lovense panel lives on `extension.lovense.com`, so its connection to the relay would be blocked. phonecam uses the same workaround as the official toolset: disable the check.

Create `~/.config/chrome-flags.conf`:

```
--disable-features=LocalNetworkAccessChecks
```

The Arch `google-chrome-stable` wrapper reads that file (Chromium reads `chromium-flags.conf` instead, and `phonecam doctor` only checks the Chrome one). Restart Chrome afterwards.

**The flag applies to the whole browser instance, not to a single profile.** Every site you open in that Chrome loses this protection. If that bothers you, run a separate Chrome only for Lovense:

```sh
google-chrome-stable --user-data-dir=$HOME/.config/chrome-lovense \
  --disable-features=LocalNetworkAccessChecks
```

and keep your main browser flag-free. The phone part of phonecam does not need Chrome at all.

## Daily use

```sh
phonecam start
```

That is the whole routine. It:

1. starts (or reuses) the relay user service,
2. creates or updates the `PhoneCam` and `Lovense Cam` browser sources,
3. puts Lovense Cam on top as a transparent overlay,
4. switches OBS to the scene,
5. prints the phone URL and the QR code.

Scan the QR with the phone, open the Lovense panel in Chrome, and stream.

`phonecam start --no-obs` starts only the relay; `--no-qr` skips the QR; `--scene NAME` uses another OBS scene.

### Room privacy

Every room has a random password (`&password=` in the phone and OBS links). VDO.Ninja uses it to encrypt the signalling handshake and to obscure the room name on its servers, so knowing the room ID alone is not enough to watch the stream.

The password is part of the URL, so the URL itself (and any screenshot of the QR code) is the secret. Keep it out of chat and screenshots.

Older configs without a password get one on the next `phonecam start`; scan the printed QR code again. To set your own, use `phonecam phone new --password <PW>` (1-49 letters and digits) or a longer room ID with `phonecam phone new --room <ID>`.

## Commands

Main:

```
phonecam start                     relay + OBS sources + QR
phonecam doctor                    environment diagnostics
phonecam phone new [--room ID] [--password PW] [--back] [--bitrate 2500] [--scene NAME]
phonecam phone qr [--open]         regenerate / open the QR image
phonecam lovense extension         install / patch the Lovense extension
phonecam lovense status            relay status (panels, widgets, OBS)
```

Everything else:

```
phonecam phone url | obs [--scene NAME] [--activate] | refresh | layout [--scene NAME]
phonecam phone status | enable-ws | all
phonecam lovense setup [--download] [--installer FILE] [--to DIR]
phonecam lovense obs [--scene NAME] [--activate]
phonecam lovense relay             run the relay in the foreground
phonecam lovense start | stop | restart | status
phonecam lovense service install | uninstall | status
```

Old top-level commands (`new`, `url`, `qr`, ...) still work and map to `phonecam phone ...`.

## Files

| Path | Purpose |
|------|---------|
| `~/.config/phonecam/config.json` | room, password, bitrate, scene, codec |
| `~/.config/phonecam/qr.png` | last QR code |
| `~/.local/share/phonecam/widget` | widget downloaded with `setup --download` |
| `~/.local/share/phonecam/extension` | extension downloaded with `lovense extension` |
| `~/.config/systemd/user/phonecam-lovense.service` | relay user service |

Environment overrides: `PHONECAM_CONFIG_DIR`, `PHONECAM_DATA_DIR`, `PHONECAM_WIDGET_DIR`, `PHONECAM_RELAY_PORT`, `PHONECAM_HTTP_PORT`. The proxy port 20996 cannot be changed: the extension hard-codes it.

## Troubleshooting

**The Lovense panel keeps spinning / says "connecting".** Check `phonecam lovense status`: it must show `panels=1` when the panel is open. If the panel is connected but OBS is not, the relay falls back to safe default answers. Make sure obs-websocket is enabled and the password in `~/.config/obs-studio/plugin_config/obs-websocket/config.json` is readable. Also check that Chrome was restarted after writing `chrome-flags.conf`.

**The extension lost the patch after an update.** Run `phonecam lovense extension` again. It is idempotent.

**Port already in use.** The relay needs 11000, 20996 and 8888. `phonecam lovense status` shows whether the relay is up. A leftover `lovense-obs-relay.service` is stopped and disabled automatically when the new service is installed.

**OBS shows a blank Lovense source.** Run `phonecam doctor`. The widget must exist and the relay must answer `http://127.0.0.1:8888/health`. Reload the source with `phonecam lovense obs --scene NAME`.

**Phone has no picture.** The phone must keep the VDO.Ninja page open. `phonecam phone status` shows the URL OBS is using. `phonecam phone refresh` forces a page reload.

## Uninstall

```sh
phonecam lovense service uninstall        # stop and remove the relay service
rm -f ~/.local/bin/phonecam
rm -rf ~/.config/phonecam ~/.local/share/phonecam
```

- Remove the `PhoneCam` and `Lovense Cam` sources from OBS by hand.
- Delete the `--disable-features=LocalNetworkAccessChecks` line from `~/.config/chrome-flags.conf` and restart Chrome.
- The extension patch has no automatic undo. Remove the extension in `chrome://extensions` and delete its folder, or delete the two added entries (`local-network-access`, `loopback-network`) from `allow=` in `pages/chrome_background.html`.

## License

The tool itself is MIT (see `LICENSE`).

The bundled `widget/` directory is the Lovense Cam OBS widget, version 2.5.6. It is proprietary software by Lovense, redistributed here unchanged so that the tool works offline, and is not covered by the MIT license. See `NOTICE` and `widget/2.5.6.txt`. If you prefer not to ship proprietary files, remove `widget/` and let `phonecam lovense setup --download` fetch it from the official installer.
