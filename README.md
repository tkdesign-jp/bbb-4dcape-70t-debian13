# BBB + 4DCAPE-70T on Debian 13.5 (Trixie)

A verified working procedure for running the **4D Systems 4DCAPE-70T** 7" LCD cape on the **BeagleBone Black** with **Debian 13.5 (Trixie)**, by downgrading the kernel to the still-maintained `5.10-bone` LTS branch.

> 🇯🇵 日本語版は [README.ja.md](README.ja.md) を参照してください。
> ⚠️ **Before following any step**, read [DISCLAIMER.md](DISCLAIMER.md) — particularly the security consideration for kernel 5.10 LTS support ending December 2026.
> 📖 Want the backstory, the people who made this possible, and the dead-ends we hit? See [FINDINGS.md](FINDINGS.md).

---

## The problem this solves

Trying to use the 4DCAPE-70T on a fresh **BeagleBone Black** with the official **Debian 13.5 (Trixie) base image (2026-05-19, kernel 6.12.x)** results in a blank LCD and these kernel errors:

```
tilcdc 4830e000.lcdc: no encoders/connectors found
pinctrl-single 44e10800.pinmux: no pins entries for pinmux_bb_lcd_pwm_backlight_pins
platform backlight: deferred probe pending: platform: supplier 48302200.pwm not ready
```

Root causes:

1. The 4DCAPE-70T was designed for kernel 3.8 (Angstrom 2013) and uses the **legacy `ti,tilcdc,panel` device-tree binding**, which the kernel 6.x `tilcdc` driver no longer recognises as a valid DRM panel.
2. Kernel 6.12 `pinctrl-single` does not interpret the cape overlay's pinmux subnodes correctly, so the PWM backlight pins are never configured.
3. As a result, no DRM connector registers, no `backlight` sysfs device is created, and the LCD stays dark.

A community-driven fix for the kernel 6.18 LCD4 overlay exists ([Colin Bester's panel-dpi rewrite](https://forum.beagleboard.org/t/4dcape-lcd-on-beaglebone-black-debian-13-5-2026-05-19-iot-v6-18-x/44044)), but on kernel 6.12 the LCD7 cape continues to fail at the `pinctrl-single` layer even after porting the `panel-dpi`/OF-graph fix.

### The fix that actually works

**Downgrade the running kernel to `bbb.io-kernel-5.10-bone`** while keeping the Debian 13.5 userspace intact. On kernel 5.10 the stock `BB-BONE-LCD7-01-00A3.dtbo` from `bb.org-overlays` works without modification, the legacy tilcdc binding still binds, and the PWM backlight comes up cleanly.

No SD-card re-flash, no kernel source-tree work, no overlay patching.

---

## Verified configuration

| Component | Value |
|---|---|
| Board | BeagleBone Black (TI AM335x Cortex-A8, 512 MB RAM, rev C) |
| Cape | 4D Systems **4DCAPE-70T** (800x480, resistive touch, ThreeFive S9700RTWV35TR / RF110-XSD-7.0 panel) |
| Cape EEPROM | `BB-BONE-LCD7-01` rev `00A3` |
| OS image | `am335x-debian-13.5-base-v6.12-armhf-2026-05-19-4gb.img.xz` |
| Userspace | Debian 13.5 Trixie (armhf) |
| Kernel (after fix) | **5.10.x-bone** (installed via `bbb.io-kernel-5.10-bone`) |
| LCD overlay | stock `BB-BONE-LCD7-01-00A3.dtbo` from `beagleboard/bb.org-overlays` |
| Power | 5V 2A AC adapter on the BBB DC barrel jack (USB-only power is **not** sufficient) |
| Date verified | 2026-06-28 |

What works after the procedure:

- LCD bright and stable
- Kernel `/dev/fb0` framebuffer present (800x480, 16bpp RGB565)
- Linux TTY console (login prompt) renders to the LCD
- DRM connector `card0-DPI-1` reports `connected` (parallel RGB/DPI output; connector name comes from tilcdc and may differ slightly on other distros)
- Touch panel events available via `/dev/input/event1` (calibration is a separate step)

---

## Prerequisites

- BeagleBone Black with the 4DCAPE-70T physically mated (cape on top of BBB, EEPROM jumpers as shipped — both shorted).
- 5V 2A power supply connected to the BBB's DC barrel jack.
- A microSD card (4 GB or larger) with the Debian 13.5 base image flashed.
- Initial network connectivity (wired Ethernet is easiest; Wi-Fi via a known-good USB dongle works too).
- A second computer (Mac/Linux/Windows) for SSH access. The BBB serial console works too if you have a 3.3V USB-UART adapter.

---

## Step-by-step procedure

### 1. Boot the stock Debian 13.5 image

Flash `am335x-debian-13.5-base-v6.12-armhf-2026-05-19-4gb.img.xz` to a microSD card (e.g. with balenaEtcher), insert into the powered-off BBB, and boot. The image runs from SD; the eMMC is untouched, so you can revert at any time by removing the SD.

Default login: `debian` / `temppwd`.

### 2. Get a network connection

Wired Ethernet via DHCP is the simplest path. Verify with:

```bash
ip a
ping -c 3 8.8.8.8
```

If you must use Wi-Fi, **first check what is already managing the network**, then install NetworkManager:

```bash
# What is currently running?
systemctl status systemd-networkd connman NetworkManager 2>/dev/null | grep -E "Active|Loaded"

# If neither connman nor NetworkManager is active, install NM:
sudo apt update
sudo apt install -y network-manager
sudo nmcli device wifi connect "YOUR_SSID" password "YOUR_PASSPHRASE"
```

If `connman` or another stack is already active, either use that or disable it before enabling NetworkManager — running two network managers concurrently leads to interfaces bouncing.

(Also be aware: some older USB Wi-Fi dongles do not support WPA3 — fall back to a WPA2 SSID if your router exposes both.)

### 3. Install the 5.10-bone kernel package

```bash
sudo apt update
sudo apt install -y bbb.io-kernel-5.10-bone
```

This installs the kernel and the matching device-tree blobs and modules under `/lib/modules/5.10.x-bone/`. It does **not** remove the 6.12 kernel — both can coexist.

Verify the package is installed:

```bash
dpkg -l | grep "kernel-5.10-bone"
ls /lib/modules/ | grep "5\.10"
```

You should see a `5.10.x-bone` directory under `/lib/modules/`.

### 4. Tell U-Boot to boot the 5.10 kernel

**Back up `/boot/uEnv.txt` first.** A typo here can prevent the BBB from booting; the backup lets you recover instantly by pulling the SD card on another machine and restoring the `.bak` file:

```bash
sudo cp /boot/uEnv.txt /boot/uEnv.txt.bak
```

Then edit:

```bash
sudo nano /boot/uEnv.txt
```

Find the line near the top:

```
uname_r=6.12.xx-bone62
```

Change it to the 5.10 version you just installed. The exact value depends on the package version; check `ls /lib/modules/` and substitute. Example:

```
uname_r=5.10.240-bone80
```

Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 5. Reboot and verify

```bash
sudo reboot
```

After the BBB comes back, SSH in again and confirm the running kernel:

```bash
uname -r          # should print 5.10.x-boneXX
```

### 6. Check the LCD

The cape should already be working at this point — the kernel TTY login prompt appears on the LCD as soon as the boot finishes.

**How the overlay gets loaded:** the BBB's U-Boot reads the cape's I²C EEPROM at power-on, detects the `BB-BONE-LCD7-01` board ID, and automatically loads the matching `BB-BONE-LCD7-01-00A3.dtbo` from `/lib/firmware/`. No edits to `/boot/uEnv.txt`'s `uboot_overlay_addr*` lines are required when the EEPROM is intact. You can confirm with:

```bash
sudo beagle-version | grep -i "loaded overlay"
# Expect a line like: UBOOT: Loaded Overlay:[BB-BONE-LCD7-01-00A3]
```

(`beagle-version` lives in the `bb-customizations` package — it's preinstalled on the BeagleBoard.org Debian images but may be absent on a stripped-down setup. If `command -v beagle-version` returns nothing, use these alternatives:)

```bash
# Direct from /proc/device-tree
ls /proc/device-tree/chosen/overlays/ 2>/dev/null
cat /proc/cmdline | tr ' ' '\n' | grep uboot_detected_capes
# Or via dmesg
dmesg | grep -i "loaded overlay"
```

If `BB-BONE-LCD7-01-00A3` doesn't show up by any of these methods, either the EEPROM is unreadable or the dtbo isn't present in `/lib/firmware/` — fall back to specifying it manually in `/boot/uEnv.txt`:

```
uboot_overlay_addr0=/lib/firmware/BB-BONE-LCD7-01-00A3.dtbo
```

Quick sysfs checks:

```bash
ls /dev/fb*                                  # expect /dev/fb0
ls /sys/class/backlight/                     # expect a backlight device (named "backlight" on this config)
ls /sys/class/drm/ | grep "^card0-"          # expect card0-DPI-1 (or similar; depends on tilcdc binding)
sudo cat /sys/class/drm/card0-*/status       # expect "connected"
dmesg | grep -iE "tilcdc|panel|backlight"    # no "no encoders/connectors found"
```

Manual backlight test (always read `max_brightness` first — the scale is driver-defined, not always 0–100):

```bash
# Find the device-specific max value
cat /sys/class/backlight/*/max_brightness
# On this verified configuration, prints: 100

# Set to max (substitute the value from above)
echo 100 | sudo tee /sys/class/backlight/*/brightness

# Off
echo 0 | sudo tee /sys/class/backlight/*/brightness
```

(On the 4DCAPE-70T with the stock dtbo, `max_brightness` is 100, so the `echo 100` form happens to be correct. On other capes or different overlays it can be 7, 255, or another value — always read first.)

### 7. (Optional) GUI on top

Out of scope for this guide. Once the framebuffer is up, any standard X11 stack on Debian 13 works — install LXQT, openbox, or run a kiosk Chromium on top of `/dev/fb0`.

```bash
# Minimal X + a window manager (example)
sudo apt install -y xserver-xorg-core xserver-xorg-video-fbdev openbox xinit
```

---

## Reverting to the 6.12 kernel

Both kernels are installed side by side. To return to 6.12:

```bash
sudo nano /boot/uEnv.txt
# Change uname_r back to the original 6.12.x-boneXX value
sudo reboot
```

(The LCD will go dark again — that is expected on 6.12.)

---

## Why kernel 5.10 specifically?

The available BeagleBoard.org kernel packages on this image, at this date, are:

```
bbb.io-kernel-5.4-bone
bbb.io-kernel-5.10-bone     ← chosen
bbb.io-kernel-5.15-bone
bbb.io-kernel-6.12-bone     ← default, doesn't work with this cape
```

**5.10 is an LTS** with the longest combination of "still maintained" and "old enough to predate the tilcdc / pinctrl-single rewrites that broke this cape." 5.4 is older but receives fewer updates. 5.15 is closer to the breaking point and was not tested by the author.

If 5.10 reaches EOL (currently December 2026, see DISCLAIMER) and is removed from the BeagleBoard.org apt feed, this guide will need updating. The 5.15-bone branch may still work and would be the next fallback.

---

## Acknowledgements

This write-up stands on the work of others:

- **[Colin Bester](https://forum.beagleboard.org/u/Colin_Bester)** — for the panel-dpi / OF-graph diagnosis on the LCD4 cape that named the problem, even though that specific fix turned out not to be sufficient for LCD7 on kernel 6.12.
- **[Robert C. Nelson](https://forum.beagleboard.org/u/RobertCNelson)** — for maintaining `bbb.io-kernel-5.10-bone` and keeping the multi-kernel apt feed alive, which is what made the kernel-only downgrade possible.
- **[4D Systems](https://www.4dsystems.com.au/)** — for the 4DCAPE-70T hardware and its 2014 datasheet that confirmed the cape was designed for kernel 3.8-era Linux.
- **[bb.org-overlays](https://github.com/beagleboard/bb.org-overlays)** project — for the stock `BB-BONE-LCD7-01-00A3.dts` that simply works on the right kernel.
- The **BeagleBoard forum thread** [4DCape LCD on BeagleBone Black Debian 13.5 (v6.18.x)](https://forum.beagleboard.org/t/4dcape-lcd-on-beaglebone-black-debian-13-5-2026-05-19-iot-v6-18-x/44044) — for confirming this is a real problem and for documenting the LCD4-side fix.

This guide does not modify any of those works; it only describes how to compose them to solve a specific real-world combination.

---

## License

[MIT](LICENSE) — knowledge belongs to whoever needs it. No warranty. See also [DISCLAIMER.md](DISCLAIMER.md) for the kernel 5.10 EOL caveat.
