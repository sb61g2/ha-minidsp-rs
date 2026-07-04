# MiniDSP RS — Home Assistant Integration

Home Assistant add-on and custom integration for [MiniDSP RS](https://github.com/sb61g2/minidsp-rs), a control daemon for MiniDSP DSP devices.

## What's included

| Component | Purpose |
|---|---|
| **Add-on** (`addon/`) | Runs the MiniDSP RS daemon inside HAOS. Provides the HTTP API and TCP server the integration connects to. |
| **Custom integration** (`custom_components/minidsp/`) | Exposes volume, mute, source, preset, and Dirac Live as HA entities. HACS-installable. |
| **Watchdog Blueprint** (`blueprints/`) | Automation that detects integration unavailability, power-cycles the device, and restarts the daemon to recover. |

## Supported devices

Any device supported by [MiniDSP RS](https://github.com/sb61g2/minidsp-rs), including:

- MiniDSP Flex HTx
- MiniDSP Flex / Flex DL
- MiniDSP 2x4 HD / DDRC-24
- MiniDSP SHD
- MiniDSP 4x10 HD
- and others (see upstream for the full list)

## Installation

### Step 1 — Add-on

1. In HA go to **Settings → Add-ons → Add-on Store** (three-dot menu) **→ Repositories**
2. Add: `https://github.com/sb61g2/ha-minidsp-rs`
3. Refresh — **MiniDSP RS** will appear in the store
4. Install and start the add-on

The add-on connects to your MiniDSP device over USB. For USB devices on HAOS, also deploy the udev rule (Step 4) to prevent the Linux audio driver from interfering with the HID interface.

### Step 2 — Custom integration (via HACS)

1. In HACS go to **Integrations → three-dot menu → Custom repositories**
2. Add `https://github.com/sb61g2/ha-minidsp-rs` with category **Integration**
3. Install **MiniDSP RS** from HACS
4. Restart Home Assistant
5. Go to **Settings → Devices & Services → Add Integration → MiniDSP RS**
6. Enter host `localhost` and port `5380`

### Step 3 — Watchdog automation (recommended for USB devices)

Import the Blueprint to automatically recover from HID interface failures:

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fsb61g2%2Fha-minidsp-rs%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fminidsp%2Fwatchdog.yaml)

You will need:
- The **MiniDSP RS** media player entity (created by the integration)
- A **smart switch** controlling power to the MiniDSP device
- Your **add-on ID** — find it in Developer Tools → Actions → `hassio.addon_restart`
- Your **config entry ID** — find it in Settings → Devices & Services → MiniDSP RS → three-dot menu → System information

### Step 4 — udev rule (HAOS + USB only)

On HAOS, the Linux `snd_usb_audio` kernel module can bind to the MiniDSP's USB audio interfaces and corrupt the HID interface the daemon uses. Deploy this udev rule on the HAOS host to permanently block it:

```bash
# SSH into HAOS host (not into the add-on container)
# Adjust VID/PID for your device — 2752:004b is the FlexHTx
sudo tee /etc/udev/rules.d/99-minidsp-no-audio.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="usb", DEVTYPE=="usb_interface", ATTRS{idVendor}=="2752", ATTRS{idProduct}=="004b", ATTR{bInterfaceClass}=="01", ENV{MODALIAS}=""
ACTION=="bind", SUBSYSTEM=="usb", ATTRS{idVendor}=="2752", ATTRS{idProduct}=="004b", DRIVER=="snd-usb-audio", RUN+="/bin/sh -c 'echo %k > /sys/bus/usb/drivers/snd-usb-audio/unbind && rmmod snd_usb_audio 2>/dev/null || true'"
EOF
sudo udevadm control --reload-rules
```

The rule survives reboots. You do not need to rebuild the add-on or modify the host in any other way.

**Common VID/PID values:**

| Device | idVendor | idProduct |
|---|---|---|
| Flex HTx | 2752 | 004b |
| Flex / Flex DL | 2752 | 0031 |
| 2x4 HD / DDRC-24 | 2752 | 0011 |
| SHD | 2752 | 002e |
| 4x10 HD | 2752 | 002a |

## Entities created

For each discovered MiniDSP device the integration creates:

| Entity | Type | Notes |
|---|---|---|
| Volume Control | `media_player` | Volume + mute for dashboard cards and remote volume via CEC |
| Volume | `number` | Master volume in dB (−127 to 0, 0.5 dB steps) |
| Mute | `switch` | Master mute |
| Input Source | `select` | Device-specific inputs (Analog, HDMI, Toslink, etc.) |
| Preset | `select` | Configuration presets 1–4 |
| Dirac Live | `switch` | Disabled by default; enable if your device has Dirac |
| Input/Output levels | `sensor` | Signal levels in dBFS; disabled by default |

## Wi-DG (network) devices

Enter the Wi-DG IP in the add-on configuration (`widg_ip`). The udev rule is not needed for network-connected devices.

## Source

- Add-on / daemon: [sb61g2/minidsp-rs](https://github.com/sb61g2/minidsp-rs)
- Upstream daemon: [mrene/minidsp-rs](https://github.com/mrene/minidsp-rs)
