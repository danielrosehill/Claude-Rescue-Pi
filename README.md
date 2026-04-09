# Claude-Rescue-Pi

> **Status:** WIP / exploratory

A Raspberry Pi acting as an out-of-band management (OOBM) node whose sole purpose is running the Claude Code CLI, reachable over **Tailscale + Cloudflare Access**, and physically wired to a primary workstation to provide an **emergency recovery surface** when that workstation can't boot (e.g. a broken Linux box).

Also runs an always-on **MQTT listener** for fire-and-forget control (Wake-on-LAN, power cycle, triage, serial commands) from Home Assistant, a phone, or any script that can publish to the broker.

## Idea

- Tiny always-on Pi, hardened and minimal.
- Exposed only via Tailscale + Cloudflare Access (zero-trust, no open ports).
- Claude Code CLI pre-installed and authenticated; SSH lands directly in a Claude session.
- **Interactive recovery surface:** SSH in → Claude runs pre-loaded skills against the host over serial + direct-link ethernet + smart plug.
- **Passive control surface:** MQTT listener dispatches WoL, power, and triage commands without needing an interactive session.

Think of it as a text-only, Claude-powered lights-out appliance for a Linux workstation.

## Lineage

Adaptation of [danielrosehill/Claude-Rescue](https://github.com/danielrosehill/Claude-Rescue).

## Documentation

| Doc | Contents |
|---|---|
| [`docs/design.md`](docs/design.md) | Architecture, failure model, dependency layering, SSH-into-Claude UX, on-host manifest |
| [`docs/host-bridge-hardware.md`](docs/host-bridge-hardware.md) | The three-layer wired bridge (serial, direct-link ethernet, smart plug), one-piece cable recommendation, Claude skills needed for serial |
| [`docs/hardware-spec.md`](docs/hardware-spec.md) | SBC board comparison, recommended spec, OS recommendation |
| [`docs/mqtt-and-wol.md`](docs/mqtt-and-wol.md) | MQTT listener architecture, topic layout, Wake-on-LAN setup and policy |
| [`docs/bom.md`](docs/bom.md) | Full project bill of materials |
| [`skills/`](skills/) | Claude skills loaded on the Pi (Layer A serial, Layer B network debug, Layer C control) |

## Open questions

- Does the target host's motherboard actually have a COM header? (Many do; confirm per host.)
- Exact one-piece FTDI-to-motherboard-COM-header cable SKU worth endorsing.
- WoL reliability across suspend/resume cycles on the target NIC.
- Where to run the MQTT broker if HA's Mosquitto is not available on a given network.
