# Zer0-Po!nT — Secure Enterprise Network Infrastructure

**Graduation Project — Full Technical Documentation**

---

## 1. Project Overview

**Zer0-Po!nT** is a security-focused company that provides **secure, highly available enterprise network infrastructure solutions**.

This project demonstrates the design and implementation of a **resilient HQ campus network** with:

- **High Availability (HA)**
- **No Single Point of Failure (NSPF)**
- **Centralized Security & Routing**
- **Scalability for future branches**
- **Enterprise-grade best practices**

The infrastructure is designed to support:

- SOC operations
- Secure internet access
- Centralized firewalling
- Future SD-WAN and branch connectivity

---

## 2. High-Level Architecture

The HQ network follows a **Dual-Core Campus Design** with centralized security at the firewall layer.

### Main Components

| Component | Quantity | Role |
|-----------|----------|------|
| Core Layer Switches (L3) | 2 | Inter-VLAN routing, HSRP |
| Access Layer Switches | Multiple | End-device connectivity |
| FortiGate Firewalls | 2 (HA Pair) | Perimeter security, SD-WAN |
| ISPs | 2 (WE & ORANGE) | Internet redundancy |

### Architecture Diagram

```
                    ┌─────────────┐     ┌─────────────┐
                    │   ISP-WE    │     │  ISP-ORANGE │
                    └──────┬──────┘     └──────┬──────┘
                           │                    │
                    ┌──────┴────────────────────┴──────┐
                    │      FortiGate HA Cluster        │
                    │    (Active/Passive)              │
                    └──────┬────────────────────┬──────┘
                           │                    │
              ┌─────────────┘                    └─────────────┐
              │                                              │
     ┌────────┴────────┐                            ┌────────┴────────┐
     │ SW-HQ-CORE-01  │◄─────Trunk (802.1Q)─────►│ SW-HQ-CORE-02  │
     │   (Active)     │                            │   (Standby)    │
     └────────┬────────┘                            └────────┬────────┘
              │                                              │
     ┌────────┴────────┬────────┬────────┬────────┬────────┐
     │  ACC-MGMT      │ACC-SRV │ACC-USER│ ACC-SOC│ACC-WIFI│ ...
     │   (VLAN 10)    │(VLAN20)│(VLAN30)│(VLAN40)│(VLAN60)│
     └─────────────────┴────────┴────────┴────────┴────────┘
```

---

## 3. Network Design Principles

| Principle | Implementation |
|-----------|----------------|
| High Availability | Dual Core, Firewall HA |
| No Single Point of Failure | Redundant links & devices |
| Layered Design | Core / Access |
| Centralized Routing | Inter-VLAN routing at Core |
| Secure Perimeter | FortiGate Firewalls |
| Scalability | VLAN-based & modular design |

---

## 4. Project Scope

### Completed Phases

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Core Layer (HSRP, STP, VLANs) | ✅ Complete |
| 2 | Access Layer (Trunks, Port Security) | ✅ Complete |
| 3 | Firewall Transit (Static Routes) | ✅ Complete |
| 4 | Internet Access (Static → SD-WAN) | ✅ Complete |
| 5 | Firewall HA | ✅ Complete |
| 6 | DMZ Setup | ✅ Complete |
| 7 | Branch-01 (BR1) — Full Stack | ✅ Complete |
| 8 | Branch-02 (BR2) — Full Stack | ✅ Complete |
| 9 | Site-to-Site VPN (IPSec) | ✅ Complete |
| 10 | Active Directory Integration | ✅ Complete |
| 11 | FSSO / LDAP | ✅ Complete |
| 12 | Security Profiles (HQ, BR1, BR2) | ✅ Complete |
| 13 | SOC / Wazuh SIEM | ✅ Complete |

---

## 5. Repository Structure

```
zer0-point/
├── 00-Project-Overview/
│   └── README.md
├── 01-Network-Architecture/
├── 02-IP-Addressing/
├── 03-Core-Layer/
├── 04-Access-Layer/
├── 05-Firewall/
├── 06-SD-WAN/
├── 07-VPN-Site-to-Site/
├── 08-DMZ/
├── 09-Active-Directory/
├── 10-SOC/
├── 11-Security-Profiles/
├── 12-HA-Redundancy/
├── 13-Testing-Validation/
└── 14-Appendix/
```

---

**Author:** Amer Ehab Amer  
**Project:** Zer0-Po!nT — Secure Enterprise Network Infrastructure  
**Institution:** Faculty of Electronic Engineering, Menoufia University  
**Year:** 2026
