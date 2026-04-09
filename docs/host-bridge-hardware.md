# Host ↔ Rescue-Pi Bridge — Hardware Recommendation

> Goal: define the **lowest-level wired bridge** between a primary workstation and a Claude-Rescue-Pi that (a) doesn't take the workstation's primary ethernet port, and (b) exposes a surface Claude on the Pi can interact with via its Bash tool.

## Design constraints

1. **Don't consume the workstation's primary NIC.** On most single-ethernet desktops that port carries the LAN + VPN and has to stay on the LAN. The bridge must use a different physical path.
2. **Must work when the host's kernel networking stack is offline.** (See `design.md` §3.1.) This rules out anything that depends on `sshd` or the NIC driver being alive.
3. **Must be a surface Claude can drive.** Something Claude can `read()` and `write()` to from its Bash tool — i.e. a character device (`/dev/ttyUSB0`) or a socket on the Pi. No GUI, no video.
4. **Must survive a full OS reinstall or an unmountable root filesystem.** Ideally observable from BIOS POST onward.
5. **Should be cheap, boring, and not depend on any specific CPU/chipset feature the host might not have.**

## The answer: three-layer wired bridge

The bridge is not one cable — it's a small stack of independent wired connections, each covering a different layer of the failure model:

```
                                 ┌────────────────────────┐
                                 │     Rescue-Pi (Pi 5)   │
                                 │                        │
          ┌──── RS-232 ──────────┤ USB-RS232 (FTDI)  (A)  │
          │  (null-modem DB9)    │                        │
          │                      │ USB-2.5GbE dongle (B)  │
          │                      │                        │
          │     ┌── Cat6 ────────┤                        │
          │     │  (direct)      │ Wi-Fi / Tailscale (C)  │
          │     │                └────────────────────────┘
          │     │
          │     │                     ┌──────────────────┐
  JCOM1   │     │                     │    Smart Plug    │────── mains
  header ─┘     │                     │  (Shelly/Tasmota)│
 (on mobo)      │                     └──────────────────┘
                │                              │
                │                              │ switches power to
                │                              ▼
         USB 2.5GbE  ┌────────────────────────────────────┐
         dongle  ────┤           Host workstation         │
         (extra)     │                                    │
                     │  Primary ethernet stays on LAN ────┼──→ LAN/router
                     └────────────────────────────────────┘
```

### Layer A — Serial console over the motherboard UART  *(primary rescue path)*

- **What it is:** the motherboard's internal UART header (on MSI boards: `JCOM1`; on ASUS: `COM_1`; on Gigabyte: `COMA`). Still present on a surprising number of consumer mATX boards. It's a real 16550A UART hanging off the PCH, not USB.

- **The one-piece cable (recommended):** a single cable with **USB-A on one end and the 9-pin motherboard COM header (IDC 2×5 or 1×9) on the other**, with an inline FTDI + MAX3232 RS-232 level shifter. Plugs directly from the Pi's USB port onto the host's JCOM1 header — no bracket, no DB9, no null-modem cable, no second adapter. One SKU, two plugs.

  Search terms that find the right product category:
  - "USB to motherboard 9-pin COM header RS232 serial cable FTDI"
  - "USB to internal COM header cable"
  - "FTDI USB to 9-pin motherboard serial cable"

  What to look for when buying:
  - **Chipset: FTDI FT232RL** (printed on the listing). Avoid CH340/CH341 for a rescue device — the drivers are fine on Linux but FTDI is more trouble-free across initramfs and minimal rescue images.
  - **Inline RS-232 level converter** (not raw 3.3 V TTL). The JCOM1 header on consumer boards carries true RS-232 voltages (±12 V swing) — a TTL-only cable will either read garbage or damage the chip. Listings that say "RS232" or "MAX3232" are correct; listings that say only "TTL" or "UART 3.3V" are wrong for this job.
  - **Header pinout matches DTK/AT standard.** MSI, ASUS, and Gigabyte all use the same 9-pin DTK pinout for internal COM headers. Avoid cables for laptop/industrial boards with nonstandard pinouts.
  - **Length:** 50–100 cm is plenty. The Pi sits next to the workstation.

  Known-decent makers in this category: **DTECH**, **CableCreation**, **StarTech** (though StarTech's 9-pin-header-to-DB9 bracket is a separate SKU from their USB-RS232 adapter — their catalog splits them). Ordering from AliExpress/Amazon "FTDI motherboard COM header USB cable" gets the integrated version for ~$15–25.

  This is strictly one piece: **Pi USB ←→ motherboard JCOM1 header**. Nothing else to assemble.

- **Fallback if you can't find the one-piece cable:** the three-piece build still works and uses only off-the-shelf parts:
  1. 9-pin header → DB9 rear bracket cable (~$5).
  2. DB9 null-modem cable (~$5). Must be null-modem (TX/RX crossed).
  3. USB-RS232 adapter on the Pi side (~$15, FTDI FT232RL).

- **What Claude sees on the Pi:** `/dev/ttyUSB0` as a character device, driven with `tio -b 115200 /dev/ttyUSB0` or `picocom`. Claude reads/writes it via Bash. Full terminal, scriptable, can capture to a log file. See §5 below for the skills needed to actually operate it.
- **Coverage:** BIOS POST messages, GRUB menu, kernel early printk, initramfs emergency shell, systemd rescue target, `getty` login. **Does not require the host's kernel networking stack, NIC driver, USB stack, display, or userspace to be alive.** This is the layer that survives when everything else is broken.
- **Does it take the ethernet port?** No. It uses a rear slot cutout, not the NIC.
- **Does it take a PCIe slot?** No. Just a blank slot cover.
- **Prerequisite work on the host (one-time):**
  - BIOS → enable Serial Port / COM1.
  - `/etc/default/grub`: set `GRUB_TERMINAL="console serial"`, `GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200"`, append `console=tty0 console=ttyS0,115200n8` to `GRUB_CMDLINE_LINUX_DEFAULT`, then `update-grub`.
  - `systemctl enable --now serial-getty@ttyS0.service`.
  - Add a dedicated local `rescue` user with a password (so you can log in over serial when the network — and therefore any network-backed auth — is offline).

- **If the host has no COM header at all:** the fallback of equivalent directness is a **PCIe serial card** (~$20, e.g. StarTech PEX1S553). Same 16550A-class UART, just as an add-in card. It *does* consume a PCIe x1 slot but not the ethernet port. Functionally identical once the kernel enumerates it as a `ttyS*`.

### Layer B — Direct-link ethernet via USB dongle  *(fallback for misconfigured networking only)*

> **Status demotion:** Layer B is **not** an independent rescue path. It shares its single point of failure — the host's kernel networking stack, USB stack, and `sshd` — with every other network-based approach. It is included only because it cheaply rescues the narrow-but-real scenario where the host is *fully booted* but its *primary* network config is broken (bad DNS, corrupt NetworkManager profile, borked `iptables`, broken Tailscale update, fat-fingered `/etc/network/interfaces`). For anything below userspace — kernel panic, initramfs drop, offline NIC driver, hung init — Layer B is useless weight and Layer A does the work.

- **What it is:** a **second, dedicated** NIC on each side connected by a short cable, carrying a private `/30` subnet that is never routed to the LAN. The key trick is that the workstation's extra NIC is a **USB 2.5GbE dongle**, so:
  - It doesn't touch the primary onboard ethernet port.
  - It doesn't consume a PCIe slot.
  - It can be unplugged if it ever causes issues without touching the primary network config.
- **Hardware:**
  - **USB 2.5GbE dongle** on the workstation (~$15). Prefer **Realtek RTL8156B** or **RTL8156BG** — mainline kernel driver (`r8152`), works out of the box on modern distros. Avoid ASIX AX88179A for a rescue role (driver is fine but less consistently present in minimal initramfs).
  - A second USB NIC on the Pi (or use the Pi 5's onboard gigabit, with the LAN moving to a USB dongle on the Pi — depends on how the Pi itself is networked).
  - **Short Cat6 cable** (~$4), 0.3–0.5 m, just the length between the two devices. No switch in the middle.
- **Configuration:**
  - Workstation side: persistent `systemd-networkd` profile pinning the dongle to `192.168.250.2/30`, independent of NetworkManager. `sshd` bound to that interface with a dedicated `Match Address 192.168.250.1` block, key-only auth, rescue user.
  - Pi side: mirror config at `192.168.250.1/30`.
- **What Claude sees on the Pi:** normal SSH (`ssh rescue@192.168.250.2`), plus `scp`/`rsync` for pushing rescue tools and pulling logs at full wire speed.
- **Coverage:** L1 + the network-stack-broken subset of L2 where the *main* network config is broken but the kernel is still alive enough to bring up USB and run `sshd`. Lets you push a 2 GB log bundle to the Pi quickly, which serial can't.
- **Does it take the ethernet port?** No — the primary onboard NIC is untouched.
- **Honest limitation:** this layer still depends on the host's USB stack, kernel networking, and `sshd` all being alive. It is **not** a substitute for Layer A; it's a convenience layer on top of it for when you need bandwidth.

### Layer C — Power control via smart plug  *(force-reboot path)*

- **What it is:** a Wi-Fi smart plug on the workstation's power inlet, controllable from the Pi via a local HTTP API (no cloud).
- **Recommended device:** **Shelly Plug S** (~$20). Local REST API, works on a LAN without any cloud account, open firmware. Second choice: a Tasmota-flashed plug.
- **Wiring:** smart plug goes **between the UPS output and the workstation PSU**, not between the wall and the UPS (otherwise you lose the UPS's main job). The Pi itself must be on a different plug — never on the same smart plug it controls.
- **What Claude sees on the Pi:** an HTTP endpoint. A `/powercycle` slash command that does `curl -s http://shelly-desk.lan/rpc/Switch.Set?id=0&on=false && sleep 10 && curl ...&on=true` with a confirmation gate.
- **Coverage:** the "host is fully wedged, serial shows nothing, no amount of magic SysRq helps" case. Doesn't require the host to be alive at all.
- **Does it take the ethernet port?** No.

## 5. Claude skills needed to actually use Layer A

Layer A gives Claude a character device (`/dev/ttyUSB0`) — a raw, stateful, full-duplex serial stream. This is **not** the shape of input Claude's Bash tool handles well by default. Running `tio /dev/ttyUSB0` as a one-shot Bash command would hang forever; `cat /dev/ttyUSB0` dumps bytes until interrupted; keystrokes have to be sent without waiting for a "prompt" that may never come.

So yes — Layer A needs **purpose-built skills** on the Pi. Without them, Claude can technically talk to the serial port but can't reliably hold a conversation over it.

### The serial interaction problem

- Serial output is **asynchronous and unbounded**. GRUB, kernel, and initramfs emit bytes on their own schedule. Claude has to *poll* or *tail* the stream, not run a command and wait for exit.
- Serial input has to be **typed in, not piped**. Bootloaders and emergency shells expect interactive input — sometimes arrow keys, sometimes single-letter menu choices, sometimes a login prompt that appears only after the kernel is done.
- **There is no shell on the other end to detect "command complete".** Claude has to reason about output windows: "wait until I see the string `initramfs>`", "wait until there's been 2 seconds of silence", "wait until I see a `:~#` prompt".
- Boot stages change the **protocol** (GRUB menu → kernel dmesg → initramfs → LUKS prompt → systemd → getty). Each stage needs different handling.

### Skills to build (under `.claude/skills/` in the rescue workspace)

| Skill | What it does | How it's implemented |
|---|---|---|
| `serial-attach` | Opens the serial port, starts a background `tio` / `socat` session, tees all output to a rotating log file in `logs/serial-YYYYMMDD-HHMMSS.log`. Returns a handle Claude can read from. | Background `socat` bridging `/dev/ttyUSB0` ↔ a Unix socket (or TCP `127.0.0.1:7000`), plus `tee` to a log file. Claude `cat`s the log for recent output and `echo`s into the socket to send input. |
| `serial-read-since` | Returns all bytes received since a given byte offset, or the last N lines. | `tail -c +OFFSET` or `tail -n N` on the log file. |
| `serial-wait-for` | Blocks until a regex matches in the log stream or a timeout expires, returns matched line + surrounding context. | `tail -F` piped through `grep -m1 --line-buffered` with a `timeout` wrapper. |
| `serial-send` | Sends a string (with or without trailing newline) to the host via the socat socket. | `printf '...\n' \| socat - UNIX-CONNECT:/tmp/serial.sock`. |
| `serial-send-key` | Sends a special key: arrow keys, escape, tab, Ctrl-C, Ctrl-D, Break. Needed for GRUB menu navigation and SysRq. | Maps symbolic names to the right escape sequences (e.g. `UP` → `\x1b[A`). Break sequence is `tio`'s `ctrl-t b` or a raw TIOCSBRK ioctl. |
| `serial-sysrq` | Sends a Magic SysRq command via serial break. Supports the standard letters: `s` (sync), `u` (remount ro), `b` (reboot), `e` (term all), `i` (kill all), `f` (oom-kill). Gated behind a confirmation for destructive letters. | Break + key sequence. Requires `kernel.sysrq=1` on the host. |
| `grub-menu-drive` | High-level: navigate the GRUB menu by name, optionally edit a boot entry (press `e`) to add/remove kernel parameters, then boot. | Composes `serial-send-key` calls with `serial-wait-for` checkpoints. |
| `initramfs-recover` | Recognizes the `(initramfs)` prompt, runs a canned diagnostic sequence (`blkid`, `cat /proc/modules`, `dmesg \| tail`), captures output, proposes next action. | A runbook driving `serial-send` + `serial-wait-for`. |
| `luks-unlock` | Walks the user through entering a LUKS passphrase at the initramfs prompt. Prompts Daniel over the SSH session; never stores the passphrase on the Pi. | `serial-wait-for` the passphrase prompt, ask Daniel, `serial-send` the response, confirm mount. |
| `serial-capture-bundle` | Snapshots the current serial log, `dmesg`, and triage state into a timestamped folder, optionally pushes to a private gist. | Pure shell + `gh gist create`. |

### Dependencies on the Pi

- `tio` (or `picocom` as fallback) — interactive serial client, good break-signal support.
- `socat` — to bridge the serial port to a Unix socket so multiple Claude tool calls can share one session.
- `screen` or `tmux` — optional, for persistent sessions a human could also attach to.
- `expect` — not strictly needed; the skill design above avoids it, but useful as a fallback for complex prompted interactions.

### What Claude *doesn't* need

- Any model-side support beyond the Bash tool. All skills are shell scripts / runbooks that read and write files and sockets. Claude's role is decision-making — when to send what, when to wait, how to interpret output — not low-level I/O.
- A custom MCP server. Serial is plain files and sockets; MCP would add complexity without benefit here.

### In short

**Yes, Layer A needs 8–10 purpose-built skills** — none of them individually complex, but together they form the vocabulary Claude needs to drive a serial console intelligently. Without them, the hardware bridge is there but Claude can't use it reliably. Building these skills is a P0 item alongside the hardware.

## What this bridge deliberately does *not* include

- **PiKVM / video capture.** Explicitly out of scope. No HDMI, no USB HID injection. Text-only rescue.
- **Intel AMT / vPro SOL.** Would be the best possible answer if the host's chipset + firmware supported it (it's serial-over-the-main-NIC at the ME level, independent of the OS) — but B760-class consumer boards almost never expose it. Not part of the recommendation, but worth a 5-minute BIOS check on any given host.
- **USB "gadget" from the host to the Pi.** Consumer desktops run USB controllers in host-only mode. Not available.
- **IPMI/BMC.** Server feature, not on consumer boards.
- **Taking over the primary ethernet port.** Rejected by constraint.

## Shopping list (per host)

| Item | Example | Approx cost |
|---|---|---|
| Motherboard COM header → DB9 bracket cable | Generic "motherboard COM port 9-pin header to DB9 slot bracket" | $5 |
| DB9 null-modem cable, ~1 m | Generic null-modem F-F | $5 |
| USB-RS232 adapter (FTDI) | FTDI FT232RL-based, e.g. Gearmo USB-RS232 | $15 |
| USB 2.5GbE dongle | Realtek RTL8156B-based (e.g. CableCreation, UGREEN) | $15 |
| Short Cat6 cable | 0.5 m | $4 |
| Smart plug with local API | Shelly Plug S | $20 |
| **Total per host** | | **~$65** |

This is additive to the Rescue-Pi itself (~$175–200, see `hardware-spec.md`). So a Pi plus one equipped host comes to about **$240–265** all in.

## Why this is "lowest level" in a meaningful sense

Going lower than Layer A on a consumer desktop requires either:

1. **JTAG on the CPU package** — needs an unlocked Management Engine, specialist probe, no getty on the other end. Not a practical rescue surface.
2. **Direct SPI flashing of the BIOS chip** — a clip-on SOIC8 programmer. Useful for un-bricking an offline BIOS, but it is not an interactive "Claude can type commands" surface; it's a blind reflash.

Neither produces something Claude can hold a conversation with. Layer A (motherboard UART → USB-serial → Pi `/dev/ttyUSB0`) is the **lowest layer on a commodity PC that still gives you a two-way character stream Claude can drive**. Everything above it (Layers B and C) is gravy — useful gravy, but gravy.
