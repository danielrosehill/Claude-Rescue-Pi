# Claude-Rescue-Pi Skills

Skills that the rescue agent (Claude Code running on the Pi) loads when driving a recovery session against a target workstation.

Two skillsets, mapped to the two bridge layers from `docs/host-bridge-hardware.md`:

## `layer-a-serial/` — Serial console operations

The primary rescue path. Talks to the host via the motherboard UART through a USB-RS232 bridge. These skills survive a fully-offline host network stack because they don't depend on it at all.

Used when:
- The host is not reachable over any network.
- The host is stuck at GRUB, initramfs, emergency shell, or a kernel panic.
- You need to read BIOS POST messages, kernel dmesg, or boot-time logs.
- You need to drive Magic SysRq.

| Skill | Purpose |
|---|---|
| `serial-attach` | Open the port, start a persistent background session, tee to a log |
| `serial-send` | Write text or keystrokes to the host |
| `serial-wait-for` | Block until a pattern appears in serial output (or timeout) |
| `serial-sysrq` | Send Magic SysRq commands over serial break |
| `grub-menu-drive` | Navigate and edit the GRUB boot menu |
| `initramfs-recover` | Diagnose and repair from the `(initramfs)` prompt |
| `luks-unlock` | Walk through LUKS unlock at initramfs prompt |
| `serial-capture-bundle` | Snapshot session state to `logs/` and optionally push to a private gist |

## `layer-b-network-debug/` — Network-stack debugging

The fallback path for the narrow scenario where the host *is* booted and userspace is alive but the primary network config is broken — bad DNS, borked firewall, Tailscale outage, mis-edited `/etc/network/interfaces`. Claude reaches the host either over the Layer B direct-link ethernet, or over residual LAN connectivity, then methodically narrows down what's broken.

Used when:
- You can SSH to the host (via Layer B or LAN) but something about its own networking is wrong.
- You want to diagnose before rebooting (rebooting loses state).

| Skill | Purpose |
|---|---|
| `net-triage` | Top-level: classify the failure (link / IP / route / DNS / firewall / app) |
| `link-and-driver` | Check physical link, NIC driver state, `dmesg` errors |
| `addressing-and-routes` | Inspect IPs, routes, default gateway, `rp_filter` |
| `dns-debug` | Systematic DNS diagnosis: `resolv.conf`, `systemd-resolved`, upstream reachability |
| `firewall-inspect` | Read `nftables`/`iptables`/`ufw` rules and find blocks |
| `tailscale-debug` | Tailscale-specific health checks and fixes |
| `sshd-check` | Diagnose why `sshd` isn't accepting a connection on an interface |
| `net-capture-bundle` | Snapshot full network state to `logs/` for offline analysis |

## Invariants

- All skills **read first, act second**. No destructive operation runs without a summary of current state.
- Any skill that modifies host state prompts Daniel for confirmation over the SSH session.
- All skills write a transcript to `logs/YYYY-MM-DD/<session>/` in the rescue workspace.
- Skills assume the workstation has a `/CLAUDE-RESCUE.md` manifest (see `docs/design.md` §6.3) and read it first for host-specific context.
