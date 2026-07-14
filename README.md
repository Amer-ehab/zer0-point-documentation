# Zer0-Po!nT — Secure Enterprise Network Infrastructure


> **Graduation Project — Full Technical Documentation & Configuration**

---

## 📋 Project Overview

**Zer0-Po!nT** is a security-focused enterprise network infrastructure project demonstrating the design and implementation of a **resilient multi-branch campus network** with centralized security operations.

### Key Features

- ✅ **High Availability (HA)** — No single point of failure at any layer
- ✅ **Dual-Core Campus Design** — HSRP gateway redundancy
- ✅ **FortiGate HA Cluster** — Active/Passive firewall failover
- ✅ **SD-WAN** — Intelligent dual-ISP path selection
- ✅ **Site-to-Site VPN** — Redundant IPSec tunnels to all branches
- ✅ **Active Directory Integration** — LDAP + FSSO identity-based policies
- ✅ **Layered Security Profiles** — AV, IPS, Web Filter, App Control, DNS Filter, File Filter
- ✅ **SOC / SIEM** — Wazuh-based centralized monitoring and detection
- ✅ **DMZ** — Public-facing web server with strict security controls

---

## 🏗️ Architecture

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
              │
     ┌────────┴────────┐
     │   VPN Tunnels     │◄───── IPSec over SD-WAN ─────► Branch-01 & Branch-02
     └───────────────────┘
```

---

## 📁 Repository Structure

> Each phase folder that had build screenshots in the original project log also includes an `images/` subfolder with those screenshots embedded at the bottom of its `README.md` under **Screenshots**.

```
zer0-point/
├── 00-Project-Overview/          # Project introduction & high-level design
├── 01-Network-Architecture/      # Architecture diagrams & design principles
├── 02-IP-Addressing/             # VLAN plan, HSRP IPs, transit networks
├── 03-Core-Layer/                # Core switch configs (HQ + Branches)
│   ├── SW-HQ-CORE-01.txt
│   ├── SW-HQ-CORE-02.txt
│   ├── SW-BR1-CORE-01.txt
│   ├── SW-BR1-CORE-02.txt
│   ├── SW-BR2-CORE-01.txt
│   └── SW-BR2-CORE-02.txt
├── 04-Access-Layer/              # Access switch configs (HQ + Branches)
│   ├── access-switch-template.txt
│   ├── SW-HQ-ACC-*.txt
│   ├── SW-BR1-ACC-01.txt
│   └── SW-BR2-ACC-01.txt
├── 05-Firewall/                  # FortiGate configs (interfaces, routes, HA)
│   ├── FGT-HQ-interfaces.txt
│   ├── FGT-HQ-static-routes.txt
│   ├── FGT-HQ-link-monitor.txt
│   ├── FGT-HQ-sdwan.txt
│   ├── FGT-HQ-ha.txt
│   └── FGT-HQ-policy-internet.txt
├── 06-SD-WAN/                    # SD-WAN configuration & health checks
├── 07-VPN-Site-to-Site/          # IPSec VPN tunnel configs (HQ↔BR1, HQ↔BR2)
├── 08-DMZ/                       # DMZ design, NAT, publishing policies
├── 09-Active-Directory/          # AD forest, sites, GPOs, LDAP/FSSO integration
├── 10-SOC/                       # Wazuh SIEM deployment, detection rules
├── 11-Security-Profiles/         # FortiGate security profiles (HQ, BR1, BR2, DMZ)
├── 12-HA-Redundancy/             # Failover testing & redundancy verification
├── 13-Testing-Validation/        # Comprehensive test results & validation
└── 14-Appendix/                  # Naming conventions, commands, troubleshooting
```

---

## 🌐 Network Summary

### Headquarters (HQ)

| VLAN | Name | Subnet | Gateway |
|------|------|--------|---------|
| 10 | MGMT | 10.1.10.0/24 | 10.1.10.1 (HSRP) |
| 20 | SERVERS | 10.1.20.0/24 | 10.1.20.1 (HSRP) |
| 30 | USERS | 10.1.30.0/24 | 10.1.30.1 (HSRP) |
| 40 | SOC | 10.1.40.0/24 | 10.1.40.1 (HSRP) |
| 50 | DMZ | 10.1.50.0/24 | 10.1.50.1 (FortiGate) |
| 60 | WIFI | 10.1.60.0/24 | 10.1.60.1 (HSRP) |
| 61 | WIFI-GUEST | 10.1.61.0/24 | 10.1.61.1 (HSRP) |
| 70 | CCTV | 10.1.70.0/24 | 10.1.70.1 (HSRP) |
| 99 | TRANSIT | 10.1.99.0/24 | 10.1.99.1 (FortiGate) |
| 98 | TRANSIT-BK | 10.1.98.0/24 | 10.1.98.1 (FortiGate) |

### Branch-01 (BR1)

| VLAN | Name | Subnet | Gateway |
|------|------|--------|---------|
| 110 | BR1-CLIENT | 10.2.10.0/24 | 10.2.10.1 (FortiGate) |
| 120 | BR1-MGMT | 10.2.20.0/24 | 10.2.20.1 (FortiGate) |
| 130 | BR1-IT | 10.2.30.0/24 | 10.2.30.1 (FortiGate) |
| 140 | BR1-MKT | 10.2.40.0/24 | 10.2.40.1 (FortiGate) |

### Branch-02 (BR2)

| VLAN | Name | Subnet | Gateway |
|------|------|--------|---------|
| 210 | BR2-CLIENT | 10.3.10.0/24 | 10.3.10.1 (FortiGate) |
| 220 | BR2-SERVERS | 10.3.20.0/24 | 10.3.20.1 (FortiGate) |
| 230 | BR2-IT | 10.3.30.0/24 | 10.3.30.1 (FortiGate) |
| 240 | BR2-MKT | 10.3.40.0/24 | 10.3.40.1 (FortiGate) |
| 250 | BR2-MGMT | 10.3.50.0/24 | 10.3.50.1 (FortiGate) |

---

## 🔒 Security Stack

| Layer | Technology | Profile |
|-------|-----------|---------|
| Web Filtering | FortiGuard + Static URL | HQ_WEB_FILTER_BASELINE_V1 |
| Application Control | Protocol decoding | HQ_APP_CONTROL_BASELINE_V1 |
| IPS | Signature-based | HQ_IPS_BASELINE_V1 |
| Antivirus | Flow-based scanning | HQ_AV_BASELINE_V1 |
| SSL Inspection | Certificate inspection | HQ_SSL_CERT_BASELINE_V1 |
| DNS Filtering | FortiGuard categories | HQ_DNS_FILTER_BASELINE_V1 |
| File Filtering | File type analysis | HQ_FILE_FILTER_BASELINE_V1 |
| Identity | LDAP + FSSO | AD-integrated policies |
| SIEM | Wazuh | Custom detection rules |

---

## ✅ Testing Results

| Category | Tests | Passed | Success Rate |
|----------|-------|--------|-------------|
| Layer 2 | 15 | 15 | 100% |
| Layer 3 | 20 | 20 | 100% |
| Firewall | 10 | 10 | 100% |
| Internet/SD-WAN | 8 | 8 | 100% |
| HA Failover | 12 | 12 | 100% |
| VPN | 16 | 16 | 100% |
| AD Integration | 14 | 14 | 100% |
| Security Profiles | 20 | 20 | 100% |
| SOC Alerts | 6 | 6 | 100% |
| **Total** | **121** | **121** | **100%** |

---

## 🚀 Future Enhancements

- [ ] Deploy Wazuh agents on BR1 and BR2 domain controllers
- [ ] Integrate FortiAnalyzer with Wazuh for unified visibility
- [ ] Implement deep SSL inspection with internal PKI
- [ ] Deploy video filtering profiles
- [ ] Build SOAR playbooks for automated incident response
- [ ] Add threat intelligence feeds (MISP/TAXII)
- [ ] Implement network segmentation with micro-segmentation
- [ ] Deploy endpoint detection and response (EDR)

---

## 📝 Author

**Author:** Amer Ehab Amer  
**Project:** Zer0-Po!nT — Secure Enterprise Network Infrastructure  
**Institution:** Faculty of Electronic Engineering, Menoufia University  
**Year:** 2026  
afull demo :https://drive.google.com/file/d/175cjci1oUTmhtDmLvmcvgweNVr3Rlh3b/view?usp=drive_link
**License:** This documentation is provided for educational purposes.

---

> **Note:** This repository contains configuration files and technical documentation for a graduation project. All IP addresses, passwords, and sensitive data are for lab/simulation environments only.
