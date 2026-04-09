# Claude-Rescue-Pi — Full Bill of Materials

> Combined BOM: Rescue-Pi SBC build + per-host bridge hardware. Prices are approximate, USD, 2026 street prices. Local VAT and shipping not included.

## A. Rescue-Pi SBC build (buy once)

The single-purpose appliance that runs Claude Code CLI, reachable via Tailscale + Cloudflare Access, physically next to the workstation.

| # | Item | Spec / example | Qty | Unit $ | Subtotal $ |
|---|---|---|---|---|---|
| A1 | Single-board computer | Raspberry Pi 5, 4 GB | 1 | 60 | 60 |
| A2 | Active cooler | Official Raspberry Pi 5 Active Cooler | 1 | 5 | 5 |
| A3 | Case with fan | Official Pi 5 Case **or** Argon NEO 5 | 1 | 20 | 20 |
| A4 | Power supply | Official 27 W USB-C PSU (Pi 5) | 1 | 12 | 12 |
| A5 | Primary storage | 128 GB USB3 SSD (Samsung T7 / Crucial X6) in USB3 enclosure | 1 | 30 | 30 |
| A6 | Emergency boot card | 32 GB A2 microSD, pre-flashed spare image, kept in drawer | 1 | 8 | 8 |
| A7 | Optional: NVMe path | Pi 5 NVMe HAT + 128 GB NVMe (alternative to A5, cleaner mechanically) | 0 | 45 | 0 |
| **A — SBC subtotal** | | | | | **135** |

Notes:
- Pi 5 chosen for thermals, PMIC with proper power button, and the longest software support horizon. Pi 4 4 GB is an acceptable ~$20 cheaper fallback if you already own one.
- USB3 SSD is non-negotiable for a device that must be reliable years after build. SD cards wear out silently.
- A6 (spare SD) is cheap insurance: if the SSD dies at 2 a.m., swap the card and you're running in minutes.
- A7 NVMe path is an aesthetic upgrade; functionally equivalent to A5. Pick one, not both.

## B. Host bridge — per workstation (buy one kit per protected host)

For each workstation Claude-Rescue-Pi is expected to recover. Daniel's current primary desktop is the first (and likely only) target, so buy **one** kit initially.

### B1. Layer A — Serial console (primary rescue path)

| # | Item | Spec / example | Qty | Unit $ | Subtotal $ |
|---|---|---|---|---|---|
| B1a | **One-piece cable (recommended):** USB-A → 9-pin motherboard COM header, FTDI FT232RL + MAX3232 inline | Search "FTDI USB to motherboard 9-pin COM header RS232 cable", e.g. DTECH / CableCreation | 1 | 20 | 20 |
| B1b | Fallback: 9-pin → DB9 rear bracket | Generic motherboard COM bracket cable | 0 | 5 | 0 |
| B1c | Fallback: DB9 null-modem cable, ~1 m | Generic F-F null-modem | 0 | 5 | 0 |
| B1d | Fallback: USB-RS232 adapter | FTDI FT232RL–based | 0 | 15 | 0 |
| B1e | Contingency: PCIe serial card | StarTech PEX1S553 or equivalent, only if the host has no COM header at all | 0 | 20 | 0 |
| **B1 subtotal (one-piece path)** | | | | | **20** |

Buy either B1a (one piece) **or** B1b+B1c+B1d (three pieces). If the host has no `JCOM1` header, buy B1e instead. Do not buy all of them.

### B2. Layer B — Direct-link ethernet (fallback for misconfigured networking)

| # | Item | Spec / example | Qty | Unit $ | Subtotal $ |
|---|---|---|---|---|---|
| B2a | USB 2.5GbE dongle for the host | Realtek RTL8156B–based (CableCreation / UGREEN) | 1 | 15 | 15 |
| B2b | Short Cat6 cable | 0.5 m | 1 | 4 | 4 |
| **B2 subtotal** | | | | | **19** |

The Pi side uses either its onboard gigabit or a second USB NIC (see note under B3).

### B3. Pi-side networking — allocation

Depending on how the Rescue-Pi itself reaches the LAN:

- **Option 1 (simpler):** Pi's onboard gigabit → Layer B direct link (static `192.168.250.1/30`). Pi's LAN/Tailscale connectivity rides on Wi-Fi. Zero extra hardware.
- **Option 2 (more robust):** Pi's onboard gigabit → LAN (wired, reliable), extra USB gigabit dongle → Layer B direct link. Adds ~$15 for a USB NIC.

Option 1 is the default. Upgrade to Option 2 only if Wi-Fi on the Pi proves unreliable.

| # | Item | Spec / example | Qty | Unit $ | Subtotal $ |
|---|---|---|---|---|---|
| B3a | Optional extra USB gigabit NIC for the Pi | Realtek RTL8156B or similar | 0 | 15 | 0 |
| **B3 subtotal** | | | | | **0** |

### B4. Layer C — Smart plug (force-reboot path)

| # | Item | Spec / example | Qty | Unit $ | Subtotal $ |
|---|---|---|---|---|---|
| B4a | Smart plug with local HTTP API | **Shelly Plug S** (local REST, no cloud required) | 1 | 20 | 20 |
| **B4 subtotal** | | | | | **20** |

Wired between the UPS output and the workstation PSU. Pi itself on a different outlet.

### B — Host bridge subtotal (one host)

| Sub-section | $ |
|---|---|
| B1 (serial, one-piece) | 20 |
| B2 (direct-link ethernet) | 19 |
| B3 (Pi networking extras) | 0 |
| B4 (smart plug) | 20 |
| **B — host bridge subtotal** | **59** |

## Grand total (one Rescue-Pi + one protected host)

| Block | $ |
|---|---|
| A. Rescue-Pi SBC build | 135 |
| B. Host bridge (×1) | 59 |
| **Project total** | **194** |

## Per-additional-host cost

If you later protect a second workstation (same Rescue-Pi, second host kit):

| | $ |
|---|---|
| Extra host bridge kit (B1 + B2 + B4) | 59 |

The Pi itself scales — one Pi can talk to multiple hosts, limited mostly by how many USB serial adapters you can keep plugged in and how many smart plugs your LAN can reach.

## Minimum viable build (budget path)

If you want to start with the absolute minimum and add later:

| Item | $ |
|---|---|
| Pi 5 4 GB + cooler + PSU + 128 GB USB3 SSD | 107 |
| One-piece FTDI USB-to-motherboard-COM-header cable | 20 |
| Shelly Plug S | 20 |
| **MVP subtotal** | **147** |

This gets you: serial console (Layer A) + power control (Layer C) + a Pi running Claude. It skips the case (use the PSU on a desk), the spare SD, and Layer B entirely. Everything else can be added as Phase 2.

## What's deliberately not in this BOM

- **Video / HDMI capture (PiKVM).** Out of scope by design — text rescue only.
- **LTE cellular failover.** Phase 2+; only worthwhile if home internet outages become a recovery blocker.
- **KVM-over-IP appliances** (e.g. TinyPilot, NanoKVM). They solve a different problem (remote GUI desktop access) and don't help with a non-booting Linux workstation where the right interface is a shell anyway.
- **Second workstation NIC via PCIe** instead of USB dongle. Functionally equivalent, costs more, opens the case — skip unless you already have one.
- **Cables and adapters for the Pi's own power/network** beyond what's listed. Assumed to already exist in a desk drawer.
