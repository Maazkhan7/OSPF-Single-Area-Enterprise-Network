# 🏢 Cisco Lab: OSPF Single-Area Enterprise Network — Project 117

> **Tools:** Cisco Packet Tracer · OSPF Area 0 · Router-on-a-Stick · VLANs · DHCP Server · HTTP Server · DNS Server · IT-ROOM · SERVER-ROOM · Multi-Router Topology

---

## 📌 Project Overview

**Project** is a complete enterprise-grade network simulation featuring **OSPF (Open Shortest Path First)** as the single dynamic routing protocol across a multi-site topology. The design includes two branch offices (**BRANCH-A** and **BRANCH-B**), a centralized **ISP router**, a dedicated **SERVER-ROOM** with three servers (DHCP, HTTP, DNS), and a special **IT-ROOM** management segment — all interconnected through a chain of routers in a single **OSPF Area 0**.

Every subnet — from branch VLANs to server segments — is advertised into OSPF using wildcard masks. OSPF neighbor adjacencies form over serial point-to-point links, and all routers converge to a complete topology view. End-to-end reachability is verified with pings and DNS-resolved web browsing.

---

## 🔴 OSPF Deep Dive — All Stages Explained

> This section covers every stage OSPF goes through — from the moment a router boots up to the final state where all routes are installed in the routing table. Understanding these stages is essential for CCNA and real-world network troubleshooting.

---

### 📚 What is OSPF?

**OSPF (Open Shortest Path First)** is a **link-state**, **classless**, **open-standard** interior gateway protocol (IGP) defined in RFC 2328. Unlike RIP which shares routing tables, OSPF routers share information about the **state of their links** — building a complete map of the network called the **LSDB (Link State Database)**. Every router then independently runs **Dijkstra's SPF algorithm** to calculate the shortest (lowest cost) path to every destination.

| Feature | OSPF | RIP V2 |
|---|---|---|
| Type | Link-State | Distance-Vector |
| Metric | Cost (bandwidth-based) | Hop Count |
| Admin Distance | 110 | 120 |
| Convergence | Fast (triggered updates) | Slow (30-sec periodic) |
| Scalability | Large networks | Small networks |
| Algorithm | Dijkstra SPF | Bellman-Ford |
| Updates | LSA flooding (event-driven) | Full table broadcast |
| Max hops | Unlimited | 15 hops |

---

### 🔄 OSPF 7-Stage Operation — Step by Step

OSPF routers go through **7 distinct states** before they can exchange routing information. In this lab, you can observe these states on every serial link between the ISP, BRANCH-A, and BRANCH-B routers.

---

#### ✅ Stage 1 — DOWN State

```
State: DOWN
```

This is the initial state. The router has OSPF configured but has **not yet sent or received any Hello packets** on the interface. The interface is either administratively down or OSPF has just been enabled.

**In this lab:** When we first configured `router ospf 1` on the ISP router and added `network` statements, all serial interfaces begin in DOWN state before any Hello is sent.

```cisco
! OSPF just configured — router is in DOWN state on Serial2/0
ISP(config)#router ospf 1
ISP(config-router)#network 1.1.1.0 0.0.0.255 area 0
! Serial2/0 enters OSPF but neighbor not yet discovered
```

---

#### ✅ Stage 2 — INIT State

```
State: INIT
```

The router has **sent a Hello packet** and is **waiting for a response**. It may have received a Hello from a neighbor, but that neighbor's Hello did **not yet include this router's Router ID** — meaning the neighbor has not acknowledged this router yet.

**Hello packet contains:**
- Router ID (highest IP or loopback IP)
- Area ID (must match — Area 0 in this lab)
- Hello/Dead intervals (must match — Hello 10, Dead 40)
- Authentication (if configured)
- Subnet mask (must match on broadcast links)
- Neighbor list (empty at first)

**In this lab:** Serial2/0 on ISP sends Hello to BRANCH-A. BRANCH-A receives it but has not yet replied with ISP's Router ID in its own Hello.

```
%OSPF-5-ADJCHG: Process 1, Nbr 10.1.1.1 on Serial2/0 from LOADING to INIT
```

---

#### ✅ Stage 3 — 2-WAY State

```
State: 2-WAY
```

Both routers have seen each other's Router ID in the Hello packets. This is **bidirectional communication confirmed**. The routers have verified:
- Same Area ID ✅
- Matching Hello/Dead timers ✅
- Same subnet ✅
- Authentication (if any) ✅

On **broadcast networks** (like Ethernet), a **DR (Designated Router)** and **BDR (Backup Designated Router)** election happens here. On **point-to-point serial links** (like in this lab), there is NO DR/BDR election — routers go directly to the next stage.

**In this lab:** Serial links between ISP ↔ BRANCH-A and ISP ↔ BRANCH-B are point-to-point, so DR/BDR election is skipped entirely. This is confirmed in:

```
show ip ospf interface serial 2/0
→ Network Type POINT-TO-POINT
→ No DR/BDR (shown as FULL/- in neighbor table)
```

---

#### ✅ Stage 4 — EXSTART State

```
State: EXSTART
```

Routers negotiate who will be the **Master** and who will be the **Slave** for the database exchange process. This is determined by **Router ID** — the higher Router ID becomes Master.

The Master controls the sequence numbers for Database Description (DBD) packets that follow.

**Router ID determination:**
1. Manually configured `router-id` command (highest priority)
2. Highest IP on a **loopback** interface
3. Highest IP on any **active physical** interface

**In this lab:**
- ISP Router ID = 2.1.1.1 (highest serial IP)
- BRANCH-A Router ID = 10.1.1.1
- BRANCH-B Router ID = 20.1.1.1

The router with the higher Router ID on each link becomes Master for that exchange.

---

#### ✅ Stage 5 — EXCHANGE State

```
State: EXCHANGE
```

Routers exchange **DBD (Database Description) packets** — these are summaries of their LSDB. Think of it like exchanging a **table of contents** rather than the full book.

Each DBD packet contains:
- LSA headers (Link ID, Advertising Router, Sequence number, Age)
- NOT the full LSA content — just the headers

After receiving DBDs, each router compares them against its own LSDB and creates a **Link State Request (LSR) list** — a list of LSAs it needs that the neighbor has.

**In this lab (from `show ip ospf database`):**

```
Router Link States (Area 0):
Link ID       ADV Router    Seq#        Link count
80.1.1.1      80.1.1.1      0x80000005  4
20.1.1.1      20.1.1.1      0x80000004  3
192.168.4.1   192.168.4.1   0x80000006  4
2.1.1.1       2.1.1.1       0x80000004  4     ← ISP's own LSA
10.1.1.1      10.1.1.1      0x80000004  3
192.168.2.1   192.168.2.1   0x80000006  4
100.1.1.1     100.1.1.1     0x80000003  2
```

All 7 routers' LSAs present — EXCHANGE was successful.

---

#### ✅ Stage 6 — LOADING State

```
State: LOADING
```

Routers send **LSR (Link State Request)** packets for any LSAs they are missing, and respond with **LSU (Link State Update)** packets containing the full LSA details. The receiving router acknowledges each LSU with an **LSAck (Link State Acknowledgment)**.

**OSPF Packet Types Summary:**

| Type | Packet Name | Purpose |
|------|-------------|---------|
| 1 | Hello | Neighbor discovery and keepalive |
| 2 | DBD (Database Description) | LSDB summary exchange |
| 3 | LSR (Link State Request) | Request specific LSAs |
| 4 | LSU (Link State Update) | Send full LSA content |
| 5 | LSAck (Link State Acknowledgment) | Confirm LSA receipt |

After all LSUs are received and acknowledged, the LSDB is complete and identical on both routers. The router then runs **Dijkstra's SPF algorithm** to calculate the best path to every network.

---

#### ✅ Stage 7 — FULL State ← THE GOAL

```
State: FULL
```

This is the final and desired state. Both routers have **identical LSDBs** and have calculated their routing tables using SPF. Routes are installed with the prefix `O` in the routing table.

**In this lab — ISP `show ip ospf neighbor`:**

```
ISP#show ip ospf neighbor

Neighbor ID    Pri    State       Dead Time    Address      Interface
10.1.1.1        0     FULL/ -     00:00:32     1.1.1.1      Serial2/0  ✅
20.1.1.1        0     FULL/ -     00:00:38     2.1.1.2      Serial3/0  ✅
```

`FULL/ -` means:
- **FULL** = adjacency complete, LSDBs synchronized
- **-** = no DR/BDR (point-to-point link, none elected)

Both neighbors FULL — network fully converged. ✅

---

### 🔁 OSPF Hello & Dead Timers

OSPF uses Hello packets as **keepalives** to maintain neighbor relationships. Once FULL state is reached:

```
show ip ospf interface serial 2/0
→ Hello 10 sec, Dead 40 sec (4x Hello)
```

| Timer | Value | Meaning |
|---|---|---|
| Hello Interval | 10 sec | How often Hello packets are sent |
| Dead Interval | 40 sec | How long to wait before declaring neighbor down |
| Wait Timer | 40 sec | How long to wait before DR/BDR election on broadcast |
| Retransmit | 5 sec | How long to wait before retransmitting unacked LSAs |

If the Dead timer expires (no Hello received in 40 sec), the neighbor goes back to **DOWN** state and LSAs are re-flooded.

---

### 📐 OSPF Cost Calculation

OSPF uses **cost** as its metric. Cost is calculated per interface:

```
Cost = Reference Bandwidth / Interface Bandwidth
Default Reference Bandwidth = 100 Mbps (100,000,000 bps)
```

| Interface Type | Bandwidth | Cost |
|---|---|---|
| Serial (T1) | 1.544 Mbps | 64 |
| FastEthernet | 100 Mbps | 1 |
| GigabitEthernet | 1000 Mbps | 1 |
| Ethernet | 10 Mbps | 10 |

In this lab, all WAN links are serial:
```
Cost per serial link = 100,000,000 / 1,544,000 ≈ 64
```

**Route cost example from ISP:**
```
O  192.168.1.0 [110/66] via 1.1.1.1, Serial2/0
```
- Cost 66 = 64 (ISP Serial2/0 outgoing) + 1 (BRANCH-A Fa subinterface) + 1 (switch/host)
- Each hop adds the **outgoing** interface cost

---

### 🗂️ OSPF LSA Types (Area 0 — this lab)

| LSA Type | Name | Generated By | Purpose |
|---|---|---|---|
| Type 1 | Router LSA | Every router | Describes router's own links and states |
| Type 2 | Network LSA | DR (on broadcast) | Describes all routers on a multi-access segment |
| Type 3 | Summary LSA | ABR | Advertises routes between areas |
| Type 4 | ASBR Summary | ABR | Points to ASBR location |
| Type 5 | External LSA | ASBR | External routes redistributed into OSPF |

**In this lab:** Only **Type 1 (Router LSA)** and **Type 2 (Network LSA)** are generated — single area, no ABR, no redistribution.

```
show ip ospf database
→ Router Link States (Area 0)  ← Type 1 LSAs from all 7 routers
→ Net Link States (Area 0)     ← Type 2 LSAs from DR on multi-access segments
```

---

### 🔍 OSPF Verification Command Reference

```cisco
! Check all OSPF-learned routes
show ip route ospf

! Full routing table (C = connected, O = OSPF)
show ip route

! Verify neighbor adjacency states
show ip ospf neighbor

! View complete Link State Database
show ip ospf database

! Detailed interface OSPF parameters (timers, cost, neighbor)
show ip ospf interface serial 2/0

! Summary of all OSPF-enabled interfaces
show ip ospf interface brief

! OSPF process details (Router ID, area count, SPF runs)
show ip ospf

! Filter routing table for OSPF only
show ip route ospf 1
```

---

## 🗺️ Full Network Topology

```
                          ┌─────────────────────────────────────┐
                          │           IT-ROOM (PC4)             │
                          │         100.1.1.0/24                │
                          └──────────────┬──────────────────────┘
                                         │ Router4
                                    50.1.1.0/24
                                         │
                                    ┌────┴────┐
                           10.1.1.0 │ BRANCH-A│ Router (BRANCH-A)
                          ──────────┤         ├──────────
                         Router2    └────┬────┘
                              │          │
                         ┌────┴────┐  Switch0     Switch1
                         │  ISP    │   │               │
                         │Router0  │  PC0(VLAN10)  PC1(VLAN20)
                         └────┬────┘  192.168.1.0   192.168.2.0
                         2.1.1.0/24
                              │
                         ┌────┴────┐
                         │BRANCH-B │ Router (BRANCH-B)
                         │         ├──────────
                         └────┬────┘  20.1.1.0/24
                              │        Router1 → Router6 → SERVER-ROOM
                           Switch2   Switch3        Switch4
                              │         │              │
                          PC2(VLAN30) PC3(VLAN40)  [DHCP][HTTP][DNS]
                          192.168.3.0  192.168.4.0

SERVER-ROOM Segment:
├── DHCP-SERVER  (Server0) → 60.1.1.0/24
├── HTTP-SERVER  (Server1) → 70.1.1.0/24
└── DNS-SERVER   (Server2) → 80.1.1.0/24
```

### Complete IP Addressing Table

| Device        | Interface / Segment  | IP Address       | Role                        |
|---------------|----------------------|------------------|-----------------------------|
| ISP (Router0) | Serial 2/0           | 1.1.1.2/24       | Link to BRANCH-A side       |
| ISP (Router0) | Serial 3/0           | 2.1.1.2/24       | Link to BRANCH-B side       |
| BRANCH-A      | Fa0/0.10             | 192.168.1.1/24   | VLAN 10 gateway             |
| BRANCH-A      | Fa0/0.20             | 192.168.2.1/24   | VLAN 20 gateway             |
| BRANCH-A      | Serial (WAN)         | 1.1.1.1/24       | WAN to ISP                  |
| BRANCH-A      | Link to Router2      | 10.1.1.0/24      | Internal chain              |
| BRANCH-B      | Fa0/0.30             | 192.168.3.1/24   | VLAN 30 gateway             |
| BRANCH-B      | Fa0/0.40             | 192.168.4.1/24   | VLAN 40 gateway             |
| BRANCH-B      | Serial (WAN)         | 2.1.1.1/24       | WAN to ISP                  |
| BRANCH-B      | Link to Router1      | 20.1.1.0/24      | Internal chain              |
| Router2       | Link segment         | 10.1.1.0/24      | BRANCH-A ↔ IT-ROOM path     |
| Router4       | IT-ROOM side         | 50.1.1.0/24      | IT-ROOM link                |
| IT-ROOM (PC4) | NIC                  | 100.1.1.x/24     | Management PC               |
| Router6       | SERVER-ROOM side     | 30.1.1.0/24      | Link before server room     |
| SERVER-ROOM   | Internal             | 60.1.1.0/24      | DHCP server subnet          |
| HTTP-SERVER   | FastEthernet         | 70.1.1.2/24      | Web server (www.cisco.com)  |
| DNS-SERVER    | FastEthernet         | 80.1.1.2/24      | DNS (A record resolver)     |
| DHCP-SERVER   | FastEthernet         | 60.1.1.2/24      | DHCP for all 4 VLANs        |

---

## ⚙️ Full Configuration

### 🔴 OSPF — Open Shortest Path First (Area 0)

> **Why OSPF over RIP?**
> OSPF uses **cost** (based on bandwidth) as its metric — not hop count. It converges faster, supports large networks, uses LSAs (Link State Advertisements) instead of periodic full table broadcasts, and forms neighbor adjacencies via Hello packets. This lab uses **Process ID 1**, **single Area 0** (backbone area) across all routers.

#### ISP Router (Router0) — OSPF

```cisco
Router(config)#router ospf 1
Router(config-router)#network 2.1.1.0 0.0.0.255 area 0
Router(config-router)#network 1.1.1.0 0.0.0.255 area 0
```

#### BRANCH-A Router — OSPF

```cisco
Router(config)#router ospf 1
Router(config-router)#network 10.1.1.0 0.0.0.255 area 0
Router(config-router)#network 192.168.1.0 0.0.0.255 area 0
Router(config-router)#network 192.168.2.0 0.0.0.255 area 0
Router(config-router)#network 50.1.1.0 0.0.0.255 area 0
```

#### BRANCH-B Router — OSPF

```cisco
BRANCH-2(config)#router ospf 1
BRANCH-2(config-router)#network 192.168.3.0 0.0.0.255 area 0
BRANCH-2(config-router)#network 192.168.4.0 0.0.0.255 area 0
BRANCH-2(config-router)#network 30.1.1.0 0.0.0.255 area 0
BRANCH-2(config-router)#network 20.1.1.0 0.0.0.255 area 0
```

#### SERVER-ROOM Router — OSPF

```cisco
SERVER-ROOM(config)#router ospf 1
SERVER-ROOM(config-router)#network 60.1.1.0 0.0.0.255 area 0
SERVER-ROOM(config-router)#network 70.1.1.0 0.0.0.255 area 0
SERVER-ROOM(config-router)#network 80.1.1.0 0.0.0.255 area 0
SERVER-ROOM(config-router)#network 30.1.1.0 0.0.0.255 area 0
```

---

### 📊 OSPF Verification — ISP Router (Router0)

#### `show ip route` — Full Table

```
C    1.1.1.0  directly connected, Serial2/0
C    2.1.1.0  directly connected, Serial3/0
O    10.1.1.0  [110/65] via 1.1.1.1, Serial2/0     ← BRANCH-A internal
O    20.1.1.0  [110/65] via 2.1.1.2, Serial3/0     ← BRANCH-B internal
O    30.1.1.0  [110/66] via 2.1.1.2, Serial3/0     ← toward SERVER-ROOM
O    50.1.1.0  [110/66] via 1.1.1.1, Serial2/0     ← toward IT-ROOM
O    60.1.1.0  [110/67] via 2.1.1.2, Serial3/0     ← DHCP Server
O    70.1.1.0  [110/67] via 2.1.1.2, Serial3/0     ← HTTP Server
O    80.1.1.0  [110/67] via 2.1.1.2, Serial3/0     ← DNS Server
O    100.1.1.0 [110/67] via 1.1.1.1, Serial2/0     ← IT-ROOM
O    192.168.1.0 [110/66] via 1.1.1.1, Serial2/0   ← VLAN 10
O    192.168.2.0 [110/66] via 1.1.1.1, Serial2/0   ← VLAN 20
O    192.168.3.0 [110/66] via 2.1.1.2, Serial3/0   ← VLAN 30
O    192.168.4.0 [110/66] via 2.1.1.2, Serial3/0   ← VLAN 40
```

> **Metric [110/65]** = AD 110 (OSPF), Cost 65 (serial link default cost)
> Cost increases by 1 per router hop → [110/66] = 2 hops, [110/67] = 3 hops

#### `show ip route ospf 1` — OSPF-Only Routes

```
O    10.1.1.0  [110/65]  via 1.1.1.1, Serial2/0
O    20.1.1.0  [110/65]  via 2.1.1.2, Serial3/0
O    30.1.1.0  [110/66]  via 2.1.1.2, Serial3/0
O    50.1.1.0  [110/66]  via 1.1.1.1, Serial2/0
O    60.1.1.0  [110/67]  via 2.1.1.2, Serial3/0
O    70.1.1.0  [110/67]  via 2.1.1.2, Serial3/0
O    80.1.1.0  [110/67]  via 2.1.1.2, Serial3/0
O    100.1.1.0 [110/67]  via 1.1.1.1, Serial2/0
O    192.168.1.0 [110/66] via 1.1.1.1, Serial2/0
O    192.168.2.0 [110/66] via 1.1.1.1, Serial2/0
O    192.168.3.0 [110/66] via 2.1.1.2, Serial3/0
O    192.168.4.0 [110/66] via 2.1.1.2, Serial3/0
```

#### `show ip ospf neighbor` — Neighbor Adjacencies

```
ISP#show ip ospf neighbor

Neighbor ID    Pri    State       Dead Time    Address      Interface
10.1.1.1        0     FULL/ -     00:00:32     1.1.1.1      Serial2/0
20.1.1.1        0     FULL/ -     00:00:38     2.1.1.2      Serial3/0
```

Both neighbors in **FULL** state — OSPF adjacency complete, LSDBs synchronized. ✅

#### `show ip ospf database` — Link State Database

```
ISP#show ip ospf database

OSPF Router with ID (2.1.1.1) (Process ID 1)

Router Link States (Area 0):
Link ID       ADV Router    Age    Seq#        Checksum   Link count
80.1.1.1      80.1.1.1      934    0x80000005  0x00ff5a   4
20.1.1.1      20.1.1.1      862    0x80000004  0x006f38   3
192.168.4.1   192.168.4.1   862    0x80000006  0x007281   4
2.1.1.1       2.1.1.1       847    0x80000004  0x00fee6   4
10.1.1.1      10.1.1.1      782    0x80000004  0x00d7fa   3
192.168.2.1   192.168.2.1   731    0x80000006  0x0005e3   4
100.1.1.1     100.1.1.1     731    0x80000003  0x00138e   2

Net Link States (Area 0):
Link ID       ADV Router    Age    Seq#        Checksum
30.1.1.2      80.1.1.1      934    0x80000001  0x00dce6
20.1.1.2      192.168.4.1   862    0x80000001  0x00e66a
10.1.1.2      192.168.2.1   782    0x80000001  0x00dcdf
50.1.1.1      192.168.2.1   731    0x80000002  0x00e61c
```

All routers present in the LSDB — complete OSPF topology picture. ✅

#### `show ip ospf interface serial 2/0` — Interface Detail

```
Serial2/0 is up, line protocol is up
  Internet address is 1.1.1.2/24, Area 0
  Process ID 1, Router ID 2.1.1.1, Network Type POINT-TO-POINT, Cost: 64
  Transmit Delay is 1 sec, State POINT-TO-POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Hello due in 00:00:02
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 10.1.1.1
```

Hello timer = 10 sec, Dead timer = 40 sec (4x Hello). Point-to-point network type on serial links. ✅

---

### 🔷 BRANCH-A — Router-on-a-Stick (VLAN 10 & 20)

```cisco
! Physical interface — no IP, just up
BRANCH-A(config)#interface fastEthernet 0/0
BRANCH-A(config-if)#no ip address
BRANCH-A(config-if)#no shutdown

! VLAN 10 subinterface
BRANCH-A(config)#interface fastEthernet 0/0.10
BRANCH-A(config-subif)#encapsulation dot1Q 10
BRANCH-A(config-subif)#ip address 192.168.1.1 255.255.255.0
BRANCH-A(config-subif)#ip helper-address 60.1.1.2

! VLAN 20 subinterface
BRANCH-A(config)#interface fastEthernet 0/0.20
BRANCH-A(config-subif)#encapsulation dot1Q 20
BRANCH-A(config-subif)#ip address 192.168.2.1 255.255.255.0
BRANCH-A(config-subif)#ip helper-address 60.1.1.2
```

### 🔷 BRANCH-B — Router-on-a-Stick (VLAN 30 & 40)

```cisco
BRANCH-B(config)#interface fastEthernet 0/0.30
BRANCH-B(config-subif)#encapsulation dot1Q 30
BRANCH-B(config-subif)#ip address 192.168.3.1 255.255.255.0
BRANCH-B(config-subif)#ip helper-address 60.1.1.2

BRANCH-B(config)#interface fastEthernet 0/0.40
BRANCH-B(config-subif)#encapsulation dot1Q 40
BRANCH-B(config-subif)#ip address 192.168.4.1 255.255.255.0
BRANCH-B(config-subif)#ip helper-address 60.1.1.2
```

---

### 🌐 DHCP Server — Server0 (60.1.1.2)

Centralized DHCP server in the SERVER-ROOM serves all 4 VLANs across both branches via `ip helper-address` relay on every branch subinterface.

| Pool Name  | Default Gateway | DNS Server  | Start IP     | Subnet Mask   | Max Users |
|------------|-----------------|-------------|--------------|---------------|-----------|
| VLAN 10    | 192.168.1.1     | 80.1.1.2    | 192.168.1.1  | 255.255.255.0 | 255       |
| VLAN 20    | 192.168.2.1     | 80.1.1.2    | 192.168.2.1  | 255.255.255.0 | 255       |
| VLAN 30    | 192.168.3.1     | 80.1.1.2    | 192.168.3.1  | 255.255.255.0 | 255       |
| VLAN 40    | 192.168.4.1     | 80.1.1.2    | 192.168.4.1  | 255.255.255.0 | 255       |
| serverPool | —               | —           | 60.1.1.0     | 255.255.255.0 | 512       |

> **Key detail:** Every DHCP pool points the DNS server to `80.1.1.2` — this allows PCs to resolve `www.cisco.com` automatically via the DNS server.

---

### 🖥️ HTTP Server — Server1 (70.1.1.2)

The HTTP server hosts the company web portal titled **"Project No 117"** and is accessed via domain name `www.cisco.com` — resolved through the DNS server. Any PC in any VLAN can open a browser and reach the web page.

**Route path from PC1 (VLAN 20, BRANCH-A) to HTTP Server:**
```
PC1 → Switch1 → BRANCH-A (Fa0/0.20) → Router2 → ISP → Router1 → BRANCH-B → Router6 → SERVER-ROOM → 70.1.1.2
```

Multiple hops — OSPF handles every routing decision. ✅

---

### 🔍 DNS Server — Server2 (80.1.1.2)

The DNS server resolves the domain name `www.cisco.com` to the HTTP server IP `70.1.1.2`.

```
DNS Record:
  Name:    www.cisco.com
  Type:    A Record
  Address: 70.1.1.2
```

When a PC types `http://www.cisco.com` in the browser:
1. PC sends DNS query to 80.1.1.2 (from DHCP config)
2. DNS server replies: `www.cisco.com → 70.1.1.2`
3. PC sends HTTP GET to 70.1.1.2
4. Web page "Project No 117" loads ✅

---

### 🔒 Switch Configuration — VLANs + Trunk

```cisco
! BRANCH-A Switch0 — VLAN 10 & 20
Switch(config)#vlan 10
Switch(config-vlan)#name VLAN10
Switch(config)#vlan 20
Switch(config-vlan)#name VLAN20

Switch(config)#interface fastEthernet 0/24
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk allowed vlan 10,20

Switch(config)#interface fastEthernet 0/1
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 10

Switch(config)#interface fastEthernet 0/2
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 20
```

---

## ✅ End-to-End Verification

### Cross-Site Pings — from PC0 (VLAN 10, BRANCH-A)

```
C:\>ping 192.168.2.2    → PC1 (VLAN 20, same branch)
Reply from 192.168.2.2: bytes=32 time<1ms TTL=127 ✅

C:\>ping 192.168.3.2    → PC2 (VLAN 30, BRANCH-B) — 5 hops
Reply from 192.168.3.2: bytes=32 time=34ms TTL=123 ✅

C:\>ping 192.168.4.2    → PC3 (VLAN 40, BRANCH-B) — 5 hops
Reply from 192.168.4.2: bytes=32 time=42ms TTL=123 ✅
```

### Server Pings — from PC2 (VLAN 30, BRANCH-B)

```
C:\>ping 60.1.1.2    → DHCP Server
Reply from 60.1.1.2: bytes=32 time<1ms TTL=126 ✅ (0% loss)

C:\>ping 70.1.1.2    → HTTP Server
Reply from 70.1.1.2: bytes=32 time<1ms TTL=126 ✅

C:\>ping 80.1.1.2    → DNS Server
Reply from 80.1.1.2: bytes=32 time<1ms TTL=126 ✅
```

### DNS Resolution + Web Browsing — PC1

```
URL: http://www.cisco.com
DNS resolved: www.cisco.com → 70.1.1.2
Result: "Project No 117" web page loaded ✅
```

---

## 🧠 Key Concepts Demonstrated

| Feature | Implementation Detail |
|---|---|
| OSPF Area 0 | Single backbone area, Process ID 1, wildcard masks on all networks |
| OSPF Cost Metric | [110/65] = 1 cost, [110/66] = 2 costs, [110/67] = 3 costs — cost-based path selection |
| OSPF Neighbor FULL | Point-to-point serial links, Hello 10 / Dead 40 timers, no DR/BDR election |
| OSPF LSDB | All 7 routers present as Router LSAs in Area 0 |
| Router-on-a-Stick | dot1Q subinterfaces on BRANCH-A (VLAN 10/20) and BRANCH-B (VLAN 30/40) |
| DHCP Relay | `ip helper-address 60.1.1.2` on all 4 subinterfaces → single server, 4 VLANs |
| DNS Service | A Record: `www.cisco.com → 70.1.1.2` — name resolution across OSPF routes |
| HTTP Service | Web portal "Project No 117" accessible by domain name from any VLAN |
| IT-ROOM Isolation | Dedicated 100.1.1.0/24 management segment via Router4, advertised into OSPF |
| SERVER-ROOM | Dedicated zone: DHCP (60.x), HTTP (70.x), DNS (80.x) — all reachable via OSPF |
| Multi-Router Chain | 7 routers total — ISP, BRANCH-A, BRANCH-B, Router2, Router4, Router6, SERVER-ROOM |

---

## 📁 Repository Structure

```
📦 Project-117-OSPF-Enterprise/
├── 📄 README.md
├── 📁 screenshots/
│   ├── topology.png
│   ├── ospf-isp-show-ip-route.png
│   ├── ospf-isp-route-ospf-only.png
│   ├── ospf-lsdb.png
│   ├── ospf-neighbor-full.png
│   ├── ospf-interface-serial.png
│   ├── dhcp-server-pools.png
│   ├── dns-server-record.png
│   ├── web-browser-project117.png
│   ├── pc0-cross-branch-ping.png
│   └── pc2-server-ping.png
└── 📄 Project117.pkt
```

---

## 👤 Author

**Maaz Khan**
CCNA Certified · Network Engineer · NOC Engineer
📍 Lower Dir, KPK, Pakistan
🔗 [LinkedIn](https://linkedin.com/in/maazkhanms) 

---

> *"OSPF doesn't just find the path — it understands the entire map."*
