# Cockpit Inferno

A custom [Cockpit](https://cockpit-project.org/) plugin for the **Inferno AoIP** appliance — a Fedora bootc-based embedded audio system that bridges Spotify Connect and analog audio to the Dante AoIP network.

Built with plain HTML/CSS/JavaScript and the Cockpit bridge API. No frameworks, no build step.

---

## Features

The UI is organised into four tabs:

### 🔧 Config (default tab)
- Set **Spotify name**, **Dante TX name**, **audio mode**, **NIC**, **audio cards**, **TX/RX channel count**, and **loop latency**
- Disabled fields (Bitrate, Normalise) shown read-only with explanatory hints — not supported at runtime
- **💾 Save & Apply** writes `/etc/inferno.conf` and restarts affected services
- **Preview Changes** modal shows a diff of pending vs saved values before applying
- Unsaved-changes badge in the header

### ⚙ Services
- Live status of all Inferno systemd services (active / inactive / failed)
- Start / Stop / Restart per service
- **⟳ Restart All** with animated per-service progress indicator
- Multi-service journal viewer (select any service or "All")
- Auto-refresh every 20 seconds

### 🔊 Audio
- Per-card ALSA volume sliders with dB readout (auto-detects Master → Headphone → PCM)
- **Set All to 100%** normalises all cards simultaneously; changes persist via `alsactl store`
- ALSA device enumeration (capture 🎙 / playback 🔊) with HDMI outputs filtered out
- Peak-level meter panel

### 📊 Monitoring
- **Signal Chain** card — reads `inferno.conf` and `.asoundrc` to show mode, Dante TX name, NIC, TX/RX channels, ALSA format, sample rate, and latency
- **PTP sparkline** — live offset graph from `statime-inferno` journal
- **System Info** — hostname, IP, image version, uptime, disk usage (reads `/sysroot` for real disk size on bootc), NIC traffic in KB/s or MB/s
- **Health Check** panel — run on demand; checks: snd-aloop loaded, PTP locked, inferno-bridge active, librespot active, statime-inferno active, disk < 80%, NIC has IP
- Dante device discovery scanner

### Global
- Keyboard shortcuts: `Ctrl+S` (save), `Ctrl+R` (refresh), `1–4` (switch tabs)
- Collapsible cards (state persisted in `localStorage`)
- Sticky header with mode badge, unsaved indicator, animated service dots
- Credits card acknowledging the upstream [Inferno project](https://gitlab.com/lumifaza/inferno)

### Audio routing modes

- 🎵 **Spotify Connect → Dante TX** — librespot receives audio, inferno-bridge sends it to Dante
- 🎙 **Analog In → Dante TX** — capture from physical soundcard(s), transmit over Dante
- 🔈 **Dante RX → Analog Out** — receive from Dante, play to physical soundcard(s)
- ⇄ **Bidirectional** — both analog in and out simultaneously, independent channel counts

Multi-card support: up to two soundcards merged into a single Dante device (channels 1–2 from card 1, 3–4 from card 2).

---

## Installation

The plugin ships baked into the Inferno AoIP container image. Cockpit auto-discovers it at `/usr/share/cockpit/inferno/`.

For development iteration on a live node (bootc — `/usr` is read-only):

```bash
# Enable writable /usr overlay (discarded on reboot)
sudo rpm-ostree usroverlay

# Copy files
scp src/{index.html,inferno.js,inferno.css} \
    core@<node-ip>:/tmp/
ssh core@<node-ip> "sudo cp /tmp/index.html /tmp/inferno.css /tmp/inferno.js \
    /usr/share/cockpit/inferno/ && sudo systemctl restart cockpit.service"
```

> **Note:** Do not use the user-local override path (`~/.local/share/cockpit/inferno/`) — it takes precedence over the system package and can mask deployments if left behind.

---

## Repository layout

```
src/
├── inferno.js      Main application (~2000 lines)
├── index.html      UI structure (tab nav, all card panels, dialog)
├── inferno.css     Styles (~560 lines)
└── manifest.json   Cockpit plugin manifest
docs/
├── architecture.md How the app is structured and how it communicates
├── usage.md        End-user guide for each tab/panel
└── development.md  Development workflow, CSP rules, ALSA debugging
```

---

## Architecture notes

- **CSP-strict:** no `style=""` attributes, no `onclick=`/`onchange=` in HTML — all events bound in `init()` via `addEventListener`
- **Three spawn helpers:** `sp()` (non-privileged), `spUser()` (user systemd/env), `spSudo()` (root ops via `sudo -n`)
- **Config file:** `/etc/inferno.conf` — KEY=VALUE shell format, written via `sudo tee`
- **Signal chain parsed from:** `/var/home/core/.asoundrc` (ALSA loopback PCM definitions)

---

## Related repositories

| Repo | Purpose |
|------|---------|
| [`legopc/inferno-aoip-releases`](https://github.com/legopc/inferno-aoip-releases) | Container image build, ISO, upgrade bundles |
| [`legopc/cockpit-iot-updater`](https://github.com/legopc/cockpit-iot-updater) | Cockpit plugin for OTA image upgrades |
| [`gitlab.com/lumifaza/inferno`](https://gitlab.com/lumifaza/inferno) | Upstream Inferno AoIP project |

---

## Licence

MIT
