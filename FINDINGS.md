# Findings & Background

Everything we learned along the way to making this work. Read this if you want context — the **README** is enough to follow the recipe; this document explains *why* the recipe is what it is.

> 🇯🇵 日本語版は [FINDINGS.ja.md](FINDINGS.ja.md) を参照してください。

---

## 1. Timeline — how we got here

| Year | Event |
|---|---|
| 2011 | BeagleBone (white) launched by the BeagleBoard.org foundation |
| 2013 | **BeagleBone Black** released — TI AM335x Cortex-A8 720 MHz, 512 MB RAM, 4 GB eMMC, $45 |
| 2014-02 | **4D Systems publishes the 4DCAPE-70T datasheet** (rev 1.2). Designed for and tested against **Angstrom 2013.06.20** with kernel **3.8.13-bone37** |
| ~2015-2016 | **Angstrom distribution effectively EOL'd**. Successor: Debian images maintained by Robert C. Nelson |
| 2017-2020 | Debian 9 / 10 (kernels 4.9 / 4.19) — LCD7 cape works out-of-the-box with the stock dtbo. This is the cape's golden age |
| 2020-2022 | Debian 11 (kernel 5.10) — still works, considered the "known-good combination" |
| 2023-2024 | Debian 12 (kernel 6.1) — works but starting to show edge cases |
| 2025-2026 | **Debian 13 (kernel 6.12) — LCD7 cape silently stops working.** No DRM connector, no backlight, blank screen |
| 2026-06-05 | Colin Bester opens a forum thread on the same problem (for the LCD4 cape) |
| 2026-06-15 | Colin's `panel-dpi` + OF-graph fix for LCD4 is merged upstream by Robert C. Nelson |
| **2026-06-28** | **This repo documents that for LCD7 on kernel 6.12, even the panel-dpi fix isn't sufficient — but the kernel-downgrade workaround restores everything in 30 minutes** |

The pattern: **the cape itself never changed. The kernel did.** A piece of hardware from 2013, designed against kernel 3.8 conventions, kept working as long as the conventions held. When the kernel rewrote `tilcdc` and `pinctrl-single` for newer DRM standards, the cape's stock overlay silently fell off the support map.

---

## 2. What actually broke between kernel 5.10 and 6.12

### 2.1. `tilcdc` no longer accepts the legacy panel binding

The stock `BB-BONE-LCD7-01-00A3.dts` describes the panel like this:

```dts
panel {
    compatible = "ti,tilcdc,panel";
    panel-info {
        ac-bias = <255>;
        bpp = <16>;
        ...
    };
    display-timings {
        ...
    };
};
```

This is the **legacy `ti,tilcdc,panel` binding**, which was the standard way to describe a DPI panel for the TI LCDC controller in the 3.x / 4.x era.

In kernel 6.x, the `tilcdc` DRM driver expects the **modern OF-graph + `panel-dpi`** binding:

```dts
&lcdc {
    port {
        lcdc_0: endpoint {
            remote-endpoint = <&panel_0>;
        };
    };
};

panel: panel {
    compatible = "panel-dpi";
    panel-timing { ... };
    port {
        panel_0: endpoint {
            remote-endpoint = <&lcdc_0>;
        };
    };
};
```

The legacy binding silently produces no DRM connector. The kernel logs `tilcdc: no encoders/connectors found` and the screen stays blank.

Robert C. Nelson exposed `CONFIG_DRM_TILCDC_PANEL_LEGACY=y` to keep the legacy binding alive on his builds, but the upstream `bb.org-overlays` source itself was never updated to the new binding — and as of mid-2026 still carries a `// FIXME - LCD doesn't init` comment with the OF-graph endpoints stubbed out.

### 2.2. `pinctrl-single` subnodes don't get scanned

A separate, more obscure failure: on kernel 6.12 (verified empirically in this work), the `pinctrl-single` driver reports:

```
pinctrl-single 44e10800.pinmux: no pins entries for pinmux_bb_lcd_pwm_backlight_pins
```

…even when the overlay's compiled `.dtbo` contains the correct pin entries:

```
pinmux_bb_lcd_pwm_backlight_pins {
    pinctrl-single,pins = <0x48 0x06>;
    phandle = <0x01>;
};
```

(0x48 = pin offset for `gpmc_a2`, 0x06 = MUX_MODE6 + PULLDOWN, i.e. EHRPWM1A on P9_14, the backlight PWM pin.)

The dtbo is correct. The driver doesn't pick it up. Whether this is a kernel-side regression or an overlay-application order problem isn't fully diagnosed — and it didn't need to be, because kernel 5.10 doesn't have this problem.

### 2.3. Result: the cascade fails

Both regressions feed into the same dead end:

```
No DRM connector  →  no display path
No pinmux applied →  PWM never enables  →  no backlight  →  panel probe defers forever
```

Replacing one piece (e.g., panel-dpi for the connector) without the other (pinmux working) leaves the panel still deferred-probing. Both layers need to work — which they do, on 5.10.

---

## 3. Why kernel 5.10 was the right choice

When BeagleBoard.org's apt repository offered:

```
5.4, 5.10, 5.15, 6.12
```

(no 6.18 in this image) the calculus was:

- **5.4** — older, less actively maintained.
- **5.10** — LTS, maintained until December 2026 upstream, still in Robert's apt feed. Predates the tilcdc / pinctrl-single rewrites that broke the cape.
- **5.15** — closer to the breaking point. Untested.
- **6.12** — confirmed broken with this cape (the original problem).

5.10 is the **newest "definitely works" kernel for this hardware combination**. It's the same kernel that shipped with Debian 11 (Bullseye), so there's years of community memory confirming it works with BBB capes.

### Crucial Linux fact: kernel ABI is stable

Debian 13.5's userspace (glibc 2.40, systemd 256, etc.) runs perfectly on kernel 5.10 because **Linus Torvalds has enforced for over twenty years**: "**Don't break userspace**." The kernel-userspace boundary is a stable API. The kernel-driver boundary is not.

This is why the procedure works: you can downgrade the kernel (which talks to the hardware, and where the cape regression lives) while keeping the userspace (which talks to apps and which has security fixes you want) up to date.

---

## 4. The people whose work made this possible

This repo is a thin wrapper around years of other people's hard work.

### Robert C. Nelson — [`@RobertCNelson`](https://forum.beagleboard.org/u/RobertCNelson)

The maintainer of [bb-kernel](https://github.com/RobertCNelson/bb-kernel) and the BeagleBoard.org Debian images. Almost every BeagleBone Linux user on Earth runs his kernels via apt without realising it. He is the single point of failure (in the best sense — a benevolent dictator) for the entire BBB ecosystem's Linux story.

He keeps `bbb.io-kernel-5.10-bone` published *and updated* in the BeagleBoard.org apt feed years after most distros would have dropped it. This whole workaround exists only because he does that.

### Colin Bester — [`@Colin_Bester`](https://forum.beagleboard.org/u/Colin_Bester)

In June 2026 Colin posted a meticulous five-layer write-up on the BeagleBoard forum about getting the related 4DCAPE-43T (480x272) to work on Debian 13.5 + kernel 6.18. His diagnosis named the problem we hit: legacy `ti,tilcdc,panel` binding silently failing on new kernels. Even though his `panel-dpi` fix wasn't sufficient for LCD7 on kernel 6.12 (we still had the pinctrl-single layer to deal with), his thread is what convinced us the bug was real and not user error.

Source: [4DCape LCD on BeagleBone Black Debian 13.5 (v6.18.x)](https://forum.beagleboard.org/t/4dcape-lcd-on-beaglebone-black-debian-13-5-2026-05-19-iot-v6-18-x/44044).

### 4D Systems (Australia)

Designed and sells the 4DCAPE-70T. Their 2014 datasheet (rev 1.2, 11 pages) frankly states the cape was developed against Angstrom 2013.06.20 and explicitly disclaims any responsibility for software changes after that. That honesty is worth respecting — they shipped good hardware and documented its assumptions clearly. The reuse of CircuitCo's LCD7-00A3 driver (acknowledged in the datasheet) is also what lets the bb.org-overlays project ship a single dtbo that works for both capes.

### CircuitCo

Designed the original LCD7-00A3 cape that the 4DCAPE-70T is software-compatible with. Without their dtbo work in the BBB era, none of this would have a starting point.

### The `bb.org-overlays` project

[github.com/beagleboard/bb.org-overlays](https://github.com/beagleboard/bb.org-overlays) — the central source of every cape's device-tree overlay. Maintained by the BeagleBoard community. The stock `BB-BONE-LCD7-01-00A3.dtbo` we use here comes from this repo, unmodified.

### And from the wider hardware-hacking culture

A philosophical nod to **Trammell Hudson** ([`@osresearch`](https://github.com/osresearch)) — Magic Lantern, Heads, Thunderstrike, hcpy, papercraft. Not directly involved in this specific work, but the *idea* of "take apart what the vendor stopped supporting, and make it keep working in the modern world" — that's the same instinct that made this repo possible. The `BBB + 4DCAPE-70T` is, in spirit, a small entry in the same tradition.

---

## 5. Things that did NOT work (recorded so others can skip them)

| Approach | Result |
|---|---|
| Stock Debian 13.5 + stock `BB-BONE-LCD7-01-00A3.dtbo` | Blank screen, `no encoders/connectors found` |
| `am33xx_pwm-00A0.dtbo` added — PWM parent enabled | `pwmchip0–3` show up, but `48302200.pwm` (EHRPWM1) still doesn't bind. Backlight still missing |
| `BB-PWM1-00A0.dtbo` added (PWM1 specifically) | Same — EHRPWM1 still doesn't bind. The `pinmux_bb_lcd_pwm_backlight_pins` "no pins entries" warning persists |
| `enable_uboot_cape_universal=1` | Doesn't fix the LCD7 path; cape-universal is a *different* pin assignment scheme that conflicts with the LCD7 overlay |
| GPIO22 manual export to force backlight high | `Device or resource busy` — the panel driver has already claimed P9_14/GPIO22 even though it never actually drove the backlight |
| Custom `panel-dpi` + OF-graph rewrite of LCD7 dtso (Colin's pattern, ported to LCD7) | Gets past `no encoders/connectors found` — `card0-LVDS-1` appears as `connected`, `/dev/fb0` is created. (Side note: the connector name is `LVDS-1` here on kernel 6.12 + our `panel-dpi` rewrite, but `DPI-1` on kernel 5.10 + stock dtbo. The tilcdc driver picks the connector name from how the panel binding declares itself, so different kernel/binding combinations get different names for the *same* DPI output.) But `pinmux_bb_lcd_pwm_backlight_pins` still won't pick up its pins on kernel 6.12, so backlight still never comes up, so panel probe still defers |
| `sudo apt-get dist-upgrade` (per Robert's recommendation in the forum thread) | Only bumps within the 6.12.x series (`6.12.90 → 6.12.93-bone62`). Does not move to 6.18.x on this image. |
| `disable_uboot_overlay_video=1` | Necessary auxiliary fix — the BBB's onboard TDA998x HDMI transmitter otherwise grabs `&lcdc` before our panel can. Combine this with the kernel downgrade for a clean path |

What we learned: on kernel 6.12 specifically, **no combination of overlay tweaks gets past the `pinctrl-single` regression**. Either you patch the kernel itself, or you change kernels. We chose the latter.

---

## 6. Hardware facts uncovered

| Fact | Detail |
|---|---|
| 4DCAPE-70T uses 4D Systems' rebadge of CircuitCo's LCD7-00A3 | EEPROM identifies the cape as `BB-BONE-LCD7-01 / 00A3` — the BBB loads the same overlay either way |
| LCD panel is the **ThreeFive S9700RTWV35TR** (or compatible: see `RF110-XSD-7.0` silkscreen on the panel back) | 800×480, 30 MHz pixel clock, 24-bit but driven as 16bpp RGB565 |
| Backlight is **PWM-controlled via EHRPWM1A on P9_14** (AM335x register 0x48302200) | This is the pin that needs `pinctrl-single` to apply MUX_MODE6 |
| LCD active area: 154.1 mm × 85.9 mm | from 4D Systems mechanical drawing |
| 4DCAPE-70T draws non-trivial current — **5V/2A external DC adapter is required** | USB-only power (500 mA) cannot drive the LCD backlight reliably |
| **EEPROM jumpers must both be closed** | Per 4D Systems datasheet, otherwise the cape ID may not be detected |
| BBB's onboard HDMI transmitter (**NXP TDA998x**) shares the LCDC | If you don't disable the HDMI virtual cape, tilcdc binds to tda998x instead of your panel |
| GPIO22 = P9_14 = backlight enable | When the panel driver claims this pin, you can't manually export it from `/sys/class/gpio/` |

---

## 7. Lessons (the meta-takeaways)

### 7.1. Old hardware ages with its kernel

Every hardware accessory designed in 2013-2014 has an implicit assumption: "this is how Linux drivers work today." When the kernel rewrites those drivers a decade later, the hardware doesn't break — its *driver glue* breaks. The hardware was fine the whole time.

For obsolete hardware you want to keep using, **the kernel you run is part of the hardware compatibility list**, just as much as the cape itself.

### 7.2. Userspace stability is a gift; preserve the option to use it

Linus' "Don't break userspace" rule is why we could downgrade *just the kernel*. On nearly any other operating system this would be impossible — kernel and userland ship as a unit. On Linux you can mix and match within a generation or two without explosions. Use that freedom.

### 7.3. Individual maintainers are infrastructure

The reason this fix is a 30-minute exercise instead of a multi-day porting effort is that Robert C. Nelson personally keeps `bbb.io-kernel-5.10-bone` updated and reachable via apt. If he stops, all of this becomes harder overnight. Consider sponsoring people like Robert — they hold up entire ecosystems alone.

### 7.4. "Doesn't display" is not "doesn't work"

A subtle one. We had a working `/dev/fb0` long before the LCD ever lit up. The kernel was perfectly happy. The cape's *display path* was broken. Don't confuse one for the other when diagnosing — separate "framebuffer exists" from "panel signal is being generated" from "backlight is on" from "you can see pixels."

### 7.5. Persistence beats triage

At several points during this debug session, the rational engineering call was to declare the kernel 6.12 path dead and switch to a known-good older image. Pursuing it further and finding the kernel-only downgrade saved hours every time someone else hits this same problem in the future. Sometimes the engineering-rational answer ("cut your losses, fall back") and the right answer ("there's one more thing to try") diverge.

---

## 8. Where to learn more

- **BeagleBone Black wiki**: [elinux.org/Beagleboard:BeagleBoneBlack](https://elinux.org/Beagleboard:BeagleBoneBlack)
- **Cape device tree overlay docs**: [elinux.org/Beagleboard:BeagleBoneBlack_Debian#U-Boot_Overlays](https://elinux.org/Beagleboard:BeagleBoneBlack_Debian#U-Boot_Overlays)
- **bb.org-overlays repo**: [github.com/beagleboard/bb.org-overlays](https://github.com/beagleboard/bb.org-overlays)
- **Robert C. Nelson's bb-kernel**: [github.com/RobertCNelson/bb-kernel](https://github.com/RobertCNelson/bb-kernel)
- **BeagleBoard forum** (where Colin's thread lives): [forum.beagleboard.org](https://forum.beagleboard.org)
- **Linux kernel `tilcdc` driver source**: `drivers/gpu/drm/tilcdc/` in any kernel tree
- **`pinctrl-single` binding doc**: `Documentation/devicetree/bindings/pinctrl/pinctrl-single.txt`
- **4D Systems 4DCAPE-70T datasheet**: [resources.4dsystems.com.au/datasheets/cape/4DCAPE-70T/](https://resources.4dsystems.com.au/datasheets/cape/4DCAPE-70T/)
- **Linus on "Don't break userspace"** (the canonical rant): [lkml.org/lkml/2012/12/23/75](https://lkml.org/lkml/2012/12/23/75)

---

*This document records what was learned during one afternoon of debugging — 2026-06-28, Kumamoto, Japan. Corrections and additions welcome via pull request.*
