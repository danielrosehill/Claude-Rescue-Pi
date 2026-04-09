# MQTT Listener + Wake-on-LAN

> Expansion of scope: the Rescue-Pi is not only an SSH-driven interactive recovery surface — it also runs a passive MQTT listener and can drive Wake-on-LAN, so that recovery actions can be triggered from anywhere an MQTT publish can reach (Home Assistant dashboard, phone, shell script, another agent) without needing an interactive SSH session.

## Why add these

Two gaps in the SSH-only design:

1. **Trigger surface is too narrow.** SSH-into-Claude is great for interactive recovery, but many useful actions are one-shot and fire-and-forget: "wake the workstation now", "power-cycle it", "run triage and publish results". Needing to open a terminal, SSH in, wait for Claude to start, and type a command is overkill for these. MQTT gives a tiny control plane over them.
2. **Clean power-on is missing.** The smart plug (Layer C) can force-cycle the workstation but that's a dirty reboot. Wake-on-LAN cleanly boots a host that's in S5/hibernate, uses the motherboard's own power-up path, and is the correct tool when the host was deliberately shut down rather than crashed.

Together, MQTT + WoL turn the Rescue-Pi into a small always-on **command-and-control appliance** for the workstation, on top of its interactive recovery role.

## Architecture

```
   ┌──────────────────────────────┐
   │  Home Assistant / phone /    │
   │  shell script / other agent  │
   └──────────────┬───────────────┘
                  │  MQTT publish
                  ▼
   ┌──────────────────────────────┐
   │   Mosquitto broker (on LAN)  │
   │   (existing infra, re-used)  │
   └──────────────┬───────────────┘
                  │  subscribe
                  ▼
   ┌──────────────────────────────┐
   │         Rescue-Pi            │
   │                              │
   │  ┌────────────────────────┐  │
   │  │  mqtt-listener daemon  │──┼──▶ dispatches to skills
   │  │   (systemd service)    │  │       │
   │  └────────────────────────┘  │       ├──▶ WoL magic packet
   │                              │       ├──▶ smart-plug HTTP call
   │                              │       ├──▶ serial-attach + capture
   │                              │       └──▶ triage + publish result
   │                              │
   │  publishes status back ──────┼──▶ MQTT topics
   └──────────────────────────────┘
```

The listener is a small long-running process (Python + `paho-mqtt`, or a shell loop using `mosquitto_sub`). It is **not** a Claude session. Claude is invoked by the listener only when a command's dispatch rule says so — e.g. "on `rescue/cmd/triage`, spawn `claude -p 'run /triage and publish the result'` and capture stdout to a topic".

## Topic layout

Prefix everything under `rescue/<hostname>/`, where `<hostname>` is the target workstation (allows one Rescue-Pi to manage multiple hosts later).

### Commands (Pi subscribes)

| Topic | Payload | Action |
|---|---|---|
| `rescue/<host>/cmd/wol` | `{}` or empty | Send WoL magic packet to the host's MAC |
| `rescue/<host>/cmd/power/on` | `{}` | Smart plug ON |
| `rescue/<host>/cmd/power/off` | `{}` | Smart plug OFF (**guarded** — see below) |
| `rescue/<host>/cmd/power/cycle` | `{}` | Off → delay → on |
| `rescue/<host>/cmd/triage` | `{}` | Run triage, publish result on `rescue/<host>/state/triage` |
| `rescue/<host>/cmd/serial/attach` | `{}` | Open the serial port, start logging |
| `rescue/<host>/cmd/serial/send` | `{"text":"..."}` | Send a string to the serial port |
| `rescue/<host>/cmd/serial/sysrq` | `{"key":"s"}` | Send a Magic SysRq letter over serial break |
| `rescue/<host>/cmd/capture` | `{}` | Snapshot session state to `logs/` |
| `rescue/<host>/cmd/claude` | `{"prompt":"..."}` | Free-form: spawn a Claude session with the given prompt, publish result |

### State (Pi publishes)

| Topic | Payload | Retained? |
|---|---|---|
| `rescue/<host>/state/reachability` | `{"lan":true,"tailscale":true,"serial":true,"power":"on"}` | yes |
| `rescue/<host>/state/triage` | result of last triage (JSON) | yes |
| `rescue/<host>/state/power` | `"on"` / `"off"` | yes |
| `rescue/<host>/state/serial/attached` | `true` / `false` | yes |
| `rescue/<host>/state/serial/last-lines` | last N lines of serial output (rolling) | no |
| `rescue/<host>/events/cmd-result` | `{"cmd":"triage","ok":true,"summary":"..."}` | no |
| `rescue/<host>/events/alerts` | alerts raised by the Pi (e.g. "workstation unreachable on all paths") | no |
| `rescue/pi/health` | Rescue-Pi's own heartbeat | retained, with `will` |

### Last-will-and-testament

The listener registers an MQTT LWT on `rescue/pi/health` with payload `offline`. If the Pi drops off the broker, HA and anything else subscribed knows within the keepalive window.

## Security and safety

MQTT is a powerful control surface. A few non-negotiables:

1. **Broker must require auth.** Username/password at minimum; TLS if the broker isn't on the trusted LAN only. Use Mosquitto's `acl_file` to restrict the `rescue/` prefix to specific users.
2. **Dedicated MQTT user for the Pi**, with read/write only on `rescue/#`. No wildcard across the whole broker.
3. **Destructive commands are guarded.** `power/off`, `power/cycle`, and any `sysrq` letter in `{b, e, i, f, o}` must carry a `"confirm": "<token>"` field in the payload, where the token is a short rolling value the Pi publishes on a separate retained topic (`rescue/<host>/state/confirm-token`) and rotates after each use. Without a matching token, the Pi logs the attempt and refuses.
4. **No free-form shell over MQTT.** The `cmd/claude` topic accepts a prompt, not a shell command. Claude mediates what actually runs, with its usual confirmation gates for destructive operations.
5. **Audit trail.** Every command received on `rescue/<host>/cmd/#` is logged to `logs/mqtt/YYYY-MM-DD.jsonl` with timestamp, topic, payload, and outcome. Rotated daily.
6. **Rate limit.** Listener refuses more than N commands per minute per topic. Defends against a stuck HA automation spamming the Pi.

## Wake-on-LAN details

### Prerequisites on the workstation

WoL needs three things all aligned:

1. **BIOS:** "PCIe Device Power On" / "Wake on LAN" / "ErP Ready" configured to allow NIC wake. On MSI boards this is under `Settings → Advanced → Wake Up Event Setup`. ErP must typically be `Disabled` or set to a mode that still provides standby power to the NIC.
2. **PSU standby rail present.** Most ATX PSUs provide 5 V standby; confirm the NIC's link LED stays on when the host is shut down. If the LED is off, WoL will not work.
3. **OS-level WoL enabled on the NIC driver.** For the onboard Realtek RTL8125 this is:
   ```
   sudo ethtool -s enp6s0 wol g
   ```
   This must be made persistent. On systemd-networkd, add `WakeOnLan=magic` to the `.network` file. On NetworkManager, `nmcli connection modify <con> 802-3-ethernet.wake-on-lan magic`. Some Realtek drivers lose this across suspend — a systemd service re-applying it on boot and on resume is the safe belt-and-braces approach.

### Sending the packet from the Pi

Two equivalent tools, pick one:

- `wakeonlan <MAC>` — simplest, uses broadcast UDP 9.
- `etherwake -i <iface> <MAC>` — lets you pick the egress interface, useful if the Pi has multiple NICs and you want to WoL over the Layer B direct link specifically.

The MAC address lives in the workstation's `/CLAUDE-RESCUE.md` manifest, so the Pi reads it from there after it last managed to connect. It is **not** hardcoded in the Rescue-Pi's config — that would break when you replace a NIC.

### Which interface to WoL over

If the host has:
- Only its primary NIC on the LAN → WoL over LAN broadcast from the Pi's own LAN interface.
- A Layer B USB dongle on a direct `/30` link to the Pi → WoL over that interface is more reliable because it skips the LAN switch entirely and there's no routing/VLAN/broadcast-domain issue.

Both should be attempted, in that order, with a small delay between.

### WoL vs smart plug — when to use which

| Situation | Use |
|---|---|
| Host is cleanly shut down (S5) | **WoL** — clean power-on, POST runs normally |
| Host is hibernated (S4) | **WoL** — NIC is still powered from standby |
| Host is suspended (S3) | **WoL** — same |
| Host is hard-hung, not responding | **Smart plug off → delay → on** (hard cycle) |
| Host is booted but wedged in userspace | **Magic SysRq** over serial first; fall back to smart plug if SysRq is off |
| Host's PSU has no standby power or BIOS WoL disabled | **Smart plug** is the only option |

Rule of thumb: **try WoL first, smart-plug second.** WoL is gentler and leaves the host in a known state.

## Skills to add

Under `skills/layer-c-control/` (new skillset, or fold into existing Layer A/B depending on preference):

| Skill | Purpose |
|---|---|
| `wol-send` | Send a WoL packet to a given MAC, pick the right interface, wait N seconds, verify the host came up (ping / ARP / Tailscale status) |
| `power-cycle` | Smart-plug hard cycle, with confirmation gate |
| `power-on` | Smart-plug on |
| `power-off` | Smart-plug off (guarded) |
| `mqtt-publish-state` | Helper that publishes a JSON state blob to the right retained topic |
| `mqtt-audit-recent` | Read the last N lines of `logs/mqtt/*.jsonl` and summarise |

These are thin wrappers around `etherwake`, `curl`, and `mosquitto_pub`.

## Software on the Pi

Add to the base image:

| Package | Purpose |
|---|---|
| `mosquitto-clients` | `mosquitto_pub`, `mosquitto_sub` for skills and the listener loop |
| `wakeonlan` **or** `etherwake` | Send WoL magic packets |
| `python3-paho-mqtt` (optional) | Cleaner listener implementation than a shell loop |
| `jq` | JSON parsing in shell |
| `curl` | Smart plug HTTP calls |

## Systemd service for the listener

Runs as the `rescue` user, restarts on failure, registers the MQTT LWT:

```ini
# /etc/systemd/system/rescue-mqtt-listener.service
[Unit]
Description=Claude-Rescue-Pi MQTT listener
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=rescue
Group=rescue
ExecStart=/home/rescue/claude-rescue-workspace/bin/mqtt-listener.sh
Restart=on-failure
RestartSec=5
StandardOutput=append:/home/rescue/claude-rescue-workspace/logs/mqtt/listener.log
StandardError=append:/home/rescue/claude-rescue-workspace/logs/mqtt/listener.log

[Install]
WantedBy=multi-user.target
```

The listener script itself is a thin dispatcher — it reads topic + payload, looks up the command in a table, and either runs a skill directly or invokes `claude -p "<prompt>"` for commands that need reasoning.

## Failure modes to be aware of

- **Broker down.** The listener's LWT fires; HA shows the Pi as offline. SSH path still works — MQTT is additive, not a replacement.
- **Pi reboots mid-command.** Systemd restarts the listener. Any in-flight action (e.g. a serial capture session) is lost; state is reconstructed from retained topics on reconnect.
- **WoL silently fails.** Most common cause: ErP setting in BIOS, or the NIC driver lost its `wol g` flag after a suspend cycle. The `wol-send` skill verifies the host came up within a timeout and raises an alert on `rescue/<host>/events/alerts` if it didn't.
- **HA automation loop.** Rate-limit on the listener defends against this.

## BOM impact

**Zero additional hardware.** MQTT reuses the existing Mosquitto broker (part of the HA infrastructure on the LAN). WoL uses the existing NICs on both sides. No new purchases.
