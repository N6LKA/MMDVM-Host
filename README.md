# MMDVM-Host — N6LKA Fork (P25 AES Passthrough)

> **License Notice**: The modifications in this fork enable P25 AES encryption passthrough for use on **licensed Part 90 business/public safety frequencies**. P25 encryption is **not permitted on amateur radio (Part 97)** frequencies. Ensure you operate only on frequencies for which you hold a valid FCC license authorizing encrypted voice communications.

This is a fork of [M17-Project/MMDVMHost](https://github.com/M17-Project/MMDVMHost) with modifications to enable P25 AES (Advanced Encryption Standard) voice passthrough on a Pi-Star repeater. The upstream base preserves full **M17, DMR, D-Star, P25, NXDN, System Fusion, POCSAG, FM, and AX.25** functionality.

## What This Fork Changes

MMDVMHost's P25 AES handling had three issues that prevented encrypted audio from passing through a repeater correctly:

1. **P25Data.cpp** — `decodeHeader()` and `decodeLDU2()`: Code to read AlgID, Message Indicator (MI), and Key ID (KID) from incoming frames was commented out. Without this, every transmission was treated as unencrypted (algId=0, MI=zeros, kId=0).

2. **P25Control.cpp** — `m_audio.process()` (IMBE FEC): Forward error correction was applied to every audio frame, including encrypted ones. FEC corrupts AES ciphertext — encrypted IMBE frames must pass through untouched.

3. **P25Control.cpp** — HDU/LDU1 sequencing: The AlgID decoded from the HDU (header) was cleared by `m_rfData.reset()` before the first LDU1 was processed, causing the first voice frame to always be treated as unencrypted.

**Commits applied:**
- `7e96eda` — P25Data.cpp: Uncomment algId/MI/kId decode blocks in `decodeHeader()` and `decodeLDU2()`
- `cf69c53` — MMDVMHost.cpp: Add `initgroups()` call before privilege drop so daemon mode inherits the `dialout` supplementary group (required to open the serial port on Pi-Star)
- `9487b32` — P25Control.cpp: Guard IMBE FEC behind unencrypted check; save/restore HDU algId across reset; uncomment `decodeLDU2()` call in AUDIO state

---

## Installation on Pi-Star

These instructions assume Pi-Star 4.1.x on a Raspberry Pi with an MMDVM board.

### Prerequisites

- SSH access to the Pi-Star node
- Pi-Star filesystem must be writable (`rpi-rw`) for all write operations
- Sufficient RAM for compilation (~256 MB free recommended)

> **Important**: Build from the home directory (`~/`), **not** `/tmp`. Pi-Star's `/tmp` is a small tmpfs RAM disk that runs out of space during compilation.

### Step 1 — Clone and Build

```bash
rpi-rw
cd ~
git clone https://github.com/N6LKA/MMDVM-Host
cd MMDVM-Host
make
```

Compilation takes several minutes on a Raspberry Pi. Only the `MMDVMHost` binary is needed from the build output.

### Step 2 — Stop the Running Service

```bash
sudo systemctl stop mmdvmhost.timer
sudo systemctl stop mmdvmhost.service
```

### Step 3 — Backup and Install

```bash
sudo cp /usr/local/bin/MMDVMHost /usr/local/bin/MMDVMHost.bak
sudo cp ~/MMDVM-Host/MMDVMHost /usr/local/bin/MMDVMHost
```

### Step 4 — Protect the Binary from Pi-Star Auto-Updates

Pi-Star's daily cron (`/etc/cron.daily/pistar-daily`) runs `git pull` on `/usr/local/bin`, which would overwrite the custom binary with the upstream pre-built. Protect against this with `skip-worktree`:

```bash
cd /usr/local/bin
sudo git update-index --skip-worktree MMDVMHost
```

Verify it's set (should show `S MMDVMHost`):

```bash
sudo git -C /usr/local/bin ls-files -v MMDVMHost
```

### Step 5 — Start and Verify

```bash
rpi-ro
sudo systemctl start mmdvmhost.service
sudo systemctl status mmdvmhost.service
```

Check the log for successful startup:

```bash
tail -f /var/log/pi-star/MMDVM-$(date +%Y-%m-%d).log
```

You should see mode lines such as `P25, Starting` and no serial port errors.

> **Note**: On the very first boot after installation, the log file may be owned by `root` (created by the old binary before it was replaced). If MMDVMHost fails to open the log, fix ownership and restart:
> ```bash
> sudo chown mmdvm:mmdvm /var/log/pi-star/MMDVM-$(date +%Y-%m-%d).log
> sudo systemctl restart mmdvmhost.service
> ```
> This is a one-time issue; the log directory is a tmpfs and clears on every reboot.

---

## Allowing Any Radio ID to Key the Repeater

Pi-Star downloads a Radio ID list from RadioID.net and by default rejects any transmission whose source ID is not on that list. To bypass this check and allow any Radio ID (including custom or non-registered IDs required for encrypted channels):

Edit `/etc/mmdvmhost` (Pi-Star dashboard → Configuration → Expert → MMDVMHost) and add or change in the `[P25]` section:

```ini
[P25]
OverrideUIDCheck=1
```

- `0` (default) — checks the source Radio ID against the RadioID.net downloaded list; IDs not on the list are rejected
- `1` — ignores the RadioID.net list and allows transmissions from any Radio ID

---

## Uninstalling / Rolling Back to the Original Binary

If you need to revert to the original Pi-Star MMDVMHost binary:

```bash
rpi-rw

# Remove skip-worktree protection first
sudo git -C /usr/local/bin update-index --no-skip-worktree MMDVMHost

# Stop the service
sudo systemctl stop mmdvmhost.timer
sudo systemctl stop mmdvmhost.service

# Restore the backup
sudo cp /usr/local/bin/MMDVMHost.bak /usr/local/bin/MMDVMHost

rpi-ro

# Restart
sudo systemctl start mmdvmhost.service
sudo systemctl status mmdvmhost.service
```

Alternatively, to pull a fresh upstream pre-built via Pi-Star's normal update mechanism (without restoring from backup):

```bash
rpi-rw
sudo git -C /usr/local/bin update-index --no-skip-worktree MMDVMHost
cd /usr/local/bin
sudo git pull
rpi-ro
sudo systemctl restart mmdvmhost.service
```

---

## Re-enabling Auto-Updates

To allow Pi-Star's daily update to overwrite the custom binary (e.g., for a Pi-Star version upgrade):

```bash
rpi-rw
sudo git -C /usr/local/bin update-index --no-skip-worktree MMDVMHost
rpi-ro
```

After Pi-Star updates and you want to re-protect the binary, repeat Steps 1–4 from the installation instructions and re-apply `--skip-worktree`.

To check protection status at any time:

```bash
sudo git -C /usr/local/bin ls-files -v MMDVMHost
```

- `S MMDVMHost` — protected (skip-worktree set; auto-updates will not overwrite)
- `H MMDVMHost` — unprotected (auto-updates will overwrite on next `git pull`)

---

## Original README

These are the source files for building the MMDVMHost, the program that
interfaces to the MMDVM or DVMega on the one side, and a suitable network on
the other. It supports D-Star, DMR, P25 Phase 1, NXDN, System Fusion, M17,
POCSAG, FM, and AX.25 on the MMDVM, and D-Star, DMR, and System Fusion on the DVMega.

On the D-Star side the MMDVMHost interfaces with the ircDDB Gateway, on DMR it
connects to the DMR Gateway to allow for connection to multiple DMR networks,
or a single network directly. on System Fusion it connects to the YSF Gateway to allow
access to the FCS and YSF networks. On P25 it connects to the P25 Gateway. On
NXDN it connects to the NXDN Gateway which provides access to the NXDN and
NXCore talk groups. On M17 it uses the M17 Gateway to access the M17 reflector system.
It uses the DAPNET Gateway to access DAPNET to receive
paging messages. Finally it uses the FM Gateway to interface to existing FM
networks.

It builds on 32-bit and 64-bit Linux as well as on Windows using Visual Studio
2019 on x86 and x64. It can optionally control various Displays. Currently
these are:

- HD44780 (sizes 2x16, 2x40, 4x16, 4x20)
	- Support for HD44780 via 4 bit GPIO connection (user selectable pins)
	- Adafruit 16x2 LCD+Keypad Kits (I2C)
	- Connection via PCF8574 GPIO Extender (I2C)
- Nextion TFTs (all sizes, both Basic and Enhanced versions)
- OLED 128x64 (SSD1306)
- LCDproc

The Nextion displays can connect to the UART on the Raspberry Pi, or via a USB
to TTL serial converter like the FT-232RL. It may also be connected to the UART
output of the MMDVM modem (Arduino Due, STM32, Teensy).

The HD44780 displays are integrated with wiringPi for Raspberry Pi based
platforms.

The OLED display needs an extra library see OLED.md

The LCDproc support enables the use of a multitude of other LCD screens. See
the [supported devices](http://lcdproc.omnipotent.net/hardware.php3) page on
the LCDproc website for more info.

This software is licenced under the GPL v2 and is primarily intended for amateur and
educational use.
