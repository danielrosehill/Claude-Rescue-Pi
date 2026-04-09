# Workstation Connection Points — Daniel's Primary Desktop

> Inventory of physical/logical paths a Claude-Rescue-Pi could use to reach this specific workstation. Scenario: **workstation is powered on, networking may or may not be up, but the OS is not booting to a usable environment.**

## Hardware summary

- **Motherboard:** MSI PRO B760M-A WIFI (MS-7D99), BIOS version 2.0
- **Chipset:** Intel B760 (Raptor Lake-S PCH)
- **GPU:** AMD Radeon RX 7700 XT / 7800 XT (Navi 32)
- **Storage:** Micron 2550 NVMe SSD (root on btrfs, subvol `@`)
- **NIC (wired):** Realtek RTL8125 2.5 GbE — `enp6s0`, MAC `d8:43:ae:1d:98:40`
- **NIC (wireless):** Intel CNVi WiFi (Raptor Lake-S PCH) — `wlo1`, currently down
- **USB:** Intel USB 3.2 Gen 2x2 XHCI controller, many ports populated (keyboard, mouse, Stream Deck, card readers, UPS, audio, hubs)
- **UPS:** PowerCom offline UPS attached via USB (`06da:ffff PPC Offline UPS`) — good, means the workstation is on battery-backed power
- **Kernel cmdline:** `quiet splash`, amdgpu tuning flags, `intel_pstate=passive`
- **GRUB:** `GRUB_TERMINAL` **not** enabled for serial (commented out) — serial console at boot would need to be turned on
- **sshd:** enabled ✓
- **serial-getty@ttyS0:** disabled ✗

## 🔑 Most important finding: the motherboard has a real COM header

DMI/SMBIOS reports:

```
External Reference Designator: COM A
External Connector Type: DB-9 male
Port Type: Serial Port 16550A Compatible
```

And the kernel actually instantiated it:

```
00:01: ttyS0 at I/O 0x3f8 (irq = 4, base_baud = 115200) is a 16550A
```

That means the B760M-A WIFI has an internal **COM1 pin header** on the board (MSI typically labels it `JCOM1`). This is the single best news for this project — it's a proper 16550A UART, not a USB-serial emulation, usable all the way from GRUB through initramfs through userspace. Most modern consumer boards don't have this. Yours does.

**Action item:** confirm by looking at the board (or the manual) for a 9- or 10-pin header labelled `JCOM1`. You'd wire it to the Rescue-Pi via:

```
MSI JCOM1 header  ──(9-pin→DB9 bracket cable, ~$5)──  DB9 null-modem  ──  USB-to-serial adapter (FTDI)  ──  Rescue Pi USB
```

Or skip the DB9 entirely with a direct dupont-to-USB-TTL adapter, being careful about RS-232 vs TTL **voltage levels** (the JCOM1 header on most MSI boards is true RS-232 on a 2.54 mm header, so you need a proper RS-232 adapter, not a raw 3.3 V TTL FTDI board). A cheap `MAX3232`-based USB-RS232 adapter handles this.

## Connection points, ranked by usefulness for the "powered on but not booting" scenario

### 1. Serial console via `JCOM1` header — **best**

- **Covers:** GRUB menu, kernel boot messages, initramfs emergency shell, single-user, systemd rescue/emergency targets, broken networking, broken display driver, kernel panics.
- **Why it's perfect here:** amdgpu is a frequent cause of "powered on, not booting to a usable environment" (black screen after DRM init). A serial console bypasses the GPU entirely.
- **Workstation-side setup required (do this *now*, before you need it):**
  1. Enable COM1 in BIOS (Settings → Advanced → Super IO → Serial Port → Enabled).
  2. In `/etc/default/grub`:
     ```
     GRUB_TERMINAL="console serial"
     GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200"
     GRUB_CMDLINE_LINUX_DEFAULT="quiet splash ... console=tty0 console=ttyS0,115200n8"
     ```
     Then `sudo update-grub`.
  3. `sudo systemctl enable --now serial-getty@ttyS0.service`
  4. Ensure your user (or a dedicated `rescue` local user) has a password set, so you can actually log in over serial when the network is dead.
- **Pi-side:** `tio /dev/ttyUSB0 -b 115200`.

### 2. Wired Ethernet (`enp6s0`, Realtek 2.5GbE) — **primary network path**

- **Covers:** L1 (OS up, services broken) and any scenario where the kernel booted far enough to bring up networking.
- Current IP lives on this interface; Tailscale (`tailscale0`) rides on top.
- **Rescue-Pi side:** reach workstation on LAN by IP, or via Tailscale MagicDNS.
- **Caveat:** useless for the scenario you described. If the box "isn't booting to a usable environment", networking is probably not up, or is up but sshd isn't running yet.
- **Mitigation:** enable a **dropbear-initramfs** SSH server so that *even during initramfs*, the box has a tiny SSH listener on `enp6s0`. Combined with serial, this gives you two independent ways in during early boot.

### 3. Dedicated direct-link ethernet — **not currently possible**

- Workstation has **only one wired NIC** (the onboard Realtek). There is no spare ethernet port for a dedicated Pi↔workstation static link.
- **Options to add one:**
  - **USB 2.5GbE dongle** permanently attached to the workstation (e.g. Realtek RTL8156B-based), configured with a static IP via systemd-networkd. ~$15. Easy. **Recommended.**
  - **PCIe NIC** in a free slot — cleaner, requires opening the case and a free slot. DMI shows multiple PCIe slots "in use" but that likely includes M.2 and the GPU; worth checking physically if there's a free x1 slot.
- Either way, this becomes the L2 fallback when the main NIC's software config is broken but the kernel still boots.

### 4. WiFi (`wlo1`, Intel CNVi) — **not useful as a rescue path**

- Currently DOWN and probably unconfigured. Consumer WiFi on Linux is flaky for early-boot rescue. Skip for this purpose.

### 5. USB "gadget" from workstation to Pi — **not available**

- The Intel Raptor Lake XHCI on this board is **host-only**, not OTG. The workstation cannot present itself as a USB ethernet or USB serial gadget to the Pi. Rule this out.

### 6. IPMI / BMC / vPro / AMT — **probably not**

- B760M-A WIFI is a consumer board — no BMC/IPMI. Intel **AMT** *may* be present on Raptor Lake CPUs with vPro, but B760 chipset consumer boards typically don't expose MEBx. Worth a 30-second check in BIOS ("Intel ME / AMT / MEBx"), but don't count on it.

### 7. Power cycle via smart plug — **adjacent, critical**

- You noted the workstation is "powered on" in this scenario, but for the general case: the UPS is between wall and PSU. A Shelly Plug S on the **UPS input** (or on the workstation's own power tail before the UPS) would let the Rescue-Pi force a hard cycle. Without this, serial is great for *diagnosis* but you still need to physically reach the case to recover from a full hang.
- **Recommendation:** Shelly on the workstation's PSU input side (not the UPS input — you want the UPS to still protect against mains glitches).

### 8. BIOS/UEFI remote update & recovery — **not applicable**

- MSI Flash BIOS Button exists on higher-end boards but not on B760M-A WIFI (check the rear I/O — likely not present). Irrelevant to runtime rescue anyway.

## Logical paths, in order Claude should try during the "not booting to a usable environment" scenario

Given the current hardware state (serial **not yet configured**, only one wired NIC, no smart plug):

1. **Tailscale / LAN SSH** to `enp6s0` — covers "boots to multi-user, sshd is up, just some apps broken".
2. **Initramfs SSH (dropbear)** on `enp6s0` — if you install it. Covers "stuck at LUKS prompt / can't mount root".
3. **Serial console** via JCOM1 → Pi USB-RS232 — the universal path. Covers everything from BIOS POST messages onward once serial is enabled on both ends.
4. **Physical intervention** — currently the only fallback for a full hang, because there's no smart-plug power control yet.

## Concrete pre-work to do on the workstation *now*, before Rescue-Pi exists

So that Rescue-Pi has something to talk to when it's built:

1. **BIOS:** enable Serial Port (COM1).
2. **GRUB:** enable serial terminal + add `console=tty0 console=ttyS0,115200n8` to cmdline.
3. **systemd:** `sudo systemctl enable --now serial-getty@ttyS0.service`.
4. **dropbear-initramfs:** `sudo apt install dropbear-initramfs`, drop your Rescue-Pi pubkey into `/etc/dropbear/initramfs/authorized_keys`, set a static IP hint in `/etc/initramfs-tools/initramfs.conf` (`IP=...`), `sudo update-initramfs -u`.
5. **Optional now, strongly recommended later:** add a second NIC (USB2.5G dongle ~$15), give it a static `192.168.250.2/30`, persistent systemd-networkd profile, and a dedicated `sshd` match block listening on that interface.
6. **Document LUKS / btrfs layout** in `context/workstation.md` in the Rescue-Pi repo — disks, subvolumes, root UUID `0f8f5fa9-e838-4843-a681-5b0087d4fbd0`, rootflags `subvol=@` — so Claude on the Pi has the facts to reason with during a recovery.

## What's missing for full coverage

| Gap | Fix | Priority |
|---|---|---|
| No smart plug for power control | Shelly Plug S | P0 |
| No serial console enabled | BIOS + GRUB + getty config above | P0 |
| No initramfs SSH | `dropbear-initramfs` | P0 |
| Only one wired NIC | USB 2.5GbE dongle as dedicated static link | P1 |
| No documented disk/LUKS layout | `context/workstation.md` | P1 |
| No physical JCOM1 cable | 9-pin→DB9 bracket + USB-RS232 adapter | P1 |
