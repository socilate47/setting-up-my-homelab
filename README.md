# Building 6 VLANs on OPNsense with One NIC and No Switch (Proxmox Homelab)

This guide explains how to segment a homelab into 6 VLANs (**Guest, AI, SOC, Cloud, Deception, Targets**) using OPNsense as a virtual router inside Proxmox, with only **one physical NIC** and **no managed switch**. It includes practical configuration details and troubleshooting steps validated during implementation.

## Why this guide exists

Most OPNsense and Proxmox VLAN guides assume a managed switch is already handling VLAN tagging. This walkthrough documents a common starting point for homelabs: **one NIC and no additional network hardware**. The initial design is fully virtual inside Proxmox; physical VLAN access can be added later with a second NIC.

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

With a single NIC, that NIC is used for OPNsense WAN uplink to the home router. All six VLANs are hosted on `vmbr1`, which has **no physical bridge port** and exists only inside Proxmox. Until a second NIC is added, VLAN access is limited to VMs hosted on Proxmox.

---

## Part 1: Proxmox Networking

### 1.1 Create the internal VLAN-aware bridge

`Proxmox → node → System → Network → Create → Linux Bridge`

- Name: `vmbr1`
- Bridge port: **leave empty** (this is an internal-only bridge)
- Check **VLAN aware**
- No IP address needed
- Apply

The screenshot below shows this bridge configuration in context, including `vmbr0` remaining mapped to the physical NIC for WAN traffic.

![Proxmox Create Linux Bridge dialog](./images/06-proxmox-create-linux-bridge.png)
*`vmbr1` is configured as a VLAN-aware internal bridge with no bridge ports; `vmbr0` remains the physical uplink bridge.*

### 1.2 Give OPNsense two virtual NICs

In the OPNsense VM hardware settings:
- `net0` → bridge `vmbr0` (WAN)
- `net1` → bridge `vmbr1`, **VLAN Tag left blank** (trunk carrying all VLAN tags)

The hardware view confirms the WAN/LAN trunk split required for this topology.

![OPNsense VM Hardware tab](./images/07-proxmox-opnsense-vm-hardware.png)
*`net0=vmbr0` provides WAN connectivity, while `net1=vmbr1` carries the internal VLAN trunk without a tag on the parent interface.*

---

## Part 2: OPNsense — Creating the 6 VLANs

### 2.1 Create VLAN interfaces

`Interfaces → Other Types → VLAN → Add` and repeat six times. Set the parent interface to your LAN trunk NIC (shown as `vtnet0` in this environment):

| VLAN Tag | Name | Purpose |
|---|---|---|
| 10 | ai | AI workloads |
| 20 | blue | SOC / blue team |
| 30 | cloud | Cloud lab |
| 40 | deception | Deception / honeypots |
| 50 | guests | Guest devices |
| 60 | targets | Target systems |

This screenshot shows all six VLAN definitions attached to the same parent trunk interface.

![OPNsense VLAN creation](./images/08-opnsense-vlan-creation.png)
*Each VLAN tag (10–60) is created under the trunk parent interface, which enables per-VLAN interface assignment in the next step.*

### 2.2 Assign each VLAN as an interface

`Interfaces → Assignments` — add each `VLAN00xx on vtnet0` entry from the dropdown. Do **not** assign the untagged parent interface and do **not** reuse the original default LAN as a substitute for tagged VLAN interfaces.

The assignment table below confirms each `optX` interface is mapped to the correct VLAN sub-interface.

![OPNsense interface assignments](./images/09-opnsense-interface-assignments.png)
*Each interface slot is explicitly bound to a tagged VLAN sub-interface (10, 20, 30, 40, 50, 60).*

Static IPs used:

| Interface | IP |
|---|---|
| ai | 192.168.10.1/24 |
| blue | 192.168.20.1/24 |
| cloud | 192.168.30.1/24 |
| deception | 192.168.40.1/24 |
| guests | 192.168.50.1/24 |
| targets | 192.168.60.1/24 |

After assigning and enabling interfaces, verify that each VLAN gateway address is active.

![OPNsense dashboard showing all interfaces up](./images/01-dashboard-interfaces.png)
*The dashboard confirms all VLAN interfaces are enabled and holding their configured gateway IPs.*

A `/24` per VLAN is intentionally used for operational simplicity: the addressing pattern remains consistent across all segments (`.1` gateway, `.100–.200` DHCP pool) while still leaving ample host capacity.

---

## Part 3: DHCP

### 3.1 Kea vs Dnsmasq — which to use

OPNsense provides both Kea and Dnsmasq. For this homelab size, **Dnsmasq** is the practical default:

- Dnsmasq combines DNS and DHCP, enabling local hostname resolution with minimal setup.
- Kea is designed for larger deployments and usually requires additional integration work for comparable DNS behavior.
- For six small VLANs, Dnsmasq is simpler to operate and sufficient for typical lab requirements.

If migrating from Kea to Dnsmasq, disable Kea DHCP on the interface first; only one DHCP service can bind to the same interface at a time.

### 3.2 Configure Dnsmasq

Two areas are essential:

**General** — enable Dnsmasq and include **all 6 VLAN interfaces plus LAN** in the interface selection.

**DHCP ranges** — create one range per VLAN:

| Interface | Start | End |
|---|---|---|
| ai | 192.168.10.100 | 192.168.10.200 |
| blue | 192.168.20.100 | 192.168.20.200 |
| cloud | 192.168.30.100 | 192.168.30.200 |
| deception | 192.168.40.100 | 192.168.40.200 |
| guests | 192.168.50.100 | 192.168.50.200 |
| targets | 192.168.60.100 | 192.168.60.200 |

Leave Domains, Hosts, DHCP options, DHCP boot, and DHCP tags at defaults unless you have a specific requirement.

The following view shows a representative DHCP range entry for the `ai` interface.

![Dnsmasq DHCP range dialog](./images/02-dnsmasq-dhcp-range.png)
*Dnsmasq DHCP range configuration example for `ai`: `192.168.10.100` to `192.168.10.200`.*

---

## Part 4: NAT + Firewall Rules

### 4.1 NAT Outbound

`Firewall → NAT → Outbound` — verify mode is **Automatic**. If set to Manual without full outbound coverage for each subnet, newly created VLANs may not have internet egress.

### 4.2 Firewall pass rules — required on every VLAN tab

A newly created OPNsense interface starts with no allow rules (default deny). For each VLAN interface tab (`Firewall → Rules → [interface]`), add:

- Action: **Pass**
- Protocol: **any**
- Source: **`[interface] net`**
- Destination: **any**
- Save and apply

A common misconfiguration is using **`[interface] address`** as source. That only matches traffic originating from OPNsense itself and will not match VM traffic on the subnet.

![Firewall rule - address vs net mistake](./images/03-firewall-rule-address-vs-net.png)
*Use `[interface] net` for VLAN client traffic. `[interface] address` targets only the firewall interface IP and blocks expected VM flows.*

---

## Part 5: Attaching VMs to VLANs

For each VM:
- Hardware → Network Device → bridge = `vmbr1`
- **VLAN Tag** = VLAN number for that VM (for example, `30` for the Cloud VLAN)
- Inside the guest OS: use DHCP

No per-VM trunk configuration is required in the guest; Proxmox applies tagging at the virtual NIC configuration level.

The network device settings below show a VM mapped to VLAN 10 through the VLAN tag field.

![Proxmox VM network device with VLAN tag](./images/10-proxmox-vm-vlan-tag.png)
*Setting bridge `vmbr1` and VLAN Tag `10` places the VM directly into the `ai` VLAN.*

### Verifying from inside a VM

Before adding physical VLAN access, validate connectivity from the VM console (Proxmox noVNC):

```bash
ip a                    # should show an IP in the right VLAN subnet
ping 192.168.30.1        # VLAN gateway
ping 8.8.8.8              # internet by IP
ping google.com           # DNS resolution
```

The output below demonstrates successful DHCP assignment and internet connectivity from a VLAN-attached VM.

![Verified working VM on the ai VLAN](./images/11-wazuh-vm-verified-internet.png)
*The VM receives `192.168.10.129/24` from the `ai` DHCP pool and reaches external IPs, confirming DHCP, firewall policy, and NAT are operating correctly.*

---

## Errors Encountered (Troubleshooting Log)

### VM got no IP at all (APIPA / 169.254.x.x)
**Cause:** DHCP broadcast did not reach OPNsense, most commonly due to a VLAN tag mismatch.

**Fix checklist:**
1. Confirm `vmbr1` is VLAN aware and has no bridge port.
2. Confirm VM vNIC bridge is `vmbr1` (not `vmbr0`).
3. Confirm VM VLAN Tag exactly matches the VLAN tag configured in OPNsense.
4. Confirm a DHCP pool exists for that interface.
5. Confirm the guest OS is configured for DHCP.

### VM got IP + could ping gateway, but no internet
**Cause:** Firewall rule source set to `[interface] address` instead of `[interface] net`.

**Also check:** NAT Outbound mode is set to Automatic.

### SSH to OPNsense timed out
**Cause:** OPNsense disables SSH by default and does not allow WAN-side SSH access without an explicit rule.

**Fix:**
1. `System → Settings → Administration` → enable Secure Shell.
2. Add `Firewall → Rules → WAN` pass rule for TCP/22.
3. Restrict source IP as tightly as possible.
4. After use, remove or disable broad temporary SSH access rules.
5. Alternative: use the Proxmox VM console (`Console → option 8) Shell`).

The example rule below limits SSH access to a specific source host.

![Firewall WAN SSH rule](./images/04-firewall-wan-ssh-rule.png)
*Example WAN rule for SSH with source restriction to reduce management exposure.*

### `tailscale status` — "Failed to connect to local Tailscale daemon"
**Cause:** `tailscaled` service was not running.

**Fix:**
```bash
service tailscaled status
service tailscaled start
```

### Tailscale not showing up in the admin console machines list
**Cause:** The login URL produced by `tailscale up` was not completed in a browser.

**Fix:** Re-run `tailscale up --advertise-routes=...`, open the printed URL, and complete authorization.

### After advertising Tailscale routes, home network gateway became unreachable
**Cause:** A conflicting route was advertised (for example, the home LAN subnet) or an exit node setting introduced a routing conflict.

**Fix:** Advertise only the six VLAN subnets and confirm exit node settings are disabled where not required.

### Tailscale does not survive an OPNsense reboot
**Cause:** Known behavior with some `os-tailscale` plugin versions when relying on manual service enablement via shell-level configuration.

**Fix that worked in this setup:** `VPN → Tailscale → Settings` — enable Tailscale in the GUI and save from that page so OPNsense manages service startup through its own configuration workflow.

If required, cron `@reboot` or `os-shellcmd` can be used as fallback automation.

### `sysrc: unknown variable 'tailscaled_enable'`
**Cause:** Querying `sysrc tailscaled_enable` before the variable is defined.

**Fix:** Verify rcvar name first:
`grep -i rcvar /usr/local/etc/rc.d/tailscaled`

Then set it:
`sysrc tailscaled_enable="YES"`

### `crontab -e` — ":wq is not a vi command"
**Cause:** Command entered while still in insert mode.

**Fix:** Press `Esc`, confirm insert mode is exited, then run `:wq` and Enter. Use `:q!` to exit without saving.

---

## Accessing VLAN Services From a Laptop (Before Adding a Second NIC)

With one NIC and no switch, the laptop remains on the home router network (same side as OPNsense WAN) and does not have a direct physical path to VLAN interfaces. Two interim access methods are practical:

**Option A — Port forward (single service access)**
`Firewall → NAT → Port Forward` to map WAN ports to internal VLAN services.

**Option B — Tailscale subnet routes (recommended for multiple services)**
Advertise all VLAN subnets from OPNsense through Tailscale, approve routes in the admin console, and enable subnet route acceptance on the laptop client.

- OPNsense: Advertise routes = YES (six VLAN CIDRs), Exit node = NO
- Laptop: Accept subnet routes = YES, Use exit node = NO

The configuration below shows the VLAN route advertisements used for remote access without exposing each service through individual port forwards.

![OPNsense Tailscale advertised routes](./images/05-tailscale-advertised-routes.png)
*Advertise only VLAN CIDRs to avoid overlap with the home LAN and prevent local gateway routing conflicts.*

The long-term solution is to add a second NIC (or USB Ethernet adapter) and bridge it into `vmbr1` so physical devices can join the VLAN network directly.

---

## What's Next

- [ ] Add a second NIC / USB-Ethernet adapter for direct physical VLAN access
- [ ] Enable Suricata (built into OPNsense) on Deception + Targets interfaces for IDS/IPS
- [ ] Forward OPNsense logs (firewall + Suricata alerts) to Wazuh via syslog for SOC-VLAN log correlation
- [ ] Tighten inter-VLAN rules (currently any/any per VLAN to internet — add explicit VLAN-to-VLAN deny rules once base connectivity is confirmed working)

---

*Built on OPNsense 26.x, Proxmox VE 8.x. Corrections and pull requests are welcome.*
