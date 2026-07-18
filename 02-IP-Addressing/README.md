# 02 — IP Addressing & VLAN Plan

## Table of Contents

1. [HQ VLANs & Subnets](#1-hq-vlans--subnets)
2. [HSRP IP Allocation](#2-hsrp-ip-allocation)
3. [Transit VLANs](#3-transit-vlans)
4. [Branch-01 IP Plan](#4-branch-01-ip-plan)
5. [Branch-02 IP Plan](#5-branch-02-ip-plan)
6. [IP Addressing Summary](#6-ip-addressing-summary)
---

## 1. HQ VLANs & Subnets

### User & Service VLANs

| VLAN ID | Name | Network | Gateway (HSRP VIP) | Purpose |
|---------|------|---------|------------------|---------|
| 10 | MGMT | 10.1.10.0/24 | 10.1.10.1 | Network management |
| 20 | SERVERS | 10.1.20.0/24 | 10.1.20.1 | Server farm |
| 30 | USERS | 10.1.30.0/24 | 10.1.30.1 | End-user workstations |
| 40 | SOC | 10.1.40.0/24 | 10.1.40.1 | Security Operations Center |
| 50 | DMZ | 10.1.50.0/24 | 10.1.50.1 | Demilitarized Zone |
| 60 | WIFI | 10.1.60.0/24 | 10.1.60.1 | Corporate WiFi |
| 61 | WIFI-GUEST | 10.1.61.0/24 | 10.1.61.1 | Guest WiFi (Internet only) |
| 70 | CCTV | 10.1.70.0/24 | 10.1.70.1 | Surveillance cameras |

### HQ IP Allocation per VLAN

| VLAN | VIP (.1) | Core-01 (.2) | Core-02 (.3) | Range |
|------|----------|--------------|--------------|-------|
| 10 | 10.1.10.1 | 10.1.10.2 | 10.1.10.3 | .10 - .254 |
| 20 | 10.1.20.1 | 10.1.20.2 | 10.1.20.3 | .10 - .254 |
| 30 | 10.1.30.1 | 10.1.30.2 | 10.1.30.3 | .10 - .254 |
| 40 | 10.1.40.1 | 10.1.40.2 | 10.1.40.3 | .10 - .254 |
| 50 | 10.1.50.1 | — | — | .10 - .254 |
| 60 | 10.1.60.1 | 10.1.60.2 | 10.1.60.3 | .10 - .254 |
| 61 | 10.1.61.1 | 10.1.61.2 | 10.1.61.3 | .10 - .254 |
| 70 | 10.1.70.1 | 10.1.70.2 | 10.1.70.3 | .10 - .254 |

> **Note:** VLAN 50 (DMZ) does **not** use HSRP. The FortiGate acts as the gateway for DMZ hosts.

---

## 2. HSRP IP Allocation

### HSRP Design (User VLANs Only)

| VLAN | Virtual IP | Core-01 (Active) | Core-02 (Standby) | Priority |
|------|------------|------------------|-------------------|----------|
| 10 | 10.1.10.1 | 10.1.10.2 | 10.1.10.3 | 110 / 100 |
| 20 | 10.1.20.1 | 10.1.20.2 | 10.1.20.3 | 110 / 100 |
| 30 | 10.1.30.1 | 10.1.30.2 | 10.1.30.3 | 110 / 100 |
| 40 | 10.1.40.1 | 10.1.40.2 | 10.1.40.3 | 110 / 100 |
| 60 | 10.1.60.1 | 10.1.60.2 | 10.1.60.3 | 110 / 100 |
| 61 | 10.1.61.1 | 10.1.61.2 | 10.1.61.3 | 110 / 100 |
| 70 | 10.1.70.1 | 10.1.70.2 | 10.1.70.3 | 110 / 100 |

### HSRP Features

- **Preempt enabled** — Active core reclaims gateway role after recovery
- **Priority-based failover** — Core-01 (110) vs Core-02 (100)
- **Virtual IP (.1)** — Used by all end devices as default gateway

---

## 3. Transit VLANs

### Core ↔ Firewall Transit

| VLAN ID | Name | Network | Core IP | FortiGate IP | Purpose |
|---------|------|---------|---------|--------------|---------|
| 99 | TRANSIT_FW_PORT5 | 10.1.99.0/24 | 10.1.99.2 (CORE-01) | 10.1.99.1 | Primary transit to FortiGate port5 |
| 98 | TRANSIT_FW_PORT4 | 10.1.98.0/24 | 10.1.98.2 (CORE-02) | 10.1.98.1 | Backup transit to FortiGate port4 |

> **Important:** HSRP is **NOT** used on VLAN 98/99. These are pure Layer-3 transit links.

### Default Routes on Core Switches

| Core Switch | Default Route | Next-Hop |
|-------------|---------------|----------|
| SW-HQ-CORE-01 | 0.0.0.0/0 | 10.1.99.1 (FortiGate port5) |
| SW-HQ-CORE-02 | 0.0.0.0/0 | 10.1.98.1 (FortiGate port4) |

### FortiGate LAN Summary Route

| Destination | Primary Next-Hop | Backup Next-Hop |
|-------------|------------------|-----------------|
| 10.1.0.0/16 | 10.1.99.2 via port5 (Distance: 10) | 10.1.98.2 via port4 (Distance: 20) |

---

## 4. Branch-01 IP Plan

| VLAN | Name | Subnet | Gateway (FortiGate) |
|------|------|--------|---------------------|
| 110 | BR1-CLIENT | 10.2.10.0/24 | 10.2.10.1 |
| 120 | BR1-MGMT | 10.2.20.0/24 | 10.2.20.1 |
| 130 | BR1-IT | 10.2.30.0/24 | 10.2.30.1 |
| 140 | BR1-MKT | 10.2.40.0/24 | 10.2.40.1 |

### VPN Tunnel IPs (HQ ↔ BR1)

| Tunnel | HQ Endpoint | BR1 Endpoint | Network |
|--------|-------------|--------------|---------|
| Tunnel 1 | 172.16.1.1 | 172.16.1.2 | /30 |
| Tunnel 2 | 172.16.2.1 | 172.16.2.2 | /30 |

### WAN IPs

| Site | ISP-WE | ISP-ORANGE |
|------|--------|------------|
| HQ | 192.168.1.10 | 192.168.218.10 |
| BR1 | 192.168.1.110 | 192.168.218.110 |

---

## 5. Branch-02 IP Plan

| VLAN | Name | Subnet | Gateway (FortiGate) |
|------|------|--------|---------------------|
| 210 | BR2-CLIENT | 10.3.10.0/24 | 10.3.10.1 |
| 220 | BR2-SERVERS | 10.3.20.0/24 | 10.3.20.1 |
| 230 | BR2-IT | 10.3.30.0/24 | 10.3.30.1 |
| 240 | BR2-MKT | 10.3.40.0/24 | 10.3.40.1 |
| 250 | BR2-MGMT | 10.3.50.0/24 | 10.3.50.1 |

---

## 6. IP Addressing Summary

### Summary Routes

| Site | Summary Network | Purpose |
|------|-----------------|---------|
| HQ | 10.1.0.0/16 | All HQ VLANs |
| BR1 | 10.2.0.0/16 | All BR1 VLANs |
| BR2 | 10.3.0.0/16 | All BR2 VLANs |

### DNS / AD Server IPs

| Server | IP Address | Site | Role |
|--------|------------|------|------|
| DC1-HQ | 10.1.20.10 | HQ | Primary Domain Controller |
| DC2-HQ | 10.1.20.11 | HQ | Secondary Domain Controller |
| DC1-BR1 | 10.2.20.10 | BR1 | Child Domain Controller |
| DC1-BR2 | 10.3.20.10 | BR2 | Child Domain Controller |

### SOC Infrastructure

| Component | IP Address | VLAN | Role |
|-----------|------------|------|------|
| Wazuh SIEM | 10.1.40.10 | 40 (SOC) | Log collection & analysis | 
| FortiAnalyzer | 10.1.40.11 | 40 (SOC) | Firewall log aggregation |
| SOC Server | 10.1.40.x | 40 (SOC) | Monitoring & dashboards |
