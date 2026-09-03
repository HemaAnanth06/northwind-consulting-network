# IP Addressing Plan — Northw
## Design Approach
VLSM (Variable Length Subnet Masking) was used instead of a single fixed 
mask, so each department's subnet is sized to its actual host count plus 
headroom — avoiding wasted address space. Subnets were allocated 
largest-to-smallest to keep block alignment clean.

## Addressing Table
Site	Department		Subnet		Mask	Usable Range	Broadcast
HQ	Sales (20)		10.10.0.0/27	.224	.1–.30		.31
HQ	Guest/Wireless (15)	10.10.0.32/27	.224	.33–.62		.63
HQ	IT/Management (10)	10.10.0.64/28	.240	.65–.78		.79
HQ	Servers (5)		10.10.0.80/29	.248	.81–.86		.87
Branch	Sales (10)		10.20.0.0/28	.240	.1–.14		.15
Branch	IT (5)			10.20.0.16/29	.248	.17–.22		.23

## Sizing Notes
- **Sales (20 hosts):** 5 host bits (/27) → 30 usable IPs, comfortable 
  headroom for growth.
- **Guest/Wireless (15 hosts):** 5 host bits (/27) → matches Sales' block 
  size since headroom needs were similar.
- **IT/Management (10 hosts):** 4 host bits (/28) → 14 usable IPs.
- **Servers (5 hosts):** deliberately right-sized to 3 host bits (/29) 
  instead of matching IT/Management's /28 — only 6 usable IPs needed, 
  so using a /28 here would have wasted 8 addresses.
- **Branch Sales (10 hosts):** 4 host bits (/28).
- **Branch IT (5 hosts):** right-sized to 3 host bits (/29), same 
  reasoning as HQ Servers.

## Notes
A dedicated /30 subnet for the HQ–Branch WAN link will be added in a 
later phase.
