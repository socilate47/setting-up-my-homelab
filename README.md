# Building 6 VLANs on OPNsense with One NIC and No Switch (Proxmox Homelab)

This guide documents a complete OPNsense and Proxmox VLAN lab built with a single physical NIC and no managed switch. It segments traffic into six VLANs — **Guest, AI, SOC, Cloud, Deception, and Targets** — and adds secure remote reachability using Tailscale subnet routing.

For host and guest monitoring, see the [Prometheus and Grafana on Proxmox guide](docs/Prometheus-Proxmox-Guide.docx).

The procedure and troubleshooting notes are based on a working implementation and focus on practical setup details that are easy to miss in first-pass deployments.

## Why this guide exists

Most VLAN tutorials assume a managed switch is already available for physical tagging. This guide covers a common starting point for homelabs: one NIC and no additional switching hardware. In this model, VLAN segmentation still works by carrying tagged traffic on an internal Proxmox bridge that never leaves the host physically.

## Architecture overview

```text
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

With one NIC, that physical link is dedicated to OPNsense WAN uplink traffic. All six VLANs are carried on `vmbr1`, a VLAN-aware internal bridge with no physical bridge port. As a result, only VMs attached to `vmbr1` with matching VLAN tags participate in those networks.

## Part 1: Proxmox networking

### 1.1 Create the internal VLAN-aware bridge

Open `Proxmox → node → System → Network → Create → Linux Bridge` and configure:

- **Name:** `vmbr1`
- **Bridge port:** leave empty (intentional; internal-only bridge)
- **VLAN aware:** enabled
- **IP address:** none required
- Apply changes

`vmbr0` remains unchanged and continues to carry WAN connectivity over the physical NIC.

![Proxmox Create Linux Bridge dialog](./images/06-proxmox-create-linux-bridge.png)
*Proxmox Network view with `vmbr1` as a VLAN-aware internal bridge and `vmbr0` mapped to the physical NIC/gateway.*

### 1.2 Attach two virtual NICs to OPNsense

In `VM → Hardware`:

- `net0` → bridge `vmbr0` (WAN)
- `net1` → bridge `vmbr1`, **VLAN Tag left blank** (trunk carrying all VLAN tags)

![OPNsense VM Hardware tab](./images/07-proxmox-opnsense-vm-hardware.png)
*OPNsense VM hardware configuration with `net0=vmbr0` and `net1=vmbr1`.*

## Part 2: OPNsense VLAN interfaces

### 2.1 Create VLAN definitions

Go to `Interfaces → Other Types → VLAN → Add` and create six entries using the LAN trunk parent (shown as `vtnet0` in this environment):

| VLAN Tag | Name | Purpose |
|---|---|---|
| 10 | ai | AI workloads |
| 20 | blue | SOC / blue team |
| 30 | cloud | Cloud lab |
| 40 | deception | Deception / honeypots |
| 50 | guests | Guest devices |
| 60 | targets | Target systems |

![OPNsense VLAN creation](./images/08-opnsense-vlan-creation.png)
*Six VLANs created on the trunk parent with tags 10 through 60.*

### 2.2 Assign VLANs as interfaces

In `Interfaces → Assignments`, add each `VLAN00xx on vtnet0` interface.

Do not assign the untagged parent interface (`vtnet0` with no tag) as a VLAN interface. Only assign the tagged VLAN sub-interfaces.

![OPNsense interface assignments](./images/09-opnsense-interface-assignments.png)
*Each VLAN mapped to an interface assignment slot (`optX`) for tags 10, 20, 30, 40, 50, and 60.*

Static gateway IPs:

| Interface | IP |
|---|---|
| ai | 192.168.10.1/24 |
| blue | 192.168.20.1/24 |
| cloud | 192.168.30.1/24 |
| deception | 192.168.40.1/24 |
| guests | 192.168.50.1/24 |
| targets | 192.168.60.1/24 |

![OPNsense dashboard showing all interfaces up](./images/01-dashboard-interfaces.png)
*Dashboard confirmation that LAN and all six VLAN interfaces are enabled and assigned expected gateway addresses.*

A `/24` per VLAN keeps addressing straightforward (`.1` gateway, `.100-.200` DHCP pool) while providing ample capacity for homelab workloads.

## Part 3: DHCP

### 3.1 Dnsmasq vs Kea

For this lab size, Dnsmasq is the practical default:

- Integrated DNS + DHCP for local hostname resolution
- Lower operational complexity for small VLAN counts
- No plugin dependency for basic local name registration behavior

If migrating from Kea, disable Kea DHCP on the target interface before enabling Dnsmasq. Only one DHCP server can bind per interface.

### 3.2 Configure Dnsmasq scopes

Two areas are essential:

1. **General**: enable Dnsmasq and select **LAN + all six VLAN interfaces**.
2. **DHCP ranges**: define one range per VLAN.

| Interface | Start | End |
|---|---|---|
| ai | 192.168.10.100 | 192.168.10.200 |
| blue | 192.168.20.100 | 192.168.20.200 |
| cloud | 192.168.30.100 | 192.168.30.200 |
| deception | 192.168.40.100 | 192.168.40.200 |
| guests | 192.168.50.100 | 192.168.50.200 |
| targets | 192.168.60.100 | 192.168.60.200 |

Default values for Domains, Hosts, DHCP options, DHCP boot, and DHCP tags are sufficient for base functionality.

![Dnsmasq DHCP range dialog](./images/02-dnsmasq-dhcp-range.png)
*Dnsmasq DHCP range configuration example for interface `ai`.*

## Part 4: NAT and firewall rules

### 4.1 NAT outbound mode

Navigate to `Firewall → NAT → Outbound` and confirm mode is **Automatic**.

If set to Manual without corresponding per-subnet outbound rules, new VLANs will not have internet access.

### 4.2 Required pass rule on each VLAN interface

Each new OPNsense interface starts with no allow rules. For every VLAN tab under `Firewall → Rules → [interface]`, create:

- **Action:** Pass
- **Protocol:** any
- **Source:** `[interface] net`
- **Destination:** any

Save each rule and apply changes.

A frequent misconfiguration is using `[interface] address` as Source. That value only matches the firewall interface IP, not client hosts in the VLAN, and therefore blocks VM traffic.

![Firewall rule - address vs net mistake](./images/03-firewall-rule-address-vs-net.png)
*Example of the `address` vs `net` source mismatch that prevents client traffic from matching the pass rule.*

## Part 5: Attach VMs to VLANs

For each VM network device:

- Bridge: `vmbr1`
- VLAN Tag: matching VLAN ID (for example, `30` for a Cloud/SOC VM on VLAN 30)
- Guest OS network: DHCP

No per-VM trunk configuration is required; Proxmox applies VLAN tagging at the vNIC.

![Proxmox VM network device with VLAN tag](./images/10-proxmox-vm-vlan-tag.png)
*VM network adapter assigned to `vmbr1` with VLAN tagging at the virtual NIC.*

### Validate from inside a VM

Because there is no physical VLAN path yet, validate from VM console access (Proxmox noVNC):

```bash
ip a                    # expect an IP in the correct VLAN subnet
ping 192.168.30.1       # test VLAN gateway
ping 8.8.8.8            # test internet by IP
ping google.com         # test DNS resolution
```

![Verified working VM on the ai VLAN](./images/11-wazuh-vm-verified-internet.png)
*Example validation output showing successful DHCP assignment and internet connectivity from a VLAN guest VM.*

## Troubleshooting

### 1) VM receives APIPA address (`169.254.x.x`)

**Cause:** DHCP broadcast is not reaching OPNsense, usually due to VLAN tag or bridge mismatch.

**Checks:**

1. Confirm `vmbr1` is VLAN aware and has no bridge port.
2. Confirm VM bridge is `vmbr1` (not `vmbr0`).
3. Confirm VM VLAN Tag exactly matches the OPNsense VLAN ID.
4. Confirm a DHCP range exists for that interface.
5. Confirm guest OS NIC is set to DHCP (not static).

### 2) VM can ping gateway but has no internet

**Likely cause:** Interface rule source uses `[interface] address` instead of `[interface] net`.

**Also verify:** NAT Outbound mode is Automatic.

### 3) SSH to OPNsense times out

**Cause:** SSH is disabled by default, and WAN access is blocked unless explicitly allowed.

**Resolution:**

1. Enable SSH in `System → Settings → Administration`.
2. Add a temporary `Firewall → Rules → WAN` TCP/22 pass rule restricted to a specific source IP.
3. Remove or tighten the WAN SSH rule after use.
4. Use the Proxmox VM console (`Console → option 8) Shell`) as a zero-exposure alternative.

![Firewall WAN SSH rule](./images/04-firewall-wan-ssh-rule.png)
*WAN SSH rule restricted to a single source IP.*

### 4) `tailscale status` returns “Failed to connect to local Tailscale daemon”

**Cause:** `tailscaled` service is not running.

**Resolution:**

```bash
service tailscaled status
service tailscaled start
```

### 5) Tailscale node does not appear in admin console

**Cause:** `tailscale up` was executed, but the authentication URL was not opened and approved.

**Resolution:** Re-run `tailscale up --advertise-routes=...`, open the generated URL, and complete authorization.

### 6) Home gateway becomes unreachable after advertising routes

**Cause:** Advertised routes overlap the home LAN subnet, or an exit node was enabled unintentionally.

**Resolution:**

- Advertise only the six VLAN subnets
- Do not advertise the home router subnet
- Keep exit node disabled on both OPNsense and client

### 7) Tailscale does not survive OPNsense reboot

**Cause:** Community `os-tailscale` plugin behavior can overwrite manual `sysrc` persistence during boot.

**Resolution that worked in this setup:** Use `VPN → Tailscale → Settings`, enable the plugin there, and save from the plugin settings page. This hands startup management back to OPNsense service integration.

Fallback options such as `@reboot` jobs or `os-shellcmd` remain valid if needed.

### 8) `sysrc: unknown variable 'tailscaled_enable'`

**Cause:** Variable queried before being defined.

**Resolution:**

```bash
grep -i rcvar /usr/local/etc/rc.d/tailscaled
sysrc tailscaled_enable="YES"
```

### 9) `crontab -e` shows `:wq is not a vi command`

**Cause:** Command entered while still in insert mode.

**Resolution:** Press `Esc`, verify `-- INSERT --` is gone, then run `:wq`. Use `:q!` to exit without saving.

## Access VLAN services from a laptop before adding a second NIC

With one NIC and no managed switch, laptops remain on the home LAN (OPNsense WAN side) and have no direct layer-2 path to VLAN networks.

### Option A: Port forward (quick, per-service)

Use `Firewall → NAT → Port Forward` to publish a specific WAN port to an internal VLAN service IP/port.

### Option B: Tailscale subnet routes (recommended)

Advertise all six VLAN CIDRs from OPNsense, approve them in the Tailscale admin console, and enable subnet route acceptance on the laptop client.

Recommended settings:

- **OPNsense:** Advertise routes = Yes (six VLAN subnets), Exit node = No
- **Laptop:** Accept subnet routes = Yes, Use exit node = No

![OPNsense Tailscale advertised routes](./images/05-tailscale-advertised-routes.png)
*Tailscale advertised routes for the six VLAN subnets without overlap to home LAN ranges.*

The long-term fix is to add a second NIC (or USB-to-Ethernet adapter) and bridge it into `vmbr1` for direct physical VLAN access.

## Next steps

- [ ] Add a second NIC or USB-to-Ethernet adapter for direct physical VLAN access
- [ ] Enable Suricata on Deception and Targets interfaces for IDS/IPS
- [ ] Forward OPNsense firewall and Suricata logs to Wazuh via syslog for SOC correlation
- [ ] Tighten inter-VLAN policy after baseline connectivity validation

Built on OPNsense 26.x and Proxmox VE 8.x. Corrections and pull requests are welcome.
