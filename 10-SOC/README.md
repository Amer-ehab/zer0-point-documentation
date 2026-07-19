# 10 — Security Operations Center (SOC)

## Table of Contents

1. [Overview](#overview)
2. [SOC Architecture](#soc-architecture)
3. [Wazuh SIEM Deployment](#wazuh-siem-deployment)
4. [Agent Deployment](#agent-deployment)
5. [Detection Use Cases](#detection-use-cases)
6. [FortiAnalyzer Integration](#fortianalyzer-integration)
7. [Log Flow](#log-flow)
8. [Alert Validation](#alert-validation)
9. [Future Enhancements](#future-enhancements)

---

## 1. Overview

The **Security Operations Center (SOC)** at HQ provides **centralized monitoring, detection, and response** for security events across all Zer0-Po!nT sites.

### SOC Components

| Component | Tool | Role |
|-----------|------|------|
| SIEM | Wazuh | Endpoint & server log collection, detection rules, alerting |
| Log Aggregator | FortiAnalyzer | Firewall log collection and analysis |
| Domain Controller | Windows Server | Identity & authentication logs |
| VPN | IPSec | Secure log transmission from branches |

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
   ┌────┴────┐         ┌────┴────┐         ┌────┴────┐
   │ Wazuh   │         │Forti    │         │ SOC     │
   │ SIEM    │         │Analyzer│         │ Server  │
   │10.1.40.10│        │10.1.40.x│        │10.1.40.x│
   └────┬────┘         └─────────┘         └─────────┘
        │
        │         VPN Tunnels
        │    ┌────────┬────────┐
        └────┤  BR1   │  BR2   ├────┘
             └────────┴────────┘
```

---

## 3. Wazuh SIEM Deployment

### Components Installed

| Component | Version | Purpose |
|-----------|---------|---------|
| Wazuh Manager | 4.14.5 | Central log collection & rule engine |
| Wazuh Indexer | 4.14.5 | Log storage & search |
| Wazuh Dashboard | 4.14.5 | Visualization & alerting UI |

### Service Verification

```bash
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

### Wazuh Manager IP

| Parameter | Value |
|-----------|-------|
| IP Address | 10.1.40.10 |
| VLAN | 40 (SOC) |
| Port (Registration) | 1515 |
| Port (Communication) | 1514 |

---

## 4. Agent Deployment

### HQ Domain Controller Agent (HQ-DC1)

#### Installation (PowerShell)

```powershell
# Download agent
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi" `
    -OutFile "$env:TEMP\wazuh-agent.msi"

# Install agent
msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /qn `
    WAZUH_MANAGER="10.1.40.10" `
    WAZUH_AGENT_NAME="HQ-DC1"
```

#### Agent Registration

```powershell
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" `
    -m 10.1.40.10 `
    -p 1515 `
    -A HQ-DC1
```

#### Agent Verification

```bash
# On Wazuh Manager
/var/ossec/bin/agent_control -l

# Check agent status
/var/ossec/bin/agent_control -i <agent_id>
```

### Future Agent Deployments

| Target | Agent Name | Priority |
|--------|-----------|----------|
| DC2-HQ | HQ-DC2 | High |
| DC1-BR1 | BR1-DC1 | High |
| DC1-BR2 | BR2-DC1 | High |
| HQ Servers | HQ-SRV-01... | Medium |
| BR1 Servers | BR1-SRV-01... | Medium |
| BR2 Servers | BR2-SRV-01... | Medium |

---

## 5. Detection Use Cases

### Detection Rule Files

| File | Rule ID | Description | MITRE ID |
|------|---------|-------------|----------|
| [wazuh-rule-brute-force.xml](wazuh-rule-brute-force.xml) | 100001 | Brute force detection (5 failed logins / 60s) | T1110 |
| [wazuh-rule-log-clearing.xml](wazuh-rule-log-clearing.xml) | 100002 | Security log clearing detection | T1070 |
| [wazuh-rule-privilege-escalation.xml](wazuh-rule-privilege-escalation.xml) | 100003-100004 | Privilege escalation detection | T1078 |
| [wazuh-rule-lateral-movement.xml](wazuh-rule-lateral-movement.xml) | 100005-100006 | Lateral movement detection | T1021 |
| [wazuh-rule-suspicious-admin.xml](wazuh-rule-suspicious-admin.xml) | 100007-100008 | Suspicious admin activity | T1098 |

### Use Case 01 — Brute Force Detection

#### Objective
Detect repeated failed login attempts on the Domain Controller.

#### Data Source
- Windows Security Logs
- Event ID: **4625**

#### Detection Logic
- **5 failed logins** within **60 seconds** → Brute Force Alert (Level 10)
- Maps to MITRE ATT&CK technique **T1110** (Brute Force)

---

### Use Case 02 — Security Log Clearing Detection

#### Objective
Detect attempts to erase security audit logs.

#### Data Source
- Windows Security Logs
- Event ID: **1102**

#### Detection Logic
- Any **Event ID 1102** → Critical Alert (Level 12)
- Maps to MITRE ATT&CK technique **T1070** (Indicator Removal)

---

### Future Use Cases

| Use Case | MITRE ID | Event IDs | Status |
|----------|----------|-----------|--------|
| Privilege Escalation | T1078 | 4672, 4728 | Configured |
| Lateral Movement | T1021 | 4624, 4648 | Configured |
| Suspicious Admin Activity | T1098 | 4738, 4781 | Configured |
| Malware Detection | T1204 | Various | Planned |
| Data Exfiltration | T1041 | Network logs | Planned |

---

## 6. FortiAnalyzer Integration

### Purpose
FortiAnalyzer collects and analyzes **FortiGate firewall logs** from all sites, providing:

- Centralized log storage
- Advanced reporting
- Threat intelligence correlation
- Compliance reporting

### Log Sources

| Source | IP | Log Type |
|--------|-----|----------|
| HQ FortiGate | 10.1.99.1 | Traffic, Event, UTM |
| BR1 FortiGate | 10.2.x.x | Traffic, Event, UTM |
| BR2 FortiGate | 10.3.x.x | Traffic, Event, UTM |

### FortiGate Log Forwarding

```fortios
config log fortianalyzer setting
    set status enable
    set server 10.1.40.x          ! FortiAnalyzer IP
    set reliable enable
end
```

---

## 7. Log Flow

### Branch-to-HQ Log Transmission

```
Branch Device → Wazuh Agent → VPN Tunnel → HQ Wazuh Manager
                                    ↓
                              Wazuh Indexer
                                    ↓
                              Wazuh Dashboard
                                    ↓
                              SOC Analyst
```

### FortiGate Log Flow

```
FortiGate → FortiAnalyzer (HQ)
    ↓
Wazuh (via syslog forwarding if needed)
```

---

## 8. Alert Validation

### Brute Force Test

1. Generate multiple failed login attempts on DC1-HQ
2. Verify alert appears in Wazuh Dashboard
3. Confirm alert details (source IP, username, timestamp)

### Log Clearing Test

1. Manually clear the Security log on DC1-HQ
2. Verify critical alert triggers immediately
3. Confirm alert severity (Level 12)

### Alert Screenshots

| Test | Result | Alert Level |
|------|--------|-------------|
| Brute Force | ✅ Triggered | 10 |
| Log Clearing | ✅ Triggered | 12 |

---

## 9. SOC Operational Model

```
Log Collection → Detection Rules → Alerts → Investigation → Response
      ↑                                                    ↓
   Wazuh Agents                                      Incident Report
   FortiAnalyzer                                    Remediation Actions
   Windows Events                                   Policy Updates
```

---

## 10. Future Enhancements

| Enhancement | Description | Priority |
|-------------|-------------|----------|
| Branch Agent Deployment | Install Wazuh agents on BR1/BR2 DCs | High |
| FortiAnalyzer Dashboards | Build custom SOC dashboards | High |
| Advanced Use Cases | Privilege escalation, lateral movement | Medium |
| Incident Response Playbooks | Document response procedures | Medium |
| Threat Intelligence Feeds | Integrate MISP/TAXII | Low |
| SOAR Integration | Automate response actions | Low |

---

## 11. Verification Commands

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
