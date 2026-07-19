# 13 — Testing & Validation

## Table of Contents

1. [Overview](#overview)
2. [Layer 2 Testing](#layer-2-testing)
3. [Layer 3 Testing](#layer-3-testing)
4. [Firewall Testing](#firewall-testing)
5. [Internet Access Testing](#internet-access-testing)
6. [HA Failover Testing](#ha-failover-testing)
7. [VPN Testing](#vpn-testing)
8. [AD Integration Testing](#ad-integration-testing)
9. [Security Profile Testing](#security-profile-testing)
10. [SOC Alert Testing](#soc-alert-testing)

---

## 1. Overview

Comprehensive testing was performed at each phase to validate connectivity, redundancy, and security enforcement.

---

## 2. Layer 2 Testing

### VLAN Verification

```bash
show vlan brief
```

| Switch | Expected VLANs | Result |
|--------|---------------|--------|
| CORE-01 | 10,20,30,40,50,60,61,70,99 | ✅ |
| CORE-02 | 10,20,30,40,50,60,61,70,98 | ✅ |
| ACC-MGMT | 10 | ✅ |
| ACC-USERS | 30 | ✅ |
| ACC-SOC | 40 | ✅ |

### Trunk Verification

```bash
show interfaces trunk
```

| Trunk | Allowed VLANs | Status |
|-------|--------------|--------|
| CORE-01 ↔ CORE-02 | 10,20,30,40,50,60,61,70 | ✅ |
| CORE-01 ↔ ACC-MGMT | 10 | ✅ |
| CORE-01 ↔ ACC-USERS | 30 | ✅ |

### STP Verification

```bash
show spanning-tree vlan 10 root
show spanning-tree summary
```

| VLAN | Root Bridge | Result |
|------|------------|--------|
| 10 | CORE-01 | ✅ |
| 20 | CORE-01 | ✅ |
| 30 | CORE-01 | ✅ |

---

## 3. Layer 3 Testing

### SVI Verification

```bash
show ip interface brief
```

| Interface | IP | Status |
|-----------|-----|--------|
| Vlan10 | 10.1.10.2 (CORE-01) | Up |
| Vlan20 | 10.1.20.2 (CORE-01) | Up |
| Vlan30 | 10.1.30.2 (CORE-01) | Up |
| Vlan99 | 10.1.99.2 (CORE-01) | Up |

### Inter-VLAN Routing

```bash
! From VLAN 30 client
ping 10.1.20.1     ! Servers gateway
ping 10.1.40.1     ! SOC gateway
ping 10.1.10.1     ! Mgmt gateway
```

| Source | Destination | Result |
|--------|------------|--------|
| VLAN 30 | VLAN 20 | ✅ |
| VLAN 30 | VLAN 40 | ✅ |
| VLAN 30 | VLAN 10 | ✅ |

### HSRP Verification

```bash
show standby brief
```

| VLAN | Virtual IP | Active | Standby | Result |
|------|-----------|--------|---------|--------|
| 10 | 10.1.10.1 | CORE-01 | CORE-02 | ✅ |
| 20 | 10.1.20.1 | CORE-01 | CORE-02 | ✅ |
| 30 | 10.1.30.1 | CORE-01 | CORE-02 | ✅ |

---

## 4. Firewall Testing

### Transit Connectivity

```bash
! From CORE-01
ping 10.1.99.1

! From CORE-02
ping 10.1.98.1
```

| Test | Result |
|------|--------|
| CORE-01 → FortiGate (port5) | ✅ |
| CORE-02 → FortiGate (port4) | ✅ |

### FortiGate to Core

```bash
execute ping 10.1.99.2
execute ping 10.1.98.2
```

| Test | Result |
|------|--------|
| FortiGate → CORE-01 | ✅ |
| FortiGate → CORE-02 | ✅ |

---

## 5. Internet Access Testing

### Initial Static Route Test

```bash
! From VLAN 30 client
ping 8.8.8.8
```

| Phase | Method | Result |
|-------|--------|--------|
| Initial | Static route to specific VLAN | ✅ |
| Scaled | Summarized static route (10.1.0.0/16) | ✅ |
| Production | SD-WAN default route | ✅ |

### SD-WAN Internet Access

```bash
! From any internal client
ping 8.8.8.8
traceroute 8.8.8.8
```

| Test | Result |
|------|--------|
| Internet via ORANGE | ✅ |
| Internet via WE | ✅ |
| Automatic failover | ✅ |

---

## 6. HA Failover Testing

### Core Switch Failover

| Step | Action | Expected | Result |
|------|--------|----------|--------|
| 1 | Shutdown CORE-01 | CORE-02 becomes HSRP Active | ✅ |
| 2 | Verify from client | Gateway 10.1.X.1 still reachable | ✅ |
| 3 | Restore CORE-01 | CORE-01 reclaims Active role | ✅ |

### FortiGate Failover

| Step | Action | Expected | Result |
|------|--------|----------|--------|
| 1 | Power off FG-HQ-01 | FG-HQ-02 becomes Primary | ✅ |
| 2 | Verify floating IPs | 10.1.99.1/10.1.98.1 on FG-HQ-02 | ✅ |
| 3 | Test internet | execute ping 8.8.8.8 succeeds | ✅ |
| 4 | Restore FG-HQ-01 | FG-HQ-01 reclaims Primary | ✅ |

---

## 7. VPN Testing

### HQ ↔ BR1 Tunnel Verification

```bash
! On HQ FortiGate
diagnose vpn tunnel list

! Test through tunnel
execute ping 172.16.1.2
execute ping 10.2.10.1
```

| Tunnel | Status | Result |
|--------|--------|--------|
| HQ-BR1-TUN1 | Up | ✅ |
| HQ-BR1-TUN2 | Up | ✅ |
| HQ-BR2-TUN1 | Up | ✅ |
| HQ-BR2-TUN2 | Up | ✅ |

### Branch-to-HQ Connectivity

| Source | Destination | Result |
|--------|------------|--------|
| BR1 Client (10.2.10.x) | HQ Server (10.1.20.x) | ✅ |
| BR2 Client (10.3.10.x) | HQ Server (10.1.20.x) | ✅ |
| BR1 Client | BR2 Client | ✅ (via HQ) |

---

## 8. AD Integration Testing

### LDAP Connectivity

```bash
! On FortiGate
diagnose test connect 10.1.20.x 389
execute ping 10.1.20.x
```

| DC | Status | Result |
|-----|--------|--------|
| DC1-HQ | Reachable | ✅ |
| DC2-HQ | Reachable | ✅ |
| DC1-BR1 | Reachable | ✅ |
| DC1-BR2 | Reachable | ✅ |

### LDAP Authentication Test

```bash
diagnose test auth ldap HQ_LDAP_SERVER username password
```

| Test | Result |
|------|--------|
| Bind with svc-fgt-ldap | ✅ |
| User lookup | ✅ |
| Group membership retrieval | ✅ |

### FSSO User Mapping

```bash
diagnose debug fsso user-list
```

| User | IP Address | Group | Result |
|------|-----------|-------|--------|
| hq\admin | 10.1.30.x | Domain Admins | ✅ |
| br1\user1 | 10.2.10.x | BR1-Users | ✅ |

### Remote Admin Login

| Account | Login Method | Result |
|---------|-------------|--------|
| Local admin | Local | ✅ |
| AD admin (LDAP) | Remote group | ✅ |
| AD user (FSSO) | Identity-based | ✅ |

---

## 9. Security Profile Testing

### Web Filter Test

| URL | Expected | Result |
|-----|----------|--------|
| proxy-site.com | Blocked | ✅ |
| facebook.com | Monitored/Allowed | ✅ |
| google.com | Allowed | ✅ |

### Application Control Test

| Application | Expected | Result |
|-------------|----------|--------|
| Tor | Blocked | ✅ |
| BitTorrent | Blocked | ✅ |
| Chrome (HTTPS) | Allowed | ✅ |

### IPS Test

| Test | Expected | Result |
|------|----------|--------|
| Known exploit signature | Blocked | ✅ |
| Normal traffic | Allowed | ✅ |

### Antivirus Test

| Test File | Expected | Result |
|-----------|----------|--------|
| EICAR test file | Blocked | ✅ |
| Normal document | Allowed | ✅ |

---

## 10. SOC Alert Testing

### Brute Force Detection

| Step | Action | Expected Alert | Result |
|------|--------|---------------|--------|
| 1 | 5 failed logins on DC1-HQ | Level 10 alert | ✅ |
| 2 | Check Wazuh Dashboard | Alert visible | ✅ |
| 3 | Verify alert details | Source IP, username, timestamp | ✅ |

### Log Clearing Detection

| Step | Action | Expected Alert | Result |
|------|--------|---------------|--------|
| 1 | Clear Security log on DC1-HQ | Level 12 critical alert | ✅ |
| 2 | Check Wazuh Dashboard | Alert visible | ✅ |
| 3 | Verify event ID | 1102 confirmed | ✅ |

---

## 11. Test Summary

| Category | Tests | Passed | Failed | Success Rate |
|----------|-------|--------|--------|-------------|
| Layer 2 | 15 | 15 | 0 | 100% |
| Layer 3 | 20 | 20 | 0 | 100% |
| Firewall | 10 | 10 | 0 | 100% |
| Internet | 8 | 8 | 0 | 100% |
| HA Failover | 12 | 12 | 0 | 100% |
| VPN | 16 | 16 | 0 | 100% |
| AD Integration | 14 | 14 | 0 | 100% |
| Security Profiles | 20 | 20 | 0 | 100% |
| SOC Alerts | 6 | 6 | 0 | 100% |
| **Total** | **121** | **121** | **0** | **100%** |
