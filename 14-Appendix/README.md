# 14 — Appendix

## Table of Contents

1. [Device Naming Convention](#device-naming-convention)
2. [Port Reference](#port-reference)
3. [Command Reference](#command-reference)
4. [Troubleshooting Guide](#troubleshooting-guide)
5. [MTU/MSS Issue Resolution](#mtumss-issue-resolution)
6. [Glossary](#glossary)

---

## 1. Device Naming Convention

### HQ Devices

| Device | Hostname | Role |
|--------|----------|------|
| Core Switch 1 | SW-HQ-CORE-01 | Active Core |
| Core Switch 2 | SW-HQ-CORE-02 | Standby Core |
| Access Switch (MGMT) | SW-HQ-ACC-MGMT | Management VLAN |
| Access Switch (Servers) | SW-HQ-ACC-SERVERS | Servers VLAN |
| Access Switch (Users) | SW-HQ-ACC-USERS | Users VLAN |
| Access Switch (SOC) | SW-HQ-ACC-SOC | SOC VLAN |
| Access Switch (WiFi) | SW-HQ-ACC-WIFI | WiFi VLANs |
| Access Switch (CCTV) | SW-HQ-ACC-CCTV | CCTV VLAN |
| Firewall 1 | FG-HQ-01 | Active Firewall |
| Firewall 2 | FG-HQ-02 | Standby Firewall |
| DC 1 | DC1-HQ | Primary Domain Controller |
| DC 2 | DC2-HQ | Secondary Domain Controller |

### BR1 Devices

| Device | Hostname | Role |
|--------|----------|------|
| Core Switch 1 | SW-BR1-CORE-01 | STP Root Primary |
| Core Switch 2 | SW-BR1-CORE-02 | STP Root Secondary |
| Access Switch | SW-BR1-ACC-01 | User access |
| Firewall 1 | FG-BR1-01 | Active Firewall |
| Firewall 2 | FG-BR1-02 | Standby Firewall |
| DC 1 | DC1-BR1 | Child Domain Controller |

### BR2 Devices

| Device | Hostname | Role |
|--------|----------|------|
| Core Switch 1 | SW-BR2-CORE-01 | STP Root Primary |
| Core Switch 2 | SW-BR2-CORE-02 | STP Root Secondary |
| Access Switch | SW-BR2-ACC-01 | User access |
| Firewall 1 | FG-BR2-01 | Active Firewall |
| Firewall 2 | FG-BR2-02 | Standby Firewall |
| DC 1 | DC1-BR2 | Child Domain Controller |

---

## 2. Port Reference

### HQ Core-01 Ports

| Port | Description | VLAN | Type |
|------|-------------|------|------|
| E0/0 | CORE-INTERCONNECT-to-CORE02 | Trunk (all) | Trunk |
| E0/1 | TRUNK-to-ACC-MGMT | 10 | Trunk |
| E0/2 | TRUNK-to-ACC-SERVERS | 20 | Trunk |
| E0/3 | TRUNK-to-ACC-USERS | 30 | Trunk |
| E1/0 | TRUNK-to-ACC-SOC | 40 | Trunk |
| E1/1 | TRUNK-to-ACC-WIFI | 60,61 | Trunk |
| E1/2 | TRUNK-to-ACC-CCTV | 70 | Trunk |
| E1/3 | TO_FGT_HA_PORT5 | 99 | Access |
| E2/0 | TO_FGT_HA_PORT5 | 99 | Access |

### HQ FortiGate Ports

| Port | Alias | IP | Role |
|------|-------|-----|------|
| port1 | WAN_ORANGE | DHCP | WAN |
| port2 | WAN_WE | DHCP | WAN |
| port3 | HA_HEARTBEAT | 169.254.1.1/24 | HA |
| port4 | CORE-TRANSIT-BACKUP | 10.1.98.1/24 | LAN |
| port5 | CORE-TRANSIT-PRIMARY | 10.1.99.1/24 | LAN |

---

## 3. Command Reference

### Cisco IOS Commands

```bash
! Basic verification
show version
show running-config
show startup-config

! Interface verification
show ip interface brief
show interfaces status
show interfaces trunk

! VLAN verification
show vlan brief
show vlan id <id>

! STP verification
show spanning-tree vlan <id> root
show spanning-tree summary
show spanning-tree interface <interface> detail

! HSRP verification
show standby brief
show standby vlan <id>

! Routing verification
show ip route
show ip route 0.0.0.0
show ip cef

! ARP and MAC
show arp
show mac address-table

! CDP/LLDP
show cdp neighbors
show lldp neighbors

! Save config
write memory
```

### FortiOS Commands

```bash
! System status
get system status
get system performance status

! Interface status
get system interface physical
show system interface

! Routing
get router info routing-table all
get router info kernel

! HA status
get system ha status
diagnose sys ha dump-by vcluster

! VPN status
diagnose vpn tunnel list
get vpn ipsec phase1-interface
get vpn ipsec phase2-interface

! SD-WAN
get system sdwan
get system sdwan member
get system sdwan health-check

! Security profiles
get security-profile av
get security-profile webfilter
get security-profile application-list
get security-profile ips

! Logs
diagnose log filter category 0
diagnose log filter field policy_id <id>
diagnose log display

! Packet capture
diagnose sniffer packet any "host 8.8.8.8" 4 0 a

! Connectivity tests
execute ping <ip>
execute traceroute <ip>
execute telnet <ip> <port>
```

### Windows/PowerShell Commands

```powershell
# AD verification
dcdiag /q
repadmin /replsummary
repadmin /showrepl

# DNS verification
nslookup hq.zeropoint.local
nslookup dc1-hq.hq.zeropoint.local

# Group Policy
gpresult /r
gpupdate /force

# Network verification
Test-NetConnection 10.1.99.1 -Port 443
Test-NetConnection 8.8.8.8

# Wazuh agent
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m 10.1.40.10 -p 1515 -A <name>
Get-Service Wazuh
```

---

## 4. Troubleshooting Guide

### Issue: Cannot Access FortiGate Management from Internal Network

**Symptoms:**
- ICMP ping to FortiGate succeeds
- TCP connection establishes
- Web GUI does not load

**Root Cause:** MTU/MSS mismatch on transit path

**Solution:**
```fortios
config system interface
    edit "port5"
        set tcp-mss 1360
    next
    edit "port4"
        set tcp-mss 1360
    next
end
```

**Verification:**
```bash
! Test with different packet sizes
ping 10.1.99.1 -l 1472    # Fails (proves MTU issue)
ping 10.1.99.1 -l 1464    # Succeeds
```

---

### Issue: HSRP Not Preempting

**Symptoms:**
- Core-01 is up but not Active
- Core-02 remains Active after Core-01 recovery

**Solution:**
```cisco
interface VlanX
 standby X preempt
end
```

**Verification:**
```bash
show standby brief
```

---

### Issue: VPN Tunnel Not Establishing

**Symptoms:**
- Phase 1 (IKE) fails
- No IPSec SA established

**Checklist:**
1. Verify WAN IPs are reachable: `execute ping <remote-gw>`
2. Verify PSK matches on both sides
3. Verify proposal/encryption settings match
4. Check FortiGate logs: `diagnose debug application ike -1`
5. Verify NAT-T is enabled if behind NAT

---

### Issue: FSSO User Not Mapping

**Symptoms:**
- User logged in but not visible in FSSO user list
- Identity-based policy not matching

**Checklist:**
1. Verify Collector Agent is running on DC
2. Verify FortiGate can reach DC on port 8002
3. Check FSSO service account permissions
4. Verify user is in monitored group
5. Check FSSO logs: `diagnose debug fsso server-status`

---

## 5. MTU/MSS Issue Resolution

### Problem Discovery

During internal FortiGate management access testing, HTTP sessions to the firewall management page failed despite successful ICMP and TCP connectivity.

### Diagnosis

```bash
! MTU test from client
ping 10.1.99.1 -f -l 1472    # Fragmentation needed, fails
ping 10.1.99.1 -f -l 1464    # Succeeds
```

**Finding:** Path MTU is approximately 1492 bytes (1500 - 8 byte PPPoE overhead from ISP).

### Solution Applied

Reduced TCP MSS on FortiGate transit interfaces to accommodate overhead:

```fortios
config system interface
    edit "port5"
        set tcp-mss 1360
    next
    edit "port4"
        set tcp-mss 1360
    next
end
```

**Result:** Internal management access restored successfully.

---

## 6. Glossary

| Term | Definition |
|------|-----------|
| **HSRP** | Hot Standby Router Protocol — Cisco gateway redundancy protocol |
| **STP** | Spanning Tree Protocol — Prevents Layer 2 loops |
| **Rapid-PVST** | Rapid Per-VLAN Spanning Tree — Fast-converging STP variant |
| **SVI** | Switched Virtual Interface — Layer 3 interface on a VLAN |
| **BPDU Guard** | Blocks ports that receive BPDUs (prevents rogue switches) |
| **PortFast** | Immediately transitions port to forwarding state |
| **SD-WAN** | Software-Defined Wide Area Network |
| **IPSec** | Internet Protocol Security — VPN encryption standard |
| **FSSO** | Fortinet Single Sign-On — Identity integration with AD |
| **LDAP** | Lightweight Directory Access Protocol |
| **SIEM** | Security Information and Event Management |
| **MTU** | Maximum Transmission Unit — Largest packet size |
| **MSS** | Maximum Segment Size — Largest TCP payload |
| **DNAT** | Destination Network Address Translation |
| **VIP** | Virtual IP (FortiGate) or Virtual IP (HSRP) |
| **GC** | Global Catalog — AD replica for fast queries |
| **OU** | Organizational Unit — AD container for objects |
| **GPO** | Group Policy Object — AD policy configuration |
