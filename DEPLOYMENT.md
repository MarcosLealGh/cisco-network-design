# Deployment guide

How to apply these configs to real Cisco equipment, and what to verify first.

## Applying a config

1. Connect a console cable to the device.
2. Enter privileged mode: `enable`
3. Enter configuration mode: `configure terminal`
4. Paste the config block.
5. Save: `end` → `write memory` (or `copy running-config startup-config`)

> **Set the passwords first.** Every config has `<SET_STRONG_PASSWORD>` placeholders for `enable secret` and the local `admin` user — replace them with real credentials before applying. Never commit real passwords.

## Per-device checklist

**`1941_core_router.txt`**
- Gi0/0 = trunk toward the distribution switch
- Gi0/1 = WAN toward the border router
- Se0/0/0 = serial toward the branch router (WIC module)
- The 1941 is DCE on the serial link → `clock rate` belongs here

**`1841_branch_router.txt`**
- Se0/0/0 = serial toward the core
- Fa0/0 = branch LAN
- The 1841 is DTE → no `clock rate` (already omitted)

**`SWD_distribution.txt`**
- Gi0/1 = uplink to the core
- Fa0/1 → SW-A1, Fa0/2 → SW-A2, Fa0/3 → SW-A3
- On some 2960 models `switchport trunk encapsulation dot1q` is not accepted (dot1q is the only option) — omit that line if the IOS rejects it

**`SWA1_access.txt` / `SWA2_access.txt` / `SWA3_access_aps.txt`**
- Fa0/1 = uplink trunk to SW-D
- Adjust the access port ranges to the real number of hosts
- SW-A2: the server ports (VLAN 50) use static IP, not DHCP
- SW-A3: one AP per trunk port; shut down the ports for APs you don't have

## IP addressing reference

| VLAN | Purpose | Network | Gateway |
|---|---|---|---|
| 10 | Corporate | 192.168.10.0/24 | .1 |
| 20 | Sales | 192.168.20.0/24 | .1 |
| 30 | IT / Operations | 192.168.30.0/24 | .1 |
| 40 | Guest WiFi | 192.168.40.0/24 | .1 |
| 50 | Servers | 192.168.50.0/24 | .1 |
| 99 | Management | 192.168.99.0/24 | .1 |

| Link | Addressing |
|---|---|
| Core ↔ border (WAN) | 10.0.0.2/30 ↔ 10.0.0.1/30 |
| Core ↔ branch (serial) | 172.16.0.1/30 ↔ 172.16.0.2/30 |
| Branch LAN | 192.168.100.0/24 |

**Switch management IPs (SVI VLAN 99):** SW-D `.2`, SW-A1 `.3`, SW-A2 `.4`, SW-A3 `.5`
**Servers (VLAN 50, static):** helpdesk `192.168.50.10`, SIEM `192.168.50.11`

## ACL ordering — important

The four ACLs on the core router are evaluated top-down. **PERMIT lines must come before DENY lines**, or the exceptions never match.

- **VLAN10_IN / VLAN20_IN** (corporate / sales): permit helpdesk (TCP 443) and SIEM (TCP/UDP 1514) → deny the rest of the server VLAN → deny management VLAN → permit everything else (internet, other VLANs).
- **VLAN40_IN** (guest WiFi): deny every internal VLAN and both WAN links → permit only internet egress.
- **VLAN50_IN** (servers): permit reaching internal VLANs (to answer requests) → deny any (no internet egress).

If a DENY is placed before its PERMIT, the intended exception (e.g. a monitoring agent reaching the SIEM) is silently blocked.
