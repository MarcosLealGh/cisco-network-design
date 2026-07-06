# Segmented Enterprise Network — Cisco IOS Design

A complete, production-style Cisco IOS configuration for a segmented small-enterprise network: six VLANs, inter-VLAN routing, OSPF, NAT, layered ACLs, and a hardened switching layer. Built as an academic capstone project (Duoc UC, Network Administration), running on Cisco 1941 / 1841 routers and 2960 switches.

## The design problem

A flat network — every device in one broadcast domain — has no internal boundaries: a compromised guest laptop can reach the servers, and the helpdesk traffic competes with everything else. The goal here was a network where **each department is isolated, guests are walled off from internal systems, and only the traffic that should cross a boundary does.**

## The architecture

```
                    Internet
                       │
                 [ Border router ]
                       │ 10.0.0.0/30 (WAN)
                       │
               ┌───────────────┐        serial 172.16.0.0/30      ┌──────────────────┐
               │  CORE  (1941)  │──────────────────────────────────│  BRANCH  (1841)  │
               │ router-on-a-   │                                   │  OSPF area 0     │
               │  stick + OSPF  │                                   │  LAN .100.0/24   │
               │  + NAT + ACLs  │                                   └──────────────────┘
               └───────┬───────┘
                       │ trunk (all VLANs)
               ┌───────────────┐
               │  SW-D (2960)   │  spanning-tree root
               │  distribution  │
               └───┬───────┬───────┬───┘
        trunk 10,20│  30,50│  10,40│ trunk
            ┌──────┘   ┌───┘   └────┐
       ┌─────────┐ ┌─────────┐ ┌─────────┐
       │  SW-A1  │ │  SW-A2  │ │  SW-A3  │
       │ Corp+   │ │ IT +    │ │ WiFi    │
       │ Sales   │ │ Servers │ │ APs     │
       └─────────┘ └─────────┘ └─────────┘
```

**Six VLANs:** Corporate (10), Sales (20), IT/Operations (30), Guest WiFi (40), Servers (50), Management (99). Full addressing table in [`DEPLOYMENT.md`](DEPLOYMENT.md).

## What each layer demonstrates

| Layer | Techniques |
|---|---|
| **Core routing (1941)** | Router-on-a-stick: one 802.1Q subinterface per VLAN as its gateway. OSPF area 0 across all networks. NAT (PAT/overload) for internet egress. Static default route to the border. |
| **Access control** | Four extended ACLs enforcing the segmentation intent — corporate/sales reach only the helpdesk and SIEM on the server VLAN; guest WiFi is denied every internal VLAN and both WAN links; servers can answer internal requests but have no internet egress. **Permit-before-deny ordering** is explicit (Cisco evaluates top-down). |
| **Distribution (SW-D)** | Rapid-PVST with SW-D as the explicit spanning-tree root. Trunks carry only the VLANs each downstream switch needs. Native VLAN moved off VLAN 1 to 99. |
| **Access switches** | Per-port hardening: `port-security` (max 1 MAC, restrict on violation), PortFast + BPDU Guard on host ports, unused ports isolated to a black-hole VLAN (999) and shut down. |
| **WAN / branch (1841)** | Serial link with correct DCE/DTE roles (clock rate only on the DCE side). OSPF adjacency over the serial link. Separate branch LAN with its own DHCP scope. |
| **Management** | SSH only (RSA 2048), no Telnet, no HTTP server, `service password-encryption`, dedicated management VLAN unreachable from user VLANs. |

## Why it's built this way — a few decisions

- **Native VLAN is 99, not 1.** Leaving user or trunk traffic on the default VLAN 1 is a known VLAN-hopping vector; moving the native VLAN to an unused management VLAN closes it.
- **Unused ports aren't just "left alone."** They're placed in VLAN 999 (which routes nowhere) and administratively shut — an unplugged port shouldn't be a way onto the network.
- **The server VLAN can't reach the internet.** Servers answer internal requests but have no egress — if one is compromised, it can't phone home.
- **Guest WiFi APs use trunk ports, not access ports.** Each AP serves two SSIDs on different VLANs (corporate + guest), so the switchport has to carry both tagged.

## Verification

This isn't just a paper design — it was built on real Cisco hardware (1941/1841 routers, 2960 switches) and verified with `show` commands and connectivity tests. Screenshots in [`evidence/`](evidence/):

| Evidence | What it proves |
|---|---|
| `ospf-neighbor-up.png` | OSPF adjacency established between the core and branch routers |
| `routing-table.png` | Routes for every VLAN and the OSPF-learned branch network are present |
| `nat-translations.png` | NAT/PAT is translating internal hosts to the WAN interface |
| `vlan-brief.png` | All six VLANs exist and ports are assigned as designed |
| `spanning-tree-root.png` | SW-D is the elected spanning-tree root (as configured) |
| `intervlan-ping-ok.png` | A host in VLAN 10 reaches VLAN 30 — inter-VLAN routing works |
| `acl-blocks-server-vlan.png` | A user ping into the server VLAN is **denied** — the ACL enforces the boundary |
| `rack.jpg` | The physical rack the topology runs on |

The last two together are the point: connectivity works *where it should* and is blocked *where it shouldn't* — the segmentation is real, not just declared.

## Repository layout

```
cisco-network-design/
├── README.md            ← this file
├── DEPLOYMENT.md        ← how to apply, per-device checklist, IP table, ACL notes
├── configs/
│   ├── 1941_core_router.txt       core: subinterfaces, OSPF, NAT, ACLs
│   ├── 1841_branch_router.txt     branch: serial WAN, OSPF, DHCP
│   ├── SWD_distribution.txt        STP root, trunks
│   ├── SWA1_access.txt             VLAN 10 + 20, port-security
│   ├── SWA2_access.txt             VLAN 30 + 50 (servers)
│   └── SWA3_access_aps.txt         WiFi APs, multi-SSID trunks
└── evidence/                       show-command output + connectivity tests
```

## Notes

- Configs are ready to paste into Cisco IOS CLI. Passwords are `<SET_STRONG_PASSWORD>` placeholders — set real ones before applying.
- Addressing uses RFC 1918 private ranges (lab/reference topology).
- Config comments are in Spanish (the language of the team that built it).
