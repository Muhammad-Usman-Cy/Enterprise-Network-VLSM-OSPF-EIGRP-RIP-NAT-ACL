# Enterprise Network Design and Implementation

**VLSM · OSPF · EIGRP · RIP v2 · NAT · ACL · DHCP**

A large-scale enterprise network designed and implemented in Cisco Packet Tracer, spanning four routing domains, eleven LAN networks, twenty-two routers, and end-to-end services including DHCP, NAT/PAT, ACL-based access control, and an internal mail server.

> Final semester project for the Computer Networks course at FAST NUCES, Department of Cyber Security.

---

## Table of Contents

1. [Overview](#overview)
2. [Network Topology](#network-topology)
3. [Routing Domains](#routing-domains)
4. [IP Addressing and VLSM Design](#ip-addressing-and-vlsm-design)
5. [Route Redistribution](#route-redistribution)
6. [Services](#services)
7. [Security: ACL](#security-acl)
8. [Testing and Verification](#testing-and-verification)
9. [Files in This Repository](#files-in-this-repository)
10. [How to Run](#how-to-run)
11. [Author](#author)
12. [License](#license)

---

## Overview

The network was built from scratch given only a private IP block (`27.128.22.25`) and a host-count requirement per subnet. The objective was to demonstrate end-to-end understanding of:

- Variable Length Subnet Masking on a single contiguous block
- Multiple dynamic routing protocols (link-state, advanced distance-vector, classic distance-vector)
- Mutual route redistribution across protocol boundaries
- Centralized DHCP with `ip helper-address` relay across multiple domains
- Port Address Translation (NAT overload) on a public-facing router
- Extended Access Control Lists for host- and subnet-level filtering
- Application-layer services (SMTP / POP3 mail) over the converged network

## Network Topology

The network contains **22 routers**, **11 LAN segments (A–K)**, and **23 point-to-point WAN links** using /30 subnets.

![Topology](screenshots/topology.png)

## Routing Domains

| Domain | Routers | Networks | Boundary Router |
|---|---|---|---|
| OSPF Area 1 | R0, R3, R4, R5, R6 | A, B, C | R5 (to EIGRP 5) |
| EIGRP AS 5 | R5, R8, R9, R10, R11, R12, R13 | D, E, F | R13 (to OSPF Area 2) |
| OSPF Area 2 | R13, R14, R15, R16, R21 | G, H, I | R21 (to RIP v2) |
| RIP v2 | R17, R18, R19, R20, R21 | J, K | R21 (from OSPF Area 2) |

## IP Addressing and VLSM Design

Total required hosts across all eleven subnets: **612,229**. A `/12` block was required (`2^20 - 2 = 1,048,574` usable hosts).

- **Assigned base block:** `27.128.0.0/12`
- **Usable range:** `27.128.0.0 – 27.143.255.255`

Subnets were allocated largest-to-smallest to eliminate waste:

| Net | Hosts | CIDR | Network Address | Broadcast |
|---|---|---|---|---|
| G | 90,123 | /15 | 27.128.0.0 | 27.129.255.255 |
| I | 89,012 | /15 | 27.130.0.0 | 27.131.255.255 |
| B | 78,901 | /15 | 27.132.0.0 | 27.133.255.255 |
| K | 78,901 | /15 | 27.134.0.0 | 27.135.255.255 |
| D | 67,890 | /15 | 27.136.0.0 | 27.137.255.255 |
| F | 56,789 | /16 | 27.138.0.0 | 27.138.255.255 |
| H | 45,678 | /16 | 27.139.0.0 | 27.139.255.255 |
| A | 34,567 | /16 | 27.140.0.0 | 27.140.255.255 |
| J | 34,567 | /16 | 27.141.0.0 | 27.141.255.255 |
| C | 23,456 | /17 | 27.142.0.0 | 27.142.127.255 |
| E | 12,345 | /18 | 27.142.128.0 | 27.142.191.255 |

All WAN links use `/30` subnets allocated sequentially starting at `27.142.192.0/30`. Full WAN allocation table is in the report.

## Route Redistribution

Performed at three boundary routers to give every host visibility into every other domain:

| Boundary | Between | Direction |
|---|---|---|
| R5 | OSPF Area 1 ↔ EIGRP AS 5 | bidirectional |
| R13 | EIGRP AS 5 ↔ OSPF Area 2 | bidirectional |
| R21 | OSPF Area 2 ↔ RIP v2 | bidirectional |

EIGRP-side redistribution uses seed metric `1544 2000 255 1 1500` (bandwidth, delay, reliability, load, MTU). OSPF-side uses the `subnets` keyword to preserve subnetted external routes.

## Services

### Centralized DHCP (Network K, R19)
- **Server IP:** `27.134.0.2`
- **Pools:** 8 (A, B, C, D, E, F, J, K) — G and I are statically addressed
- `ip helper-address 27.134.0.2` is configured on every LAN-facing interface across the four domains to relay broadcast DHCP requests as unicast

### NAT / PAT (Router 20, Network J)
- **Inside network:** `27.141.0.0/16`
- **Outside (public) IP:** `198.244.120.212` (configured on Loopback0)
- **Translation type:** PAT (overload)
- The public IP is redistributed into RIP via `redistribute connected metric 1` so all routers across all domains have a return route for de-translation

### Mail Server (Network C)
- **Server IP:** `27.142.0.2`
- **Services:** SMTP + POP3
- **Domain:** `24i2119.net`
- All hosts in OSPF Area 1 (Networks A, B, C) are configured as mail clients

## Security: ACL

Extended ACL **100** on R16, applied outbound on `Fa0/0` (interface facing Network H):

| Rule | Action | Source | Destination |
|---|---|---|---|
| 1 | DENY | host `27.140.0.3` (specific host in Net A) | Web Server `27.139.0.2` |
| 2 | DENY | `27.136.0.0/15` (entire Network D) | Web Server `27.139.0.2` |
| 3 | PERMIT | any | any |

Outbound placement on R16 stops the traffic as close to the destination as possible.

## Testing and Verification

| Test | Source | Destination | Expected | Result |
|---|---|---|---|---|
| 1 | PC12 (Net J) | Laptop2 (Net A) | Success via NAT | ✅ PASS |
| 2 | Laptop2 (Net A) | PC12 (Net J) | Success | ✅ PASS |
| 3 | Host in Net D | Web Server (H) | BLOCKED by ACL | ✅ PASS |
| 4 | `27.140.0.3` (Net A) | Web Server (H) | BLOCKED by ACL | ✅ PASS |
| 5 | Host in Net B | Web Server (H) | Success | ✅ PASS |
| 6 | Host in Net G | Host in Net K | Success | ✅ PASS |
| 7 | DHCP Client | DHCP Server | IP Assigned | ✅ PASS |
| 8 | Laptop0 | Laptop2 | Email Received | ✅ PASS |

Verification commands used: `show ip route`, `show ip ospf neighbor`, `show ip eigrp neighbors`, `show ip nat translations`, `show access-lists`, `show ip dhcp binding`.

## Files in This Repository

```
.
├── README.md
├── LICENSE
├── report/
│   └── CN-Project-Report.pdf
├── packet-tracer/
│   └── Enterprise-Network.pkt
└── screenshots/
    ├── topology.png
    ├── ospf-routing-table-r3.png
    ├── eigrp-routing-table-r9.png
    ├── rip-routing-table-r20.png
    ├── nat-translations-r20.png
    ├── acl-block-test.png
    ├── dhcp-pools.png
    └── mail-server.png
```

## How to Run

1. Install **Cisco Packet Tracer 8.2 (Instructor Version)** or newer.
2. Open `packet-tracer/Enterprise-Network.pkt`.
3. Wait for routing protocols to converge (about 30–60 seconds on first load).
4. Verify with the commands listed under [Testing and Verification](#testing-and-verification).
5. The full design rationale, addressing tables, and configuration details are in `report/CN-Project-Report.pdf`.

## Author

**Muhammad Usman**
BS Cyber Security, FAST NUCES
Roll No: 24I-2119

If you found this useful or want to discuss network design, feel free to connect on LinkedIn.

## License

This project is released under the MIT License. See [LICENSE](LICENSE) for details.

Educational use is encouraged. If you reference this work for a similar coursework project, please cite it rather than copy it — your own design decisions are what make the learning stick.
