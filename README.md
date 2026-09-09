# Building 6 VLANs on OPNsense with One NIC and No Switch (Proxmox Homelab)

This guide explains how to segment a homelab into six VLANs—**Guest, AI, SOC, Cloud, Deception, and Targets**—using OPNsense as a virtual router in Proxmox with **one physical NIC** and **no managed switch**. It documents a complete working setup, including key troubleshooting outcomes.

## Why this guide exists

Many OPNsense and Proxmox VLAN guides assume a managed switch is already available for VLAN tagging. This walkthrough focuses on a common starting point: one NIC and no additional network hardware. In this design, VLAN segmentation is implemented virtually in Proxmox first. Physical device access can be added later by introducing a second NIC.

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

With a single NIC, the host NIC is used as OPNsense's WAN uplink to the home router. All VLANs are carried on `vmbr1`, an internal VLAN-aware bridge with no physical port. As a result, VLAN access is initially available to virtual machines only, not to physical endpoints on the home network.

---

## Part 1: Proxmox Networking

### 1.1 Create the internal VLAN-aware bridge

`Proxmox → node → System → Network → Create → Linux Bridge`

- Name: `vmbr1`
- Bridge port: leave empty (intentional for an internal bridge)
- Check **VLAN aware**
- No IP address needed
- Apply

Keep the existing `vmbr0` configuration unchanged, because it remains the WAN bridge tied to the physical NIC.

The screenshot below shows the bridge creation dialog and the existing bridge table.

![Proxmox Create Linux Bridge dialog](./images/06-proxmox-create-linux-bridge.png)

Observe that `vmbr1` is configured as VLAN-aware with no bridge ports, while `vmbr0` remains associated with the physical uplink. This separation is required so VLAN traffic stays internal to Proxmox in a one-NIC design.

### 1.2 Give OPNsense two virtual NICs

In `VM → Hardware`, configure:

- `net0` → bridge `vmbr0` (WAN)
- `net1` → bridge `vmbr1`, with **VLAN Tag left blank** so it operates as a trunk

The following image shows the expected NIC mapping on the OPNsense VM.

![OPNsense VM Hardware tab](./images/07-proxmox-opnsense-vm-hardware.png)

Verify that `net0` points to `vmbr0` and `net1` points to `vmbr1` without a tag. This is what allows OPNsense to receive all VLAN-tagged traffic on the LAN side.

---

## Part 2: OPNsense — Creating the 6 VLANs

### 2.1 Create VLAN interfaces

Go to `Interfaces → Other Types → VLAN → Add` and repeat six times. Use the LAN trunk parent interface (shown as `vtnet0` in this setup).

| VLAN Tag | Name | Purpose |
|---|---|---|
| 10 | ai | AI workloads |
| 20 | blue | SOC / blue team |
| 30 | cloud | Cloud lab |
| 40 | deception | Deception / honeypots |
| 50 | guests | Guest devices |
| 60 | targets | Target systems |

This screenshot shows the completed VLAN entries.

![OPNsense VLAN creation](./images/08-opnsense-vlan-creation.png)

Confirm that tags 10 through 60 are all attached to the same trunk parent. This ensures each VLAN has a defined logical interface before assignment.

### 2.2 Assign each VLAN as an interface

Go to `Interfaces → Assignments`. Add each `VLAN00xx on vtnet0` entry from the dropdown. Do not assign the raw parent interface and do not reuse the default untagged LAN entry.

A common configuration error is assigning the plain parent interface (`vtnet0` without a tag). The parent is the trunk transport and not a VLAN endpoint, so only tagged `VLAN00xx` interfaces should be assigned.

The image below shows the assignment view after mapping VLAN interfaces.

![OPNsense interface assignments](./images/09-opnsense-interface-assignments.png)

Check that each `optX` interface maps to the intended VLAN tag (10, 20, 30, 40, 50, 60). Correct mapping is necessary for DHCP, routing, and firewall policy to apply per segment.

Static IPs used:

| Interface | IP |
|---|---|
| ai | 192.168.10.1/24 |
| blue | 192.168.20.1/24 |
| cloud | 192.168.30.1/24 |
| deception | 192.168.40.1/24 |
| guests | 192.168.50.1/24 |
| targets | 192.168.60.1/24 |

After interface assignment and addressing, the dashboard should reflect all interfaces in an active state.

![OPNsense dashboard showing all interfaces up](./images/01-dashboard-interfaces.png)

Verify that LAN plus all six VLAN interfaces are present, enabled, and holding their configured gateway IPs. This is the baseline health check before DHCP and firewall work.

**Why /24 for every VLAN?** A `/24` provides 254 usable addresses per VLAN, which is usually sufficient for homelab segmentation while keeping addressing and DHCP ranges consistent across all networks.

---

## Part 3: DHCP

### 3.1 Kea vs Dnsmasq — which to use

OPNsense provides both Kea and Dnsmasq. For this scale, Dnsmasq is typically the better fit:

- Dnsmasq combines DNS and DHCP, enabling straightforward local hostname resolution
- Kea is optimized for larger deployments and usually requires additional integration for equivalent DNS behavior
- For six small VLANs, Dnsmasq is operationally simpler

If migrating from Kea, disable Kea DHCP on the interface before enabling Dnsmasq. Only one DHCP service can bind per interface, otherwise an “address already in use” error occurs.

### 3.2 Configure Dnsmasq

The two primary tabs are:

**General** — enable the service and select **all 6 VLANs plus LAN** under Interfaces. If an interface is omitted here, DHCP will not be served there even if a range exists.

**DHCP ranges** — create one range per VLAN:

| Interface | Start | End |
|---|---|---|
| ai | 192.168.10.100 | 192.168.10.200 |
| blue | 192.168.20.100 | 192.168.20.200 |
| cloud | 192.168.30.100 | 192.168.30.200 |
| deception | 192.168.40.100 | 192.168.40.200 |
| guests | 192.168.50.100 | 192.168.50.200 |
| targets | 192.168.60.100 | 192.168.60.200 |

Leave Domains, Hosts, DHCP options, DHCP boot, and DHCP tags at defaults for a baseline deployment.

The screenshot below shows a DHCP range entry.

![Dnsmasq DHCP range dialog](./images/02-dnsmasq-dhcp-range.png)

Use it to confirm the selected interface and IP range format. Replicating this pattern for each VLAN ensures consistent client lease behavior.

---

## Part 4: NAT + Firewall Rules

### 4.1 NAT Outbound

Go to `Firewall → NAT → Outbound` and verify mode is **Automatic**. If set to Manual without corresponding rules for each subnet, new VLANs will not have outbound internet access.

### 4.2 Firewall pass rules — required on every VLAN tab

Each new OPNsense interface starts with a default deny policy. For each VLAN interface tab (`Firewall → Rules → [interface]`), add:

- Action: **Pass**
- Protocol: **any**
- Source: **`[interface] net`**
- Destination: **any**
- Save, repeat for all six VLANs, then Apply

A critical detail is using `[interface] net` rather than `[interface] address`. `address` matches only the OPNsense interface IP itself, while `net` matches client traffic from the entire VLAN subnet.

The following screenshot illustrates this distinction.

![Firewall rule - address vs net mistake](./images/03-firewall-rule-address-vs-net.png)

When reviewing your rule, confirm Source is set to `<vlan> net`. This single field determines whether VM-originated traffic can pass.

---

## Part 5: Attaching VMs to VLANs

For each VM:

- Hardware → Network Device → bridge = `vmbr1`
- **VLAN Tag** = target VLAN ID (for example, `30`)
- Inside the guest OS: use DHCP

No per-VM trunk setup is needed in the guest. Proxmox applies tagging at the virtual NIC configuration.

The screenshot below shows a correctly tagged VM NIC.

![Proxmox VM network device with VLAN tag](./images/10-proxmox-vm-vlan-tag.png)

Confirm the bridge is `vmbr1` and the VLAN Tag matches the intended segment. This is the control point that places the VM into the right VLAN.

### Verifying from inside a VM

Because there is no physical VLAN path yet, validate from the VM console in Proxmox (noVNC):

```bash
ip a                    # should show an IP in the right VLAN subnet
ping 192.168.30.1        # VLAN gateway
ping 8.8.8.8              # internet by IP
ping google.com           # DNS resolution
```

The terminal output in the next image demonstrates successful end-to-end validation.

![Verified working VM on the ai VLAN](./images/11-wazuh-vm-verified-internet.png)

Check for three signals: a DHCP lease in the expected subnet, successful gateway reachability, and successful internet and DNS tests. Together, these confirm DHCP, firewall, and NAT are functioning.

---

## Errors Encountered (Troubleshooting Log)

### VM got no IP at all (APIPA / 169.254.x.x)
**Cause:** DHCP broadcast did not reach OPNsense, usually due to a VLAN tag mismatch.
**Fix checklist:**
1. Confirm `vmbr1` is VLAN aware with no bridge port
2. Confirm the VM’s vNIC bridge = `vmbr1` (not `vmbr0`)
3. Confirm the VM’s VLAN Tag matches the OPNsense VLAN tag exactly
4. Confirm a DHCP pool exists for that interface (service running alone is not sufficient)
5. Confirm the guest OS is set to DHCP rather than a stale static template config

### VM got IP + could ping gateway, but no internet
**Cause (in this setup):** Firewall rule Source was set to `[interface] address` instead of `[interface] net`.
**Also check:** NAT Outbound mode is Automatic.

### SSH to OPNsense timed out
**Cause:** SSH is disabled by default, and WAN-side management is not open by default.
**Fix:**
1. `System → Settings → Administration` → enable Secure Shell
2. Add an explicit `Firewall → Rules → WAN` pass rule for TCP/22, ideally restricted to a specific source IP
3. Remove or disable the rule after use, or keep Source tightly restricted
4. Alternative with no network exposure: use the Proxmox VM console (`Console → option 8) Shell`)

The screenshot below shows a constrained WAN SSH rule.

![Firewall WAN SSH rule](./images/04-firewall-wan-ssh-rule.png)

Use this as a reference to keep SSH exposure limited to a known source rather than opening it broadly.

### `tailscale status` — "Failed to connect to local Tailscale daemon"
**Cause:** `tailscaled` service was not running.
**Fix:**
```bash
service tailscaled status
service tailscaled start
```

### Tailscale not showing in the admin console machines list
**Cause:** `tailscale up` was run, but the login URL was not opened and authenticated.
**Fix:** Re-run `tailscale up --advertise-routes=...`, open the printed URL, and authorize.

### After advertising Tailscale routes, home network gateway became unreachable
**Cause:** The home LAN subnet was advertised by mistake (or an exit node was enabled), creating a routing conflict.
**Fix:** Advertise only the six VLAN subnets and confirm no exit node is set on OPNsense or the client.

### Tailscale does not survive an OPNsense reboot
**Cause:** Known issue in the `os-tailscale` plugin where `sysrc tailscaled_enable="YES"` may not persist because OPNsense regenerates service configuration during boot.

**Fix that worked here:** Use `VPN → Tailscale → Settings`, ensure **Enable** is checked, and click **Save** on that page. This keeps service control inside OPNsense’s native configuration workflow.

A cron `@reboot` task or the `os-shellcmd` plugin remains a fallback if the GUI toggle does not persist in your version.

### `sysrc: unknown variable 'tailscaled_enable'`
**Cause:** `sysrc tailscaled_enable` was queried before the variable was set.
**Fix:** Check the rcvar name first with `grep -i rcvar /usr/local/etc/rc.d/tailscaled`, then set it using `sysrc tailscaled_enable="YES"`.

### `crontab -e` — ":wq is not a vi command"
**Cause:** `:wq` was entered while still in insert mode.
**Fix:** Press `Esc`, ensure `-- INSERT --` is no longer shown, then enter `:wq` and press Enter. To exit without saving: `Esc` then `:q!`.

---

## Accessing VLAN Services From a Laptop (Before Adding a Second NIC)

With one NIC and no switch, the laptop remains on the home-router side of OPNsense and has no direct Layer 2 path to the internal VLANs. Two temporary options are available:

**Option A — Port forward (single-service access)**
`Firewall → NAT → Port Forward` to map a WAN port to a VLAN host and service port.

**Option B — Tailscale subnet routes (recommended for multiple services)**
Advertise the six VLAN CIDRs from OPNsense, approve them in Tailscale admin, and enable subnet route acceptance on the laptop client. This enables access to VLAN IPs directly without maintaining multiple individual forwards.

- OPNsense: Advertise routes = YES (the six VLAN CIDRs), Exit node = NO
- Laptop: Accept subnet routes = YES, Use exit node = NO

The screenshot below shows the advertised routes configuration.

![OPNsense Tailscale advertised routes](./images/05-tailscale-advertised-routes.png)

Confirm only VLAN CIDRs are advertised and that none overlap with the home LAN subnet. This avoids the gateway reachability conflict described in troubleshooting.

**Long-term fix:** add a second NIC (or USB-to-Ethernet adapter) and bridge it into `vmbr1` so physical devices can join the VLAN environment directly.

---

## What's Next

- [ ] Add a second NIC / USB-Ethernet adapter for direct physical VLAN access
- [ ] Enable Suricata (built into OPNsense) on Deception + Targets interfaces for IDS/IPS
- [ ] Forward OPNsense logs (firewall + Suricata alerts) to Wazuh via syslog for SOC-VLAN log correlation
- [ ] Tighten inter-VLAN rules (currently any/any per VLAN to internet — add explicit VLAN-to-VLAN deny rules after base connectivity is confirmed)

---

Built on OPNsense 26.x and Proxmox VE 8.x. Corrections and pull requests are welcome.
