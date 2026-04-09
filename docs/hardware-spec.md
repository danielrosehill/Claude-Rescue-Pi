# Claude-KVM — Suggested Hardware Spec

> **Status:** WIP recommendation, open to change once the serial path on the workstation is pinned down.

## TL;DR

**Raspberry Pi 5 (4 GB) + 128 GB USB3 SSD, running Raspberry Pi OS Lite 64-bit (Bookworm).**

Rationale: this device is single-purpose and must be reliable for years with no hands-on. Pick the mainstream board with the longest software support window, the best thermals, and boot-from-SSD.

## Board options, ranked

### 1. Raspberry Pi 5 (4 GB) — **recommended**

- **Why:** Current flagship, supported through ~2036. Native NVMe via HAT, gigabit ethernet, 2× USB3 (good for USB-serial + USB-ethernet dongle simultaneously at full speed), 2× USB2, proper PMIC with power button, much better thermals than Pi 4 with the official active cooler.
- **RAM:** 4 GB is plenty. Claude CLI + `tio` + a tmux session are tiny. 8 GB is overkill; 2 GB is fine but 4 GB is the sweet spot for resale / reuse.
- **Cost:** ~$60 board + ~$15 active cooler + ~$12 PSU.
- **Caveat:** Pi 5 runs hot under sustained load — use the **official active cooler** or a case with a fan. Non-negotiable for an always-on device.

### 2. Raspberry Pi 4 Model B (4 GB) — solid fallback

- **Why:** Cheaper, widely available, rock-solid, huge community. USB3 + gigabit. Fully adequate for this job.
- **Tradeoff:** Older, warmer relative to workload, shorter remaining support horizon than Pi 5.
- **Pick this if:** you already have one in a drawer. Don't buy one new in 2026.

### 3. NanoPi R5S / R6S (FriendlyElec)

- **Why it's tempting:** Two gigabit ports out of the box (one could go to the workstation direct link, the other to the LAN), metal case, low idle power, Rockchip RK3568 / RK3588S.
- **Why I'd still pass:** Weaker software ecosystem. FriendlyElec's own images (FriendlyWrt, FriendlyCore) are fine but lag behind mainline; Armbian works but isn't as battle-tested as Raspberry Pi OS. For a device whose entire purpose is "work correctly the one day per year I need it", boring + mainstream wins.
- **Pick this if:** dual NIC out of the box is worth more to you than software boringness. Legitimate choice.

### 4. NanoPi NEO3 / Zero 2 — too small

- Single USB port, limited RAM, limited thermal headroom, fiddly. Don't.

### 5. Raspberry Pi Zero 2 W — no

- USB-OTG only, no gigabit, 512 MB RAM. Claude CLI will be painful. Only interesting if you specifically want USB-gadget mode and nothing else, and then it's a secondary device, not the main one.

### 6. Mini PC (N100 / N150) — overkill but valid

- e.g. Beelink / GMKtec N100, ~$140. x86_64, 8–16 GB RAM, NVMe, two NICs on some models, runs plain Debian or Ubuntu Server, Claude CLI installs without any arm64 caveats, handles PXE rescue netboot trivially.
- **Pick this if:** you want zero ARM friction, and you're OK with ~6–10 W idle instead of ~2–3 W.
- Honestly a strong contender for this role. The only reason to prefer the Pi is form factor and the "this is just a recovery appliance" vibe.

## Recommended build (Pi 5 path)

| Component | Spec | Approx cost |
|---|---|---|
| Board | Raspberry Pi 5, 4 GB | $60 |
| Cooling | Official Active Cooler | $5 |
| Case | Official Pi 5 case (fan) **or** Argon NEO 5 | $15–25 |
| PSU | Official 27 W USB-C PSU | $12 |
| Storage | 128 GB USB3 SSD (e.g. Samsung T7, Crucial X6) in USB3 enclosure. **Not** an SD card for primary. | $25–40 |
| (Optional) Boot SD | 32 GB A2 card as emergency fallback image | $8 |
| USB-to-serial | FTDI FT232RL or CP2102 adapter with Dupont leads | $10 |
| USB ethernet dongle | Gigabit, AX88179A chipset (well supported on Linux) | $15 |
| Ethernet cable | Cat6, 0.5 m, for direct workstation link | $4 |
| Smart plug | Shelly Plug S (local API, no cloud required) | $20 |
| **Total** | | **~$175–200** |

Alternative storage: Pi 5 NVMe HAT + 128 GB NVMe (~$25 HAT + ~$20 drive) is cleaner mechanically than a USB3 SSD hanging off the side. Slightly more expensive, nicer build.

## OS recommendation

**Raspberry Pi OS Lite, 64-bit (Bookworm or the current stable).**

- Official, best-supported OS on Pi hardware. Kernel, firmware, and bootloader updates come through `apt` with no surprises.
- "Lite" = no desktop. This box is headless forever.
- Debian-based, so all the usual tools (`tio`, `minicom`, `picocom`, `cloudflared`, `tailscale`, Node.js for Claude CLI) install cleanly.
- Long-term stability > bleeding edge. Do **not** use Raspberry Pi OS with desktop, and do **not** use Ubuntu Server on Pi for this role (it works, but you lose the vendor-blessed update path for firmware).

### Alternatives considered

- **Ubuntu Server 24.04 LTS arm64** — fine, works, 5-year support. Use this only if you strongly prefer Ubuntu's userland. Slightly worse firmware story on Pi.
- **DietPi** — very minimal, great idle footprint, but an extra layer of its own scripts I'd rather not debug during a 2 a.m. recovery. Skip.
- **NixOS on Pi** — reproducible, beautiful, overkill for a 1-purpose appliance. Skip unless you already run Nix everywhere.
- **Armbian (if going NanoPi route)** — fine choice on non-Pi boards. Use the "Bookworm minimal" image.

## Boot & storage strategy

- Primary boot: **USB3 SSD** (or NVMe via HAT). SD cards wear out and corrupt silently; this device must survive years of idle with occasional writes.
- Keep a **tested SD card image** in a drawer with the Pi, flashed and ready. If the SSD ever fails, swap the SD and you're back in minutes.
- Enable `rpi-eeprom` auto-updates but pin to the "default" (stable) channel, not "latest".
- Journald to volatile storage (`Storage=volatile` in `journald.conf`) to reduce SSD writes; mirror important recovery logs into the workspace git repo instead.

## Power & uptime

- 24/7 operation. Budget ~2–3 W idle for Pi 5, ~2 W for Pi 4.
- Put the Pi on a **different circuit / UPS** than the workstation if possible, so that a PSU blow-up on the workstation doesn't also take out its recovery path.
- Do **not** put the Pi on the smart plug it controls. (Obvious, but worth writing down.)

## What to buy first if budget-constrained

If you want to start cheap and iterate:

1. Pi 5 (4 GB) + active cooler + 27 W PSU + 128 GB USB3 SSD — ~$110.
2. FTDI USB-serial adapter — $10.
3. Shelly Plug S — $20.

That's the minimum viable Claude-KVM. Everything else (second NIC, NVMe HAT, LTE fallback) can be added later.
