# Claude-KVM

> **Status:** WIP / exploratory

A Raspberry Pi acting as a networked **jump host** (bastion) whose sole purpose is running the Claude Code CLI, reachable over **Cloudflare Access + Tailscale**, and wired to a primary workstation to provide an **emergency recovery surface** when that workstation can't boot (e.g. a broken Linux box).

## Idea

- Tiny always-on Pi, hardened and minimal.
- Exposed only via Tailscale + Cloudflare Access (zero-trust, no open ports).
- Claude Code CLI pre-installed and authenticated.
- Out-of-band link to the workstation (SSH over LAN, serial/USB console, or IP-KVM passthrough) so that if the workstation won't boot, Claude on the Pi can still:
  - inspect logs / boot state
  - drive recovery commands
  - walk through repair steps interactively

Think of it as a "Claude-powered lights-out card" for a Linux workstation.

## Lineage

Adaptation of [danielrosehill/Claude-Rescue](https://github.com/danielrosehill/Claude-Rescue).

## Open questions

- Correct terminology — jump host? bastion? out-of-band management node?
- Best recovery-surface mechanism: SSH, serial console, PiKVM-style HDMI/USB capture, or Wake-on-LAN + netboot rescue image.
- How Claude authenticates safely on a remote always-on device.
