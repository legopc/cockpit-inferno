# Usage Guide

Access the Inferno panel via Cockpit at `https://<node-ip>:9090` → **Inferno AoIP** in the left sidebar.

---

## Services

Shows all Inferno systemd services with their current state (active/inactive/failed).

Each card has **Start**, **Stop**, and **Restart** buttons. The coloured left border indicates status:
- 🟢 Green — active and running
- 🔴 Red — failed
- ⚫ Grey — inactive (stopped)

**⟳ Restart All** at the top restarts every service in sequence.

> Services refresh automatically every 20 seconds. Click **↻ Refresh** in the header for an immediate update.

---

## Configuration

### Device identity

| Field | What it sets |
|-------|-------------|
| **Spotify Name** | The name shown in Spotify apps as the playback target. Only visible in Spotify mode. Changing it restarts `librespot`. |
| **Dante TX Name** | The name shown in Dante Controller for the transmitter. Changing it restarts `inferno-bridge`. |

### Audio routing

| Field | Description |
|-------|-------------|
| **Mode** | The audio routing mode (see below) |
| **TX Channels** | Number of channels sent to Dante TX (2/4/6/8) |
| **Input Card** | ALSA soundcard used for capture (→ Dante TX) |
| **Input Card 2** | Second capture card for channels 3–4 (only shown when TX Channels ≥ 4) |
| **RX Channels** | Number of channels received from Dante RX (2/4/6/8) |
| **Output Card** | ALSA soundcard used for playback (← Dante RX) |
| **Output Card 2** | Second playback card for channels 3–4 (only shown when RX Channels ≥ 4) |

#### Modes

| Mode | Description |
|------|-------------|
| 🎵 Spotify → Dante TX | Spotify Connect receiver. Audio goes: Spotify app → librespot → ALSA → inferno-bridge → Dante TX |
| 🎙 Analog In → Dante TX | Capture from physical soundcard(s) and transmit over Dante |
| 🔈 Dante RX → Analog Out | Receive from Dante and play to physical soundcard(s) |
| ⇄ In + Out (bidirectional) | Both capture and playback simultaneously, with independent channel counts |

### Network

**Interface** — the wired Ethernet interface used for Dante audio and PTP sync. Usually `eno1` or similar. Changing this requires restarting `inferno-bridge` and `statime-inferno`.

### Saving

Click **💾 Save & Apply** to write the configuration and restart affected services. No reboot needed.

The **● unsaved** badge in the header appears when there are pending changes.

---

## System Info

Displays live system information:

| Row | Source |
|-----|--------|
| Hostname | `/proc/sys/kernel/hostname` |
| IP address | `ip -4 addr show <nic>` |
| Image version | `bootc status` |
| PTP status | `journalctl -u statime-inferno` (last sync line) |
| Uptime | `uptime -p` |
| Deploy sentinel | Presence of `/var/lib/inferno/.deployed` |

---

## Volume Control

Shows a slider per active soundcard (cards currently selected in the Configuration panel).

- Slider range: 0–100%
- The dB value shown is the actual hardware level reported by ALSA
- Moving the slider updates the hardware immediately and persists the change (`alsactl store`)
- **Set All to 100%** normalises all cards to 0 dB simultaneously

> **Which control is used?** Per card, the plugin tries: `Master` → `Headphone` → `PCM`. On Intel HDA (PCH) cards, `Master` is the correct control as it gates all downstream outputs.

---

## Audio Devices

Enumerates all ALSA cards and their devices via `aplay -l` and `arecord -l`.

- Capture devices are shown with a 🎙 badge
- Playback devices with a 🔊 badge
- HDMI/DisplayPort outputs are excluded (not usable for Dante)
- Software loopback cards are shown dimmed — they cannot be selected as audio cards

Click **↻ Refresh** to re-scan (useful after plugging in a USB audio device).

---

## Journal

Select a service from the dropdown and click **↻** to load its recent log entries. Log lines are colour-coded:

- 🟢 Green — success/OK messages
- 🔴 Red — errors
- 🟠 Orange — warnings
- Grey — timestamps

---

## Actions

| Button | Effect |
|--------|--------|
| **🔄 Re-deploy binaries** | Removes `/var/lib/inferno/.deployed` (the deploy sentinel) and reboots. On next boot, `inferno-deploy.sh` re-downloads the latest Inferno binaries from the GitHub release. Use this to force a binary refresh without a full OTA image upgrade. |
| **⏻ Reboot** | Reboots the node immediately. |

Both actions show a confirmation dialog before proceeding.
