# Building 6 VLANs on OPNsense with One NIC and No Switch (Proxmox Homelab)

A step-by-step guide to segmenting a homelab into 6 VLANs — **Guest, AI, SOC, Cloud, Deception, Targets** — using OPNsense as a virtual router inside Proxmox, with only **one physical NIC** and **no managed switch**. Includes implementation issues encountered during setup and their resolutions.

## Why this guide exists

Most OPNsense/Proxmox VLAN tutorials assume a managed switch handles VLAN tagging. This guide covers a common homelab starting point: **one NIC, no additional hardware**. The environment is built virtually inside Proxmox first, with physical device access added later when a second NIC is available.

---

## Architecture Overview

```
Home Router (does real internet NAT)
        │
        │ (single physical NIC)
        ▼
   Proxmox Host
        │
   ┌────┴────┐
   │ vmbr0   │  ← physical NIC attached, WAN side
   └────┬────┘
        │
   OPNsense VM (net0 = vmbr0 = WAN)
        │
   OPNsense VM (net1 = vmbr1 = LAN trunk, no VLAN tag = trunk)
        │
   ┌────┴────┐
   │ vmbr1   │  ← VLAN-aware, NO physical port (fully virtual)
   └────┬────┘
        │
  ┌─────┼─────┬─────┬─────┬─────┬─────┐
 ai   blue  cloud decep guests targets   ← VMs, each tagged to its VLAN
(10)  (20)  (30)  (40)   (50)   (60)
```

**Key insight:** with only one NIC, that NIC becomes OPNsense's WAN uplink to your home router. All 6 VLANs live on a second bridge (`vmbr1`) that has **no physical port at all** — it only exists inside Proxmox. This means, until you add a second NIC, only VMs can reach the VLANs; physical devices (like your laptop) cannot.

---

## Part 1: Proxmox Networking

### 1.1 Create the internal VLAN-aware bridge

`Proxmox → node → System → Network → Create → Linux Bridge`

- Name: `vmbr1`
- Bridge port: **leave empty** — this is intentional, it's a purely internal bridge
- Check **VLAN aware**
- No IP address needed
- Apply

*(Your existing `vmbr0`, tied to the physical NIC, is untouched — that stays as WAN.)*

![Proxmox Create Linux Bridge dialog](./images/06-proxmox-create-linux-bridge.png)
*Proxmox → Network. `vmbr1` already exists as a VLAN-aware bridge with no bridge ports (visible in the table behind the dialog), `vmbr0` carries the physical NIC and the gateway.*

### 1.2 Give OPNsense two virtual NICs

VM → Hardware:
- `net0` → bridge `vmbr0` (WAN)
- `net1` → bridge `vmbr1`, **VLAN Tag left blank** (this makes it a trunk carrying all 6 VLANs)

![OPNsense VM Hardware tab](./images/07-proxmox-opnsense-vm-hardware.png)
*OPNsense VM Hardware page — `net0=vmbr0` (WAN), `net1=vmbr1` (LAN trunk, no tag).*

---

## Part 2: OPNsense — Creating the 6 VLANs

### 2.1 Create VLAN interfaces

`Interfaces → Other Types → VLAN → Add` — repeat 6 times, parent interface = your LAN NIC (shows as `vtnet0` in this build):

| VLAN Tag | Name | Purpose |
|---|---|---|
| 10 | ai | AI workloads |
| 20 | blue | SOC / blue team |
| 30 | cloud | Cloud lab |
| 40 | deception | Deception / honeypots |
| 50 | guests | Guest devices |
| 60 | targets | Target systems |

![OPNsense VLAN creation](./images/08-opnsense-vlan-creation.png)
*Interfaces → Other Types → VLAN. All 6 VLANs created on the trunk parent, tags 10–60.*

### 2.2 Assign each VLAN as an interface

`Interfaces → Assignments` — pick each `VLAN00xx on vtnet0` entry from the dropdown (**not** the raw parent interface, and **not** the existing default LAN) and add it.

> **Important:** Do not assign the plain parent interface (`vtnet0` with no tag) — that is the trunk itself, not a VLAN. Only assign the tagged `VLAN00xx` sub-interfaces.

![OPNsense interface assignments](./images/09-opnsense-interface-assignments.png)
*Interfaces → Assignments. Each `optX` slot mapped to its VLAN — tag 10 = ai, 20 = blue, 30 = cloud, 40 = deception, 50 = guests, 60 = targets.*

Static IPs used:

| Interface | IP |
|---|---|
| ai | 192.168.10.1/24 |
| blue | 192.168.20.1/24 |
| cloud | 192.168.30.1/24 |
| deception | 192.168.40.1/24 |
| guests | 192.168.50.1/24 |
| targets | 192.168.60.1/24 |

![OPNsense dashboard showing all interfaces up](./images/01-dashboard-interfaces.png)
*Dashboard confirming all 7 interfaces (LAN + 6 VLANs) are assigned, enabled, and holding their gateway IPs.*

**Why /24 for every VLAN?** 254 usable hosts per subnet is far more than a homelab VLAN needs, and it keeps the math identical across all six (`.1` = gateway, `.100–.200` = DHCP pool). No reason to subnet smaller at this scale.

---

## Part 3: DHCP

### 3.1 Kea vs Dnsmasq — which to use

OPNsense ships two options. **Use Dnsmasq**, not Kea, for a setup this size:

- Dnsmasq bundles DNS + DHCP together — you get local hostname resolution for free
- Kea is built for large/enterprise-scale deployments and doesn't auto-register hostnames into DNS without extra plugins
- For 6 small VLANs with a handful of VMs each, Dnsmasq is simpler and does everything you need

> **If migrating from Kea to Dnsmasq:** disable Kea DHCP on the interface first, or you will get "address already in use" — only one DHCP server can bind per interface.

### 3.2 Configure Dnsmasq

Only two tabs actually matter:

**General** — enable the service, and select **all 6 VLANs + LAN** in the Interfaces list. (Miss one here and it won't serve DHCP there no matter what you set in ranges.)

**DHCP ranges** — one entry per VLAN:

| Interface | Start | End |
|---|---|---|
| ai | 192.168.10.100 | 192.168.10.200 |
| blue | 192.168.20.100 | 192.168.20.200 |
| cloud | 192.168.30.100 | 192.168.30.200 |
| deception | 192.168.40.100 | 192.168.40.200 |
| guests | 192.168.50.100 | 192.168.50.200 |
| targets | 192.168.60.100 | 192.168.60.200 |

Leave Domains, Hosts, DHCP options, DHCP boot, and DHCP tags at default — none of these are required for basic operation.

![Dnsmasq DHCP range dialog](./images/02-dnsmasq-dhcp-range.png)
*Services → Dnsmasq DNS & DHCP → DHCP ranges. Interface `ai`, range 192.168.10.100–200.*

---

## Part 4: NAT + Firewall Rules (critical connectivity step)

### 4.1 NAT Outbound

`Firewall → NAT → Outbound` → confirm mode is **Automatic**. If it's on Manual and you don't have a specific reason, switch it back — Manual mode requires an explicit outbound rule per subnet or new VLANs get no internet at all.

### 4.2 Firewall pass rules — required on every VLAN tab

**A brand-new OPNsense interface starts with zero rules, and the default is deny-all.** For each of the 6 interface tabs (`Firewall → Rules → [interface]`):

- Action: **Pass**
- Protocol: **any**
- Source: **`[interface] net`** (see error below)
- Destination: **any**
- Save, repeat for all 6, then Apply

> **Common error:** Setting Source to **"[interface] address"** instead of **"[interface] net"**. `address` only matches traffic from OPNsense's own interface IP, so VM traffic does not match it. This blocks VM traffic even when a rule exists. **Use `net`, not `address`, for VLAN pass rules.**

![Firewall rule - address vs net mistake](./images/03-firewall-rule-address-vs-net.png)
*The exact mistake: Source set to "ai address" instead of "ai net" — looks correct at a glance, but blocks every real VM.*

---

## Part 5: Attaching VMs to VLANs

For each VM:
- Hardware → Network Device → bridge = `vmbr1`
- **VLAN Tag** = the number for that VLAN (e.g. `30` for a SOC VM)
- Inside the guest OS: set networking to DHCP

No trunk configuration is needed per VM — Proxmox tags the traffic at the vNIC level.

![Proxmox VM network device with VLAN tag](./images/10-proxmox-vm-vlan-tag.png)
*Wazuh VM's Network Device: bridge `vmbr1`, VLAN Tag `10` — this single field is what places the VM into the `ai` VLAN, no OPNsense-side trunk config needed.*

### Verifying from inside a VM

Since there's no physical path in yet, test from the VM's console (Proxmox noVNC):

```bash
ip a                    # should show an IP in the right VLAN subnet
ping 192.168.30.1        # VLAN gateway
ping 8.8.8.8              # internet by IP
ping google.com           # DNS resolution
```

![Verified working VM on the ai VLAN](./images/11-wazuh-vm-verified-internet.png)
*End-to-end confirmation on the Wazuh VM: `ip a` shows `192.168.10.129/24` (leased from the `ai` VLAN's DHCP pool), and `ping 8.8.8.8` succeeds with 0% packet loss — proof the DHCP → firewall rule → NAT chain is fully working for this VLAN.*

---

## Errors Encountered (Troubleshooting Log)

### VM got no IP at all (APIPA / 169.254.x.x)
**Cause:** DHCP broadcast never reached OPNsense — almost always a VLAN tag mismatch.
**Fix checklist:**
1. Confirm `vmbr1` is VLAN aware with no bridge port
2. Confirm the VM's vNIC bridge = `vmbr1` (not `vmbr0`)
3. Confirm the VM's VLAN Tag number matches the tag OPNsense created for that VLAN exactly
4. Confirm a DHCP pool actually exists for that interface (service "running" ≠ pool configured)
5. Confirm the guest OS is actually set to DHCP, not a leftover static IP from a template

### VM got IP + could ping gateway, but no internet
**Cause (in this setup):** Firewall rule Source set to `[interface] address` instead of `[interface] net`. See Part 4.2 above.
**Also check:** NAT Outbound mode set to Automatic.

### SSH to OPNsense timed out
**Cause:** OPNsense disables SSH by default, and even when enabled, doesn't allow WAN-side management access out of the box.
**Fix:**
1. `System → Settings → Administration` → enable Secure Shell
2. Add an explicit `Firewall → Rules → WAN` pass rule for TCP/22, ideally restricted to a specific source IP
3. Remember this opens SSH to your entire home network, not only your workstation — remove or disable the rule when finished, or restrict Source tightly
4. Alternative with zero network exposure: use the Proxmox console directly on the VM (`Console → option 8) Shell`)

![Firewall WAN SSH rule](./images/04-firewall-wan-ssh-rule.png)
*WAN rule restricting SSH (TCP/22) to a single source IP rather than opening it to the whole home network.*

### `tailscale status` — "Failed to connect to local Tailscale daemon"
**Cause:** `tailscaled` daemon wasn't running — the CLI can't be used until the background service is started.
**Fix:**
```bash
service tailscaled status
service tailscaled start
```

### Tailscale not showing up in the admin console machines list
**Cause:** `tailscale up` was run but the printed login URL was never opened/authenticated.
**Fix:** re-run `tailscale up --advertise-routes=...`, actually open the printed URL in a browser and authorize.

### After advertising Tailscale routes, home network gateway became unreachable
**Cause:** Accidentally advertised the home LAN's own subnet (or used an exit node), causing a routing conflict with the network the laptop is already directly wired into.
**Fix:** Only advertise the 6 VLAN subnets, never your home router's own subnet. Confirm no exit node is set on either the OPNsense node or the client.

### Tailscale doesn't survive an OPNsense reboot
**Cause:** Known bug in the `os-tailscale` community plugin — `sysrc tailscaled_enable="YES"` doesn't reliably persist because OPNsense regenerates system config from its own templates at boot, silently overwriting manual rc.conf-style edits. (Confirmed as an open upstream issue, not user error.)

**Working fix:** Use the GUI instead of the terminal. In `VPN → Tailscale → Settings`, ensure **Enable** is checked and click **Save** on that page directly (not only a one-off `service tailscaled start` in the shell). OPNsense's service manager then starts it on boot, because the plugin is intended to be controlled through its settings page rather than manual `sysrc`/`rc.conf` edits.

*(A cron `@reboot` job or the `os-shellcmd` plugin are still valid fallback options if the GUI toggle does not persist on your version. For this build, the GUI Enable + Save flow was sufficient.)*

### `sysrc: unknown variable 'tailscaled_enable'`
**Cause:** Ran `sysrc tailscaled_enable` (a query) before the variable was ever set anywhere — nothing existed to read.
**Fix:** confirm the real rcvar name first: `grep -i rcvar /usr/local/etc/rc.d/tailscaled`, then set it with `sysrc tailscaled_enable="YES"`.

### `crontab -e` — ":wq is not a vi command"
**Cause:** Still in insert mode when typing `:wq` — vi interpreted it as literal text, not a command.
**Fix:** Press `Esc` first (always safe), confirm `-- INSERT --` is gone from the bottom of the screen, then type `:wq` and press Enter. To bail out without saving: `Esc` then `:q!`.

---

## Accessing VLAN Services From a Laptop (Before Adding a Second NIC)

With one NIC and no switch, your laptop sits on the home router's network — the same side as OPNsense's WAN — and has no physical path to the VLANs. Two workarounds until a second NIC is added:

**Option A — Port forward (quick, one service at a time)**
`Firewall → NAT → Port Forward` → forward a WAN port to the internal VLAN IP/port. Works immediately, but doesn't scale past one or two services.

**Option B — Tailscale subnet routes (recommended for multiple services)**
Advertise your 6 VLAN subnets from OPNsense via Tailscale, approve them in the admin console, and enable "accept subnet routes" on the laptop's Tailscale client. This gives direct access to real VLAN IPs (e.g. `192.168.30.100` for a SOC service) without managing individual port forwards.

- OPNsense: Advertise routes = YES (the 6 VLAN CIDRs), Exit node = NO
- Laptop: Accept subnet routes = YES, Use exit node = NO

![OPNsense Tailscale advertised routes](./images/05-tailscale-advertised-routes.png)
*VPN → Tailscale → Settings → Advertised Routes. All 6 VLAN subnets advertised — none of them overlap with the home network's own subnet, which is what avoids the "can't reach my gateway" conflict described below.*

**Long-term fix:** once a second NIC (or USB-to-Ethernet adapter) is added and bridged into `vmbr1`, the laptop becomes physically part of the VLAN network and neither workaround is needed.

---

## What's Next

- [ ] Add a second NIC / USB-Ethernet adapter for direct physical VLAN access
- [ ] Enable Suricata (built into OPNsense) on Deception + Targets interfaces for IDS/IPS
- [ ] Forward OPNsense logs (firewall + Suricata alerts) to Wazuh via syslog for SOC-VLAN log correlation
- [ ] Tighten inter-VLAN rules (currently any/any per VLAN to internet — add explicit VLAN-to-VLAN deny rules once base connectivity is confirmed working)

---

*Built on OPNsense 26.x, Proxmox VE 8.x. Corrections and PRs welcome.*
