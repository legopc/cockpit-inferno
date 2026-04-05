# Architecture

## Overview

Cockpit Inferno is a single-page Cockpit plugin. It consists of three files that Cockpit serves statically:

```
src/
├── inferno.js    — all application logic
├── index.html    — DOM skeleton (no logic, no inline styles)
└── inferno.css   — all styles (no inline style="" attributes)
```

Cockpit loads the plugin via `manifest.json`, which registers it as a top-level menu item.

---

## Communication model

The browser runs inside a Cockpit session. JavaScript communicates with the host system through the **Cockpit bridge** (a privileged process running as the authenticated user on the host). There is no REST API, no backend server — the bridge provides:

| API | Used for |
|-----|----------|
| `cockpit.spawn(args)` | Run a command as the session user, return stdout |
| `cockpit.spawn(["bash","-c", cmd])` | Run shell commands (`sed`, `systemctl --user`, `amixer`, etc.) |
| `cockpit.file(path).replace(content)` | Write user-owned files atomically |
| `cockpit.spawn(["sudo","-n","tee", path])` | Write root-owned files via NOPASSWD sudo |

Three spawn helpers encapsulate these patterns:

```js
sp(args)       // non-privileged array spawn
spUser(cmd)    // shell command in user env (for systemctl --user, sed, etc.)
spSudo(cmd)    // sudo -n shell command (for root-owned files)
```

---

## Configuration file

All persistent settings live in `/etc/inferno.conf` — a plain `KEY=value` shell file written by the Cockpit UI:

```
INFERNO_MODE=aux-bidir
INFERNO_AUDIO_IN=PCH
INFERNO_AUDIO_OUT=PCH
INFERNO_AUDIO_IN2=G430
INFERNO_AUDIO_OUT2=G430
INFERNO_TX_CHANNELS=4
INFERNO_RX_CHANNELS=4
INFERNO_NIC=eno1
INFERNO_DANTE_NAME=Inferno-ABCDEF
INFERNO_SPOTIFY_NAME=Inferno-ABCDEF
```

On **Save & Apply**, the UI:
1. Writes the new config via `sudo tee /etc/inferno.conf`
2. Regenerates `.asoundrc` (the ALSA routing config) from a template based on mode + cards + channels
3. Stops/starts/restarts the affected systemd user services
4. Calls `systemctl --user daemon-reload`

No reboot is needed.

---

## ALSA routing

The ALSA config (`.asoundrc`) is generated dynamically in `applyConfig()` inside `inferno.js`. The shape depends on mode:

| Mode | ALSA construct |
|------|---------------|
| `spotify` | Single `pcm.inferno_spotify` plug → `plughw:CARD,DEV=0` |
| `aux-in` | Single `pcm.inferno_aux_tx` rate-converted plug |
| `aux-out` | Single `pcm.inferno_aux_rx` rate-converted plug |
| `aux-bidir` | Both TX and RX plugs |
| Multi-card | `asym` + `multi` plugin to merge two cards into one virtual device |

Card references use the stable ALSA `alsaId` (e.g. `PCH`, `G430`) rather than the numeric card number, which can change across reboots.

---

## Volume control

Volume is read and written directly via `amixer`:

```
amixer -c <cardNum> sget <control>     # read current level + dB
amixer -c <cardNum> sset <control> N%  # set level
sudo alsactl store                      # persist to /var/lib/alsa/asound.state
```

Control detection priority per card: **Master** (if present with playback volume) → **Headphone** → **PCM**. This matters because on Intel HDA (PCH) cards, `Master` is upstream of `Headphone`/`Speaker` — raising only `Headphone` while `Master` is attenuated still produces quiet output.

Persistence is handled by `alsa-state.service` (a continuous ALSA state daemon, preferred over `alsa-restore.service` when `/etc/alsa/state-daemon.conf` exists).

---

## Services managed

All services run as **systemd user services** under the `core` user:

| Service | Purpose |
|---------|---------|
| `librespot` | Spotify Connect receiver |
| `inferno-bridge` | Dante TX transmitter (reads from ALSA, sends over Dante) |
| `inferno-keepalive` | Watchdog that restarts inferno-bridge if it crashes |
| `librespot-watchdog` | Watchdog for librespot |
| `statime-inferno` | PTP grandmaster (IEEE 1588) for Dante sync |

Services are started/stopped based on the active mode. For example, `librespot` only runs in `spotify` mode; `inferno-bridge` runs in all modes.

---

## CSP constraints

Cockpit enforces a strict Content Security Policy. The plugin must comply:

- No `<style>` tags — all CSS must be in `inferno.css`
- No `style="..."` inline attributes — use CSS classes only
- No `onclick=`/`onchange=` HTML attributes — wire all events via `addEventListener` in `init()`
- External scripts are blocked — everything must be in `inferno.js`

---

## Initialisation sequence

```
init()
  └─ loadConfig()           — reads /etc/inferno.conf, populates form fields
       └─ onModeChange()    — shows/hides fields for current mode
            └─ onChannelChange() — shows/hides Card 2 fields based on channel count
  └─ refreshHeader()        — hostname, IP, PTP status, mode badge
  └─ refreshAll()           — service grid + system info table
  └─ loadLog()              — first journal fetch
  └─ refreshAudioDevices()  — enumerate ALSA cards
  └─ loadVolumes()          — probe mixer levels for active cards
```

`setInterval(refreshServices, 20000)` keeps the service grid live-updating every 20 seconds.
