# Cockpit Inferno

A custom [Cockpit](https://cockpit-project.org/) plugin for the **Inferno AoIP** appliance — a Fedora bootc-based embedded audio system that bridges Spotify Connect and analog audio to the Dante AoIP network.

Built with plain HTML/CSS/JavaScript and the Cockpit bridge API. No frameworks, no build step.

---

## Features

| Panel | What it does |
|-------|-------------|
| **Services** | Live status of all Inferno systemd services with start/stop/restart controls |
| **Configuration** | Set device names, audio routing mode, card selection, channel count, and network interface — applied without reboot |
| **System Info** | Hostname, IP, image version, PTP sync status, uptime, deploy sentinel |
| **Volume Control** | Per-card ALSA sliders with dB readout, auto-detecting primary control (Master → Headphone → PCM). Changes persist via `alsactl store` |
| **Audio Devices** | Enumerate all ALSA cards and their capture/playback devices |
| **Journal** | Live log viewer for any Inferno service |
| **Actions** | Re-deploy binaries or reboot from the UI |

### Audio routing modes

- 🎵 **Spotify Connect → Dante TX** — librespot receives audio, inferno-bridge sends it to Dante
- 🎙 **Analog In → Dante TX** — capture from physical soundcard(s), transmit over Dante
- 🔈 **Dante RX → Analog Out** — receive from Dante, play to physical soundcard(s)
- ⇄ **Bidirectional** — both analog in and out simultaneously, independent channel counts

Multi-card support: up to two soundcards merged into a single Dante device (channels 1–2 from card 1, 3–4 from card 2).

---

## Installation

The plugin ships baked into the Inferno AoIP container image. Cockpit auto-discovers it at `/usr/share/cockpit/inferno/`.

For development iteration, copy directly to the node's user override path:

```bash
scp -i ~/.ssh/your_key src/{index.html,inferno.js,inferno.css} \
    core@<node-ip>:/var/home/core/.local/share/cockpit/inferno/
```

Reload the Cockpit page — no service restart needed.

---

## Repository layout

```
src/
├── inferno.js      Main application (~1000 lines)
├── index.html      UI structure
├── inferno.css     Styles
└── manifest.json   Cockpit plugin manifest
docs/
├── architecture.md How the app is structured and how it communicates
├── usage.md        End-user guide for each panel
└── development.md  Development workflow, CSP rules, ALSA debugging
```

---

## Related repositories

| Repo | Purpose |
|------|---------|
| [`legopc/inferno-aoip-releases`](https://github.com/legopc/inferno-aoip-releases) | Container image build, ISO, upgrade bundles |
| [`legopc/cockpit-iot-updater`](https://github.com/legopc/cockpit-iot-updater) | Cockpit plugin for OTA image upgrades |

---

## Licence

MIT
