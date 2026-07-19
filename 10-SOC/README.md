# 10 — Security Operations Center (SOC)

## Table of Contents

1. [Overview](#1-overview)
2. [SOC Architecture](#2-soc-architecture)
3. [Wazuh SIEM Deployment](#3-wazuh-siem-deployment)
4. [Agent Deployment](#4-agent-deployment)
5. [Detection Use Cases](#5-detection-use-cases)
6. [FortiAnalyzer Integration](#6-fortianalyzer-integration)
7. [Log Flow](#7-log-flow)
8. [Alert Validation](#8-alert-validation)
9. [Future Enhancements](#9-future-enhancements)
10. [Verification Commands](#10-verification-commands)

---

## 1. Overview

The **Security Operations Center (SOC)** at HQ provides **centralized monitoring and detection** for security events across all Zer0-Po!nT sites (HQ, BR1, BR2).

### SOC Components

| Component | Tool | Role | Status |
|-----------|------|------|--------|
| SIEM | Wazuh | Endpoint log collection, detection rules, alerting | ✅ Deployed |
| Log Aggregator | FortiAnalyzer | Firewall log collection from all sites | ✅ Deployed |
| Domain Controller | Windows Server | Identity & authentication logs | ✅ Sending logs |
| VPN | IPSec | Secure log transmission from branches | ✅ In place |

---

## 2. SOC Architecture

```
                    ┌─────────────────┐
                    │   Internet      │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │      FortiGate HA (HQ)       │
              └──────────────┬──────────────┘
                             │
              ┌──────────────┴──────────────┐
              │      Core Switches (HQ)      │
              └──────────────┬──────────────┘
                             │
                    ┌────────┴────────┐
                    │   VLAN 40 (SOC)  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────┴────┐         ┌────┴────┐
   │ Wazuh   │         │Forti    │
   │ SIEM    │         │Analyzer│
   │10.1.40.10│        │10.1.40.x│
   └────┬────┘         └────┬────┘
        │                   │
        │ VPN Tunnels       │ Log forwarding
   ┌────┴────┬────────┐     └── HQ / BR1 / BR2 FortiGates
   │  BR1    │  BR2   │
   └─────────┴────────┘
```

---

## 3. Wazuh SIEM Deployment

### Components Installed

| Component | Version | Purpose |
|-----------|---------|---------|
| Wazuh Manager | 4.14.5 | Central log collection & rule engine |
| Wazuh Indexer | 4.14.5 | Log storage & search |
| Wazuh Dashboard | 4.14.5 | Visualization & alerting UI |

### Wazuh Manager

| Parameter | Value |
|-----------|-------|
| IP Address | 10.1.40.10 |
| VLAN | 40 (SOC) |
| Port (Registration) | 1515 |
| Port (Communication) | 1514 |

---

## 4. Agent Deployment

Wazuh agents are deployed on **all Windows servers across all three sites** (HQ, BR1, BR2), including domain controllers. All agents are actively forwarding logs to the HQ Wazuh Manager over the site-to-site VPN tunnels.

| Site | Coverage | Status |
|------|----------|--------|
| HQ | All Windows servers (incl. DC1-HQ, DC2-HQ) | ✅ Deployed |
| BR1 | All Windows servers (incl. DC1-BR1) | ✅ Deployed |
| BR2 | All Windows servers (incl. DC1-BR2) | ✅ Deployed |

### Agent Verification

```bash
# On Wazuh Manager
/var/ossec/bin/agent_control -l

# Check agent status
/var/ossec/bin/agent_control -i <agent_id>
```

---

## 5. Detection Use Cases

Three detection rules have been written, deployed, and validated.

| File | Rule ID | Description | Event ID(s) | MITRE ID | Status |
|------|---------|-------------|-------------|----------|--------|
| [wazuh-rule-brute-force.xml](wazuh-rule-brute-force.xml) | 100001 | Repeated failed logins | 4625 | T1110 | ✅ Deployed & Validated |
| [wazuh-rule-log-clearing.xml](wazuh-rule-log-clearing.xml) | 100002 | Security log cleared | 1102 | T1070 | ✅ Deployed & Validated |
| [wazuh-rule-privilege-escalation.xml](wazuh-rule-privilege-escalation.xml) | 100003-100004 | User added to Domain Admins group / RDP login | 4728, 4624 (Logon Type 10) | T1078 | ✅ Deployed & Validated |

### Use Case 01 — Brute Force Detection

**Objective:** Detect repeated failed login attempts.
**Data Source:** Windows Security Logs, Event ID **4625**.
**Detection Logic:** 5 failed logins within 60 seconds → Alert (Level 10). Maps to MITRE **T1110**.

### Use Case 02 — Security Log Clearing Detection

**Objective:** Detect attempts to erase security audit logs.
**Data Source:** Windows Security Logs, Event ID **1102**.
**Detection Logic:** Any occurrence of Event ID 1102 → Critical Alert (Level 12). Maps to MITRE **T1070**.

### Use Case 03 — Privilege Escalation Detection

**Objective:** Detect unauthorized escalation to administrative privileges.
**Data Source:** Windows Security Logs.
**Detection Logic:**
- **Event ID 4728** — a user is added to the Domain Admins security group → Alert.
- **Event ID 4624 (Logon Type 10)** — a Remote Desktop (RDP) logon occurs → Alert.

Maps to MITRE ATT&CK technique **T1078** (Valid Accounts).

---

## 6. FortiAnalyzer Integration

FortiAnalyzer is deployed at HQ and is receiving logs from **every FortiGate in the infrastructure** (HQ, BR1, BR2).

### Purpose

- Centralized firewall log storage
- Traffic and UTM log visibility across all sites

### Log Sources

| Source | Log Type | Status |
|--------|----------|--------|
| HQ FortiGate | Traffic, Event, UTM | ✅ Forwarding |
| BR1 FortiGate | Traffic, Event, UTM | ✅ Forwarding |
| BR2 FortiGate | Traffic, Event, UTM | ✅ Forwarding |

### FortiGate Log Forwarding Config

```fortios
config log fortianalyzer setting
    set status enable
    set server 10.1.40.x          ! FortiAnalyzer IP
    set reliable enable
end
```

> **Note:** FortiAnalyzer and Wazuh currently operate as two separate log destinations (FortiGate logs → FortiAnalyzer, endpoint/server logs → Wazuh). They are not yet integrated with each other — see [Future Enhancements](#9-future-enhancements).

---

## 7. Log Flow

### Endpoint/Server Log Flow

```
Windows Server (any site) → Wazuh Agent → VPN Tunnel (branches only) → HQ Wazuh Manager
                                                    ↓
                                              Wazuh Indexer
                                                    ↓
                                              Wazuh Dashboard
```

### FortiGate Log Flow

```
FortiGate (HQ / BR1 / BR2) → FortiAnalyzer (HQ)
```

---

## 8. Alert Validation

| Test | Result | Alert Level |
|------|--------|-------------|
| Brute Force (5 failed logins / 60s) | ✅ Triggered | 10 |
| Security Log Clearing | ✅ Triggered | 12 |
| Added to Domain Admins Group | ✅ Triggered | — |
| RDP Logon | ✅ Triggered | — |

---

## 9. Future Enhancements

| Enhancement | Description | Priority |
|-------------|-------------|----------|
| Integrate FortiAnalyzer with Wazuh | Unified visibility between FortiAnalyzer and Wazuh | High |
| Lateral Movement Detection | Detect lateral movement between hosts (T1021) | Medium |
| Suspicious Admin Activity Detection | Detect anomalous admin behavior (T1098) | Medium |
| FortiAnalyzer Dashboards | Build custom SOC dashboards | Medium |
| Incident Response Playbooks | Document response procedures | Low |
| Threat Intelligence Feeds | Integrate MISP/TAXII | Low |
| SOAR Integration | Automate response actions | Low |

---

## 10. Verification Commands

```bash
# Wazuh Manager
/var/ossec/bin/agent_control -l                    # List agents
/var/ossec/bin/agent_control -i <id>                # Agent details
/var/ossec/bin/ossec-logtest                        # Test rules

# Wazuh Indexer
curl -X GET "localhost:9200/_cluster/health"        # Cluster health
curl -X GET "localhost:9200/_cat/indices"           # Index list

# Wazuh Dashboard
# Access via browser: https://10.1.40.10

# FortiAnalyzer
diagnose log device status                           # Log device status
diagnose test connect 10.1.40.x 514                  # Test connectivity
```
