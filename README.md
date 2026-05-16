# BADASS — BGP At Doors of Autonomous Systems is Simple

A network administration project that simulates and configures BGP EVPN-based data center networking using GNS3 and Docker. Covers VXLAN overlay networks and BGP with EVPN route reflection across three progressive parts.

---

## Overview

| Part | Topic | Key Technology |
|------|-------|----------------|
| P1 | GNS3 + Docker setup | FRRouting (zebra, bgpd, ospfd, isisd), Alpine Linux |
| P2 | VXLAN discovery | VXLAN (VNI 10), static unicast + dynamic multicast (239.1.1.1) |
| P3 | BGP with EVPN | BGP EVPN (RFC 7432), OSPF underlay, Route Reflection |

---

## Repository Structure

```
.
├── P1/
│   ├── P1.gns3project       # GNS3 portable project (ZIP)
│   ├── Dockerfile_host      # Alpine-based host image (busybox, iproute2, tcpdump)
│   ├── Dockerfile_router    # FRRouting router image (bgpd, ospfd, isisd, zebra)
│   └── daemons.conf         # FRR daemon configuration
├── P2/
│   ├── P2.gns3project       # GNS3 portable project (ZIP)
│   ├── moouaamm-1_s         # Router 1 — static VXLAN config
│   ├── moouaamm-2_s         # Router 2 — static VXLAN config
│   ├── moouaamm-1_g         # Router 1 — multicast VXLAN config
│   ├── moouaamm-2_g         # Router 2 — multicast VXLAN config
│   ├── moouaamm_host-1      # Host 1 config
│   └── moouaamm_host-2      # Host 2 config
└── p3/
    ├── p3.gns3project       # GNS3 portable project (ZIP)
    ├── ychahbi-1            # Route Reflector (RR) config — OSPF + BGP RR
    ├── ychahbi-2            # VTEP leaf config
    ├── ychahbi-3            # VTEP leaf config
    ├── ychahbi-4            # VTEP leaf config
    ├── ychahbi-1_host       # Host behind VTEP 1
    ├── ychahbi-2_host       # Host behind VTEP 2
    └── ychahbi-3_host       # Host behind VTEP 3
```

---

## Part 1 — GNS3 Configuration with Docker

Two Docker images are built and used inside GNS3:

**Host image** (`Dockerfile_host`) — Alpine Linux with:
- `busybox-extras`, `iproute2`, `iputils`, `tcpdump`, `vim`

**Router image** (`Dockerfile_router`) — FRRouting v8.4.1 with daemons enabled:
- `zebra` — core routing engine
- `bgpd` — BGP daemon
- `ospfd` — OSPF daemon
- `isisd` — IS-IS routing engine

No IP addresses are configured by default; all addressing is done at runtime.

---

## Part 2 — Discovering VXLAN

A two-router topology using VXLAN ID (VNI) **10** over a bridge `br0`.

### Static Unicast Mode

Each router hardcodes the remote VTEP IP:

```sh
ip addr add 10.1.1.1/24 dev eth0

ip link add name vxlan10 type vxlan \
    id 10 dev eth0 \
    local 10.1.1.1 remote 10.1.1.2 \
    dstport 4789

ip link set dev vxlan10 up
ip link add br0 type bridge
ip link set dev br0 up
brctl addif br0 eth1
brctl addif br0 vxlan10
```

### Dynamic Multicast Mode

Instead of a static remote, a multicast group is used for flooding:

```sh
ip link add name vxlan10 type vxlan \
    id 10 dev eth0 \
    group 239.1.1.1 \
    dstport 4789
```

---

## Part 3 — BGP with EVPN

A small data center topology with one **Route Reflector (RR)** and three **VTEP leaves**, all in AS 1. OSPF is used as the underlay; BGP EVPN carries MAC/IP reachability (type-2 and type-3 routes) without MPLS.

### Addressing

| Node | Loopback | Role |
|------|----------|------|
| ychahbi-1 | 1.1.1.1/32 | Route Reflector |
| ychahbi-2 | 1.1.1.2/32 | VTEP (leaf) |
| ychahbi-3 | 1.1.1.3/32 | VTEP (leaf) |
| ychahbi-4 | 1.1.1.4/32 | VTEP (leaf) |

### Route Reflector FRR config (ychahbi-1)

```
router ospf
 ospf router-id 1.1.1.1
 network 1.1.1.1/32 area 0
 network 10.1.1.0/30 area 0
 network 10.1.1.4/30 area 0
 network 10.1.1.8/30 area 0

router bgp 1
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 bgp cluster-id 1.1.1.1
 neighbor 1.1.1.2 remote-as 1
 neighbor 1.1.1.2 update-source lo
 neighbor 1.1.1.3 remote-as 1
 neighbor 1.1.1.3 update-source lo
 neighbor 1.1.1.4 remote-as 1
 neighbor 1.1.1.4 update-source lo
 address-family l2vpn evpn
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 route-reflector-client
  neighbor 1.1.1.3 activate
  neighbor 1.1.1.3 route-reflector-client
  neighbor 1.1.1.4 activate
  neighbor 1.1.1.4 route-reflector-client
 exit-address-family
```

### How it works

- **Type-3 routes** (Inclusive Multicast Ethernet Tag) are pre-announced per VTEP at startup, advertising that a VTEP exists for VNI 10.
- **Type-2 routes** (MAC/IP Advertisement) are generated automatically when a host becomes active — no manual IP assignment needed. The RR reflects these to all other VTEPs.
- Verification: `ping` between hosts traverses the VXLAN tunnel; OSPF and ICMP packets are visible in captured traffic.

---

## Prerequisites

- A Linux virtual machine (mandatory)
- [GNS3](https://www.gns3.com/) installed and running
- [Docker](https://www.docker.com/) installed inside the VM
- FRRouting base image: `frrouting/frr:v8.4.1`

---

## Key Concepts

| Term | Description |
|------|-------------|
| **BGP** | Border Gateway Protocol (RFC 4271) — the Internet's routing protocol |
| **MP-BGP** | Multiprotocol BGP (RFC 4760) — carries NLRI for EVPN, VPNv4, etc. |
| **EVPN** | Ethernet VPN — BGP address family for MAC/IP reachability (RFC 7432) |
| **VXLAN** | Virtual Extensible LAN — L2 overlay over UDP/IP (RFC 7348) |
| **VNI** | VXLAN Network Identifier — equivalent to a VLAN ID in the overlay |
| **VTEP** | VXLAN Tunnel Endpoint — the device that encapsulates/decapsulates VXLAN frames |
| **Route Reflector** | BGP peer that reflects routes between iBGP clients, avoiding full mesh |
| **OSPF** | Open Shortest Path First — used here as the underlay IGP |

---

## References

- [RFC 4271 — BGP-4](https://datatracker.ietf.org/doc/html/rfc4271)
- [RFC 4760 — Multiprotocol Extensions for BGP](https://datatracker.ietf.org/doc/html/rfc4760)
- [RFC 7348 — VXLAN](https://datatracker.ietf.org/doc/html/rfc7348)
- [RFC 7432 — BGP MPLS-Based Ethernet VPN](https://datatracker.ietf.org/doc/html/rfc7432)
- [FRRouting Documentation](http://docs.frrouting.org/en/latest/setup.html)
