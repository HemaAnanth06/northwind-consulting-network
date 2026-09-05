# Northwind Consulting Group — Consolidated Technical Notes
## Phases 1-3: Addressing, VLANs, Inter-VLAN Routing

---

## PHASE 1: IP Addressing & VLSM Subnetting

### Core Concept
VLSM (Variable Length Subnet Masking) sizes each subnet to its actual
host requirement instead of using one fixed mask everywhere. This
avoids wasting address space — a critical distinction from older
fixed-length subnetting.

### The Method
1. List departments by host count, **largest to smallest**.
2. For each, find the smallest number of host bits that satisfies:
   `usable hosts = 2^(host bits) − 2` (subtract network + broadcast).
3. Remaining bits (32 − host bits) = the subnet mask.
4. **Block size** = 2^(host bits). Each subnet must start on a multiple
   of its own block size.
5. The next subnet starts immediately after the previous one's block
   ends — check alignment before assuming the address is valid.

### Final Addressing Table
| Site | Department | Subnet | Mask | Usable Range | Broadcast |
|---|---|---|---|---|---|
| HQ | Sales (20) | 10.10.0.0/27 | .224 | .1–.30 | .31 |
| HQ | Guest/Wireless (15) | 10.10.0.32/27 | .224 | .33–.62 | .63 |
| HQ | IT/Management (10) | 10.10.0.64/28 | .240 | .65–.78 | .79 |
| HQ | Servers (5) | 10.10.0.80/29 | .248 | .81–.86 | .87 |
| Branch | Sales (10) | 10.20.0.0/28 | .240 | .1–.14 | .15 |
| Branch | IT (5) | 10.20.0.16/29 | .248 | .17–.22 | .23 |

### Mistakes Made & Corrected
- **Confused /24 conventions with smaller masks** — initially used
  `.1` and `.254` (classic /24 pattern) for a /27 subnet, which was
  wrong. Every mask size has its own network/broadcast addresses
  determined by block size, not a fixed "last octet" habit.
- **Reported host count instead of the actual usable range** — when
  asked for "usable range," gave "30" (a count) instead of
  "10.10.0.1 to 10.10.0.30" (the actual addresses).
- **Off-by-one on broadcast address** — calculated broadcast as
  network + block size (e.g., 64+16=80) instead of network + block
  size − 1 (64+16−1=79). The broadcast address is always the *last*
  address in the block, not one past it.
- **Oversized the Servers and Branch IT subnets initially** — used
  the same /28 as a larger neighboring subnet instead of tightening
  to a /29. This defeats the purpose of VLSM: **right-size every
  subnet independently**, don't copy the previous subnet's size out
  of habit.

### Critical Points to Keep Sharp
- Be able to calculate any subnet **from scratch, on a whiteboard**,
  without a subnet calculator — this is a very common CCNA interview
  test.
- Be able to explain **why** VLSM saves space vs. fixed-length
  subnetting, with a concrete number (e.g., "using /28 everywhere
  would have wasted 8 addresses on the Servers subnet").
- Know the block-size alignment rule cold: subnets must start on
  multiples of their own block size, and allocation order
  (largest→smallest) exists specifically to minimize wasted space
  from misalignment.

---

## PHASE 2: VLANs & Trunking

### Core Concept
A VLAN is a **virtual/logical division of a switch** — it segments a
single physical switch into multiple isolated broadcast domains,
without needing separate physical hardware. This provides:
- **Security** (e.g., Guest traffic can't reach IT/Management)
- **Reduced broadcast congestion** (broadcasts stay within their VLAN)
- **Organizational clarity** (group by department/function)

### Why VLANs Isolate Traffic (the mechanism, not just the rule)
Communication at Layer 2 depends on MAC addresses, which are
discovered via ARP — and ARP is a broadcast. Switches only forward
broadcasts within the same VLAN. So even with correct IP
configuration, if the ARP broadcast can never leave VLAN 10, the
Sales PC can never even learn a Servers PC's MAC address in VLAN 40.
**The failure happens at Layer 2, before IP ever becomes relevant.**

### Access Ports vs. Trunk Ports
| | Access Port | Trunk Port |
|---|---|---|
| Purpose | Connects end devices (PCs, servers, APs) | Connects switch-to-switch (or switch-to-router) |
| VLANs carried | Exactly one | Multiple, tagged via 802.1Q |
| End device awareness | None — looks like a normal connection | N/A (not for end devices) |

### Per-Switch VLAN Databases
Each switch maintains its **own local VLAN database** by default.
Creating VLAN 10 on Switch-1 does **not** create it on Switch-2 —
you must explicitly configure the same VLANs on every switch that
needs them, unless using VTP (VLAN Trunking Protocol), which
synchronizes VLAN databases automatically (not used in this project,
by design, to keep VLAN control explicit and local).

### "Allowed" vs. "Active" on a Trunk (key distinction)
- **Allowed**: the trunk is technically permitted to carry that VLAN
  ID (default range is usually 1–1005).
- **Active**: the VLAN actually exists in that switch's local
  database and has member ports.
A trunk can *allow* a VLAN that isn't *active* anywhere — this is
exactly what happened when Switch-2 hadn't had its VLANs created yet.

### Mistakes Made & Corrected
- **No prior VLAN knowledge** — started from zero; required building
  the concept from the ground up (broadcast domains, ARP dependency)
  before any commands made sense. Good foundation now.
- **Assumed IP addressing alone enables cross-VLAN communication** —
  corrected: Layer 2 isolation blocks this regardless of valid IPs,
  because ARP (Layer 2) has to succeed before IP communication can
  even begin.
- **Accidentally trunked two ports on the same switch** (Fa0/1
  toward the router, Fa0/2 toward Switch-2) when only one link was
  intended to trunk at that stage. This created an apparent redundant
  path, and Spanning Tree Protocol automatically blocked one port to
  prevent a loop.
- **Forgot to create VLANs on the second switch** — trunk came up
  created there too.


### Troubleshooting Sequence (real, worth retelling in interviews)
1. Noticed `show interfaces trunk` showed one port forwarding, the

   other showing "none" (blocked).
2. Identified the blocked port was the one accidentally trunked
   toward the router — an unintended second path from the switch's

   perspective.

3. Reverted that port to access mode (matching the router's actual
   readiness at that time).

4. Confirmed the legitimate trunk immediately began forwarding all

   VLANs once the apparent loop condition was removed.


### Critical Points to Keep Sharp
- Be precise: say "different broadcast **domains**," not "different
  broadcast **addresses**" (a broadcast address is an IP-layer
  concept; broadcast domain is the Layer 2 concept being isolated).
- Know that STP blocking is **protective behavior**, not a fault —

  it's actively preventing a broadcast storm from a redundant path.
- Be able to explain switching vs. routing cleanly (see Phase 3
  section below — this distinction underlies the whole VLAN topic).

---

## PHASE 3: Inter-VLAN Routing (Router-on-a-Stick)

### Core Concept
A router is required to move traffic **between** VLANs, because
switches fundamentally cannot forward Layer 2 traffic across VLAN

boundaries. Router-on-a-stick uses **one physical interface**, split
into multiple **sub-interfaces** — each one logically dedicated to a
single VLAN via 802.1Q tagging — instead of requiring one physical
interface per VLAN.


**Key analogy worth keeping:** a router sub-interface is a logical
partition of a physical router interface, the same way a VLAN is a
logical partition of a physical switch. Both are examples of
splitting one physical resource into multiple logical ones.


### Switching vs. Routing (the foundational distinction)
| | Switching | Routing |

|---|---|---|
| Layer | Layer 2 (Data Link) | Layer 3 (Network) |
| Address used | MAC address | IP address |
| Table consulted | MAC address table | Routing table |
| Scope | Within one broadcast domain / VLAN | Between broadcast domains / VLANs |

**One-line version to have ready:** "Switching moves frames within a
broadcast domain using MAC addresses; routing moves packets between
broadcast domains using IP addresses and a routing table."

### Full Packet Path (Sales PC → Servers PC, cross-VLAN)
1. Sales PC compares destination IP against its own subnet mask —
   determines the destination is **not** in its own subnet.
2. Sales PC sends the packet to its **default gateway** (the router's
   VLAN 10 sub-interface IP).
3. Router receives the packet, checks its **routing table**, finds
   the destination subnet is directly connected via its VLAN 40
   sub-interface.
4. Router forwards the packet out that sub-interface, then ARPs (if
   not already cached) for the Servers PC's MAC address to complete
   delivery.

### Configuration Applied
```
interface gigabitEthernet0/0
 no shutdown
!
interface gigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.10.0.1 255.255.255.224
!
interface gigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.10.0.33 255.255.255.224
!
interface gigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 10.10.0.65 255.255.255.240
!
interface gigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 10.10.0.81 255.255.255.248
```

### Verification Results
- Sales PC (10.10.0.2) → Servers PC (10.10.0.82): 3/4 replies
  (1 initial timeout), TTL=127
- MGMT PC (10.10.0.66) → Servers PC (10.10.0.82): 4/4 replies,
  TTL=127

### Mistakes Made & Corrected
- **Trunked the router-facing port too early**, before the router had
  any VLAN-aware configuration — caused the Phase 2 STP-blocking
  issue. Correct sequencing: configure the router's sub-interfaces
  *conceptually* first, understand why trunking is needed, then trunk
  the link once the router side is ready to use it.
- **Jumped straight to testing instead of predicting first** — ran a
  ping before stating an expected outcome and reasoning. Prediction
  matters because it proves understanding, not just command
  execution — this is what interviewers are actually listening for
  when they ask "walk me through your troubleshooting."
- **Tested gateway reachability and called it "inter-VLAN routing
  working"** — pinging your own gateway (same subnet) only proves
  Layer 2 connectivity to the router, not that routing between VLANs
  is functioning. The real test requires a destination in a
  *different* VLAN's subnet.

### Critical Points to Keep Sharp
- **Why the first ping often times out:** ARP resolution (for the
  gateway's MAC, and later for the destination's MAC on the far side
  of the router) takes a moment before the first packet succeeds.
  Subsequent pings succeed quickly once MAC addresses are cached.
  This is a very common real troubleshooting/interview question.
- **TTL as a diagnostic signal:** TTL decrements by 1 per router hop.
  An unchanged TTL between two devices suggests they're on the same
  L2 segment (switched, not routed); a decremented TTL confirms the
  packet was actually routed. Useful for quickly telling switching
  from routing behavior just by reading ping output.
- Be able to state, unprompted, **why one physical interface can
  serve 4 VLANs** without 4 separate physical cables — this is the
  entire point of router-on-a-stick and a common follow-up question.

---

## Cross-Phase Themes to Be Ready to Discuss

1. **Layer 2 vs Layer 3 separation of concerns** — this single idea
   (switching handles delivery within a domain, routing handles
   crossing between domains) ties Phases 2 and 3 together and is
   likely to come up as a direct interview question in some form.
2. **Troubleshooting methodology over "it just worked"** — every
   mistake made in this project (STP block, oversized subnets,
   premature trunking) has a diagnosis story with a command that
   revealed the problem (`show interfaces trunk`, `show vlan brief`,
   `show ip interface brief`). These stories are stronger interview
   material than a clean first-try config would have been.
3. **Precision in terminology** — "broadcast domain" vs "broadcast
   address," "inter-VLAN routing" (a process) vs "inter-VLAN" (not a
   noun), "usable range" vs "host count." Small wording slips are
   worth catching now, since interviewers often listen for exact
   terminology as a proxy for depth of understanding.
