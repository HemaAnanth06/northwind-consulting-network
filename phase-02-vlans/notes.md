# Phase 2 Notes — VLANs & Trunking (HQ)

## Concepts Learned

### 1. Why VLANs isolate broadcast domains at Layer 2
A VLAN defines a broadcast domain at Layer 2. When a switch receives a
broadcast frame — like an ARP request — it only forwards that frame out
ports belonging to the *same* VLAN. It never crosses into another VLAN,
regardless of IP configuration. This matters because before two devices
can communicate using IP, they first need to resolve each other's MAC
address via ARP, and ARP itself relies on broadcast. If the broadcast
can't cross the VLAN boundary, the devices never even discover each
other's MAC address — so IP addressing becomes irrelevant; the failure
happens a layer below IP, at Layer 2.

### 2. Access ports vs. trunk ports
An access port belongs to exactly one VLAN and is used to connect end
devices — PCs, servers, access points. The device on that port has no
awareness that VLANs exist; it just sees a normal network connection.
A trunk port, by contrast, carries traffic for multiple VLANs over a
single physical link, using 802.1Q tagging to mark each frame with its
VLAN ID. Trunk ports are used between switches, or between a switch and
a router doing inter-VLAN routing, so that one physical cable can carry
many logically separate networks.

### 3. Why each switch has its own VLAN database
By default, VLAN configuration is local to each switch — there's no
automatic synchronization unless you explicitly configure something
like VTP (VLAN Trunking Protocol). So creating VLAN 10 on Switch-1 only
exists on Switch-1; Switch-2 has no idea it exists until you configure
it there too. I hit this directly — I created all four VLANs on my
first switch, trunked the link to my second switch, and the trunk came
up fine, but none of my VLANs were active on Switch-2 until I created
them there as well.

### 4. "Allowed" vs. "active" on a trunk
A trunk port, by default, is configured to *allow* a wide range of
VLANs — often 1 through 1005. But "allowed" just means the trunk is
technically permitted to carry that VLAN's traffic if it exists.
"Active" means the VLAN actually exists in that switch's local VLAN
database and has member ports. In my case, my trunk allowed VLANs
1–1005, but only VLAN 1 showed as active on Switch-2 — because I hadn't
created VLANs 10, 20, 30, 40 there yet. The trunk link was ready; the
VLANs simply didn't exist on that switch.

## Troubleshooting Log: Blocked Trunk Port

**What happened:** While setting up trunking between the two HQ
switches, I accidentally configured *two* ports on Switch-1 as trunks
instead of one — one toward the router (Fa0/1), one toward Switch-2
(Fa0/2). Since the router wasn't part of a VLAN-aware topology at that
point, this created what looked like a redundant path to Spanning Tree
Protocol, and STP automatically blocked the second port to prevent a
potential loop.

**How I diagnosed it:** Used `show interfaces trunk`, which showed one
port forwarding all VLANs and the other showing "none" — blocked.

**Root cause:** The router-facing port (Fa0/1) was mistakenly set to
trunk mode instead of staying as an access port.

**Fix:** Reverted Fa0/1 to `switchport mode access` /
`switchport access vlan 1`. The legitimate trunk on Fa0/2 immediately
came up and started forwarding all VLANs correctly.

**Lesson:** Spanning Tree's blocking behavior is protective, not a
bug — it actively prevents broadcast storms caused by redundant paths.
`show interfaces trunk` is the right tool to quickly separate "is the
trunk technically up" from "is it actually passing what I expect."

## Mistakes Made (and worth owning in an interview)

1. Trunked Fa0/1 (router-facing) by mistake instead of only Fa0/2 —
   misidentified which port went where when applying the trunk config.
2. Assumed VLANs created on one switch would exist everywhere —
   forgot VLAN databases are local per switch by default.

## Topics to Go Deeper On

- **Spanning Tree Protocol fundamentals** — root bridge election, port
  states (blocking/listening/learning/forwarding), why STP exists.
  Watched STP *act* in this phase, but haven't studied *how* it decides
  what to block yet.
- **VTP (VLAN Trunking Protocol)** — what it is, and that this project
  deliberately avoids it, keeping each switch's VLAN database
  independent by design.
- **802.1Q tagging mechanics** — what actually happens to a frame when
  tagged (a 4-byte tag inserted with VLAN ID).

## Final Verified State

**HQ-Switch-1**
```
VLAN Name     Status  Ports
10   SALES    active  Fa0/3
20   GUEST    active  Fa0/5
30   MGMT     active  Fa0/4
40   SERVERS  active  Fa0/6
```
Trunk: Fa0/2 → HQ-Switch-2, forwarding VLANs 1,10,20,30,40
Fa0/1 → HQ-Router, access mode, VLAN 1

**HQ-Switch-2**
```
VLAN Name     Status  Ports
10   SALES    active  Fa0/3
20   GUEST    active  Fa0/5
30   MGMT     active  Fa0/4
40   SERVERS  active  Fa0/6
```
Trunk: Fa0/1 → HQ-Switch-1, forwarding VLANs 1,10,20,30,40
