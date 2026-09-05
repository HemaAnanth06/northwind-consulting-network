# Phase 4 Notes — Wireless Configuration (HQ Guest VLAN)

## What Was Built
An Access Point connected to a switch port on VLAN 20 (Guest/Wireless,
10.10.0.32/27), broadcasting SSID "Northwind-Guest" secured with
WPA2-PSK. A wireless client was connected to this SSID, statically
addressed at 10.10.0.34/27, gateway 10.10.0.33 (the router's
Gi0/0.20 sub-interface, already configured in Phase 3).

## Core Concept
An Access Point operates as a Layer 2 bridge for a single VLAN — it
extends that VLAN wirelessly, the same way a switch port extends it
over copper. It does not perform routing, does not span multiple
VLANs, and does not create a separate network. A wireless client
connected through the AP is functionally identical, from the
network's perspective, to a wired PC plugged into a VLAN 20 access
port.

## Full Path for Wireless Cross-VLAN Traffic
Wireless client → AP (VLAN 20 bridge) → switch access port (VLAN 20)
→ trunk link (already active from Phase 2) → router's Gi0/0.20
sub-interface (VLAN 20 gateway) → routing table lookup → destination
VLAN's sub-interface → destination device.

This is the exact same path used by wired VLAN 20 traffic in Phase 3
— wireless did not introduce a new mechanism, it only introduced a
new physical entry point onto an already fully-routed VLAN.

## Prediction vs. Actual — The Key Lesson From This Phase

**Prediction made before testing:**
- Wireless client → own gateway: succeed (same VLAN) — correct
- Wireless client → different-VLAN devices (Sales, MGMT): predicted
  **fail**, reasoning given was "trunking isn't happened"

**Actual result:**
- Wireless client → gateway (10.10.0.33): 4/4 replies, TTL=255
  (unchanged — same VLAN, no routing, as expected)
- Wireless client → Sales PC (10.10.0.2): 3/4 replies, TTL=127
  (decremented — routed), one initial timeout
- Wireless client → Servers PC (10.10.0.82): 3/4 replies, TTL=127,
  one initial timeout

Both cross-VLAN pings **succeeded**, contradicting the prediction.

**Root cause of the wrong prediction:** the assumption that adding a
wireless client required *new* trunking/routing configuration. In
reality, the trunk between HQ-Switch-1 and HQ-Switch-2 (configured in
Phase 2) and the router's sub-interfaces (configured in Phase 3) were
already fully active and VLAN 20-aware before the AP was even placed.
Wireless didn't need to be specially connected to that infrastructure
— it just needed to land on VLAN 20 like any other device, and
everything downstream was already built to handle it.

**Why this is worth remembering:** it's a common instinct to treat a
new device type (wireless) as if it implies a separate or lesser-
connected part of the network. The correct mental model is the
opposite — if the underlying VLAN/trunk/routing design is sound,
adding a new access method (wireless, in this case) requires no
additional cross-VLAN configuration at all. The design was already
correct; the new device simply inherited it.

## TTL Evidence (confirms the mechanism, not just the outcome)
| Destination | TTL | Interpretation |
|---|---|---|
| 10.10.0.33 (own gateway, VLAN 20) | 255 | Same VLAN, switched, no routing |
| 10.10.0.2 (Sales, VLAN 10) | 127 | Routed — crossed the HQ-Router |
| 10.10.0.82 (Servers, VLAN 40) | 127 | Routed — crossed the HQ-Router |

The repeated "first packet times out, rest succeed" pattern on both
cross-VLAN pings is consistent with ARP resolution delay, as
established in Phase 3 — not a fault.

## Critical Points to Keep Sharp
- Be able to state plainly: "An AP bridges Layer 2 for one VLAN; it
  does not route, and it does not change VLAN isolation rules."
- Be able to name the specific wrong assumption made in this phase
  and why it was wrong — this is a strong, honest interview story
  about correcting your own reasoning against real evidence rather
  than assuming a prediction was right.
- WPA2-PSK secures *who* can join the wireless network (authentication
  and encryption), but it has no bearing on VLAN membership or
  routing — those are governed entirely by which VLAN the AP's switch
  port belongs to.

## GitHub Artifacts for This Phase
- AP configuration (SSID, WPA2-PSK)
- Switch port assignment confirming VLAN 20 membership
- Wireless client static IP configuration
- Ping test results (this document) showing prediction vs. actual,
  with TTL-based evidence
