# Northwind Consulting Group — Network Overview

## Company Scenario
Northwind Consulting Group is a small consulting firm with a headquarters (HQ) office 
and one branch office, connected via a WAN link. The company provides consulting 
services and regularly hosts clients on-site to discuss and sign contracts, which 
requires a dedicated guest wireless network separate from internal company traffic.

**HQ Departments:**
- Sales — 20 hosts (client-facing staff)
- IT/Management — 10 hosts (network administration, elevated access)
- Servers — 5 hosts (internal file/application server resources)
- Guest/Wireless — 15 hosts (visiting clients applying for contracts; isolated from internal LAN)

**Branch Office Departments:**
- Sales — 10 hosts
- IT — 5 hosts

## Goals
This project demonstrates hands-on configuration and troubleshooting of enterprise 
networking devices, including how VLANs, routing, and addressing are implemented in 
a multi-site business network. It also builds practical understanding of how data 
packets traverse a network end-to-end — from a host, through switching and routing 
decisions, across a WAN link, to a destination — reflecting the kind of troubleshooting 
and configuration work expected of a Junior Network Engineer / NOC technician.

## Topology Summary
- **HQ:** 1 router, 2 switches, end devices across 4 departments (Sales, IT/Management, 
  Servers, Guest/Wireless)
- **Branch:** 1 router, 1 switch, end devices across 2 departments (Sales, IT)
- **Site connectivity:** HQ and Branch routers connected via a serial WAN link, 
  simulating a leased-line connection between sites
