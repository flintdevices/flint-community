<div align="center">
  <img src="assets/flint_logo.png" alt="Flint" width="320" />
  <h1>Flint | Offensive Security Firmware</h1>
</div>

Offensive security firmware for the M5Stack CardputerZero, written in Python.
Pure Python stack with a pygame emulator for development and a framebuffer/evdev runtime for hardware targets.

# flint-community

This is the public issue tracker and feedback hub for **Flint** — offensive security firmware for Linux hardware.

The Flint source is closed, but this space is open. Use it to report bugs, request features, ask questions, or share ideas.

---

## Hardware Prerequisites

Several WiFi tools (deauth, beacon monitor, PMKID/handshake capture, probe scan, etc.) require monitor mode. The native BCM43438 chip on CardputerZero and RPi 3B does **not** support monitor mode out of the box. You need one of the following:

### Option A — nexmon (no extra hardware)

Firmware patch for the BCM43438. Supported on both RPi 3B and CardputerZero.

**Limitation:** while nexmon is active, `wlan0` cannot connect to networks or act as an AP — single radio, can't do both simultaneously.

**Install:**

```bash
sudo apt update
sudo apt install raspberrypi-kernel-headers
git clone https://github.com/seemoo-lab/nexmon
cd nexmon && source setup_env.sh
cd patches/bcm43430a1/7_45_41_26/nexmon
make && make backup-firmware && make install-firmware
reboot
```

After reboot, Flint auto-detects nexmon and offers enable/disable from **WiFi › Settings › Monitor Setup**.

### Option B — USB WiFi dongle (recommended for power users)

No firmware changes required. Native WiFi remains fully usable in parallel.

- **Supported adapters:** MT7612U (recommended), AR9271
- **Drivers:** `mt76x2u`, `ath9k_htc` — included in Raspberry Pi OS
- Plug in → Flint detects automatically. No configuration needed.

---

Flint's built-in UI surfaces this guide too: **WiFi › Settings › Monitor Setup**.

---

## Opening an issue

Just open one. No template required.

If you want to give more context, here are some things that help:

- **Bug report** — what you did, what you expected, what actually happened. Device and Flint version if you know it.
- **Feature request** — what you'd like to do, and why the current tools don't cover it.
- **Question** — anything about usage, hardware compatibility, or how something works.

If you're not sure which category fits, don't worry about it. Open the issue and we'll figure it out together.

→ [Open an issue](https://github.com/flintdevices/flint-community/issues/new)

---

## Links

- [flintdevices.dev](https://flintdevices.dev) — landing page, install instructions, screenshots
- [flintdevices](https://github.com/flintdevices) — GitHub organization

---

## Scope

This repo is for feedback only — no source code lives here. Pull requests won't be merged.

Issues may be closed without comment if they're out of scope (e.g. requests for help cracking networks you don't own).
