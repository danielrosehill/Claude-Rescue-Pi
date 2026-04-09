# Claude-KVM — Design Notes

> **Status:** WIP planning document.

## 1. Goal

When Daniel's primary Linux workstation is unreachable (won't boot, broken networking, wedged kernel, failed upgrade), he should be able to:

1. From any other device (laptop, phone with SSH client), reach a small always-on **recovery box** (Raspberry Pi).
2. Land **directly in a Claude Code CLI session** that already has the right skills, context, and credentials loaded for recovering the workstation.
3. Drive recovery **entirely over text / serial** — no GUI, no remote desktop, no VNC.

The recovery box is single-purpose. It is not a general dev machine. Its only job is to be a **Claude-powered out-of-band management surface** for the workstation.

## 2. Terminology

A few overlapping terms apply. Most accurate here:

- **Jump host / bastion** — a hardened intermediate host you SSH through. True, but bastions usually forward to a *reachable* target. Here the target may be network-offline.
- **Out-of-band management (OOBM) node** — industry term for a device that reaches a host via a path independent of its primary NIC / OS. This is the closest fit.
- **Lights-out / LOM surrogate** — consumer analog of iDRAC / iLO / IPMI, built from a Pi.
- **KVM-over-IP** traditionally implies video + keyboard capture (PiKVM). We are explicitly **not** doing video. So "Claude-KVM" is a slight misnomer; really this is **"Claude-LOM"** or **"Claude-OOBM"**. Keeping the repo name for now.

Working definition: **Claude-KVM is a single-purpose out-of-band management node running Claude Code CLI, reachable via zero-trust networking, physically wired to the workstation for recovery when the workstation's own network stack is unavailable.**

## 3. Threat / failure model

What we need to recover from, ranked by how much of the workstation is alive:

| Level | Workstation state | Normal SSH works? | Claude-KVM path |
|---|---|---|---|
| L1 | Booted, but app/service broken | Yes | SSH over LAN/Tailscale from Pi |
| L2 | Booted, networking stack broken (no Tailscale, no LAN) | No | USB ethernet gadget / direct ethernet link |
| L3 | Stuck in userspace (systemd hang, OOM, read-only root) | Maybe | Serial console (USB-TTL) to get a getty |
| L4 | Stuck in initramfs / emergency shell | No | Serial console into initramfs shell |
| L5 | Stuck in GRUB / pre-boot | No | Serial console to GRUB (requires GRUB serial config) |
| L6 | Won't POST / offline PSU / offline disk | No | Out of scope — physical intervention |

Claude-Rescue-Pi must credibly cover **L1–L5**. L6 is hardware.

### 3.1 Dependency layering — what each path actually needs from the host

The failure-level table above hides an important distinction: **most "network" paths still depend on the host's kernel networking stack being partially alive.** It is worth being explicit about what each rescue path assumes, so we don't fool ourselves into thinking we have redundancy when we don't.

| Rescue path | Needs host userspace? | Needs host kernel networking? | Needs host NIC driver loaded? | Needs host CPU+RAM alive? |
|---|---|---|---|---|
| SSH over LAN (Tailscale or direct) | ✅ yes (`sshd`, network manager) | ✅ yes | ✅ yes | ✅ yes |
| SSH over direct-link ethernet (dedicated NIC) | ✅ yes (`sshd` bound to that iface) | ✅ yes | ✅ yes (for that NIC) | ✅ yes |
| SSH over `dropbear-initramfs` | ❌ no | ✅ yes (initramfs network config) | ✅ yes | ✅ yes |
| USB ethernet gadget (if host supported it) | ✅ yes | ✅ yes | ✅ yes | ✅ yes |
| **Serial console (UART)** | ❌ no | **❌ no** | **❌ no** | ✅ yes |
| Smart-plug power cycle | ❌ no | ❌ no | ❌ no | ❌ no |
| PXE/netboot rescue image | ❌ no (replaces it) | ✅ yes (firmware-level) | ✅ yes (firmware-level) | ✅ yes |

**Implication:** the moment the host's networking stack is broken — bad NetworkManager config, failed VPN, kernel NIC driver crash, broken iptables/nftables rules, init not reaching the point where `sshd` starts — *every* network-based rescue path dies simultaneously. Adding "Tailscale plus LAN plus a dedicated direct-link NIC" does **not** give you three independent rescue paths against a network-stack failure; it gives you one path with three entry points, all of which share the same point of failure.

The only two paths that survive a totally-broken host networking stack are:

1. **Serial console** — the UART is a dumb hardware device. As long as the host CPU and a serial getty (or the bootloader, or the kernel) are writing bytes to it, you can read them on the Pi. Nothing about the host's network has to work. This is why the serial path is not just "one option among several" — it is **the one path that is architecturally independent of the failure mode we're most worried about**.
2. **Smart-plug power control** — doesn't talk to the host at all, just to mains electricity. Lets you force a reboot even when the host is fully wedged.

Everything else should be understood as a *convenience* layer on top of serial + power control: nice when they work, useless in the exact scenario the Rescue-Pi exists for.

**Consequence for the design:** serial console is **not optional**. If we cannot establish a working serial path to a given workstation, we should not claim that workstation is covered by Claude-Rescue-Pi. A host with no serial path is at best covered for L1 scenarios (app/service breakage), which is the scenario that needs the Rescue-Pi least.

## 4. Physical connections to the workstation

Since we cannot rely on the workstation's own NIC or its display, the Pi needs **independent physical paths** into it. At least two, for redundancy.

### 4.1 Serial console (USB → TTL → motherboard header, or USB → USB)

- **What:** A USB-to-serial adapter from the Pi, into either:
  - the workstation motherboard's COM/serial header (if present), or
  - a USB port on the workstation exposing a USB CDC-ACM gadget, or
  - a dedicated serial card.
- **Why:** This is the lowest-level, most resilient path. Works at GRUB, initramfs, and single-user. Does not depend on the workstation's network stack at all.
- **Requirements on workstation:**
  - Enable serial console in GRUB (`GRUB_TERMINAL=serial`, `GRUB_SERIAL_COMMAND=...`, kernel cmdline `console=tty0 console=ttyS0,115200n8`).
  - Enable a getty on `ttyS0` (`systemctl enable serial-getty@ttyS0`).
- **Tooling on Pi:** `tio` or `minicom` or `picocom`. Claude drives them via its Bash tool.
- **Caveat:** Most modern consumer motherboards no longer expose a COM header. Likely need a **PCIe serial card** or a **USB-to-USB "null modem"** using a second Pi-side adapter plugged into a workstation USB port configured as a serial gadget. This is the single biggest hardware design question.

### 4.2 Direct ethernet crossover (L2 rescue)

- **What:** A second NIC (USB ethernet dongle) on the Pi, connected by a short cable directly to a second NIC on the workstation. Static IPs on both sides, e.g. `192.168.250.1/30` ↔ `192.168.250.2/30`. Never touches the LAN.
- **Why:** If the workstation has booted far enough for the kernel to bring up interfaces but LAN/Tailscale is broken (DNS, routing, firewall, VPN), a dedicated static link still works. Gives full SSH, `scp`, `rsync`.
- **Requirements on workstation:**
  - A second NIC (onboard or USB).
  - A `systemd-networkd` / NetworkManager profile that **always** brings up that interface with the static IP, independent of the main network profile.
  - `sshd` listening on that interface.
- **Good for:** L2 scenarios. Also doubles as a fast file transfer channel to push rescue tools onto the workstation.

### 4.3 USB gadget mode (optional, level L2.5)

- **What:** Pi connected via USB to the workstation, exposing itself as a USB ethernet device (`g_ether`) **or** the workstation exposes itself as a USB serial gadget to the Pi.
- **Why:** Another independent path that doesn't need a spare NIC on either side.
- **Caveat:** Pi acting as USB-ethernet gadget requires Pi USB-OTG hardware (Pi Zero / Pi 4 USB-C). Useful as a backup.

### 4.4 Wake-on-LAN + PXE / netboot rescue image (optional)

- **What:** Pi sends WoL to the workstation's NIC, workstation is configured to PXE-boot from the Pi, which serves a minimal rescue image (tftp + http).
- **Why:** Covers L5 / L6-ish "root filesystem is destroyed" recovery without physical media.
- **Cost:** Significantly more setup (dnsmasq, tftp, signed rescue image). Mark as **phase 2**.

### 4.5 Power control (optional, but valuable)

- **What:** A smart plug (Tasmota / Zigbee / Shelly) or a relay HAT on the Pi wired into the workstation's power. Claude can hard-cycle the workstation.
- **Why:** L3/L4 hangs often need a power cycle. Without this, Daniel has to physically touch the machine, defeating the "remote recovery" premise.
- **Recommendation:** Smart plug is cheapest and works today.

### 4.6 Summary table

| Path | Covers | Hardware cost | Complexity | Priority |
|---|---|---|---|---|
| Serial console | L3, L4, L5 | USB-TTL adapter + possibly PCIe serial card | Medium | **P0** |
| Direct ethernet link | L2 | Spare USB NIC | Low | **P0** |
| Smart plug power cycle | L3, L4 | ~$15 | Low | **P0** |
| USB gadget serial/ethernet | L2 backup | None (cable only) | Medium | P1 |
| PXE rescue netboot | L5 root-fs destroyed | None extra | High | P2 |

## 5. Network reachability *to* the Pi

The Pi itself must be reachable from anywhere, reliably, without depending on the home LAN being healthy.

### 5.1 Primary: Tailscale

- Pi joins Daniel's tailnet as a dedicated node (`claude-kvm`).
- Node key stored on disk, MagicDNS name `claude-kvm`.
- ACL restricted: only Daniel's user/devices can reach it.
- `ssh` over Tailscale from laptop or phone → lands on the Pi.
- Failure mode: Tailscale control plane down, or home internet down entirely. For "home internet down" we are also out of recovery scope.

### 5.2 Secondary: Cloudflare Access + cloudflared tunnel (SSH)

- `cloudflared` running on the Pi exposing SSH as a Cloudflare Access–protected hostname (e.g. `kvm.dsrholdings.cloud`).
- Auth via Cloudflare Access (email OTP / SSO / hardware key).
- Client side: `cloudflared access ssh --hostname kvm.dsrholdings.cloud`.
- **Why both:** independent control planes. If Tailscale is broken on the client device or account, Cloudflare still works, and vice versa.

### 5.3 Tertiary (optional): cellular / LTE hat

- USB LTE dongle or SIM hat on the Pi for when home internet is down.
- Marked P2 — expensive, not currently justified.

### 5.4 Local fallback: LAN SSH

- Pi also listens on SSH on the LAN with a known static IP. If Daniel is physically home, a laptop on the same LAN can reach it even if both Tailscale and Cloudflare are broken.

## 6. "SSH in → straight into Claude" UX

The target flow:

```
laptop$ ssh claude-kvm
# (or: cloudflared access ssh --hostname kvm.dsrholdings.cloud)
→ lands in a Claude Code session, already cd'd into the recovery workspace,
  with recovery skills loaded, MCPs connected, and the workstation
  connection pre-tested.
```

### 6.1 Mechanism

- Dedicated Unix user on the Pi, e.g. `rescue`.
- Its login shell is **not** bash. Instead, its shell is a small wrapper script, e.g. `/usr/local/bin/claude-rescue-shell`, that:
  1. `cd ~/claude-rescue-workspace`
  2. Runs a pre-flight check (see §7).
  3. `exec claude` (Claude Code CLI) with a resume flag or a fresh session seeded by a startup prompt.
- Alternative: keep a normal shell but put the `exec claude` line in `~/.bash_profile` for the `rescue` user, so any SSH login lands in Claude. Simpler, easier to escape from (Ctrl-D back to bash for manual work). **Preferred.**
- `authorized_keys` entry for the `rescue` user contains only Daniel's keys, with `no-port-forwarding,no-agent-forwarding,no-X11-forwarding` hardening. (Do **not** add `command=` — we want Claude to launch via login shell, not a forced command, so Daniel can still drop to a raw shell if Claude itself is broken.)
- A **second** user, `admin`, with normal bash, for when Claude-KVM itself needs repair. Different SSH key ideally.

### 6.2 Workspace layout on the Pi

```
/home/rescue/claude-rescue-workspace/
├── CLAUDE.md                 # system prompt: you are a recovery operator
├── .claude/
│   ├── skills/               # recovery skills (see §8)
│   ├── commands/             # slash commands (/triage, /serial, /powercycle, ...)
│   └── settings.json         # MCPs, hooks
├── context/
│   ├── workstation.md        # hardware, disks, partitions, LUKS, boot setup
│   ├── connections.md        # how to reach the workstation on each path
│   ├── credentials.md        # pointers to 1Password / age-encrypted secrets
│   └── runbooks/             # markdown recovery runbooks
├── logs/                     # session transcripts, serial captures
└── scratch/                  # working files during a recovery session
```

`CLAUDE.md` primes Claude with: "You are operating from a recovery box. The target workstation may be in any of failure levels L1–L5. Your first action in every session is to run `/triage`."

## 6.3 On-host instruction manifest (workstation-side)

The Rescue-Pi agent should not have to guess the layout of the target workstation. Every workstation that Claude-Rescue-Pi is expected to recover should carry, on its own root filesystem, a **plain-text instruction manifest** written *for the recovery agent*.

### Conventions

- **Canonical path:** `/CLAUDE-RESCUE.md` (root of the filesystem, so it's reachable from an initramfs shell, a live USB chroot, or a serial getty without hunting through `/home`).
- **Fallback paths** the agent should also check, in order:
  1. `/CLAUDE-RESCUE.md`
  2. `/etc/claude-rescue.md`
  3. `/root/CLAUDE-RESCUE.md`
  4. `/boot/CLAUDE-RESCUE.md` (survives even if root fs is unmountable)
- **Encoding:** UTF-8, plain Markdown, no front-matter required.
- **Audience:** written in the second person, addressed to "you, the recovery agent", not to a human reader. Assume the reader is Claude Code running on the Rescue-Pi, reached the file via serial or SSH, and needs to know what it's looking at.
- **Tone:** terse, factual, imperative. No marketing, no history.

### What the manifest must contain

1. **Host identity** — hostname, OS + version, owner, contact.
2. **Disk layout** — physical disks, partitions, LUKS containers (names, not passphrases), filesystems, mount points, subvolumes.
3. **Boot chain** — bootloader, root UUID, rootflags, kernel cmdline quirks, any known-finicky drivers (e.g. "amdgpu black screen if X").
4. **Network interfaces** — names, which are wired/wireless, which have static configs, which carry Tailscale.
5. **Services that matter during recovery** — sshd config file paths, whether dropbear-initramfs is present, where the smart-plug lives on the network.
6. **Known failure modes** — "if you see X, the cause is usually Y; try Z first". Each workstation accumulates these over time.
7. **Snapshot / rollback mechanism** — snapper? timeshift? btrfs subvolume snapshots? Where and how to roll back.
8. **Credential pointers** — *where* secrets live (1Password vault name, pass store path), never the secrets themselves. The agent prompts Daniel over the SSH session when a secret is actually needed.
9. **Escalation rules** — what the agent is allowed to do autonomously vs. what requires confirmation. E.g. "you may run fsck on unmounted filesystems; you may not `dd` to any block device without explicit confirmation".
10. **Last-updated date** — so the agent knows how stale the manifest is.

### Workflow

- The agent's `/triage` command, after establishing a connection to the workstation, **immediately reads this file** and loads it into context before suggesting any action.
- If the file is missing, the agent says so loudly and refuses to run destructive skills until Daniel confirms manually.
- The file is part of the workstation's own configuration management, not the Rescue-Pi repo — each host carries its own manifest. The Rescue-Pi repo should ship a **template** (`templates/CLAUDE-RESCUE.md`) that new hosts can copy and fill in.
- The manifest is **not secret** in itself (no passphrases, no keys), but it is a detailed hardware/software fingerprint — treat it as private. Don't commit a real one to a public repo.

## 7. Pre-flight / triage

On each SSH login (or as the first Claude action), the Pi should know the current reachability state. A `/triage` slash command runs:

1. `ping` workstation LAN IP.
2. `ping` workstation direct-link IP (§4.2).
3. `tailscale status | grep workstation` — is it on the tailnet?
4. `ssh -o BatchMode=yes -o ConnectTimeout=3 workstation true` on each path.
5. Check serial device `/dev/ttyUSB0` (or whichever) — is the adapter present? Can we open it?
6. Query smart plug for power state.
7. Print a summary: "Workstation reachable via: [LAN ✗, direct ✓, tailscale ✗, serial ✓, power=ON]".

Claude then decides the recovery level and loads the relevant runbook.

## 8. Skills to preload

Under `.claude/skills/`:

| Skill | Purpose |
|---|---|
| `triage-workstation` | Run §7 checks, classify failure level, propose next step. |
| `serial-console` | Open `/dev/ttyUSB0` via `tio`, interact with GRUB / initramfs / getty, capture transcript to `logs/`. |
| `direct-link-ssh` | SSH over the §4.2 static link, run diagnostics. |
| `power-cycle` | Hard power cycle via smart plug, with confirmation gate. |
| `journal-diagnose` | Once on the workstation, pull `journalctl -b -1`, `dmesg`, `systemctl --failed`, summarize. |
| `network-stack-repair` | Runbook for broken NetworkManager / systemd-networkd / DNS. |
| `grub-repair` | Runbook for GRUB breakage, including chrooting from a rescue shell. |
| `initramfs-repair` | Runbook for emergency-shell / broken fstab / failed root mount. |
| `filesystem-fsck` | Run `fsck` safely on unmounted filesystems from rescue. |
| `luks-unlock` | Walk through unlocking an encrypted root from initramfs over serial. |
| `package-rollback` | Snapper / apt history rollback of a bad upgrade. |
| `capture-state` | Snapshot current findings into `logs/YYYY-MM-DD/` and push to a private gist for persistence. |

Most of these are thin wrappers around command sequences + decision trees. They're not magic; the value is that in an emergency, Daniel doesn't have to remember any of it.

## 9. Security

- **Attack surface minimized:** Pi runs only `sshd`, `tailscaled`, `cloudflared`. No web UI, no exposed services.
- **SSH:** key-only, no passwords, Fail2ban, separate `rescue` and `admin` users.
- **Cloudflare Access:** require hardware key or SSO for the tunnel hostname.
- **Claude CLI auth:** API key stored in `~/.config/claude/`, file mode 600, owned by `rescue`. Consider a dedicated Anthropic API key with spend limit for this device.
- **Secrets for the workstation:** do **not** store workstation LUKS passphrases on the Pi in plaintext. Use a `pass` / `age` store unlocked interactively, or prompt Daniel over the SSH session when needed.
- **Tamper:** Pi is physically in Daniel's home; full-disk encryption on the Pi SD card is optional (adds friction on reboot since there's no one to type the passphrase — skip).
- **Audit:** all Claude sessions logged to `logs/` and optionally mirrored to a private git repo.

## 10. Open questions / decisions to make

1. **Serial path:** PCIe serial card in the workstation, or USB-gadget trick, or just accept "no serial" and rely on direct ethernet + smart plug? Needs a look at the actual workstation motherboard.
2. **Which Pi:** Pi 4 (USB3, gigabit) vs Pi 5 vs Pi Zero 2 W. Recommend **Pi 4 (2 or 4 GB)** for stability, thermal headroom, and spare USB ports for serial + ethernet dongle.
3. **Storage:** SD card vs USB SSD. Recommend **USB SSD** — SD cards die and this device must be reliable when it matters most.
4. **Claude Code CLI on ARM:** verify current install path on Raspberry Pi OS (arm64). Likely via npm.
5. **Does Daniel want MCPs on the Pi?** (Context7, HF, time, etc.) Probably yes for Context7 (fetch docs during recovery) and a filesystem MCP. Keep the MCP set minimal — fewer moving parts.
6. **Autostart a Claude session 24/7?** No. Spawn on SSH login only. An idle running Claude on a recovery box is wasted tokens and a surprise.
7. **Repo hygiene:** the `claude-rescue-workspace` directory should itself be a git repo, private, pushed to GitHub, so it can be re-cloned if the Pi dies.

## 11. Phased build plan

**Phase 0 — hardware shopping list**
- Pi 4 (4 GB) + USB SSD + case + PSU
- USB-to-serial adapter (FTDI or CP2102)
- USB gigabit ethernet dongle
- Short ethernet cable
- Smart plug (Shelly or Tasmota)

**Phase 1 — base device**
- Flash Raspberry Pi OS Lite 64-bit to USB SSD.
- SSH keys, Fail2ban, unattended-upgrades.
- Tailscale up, ACL'd.
- `cloudflared` tunnel + Cloudflare Access policy.
- `admin` and `rescue` users.

**Phase 2 — Claude on the Pi**
- Install Claude Code CLI.
- Create `claude-rescue-workspace` repo (private) and clone to `/home/rescue/`.
- Seed `CLAUDE.md`, minimal skills, `/triage` command.
- `rescue` user's `.bash_profile` → `exec claude`.

**Phase 3 — workstation links**
- Configure direct ethernet link on the workstation (persistent profile).
- Decide and install serial path.
- Wire up smart plug, integrate with `power-cycle` skill.
- Configure workstation GRUB + getty for serial.

**Phase 4 — drills**
- Deliberately break the workstation in controlled ways:
  - unplug main NIC → recover via direct link.
  - break NetworkManager config → recover via serial.
  - bad GRUB entry → recover via serial at GRUB prompt.
  - bad fstab → recover from emergency shell over serial.
- Each drill updates a runbook under `context/runbooks/` and ideally becomes a skill.

**Phase 5 — polish**
- PXE rescue netboot (optional).
- Session log mirroring to private GitHub.
- Periodic health check cron that pings the workstation and alerts if reachability drops below 2 independent paths.
