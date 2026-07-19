# 07 — Site-to-Site VPN (IPSec)

## Table of Contents

1. [Overview](#overview)
2. [VPN Topology](#vpn-topology)
3. [HQ Configuration](#hq-configuration)
4. [BR1 Configuration](#br1-configuration)
5. [BR2 Configuration](#br2-configuration)
6. [Redundant Tunnels](#redundant-tunnels)
7. [SD-WAN VPN Integration](#sd-wan-vpn-integration)
8. [Firewall Policies](#firewall-policies)

---

## 1. Overview

Site-to-site IPSec VPN tunnels connect the **Headquarters** with **Branch-01** and **Branch-02**. Redundant tunnels are configured over both ISPs (WE and ORANGE) to ensure high availability.

---

## 2. VPN Topology

### HQ ↔ BR1

| Tunnel | HQ Endpoint | BR1 Endpoint | ISP | Network |
|--------|-------------|--------------|-----|---------|
| Tunnel 1 | 172.16.1.1 | 172.16.1.2 | ORANGE | /30 |
| Tunnel 2 | 172.16.2.1 | 172.16.2.2 | WE | /30 |

### HQ ↔ BR2

| Tunnel | HQ Endpoint | BR2 Endpoint | ISP | Network |
|--------|-------------|--------------|-----|---------|
| Tunnel 1 | 172.16.3.1 | 172.16.3.2 | ORANGE | /30 |
| Tunnel 2 | 172.16.4.1 | 172.16.4.2 | WE | /30 |

### WAN IPs

| Site | ISP-WE | ISP-ORANGE |
|------|--------|------------|
| HQ | 192.168.1.10 | 192.168.218.10 |
| BR1 | 192.168.1.110 | 192.168.218.110 |
| BR2 | 192.168.1.210 | 192.168.218.210 |

---

## 3. HQ Configuration

### Phase 1 (IKE)

```fortios
config vpn ipsec phase1-interface
    edit "HQ-BR1-TUN1"
        set interface "WAN_ORANGE"
        set ike-version 2
        set keylifeseconds 28800
        set peertype any
        set proposal aes256-sha256
        set remote-gw 192.168.218.110
        set psksecret ENC AK1...
        set dpd enable
    next
    edit "HQ-BR1-TUN2"
        set interface "WAN_WE"
        set ike-version 2
        set keylifeseconds 28800
        set peertype any
        set proposal aes256-sha256
        set remote-gw 192.168.1.110
        set psksecret ENC AK1...
        set dpd enable
    next
end
```

### Phase 2 (IPSec)

```fortios
config vpn ipsec phase2-interface
    edit "HQ-BR1-TUN1-P2"
        set phase1name "HQ-BR1-TUN1"
        set proposal aes256-sha256
        set keylifeseconds 3600
        set auto-negotiate enable
    next
    edit "HQ-BR1-TUN2-P2"
        set phase1name "HQ-BR1-TUN2"
        set proposal aes256-sha256
        set keylifeseconds 3600
        set auto-negotiate enable
    next
end
```

### Tunnel Interface IPs

```fortios
config system interface
    edit "HQ-BR1-TUN1"
        set ip 172.16.1.1 255.255.255.252
        set remote-ip 172.16.1.2
    next
    edit "HQ-BR1-TUN2"
        set ip 172.16.2.1 255.255.255.252
        set remote-ip 172.16.2.2
    next
end
```

### Static Routes to Branches

```fortios
config router static
    edit 10
        set dst 10.2.0.0 255.255.0.0
        set device "HQ-BR1-TUN1"
        set distance 10
    next
    edit 11
        set dst 10.2.0.0 255.255.0.0
        set device "HQ-BR1-TUN2"
        set distance 20
    next
    edit 20
        set dst 10.3.0.0 255.255.0.0
        set device "HQ-BR2-TUN1"
        set distance 10
    next
    edit 21
        set dst 10.3.0.0 255.255.0.0
        set device "HQ-BR2-TUN2"
        set distance 20
    next
end
```

---

## 4. BR1 Configuration

### Phase 1 (IKE)

```fortios
config vpn ipsec phase1-interface
    edit "BR1-HQ-TUN1"
        set interface "WAN_ORANGE"
        set ike-version 2
        set proposal aes256-sha256
        set remote-gw 192.168.218.10
        set psksecret ENC AK1...
    next
    edit "BR1-HQ-TUN2"
        set interface "WAN_WE"
        set ike-version 2
        set proposal aes256-sha256
        set remote-gw 192.168.1.10
        set psksecret ENC AK1...
    next
end
```

### Phase 2 (IPSec)

```fortios
config vpn ipsec phase2-interface
    edit "BR1-HQ-TUN1-P2"
        set phase1name "BR1-HQ-TUN1"
        set proposal aes256-sha256
        set auto-negotiate enable
    next
    edit "BR1-HQ-TUN2-P2"
        set phase1name "BR1-HQ-TUN2"
        set proposal aes256-sha256
        set auto-negotiate enable
    next
end
```

### Tunnel Interface IPs

```fortios
config system interface
    edit "BR1-HQ-TUN1"
        set ip 172.16.1.2 255.255.255.252
        set remote-ip 172.16.1.1
    next
    edit "BR1-HQ-TUN2"
        set ip 172.16.2.2 255.255.255.252
        set remote-ip 172.16.2.1
    next
end
```

### Static Route to HQ

```fortios
config router static
    edit 1
        set dst 10.1.0.0 255.255.0.0
        set device "BR1-HQ-TUN1"
        set distance 10
    next
    edit 2
        set dst 10.1.0.0 255.255.0.0
        set device "BR1-HQ-TUN2"
        set distance 20
    next
end
```

---

## 5. BR2 Configuration

Same methodology as BR1, with adjusted IPs:

| Parameter | BR2 Value |
|-----------|-----------|
| Tunnel 1 IP | 172.16.3.2 |
| Tunnel 2 IP | 172.16.4.2 |
| HQ Tunnel 1 IP | 172.16.3.1 |
| HQ Tunnel 2 IP | 172.16.4.1 |

---

## 6. Redundant Tunnels

### Design Rationale

| Tunnel | Path | Distance | Role |
|--------|------|----------|------|
| Tunnel 1 | ORANGE → ORANGE | 10 | Primary |
| Tunnel 2 | WE → WE | 20 | Backup |

If the primary tunnel fails, traffic automatically shifts to the backup tunnel. SD-WAN health checks monitor tunnel availability.

---

## 7. SD-WAN VPN Integration

### VPN as SD-WAN Members

```fortios
config system sdwan
    config members
        edit 3
            set interface "HQ-BR1-TUN1"
            set zone "virtual-wan-link"
        next
        edit 4
            set interface "HQ-BR1-TUN2"
            set zone "virtual-wan-link"
        next
    end
end
```

### SLA Monitoring for Tunnels

```fortios
config system sdwan
    config health-check
        edit "TUNNEL_CHECK"
            set server 172.16.1.2
            set members 3 4
        next
    end
end
```

---

## 8. Firewall Policies

### HQ → BR1 Policy

```fortios
config firewall policy
    edit 10
        set name "HQ_to_BR1"
        set srcintf "port5" "port4"
        set dstintf "HQ-BR1-TUN1" "HQ-BR1-TUN2"
        set srcaddr "HQ_LAN"
        set dstaddr "BR1_LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

### BR1 → HQ Policy (Reverse)

```fortios
config firewall policy
    edit 10
        set name "BR1_to_HQ"
        set srcintf "BR1-HQ-TUN1" "BR1-HQ-TUN2"
        set dstintf "LAN_TRUNK"
        set srcaddr "BR1_LAN"
        set dstaddr "HQ_LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

---

## 9. Verification Commands

```bash
! Verify VPN tunnel status
diagnose vpn tunnel list

! Verify Phase 1 (IKE)
get vpn ipsec phase1-interface

! Verify Phase 2 (IPSec)
get vpn ipsec phase2-interface

! Verify tunnel interface
get system interface

! Test connectivity through tunnel
execute ping 172.16.1.2
execute ping 10.2.10.1

! Verify routing
get router info routing-table all

! Check VPN statistics
diagnose vpn tunnel up HQ-BR1-TUN1
```

---

## Screenshots

Reference screenshots captured during the build, extracted from the original project log.

![HQ↔BR1 IPSec Phase 1 (IKE) tunnel configuration.](images/01-hq-br1-vpn-phase1-1.png)
*HQ↔BR1 IPSec Phase 1 (IKE) tunnel configuration.*

![HQ↔BR1 IPSec tunnel configuration.](images/02-hq-br1-vpn-phase1-2.png)
*HQ↔BR1 IPSec tunnel configuration.*

![HQ↔BR1 IPSec tunnel configuration.](images/03-hq-br1-vpn-phase1-3.png)
*HQ↔BR1 IPSec tunnel configuration.*

![HQ↔BR1 IPSec Phase 2 configuration.](images/04-hq-br1-vpn-phase2-1.png)
*HQ↔BR1 IPSec Phase 2 configuration.*

![HQ↔BR1 IPSec Phase 2 configuration.](images/05-hq-br1-vpn-phase2-2.png)
*HQ↔BR1 IPSec Phase 2 configuration.*

![HQ↔BR1 IPSec Phase 2 configuration.](images/06-hq-br1-vpn-phase2-3.png)
*HQ↔BR1 IPSec Phase 2 configuration.*

![HQ↔BR1 secondary (backup) tunnel — created the same way as tunnel 1.](images/07-hq-br1-vpn-tunnel2-1.png)
*HQ↔BR1 secondary (backup) tunnel — created the same way as tunnel 1.*

![HQ↔BR1 secondary tunnel configuration.](images/08-hq-br1-vpn-tunnel2-2.png)
*HQ↔BR1 secondary tunnel configuration.*

![HQ↔BR1 secondary tunnel configuration.](images/09-hq-br1-vpn-tunnel2-3.png)
*HQ↔BR1 secondary tunnel configuration.*

![HQ↔BR1 secondary tunnel configuration.](images/10-hq-br1-vpn-tunnel2-4.png)
*HQ↔BR1 secondary tunnel configuration.*

![Performance SLA created for VPN tunnel health checks.](images/11-hq-br1-vpn-sla.png)
*Performance SLA created for VPN tunnel health checks.*

![SD-WAN rule for VPN path selection.](images/12-hq-br1-vpn-sdwan-rule-1.png)
*SD-WAN rule for VPN path selection.*

![SD-WAN rule for VPN path selection.](images/13-hq-br1-vpn-sdwan-rule-2.png)
*SD-WAN rule for VPN path selection.*

![SD-WAN rule for VPN path selection.](images/14-hq-br1-vpn-sdwan-rule-3.png)
*SD-WAN rule for VPN path selection.*

![IP configuration for the VPN tunnel interface.](images/15-hq-br1-vpn-tunnel-ip-1.png)
*IP configuration for the VPN tunnel interface.*

![Tunnel interface IP configuration.](images/16-hq-br1-vpn-tunnel-ip-2.png)
*Tunnel interface IP configuration.*

![Tunnel interface IP configuration.](images/17-hq-br1-vpn-tunnel-ip-3.png)
*Tunnel interface IP configuration.*

![SLA added for tunnel monitoring — second tunnel configured identically.](images/18-hq-br1-vpn-sla-monitor.png)
*SLA added for tunnel monitoring — second tunnel configured identically.*

![Firewall policy allowing traffic across the VPN tunnel.](images/19-hq-br1-vpn-policy.png)
*Firewall policy allowing traffic across the VPN tunnel.*

![Reverse firewall policy (BR1 → HQ) for the VPN tunnel.](images/20-hq-br1-vpn-policy-reverse.png)
*Reverse firewall policy (BR1 → HQ) for the VPN tunnel.*

![Static route making the remote VPN network reachable.](images/21-hq-br1-vpn-route.png)
*Static route making the remote VPN network reachable.*
