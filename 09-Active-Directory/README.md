# 09 — Active Directory Integration

## Table of Contents

1. [Overview](#1-overview)
2. [AD Forest Structure](#2-ad-forest-structure)
3. [Domain Controllers](#3-domain-controllers)
4. [Sites & Subnets](#4-sites--subnets)
5. [OU Structure](#5-ou-structure)
6. [Group Policy Objects](#6-group-policy-objects)
7. [FortiGate LDAP Integration](#7-fortigate-ldap-integration)
8. [FSSO (Single Sign-On)](#8-fsso-single-sign-on)
9. [Identity-Based Policies](#9-identity-based-policies)
10. [Verification Commands](#10-verification-commands)
---

## 1. Overview

Active Directory provides **centralized identity management** across all Zer0-Po!nT sites. The FortiGate firewalls integrate with AD via LDAP and FSSO to enable **identity-based security policies**.

---

## 2. AD Forest Structure

```
Forest: hq.zeropoint.local
│
├── Domain: hq.zeropoint.local
│   ├── DC1-HQ (Primary)
│   ├── DC2-HQ (Secondary)
│   │
│   ├── OU: HQ-Users
│   ├── OU: HQ-Computers
│   ├── OU: HQ-Groups
│   │
│   └── Child Domain: br1.hq.zeropoint.local
│       └── DC1-BR1
│
└── Child Domain: br2.hq.zeropoint.local
    └── DC1-BR2
```

---

## 3. Domain Controllers

### HQ Domain Controllers

| Server | IP Address | Role | Domain |
|--------|------------|------|--------|
| DC1-HQ | 10.1.20.x | Primary DC, DNS, GC | hq.zeropoint.local |
| DC2-HQ | 10.1.20.x | Secondary DC, DNS, GC | hq.zeropoint.local |

### Branch Domain Controllers

| Server | IP Address | Role | Domain |
|--------|------------|------|--------|
| DC1-BR1 | 10.2.20.x | Child DC, DNS | br1.hq.zeropoint.local |
| DC1-BR2 | 10.3.20.x | Child DC, DNS | br2.hq.zeropoint.local |

### Replication Verification

```powershell
# On any DC
repadmin /replsummary
```

---

## 4. Sites & Subnets

### Sites

| Site | Domain Controllers | Subnets |
|------|-------------------|---------|
| HQ | DC1-HQ, DC2-HQ | 10.1.0.0/16 |
| BR1 | DC1-BR1 | 10.2.0.0/16 |
| BR2 | DC1-BR2 | 10.3.0.0/16 |

### Subnet Assignment

```powershell
# In Active Directory Sites and Services
New-ADReplicationSubnet -Name "10.1.0.0/16" -Site "HQ"
New-ADReplicationSubnet -Name "10.2.0.0/16" -Site "BR1"
New-ADReplicationSubnet -Name "10.3.0.0/16" -Site "BR2"
```

---

## 5. OU Structure

### HQ Organizational Units

```
DC=hq,DC=zeropoint,DC=local
├── OU=HQ-Users
│   ├── CN=Administrator
│   ├── CN=svc-fgt-ldap
│   └── CN=regular-users...
├── OU=HQ-Computers
│   └── CN=workstations...
├── OU=HQ-Groups
│   ├── CN=Domain Admins
│   ├── CN=IT-Admin
│   ├── CN=Management
│   └── CN=VPN-Users
└── OU=HQ-Servers
    ├── CN=DC1-HQ
    └── CN=DC2-HQ
```

### Branch OUs

```
DC=br1,DC=hq,DC=zeropoint,DC=local
├── OU=BR1-Users
├── OU=BR1-Computers
└── OU=BR1-Groups

DC=br2,DC=hq,DC=zeropoint,DC=local
├── OU=BR2-Users
├── OU=BR2-Computers
└── OU=BR2-Groups
```

---

## 6. Group Policy Objects

### HQ GPOs

| GPO Name | Target | Settings |
|----------|--------|----------|
| HQ-User-Restrictions | HQ-Users | Disable Control Panel, CMD, Regedit, Run menu |
| HQ-Endpoint-Security | HQ-Computers | Enable Firewall, disable Guest, anonymous enumeration |
| HQ-USB-Block | HQ-Computers | Block removable storage |

### BR1 GPOs

| GPO Name | Target | Settings |
|----------|--------|----------|
| BR1-User-Restrictions | BR1-Users | Same as HQ baseline |
| BR1-Management-Controlled | BR1-Management | Screen lock (600s), moderate restrictions |
| BR1-IT-Admin-Baseline | BR1-IT | Screen lock, disable consumer experiences |
| BR1-Endpoint-Security | BR1-Computers | Firewall, disable Guest, logon message |
| BR1-USB-Block | BR1-Computers | Block USB storage |

### BR2 GPOs

Same methodology as BR1, applied to BR2 OUs.

### Key GPO Settings

#### User Restrictions
- **Disable Control Panel and Settings** — Prevent users from changing system settings
- **Disable Command Prompt (CMD)** — Block command-line access
- **Disable Registry Editor (Regedit)** — Prevent registry modifications
- **Hide Run from Start Menu** — Remove Run dialog
- **Disable Network Adapter Settings** — Prevent network configuration changes

#### Endpoint Security
- **Enable Windows Firewall** — All profiles (Domain, Private, Public)
- **Disable Guest Account** — Remove unauthorized access vector
- **Disable Anonymous Enumeration** — Prevent SID enumeration
- **Do Not Store Passwords Using Reversible Encryption** — Protect credential storage
- **Interactive Logon Message** — Display security warning at login

#### USB Blocking
- **Removable Storage: Deny Write Access** — Block data exfiltration
- **Removable Storage: Deny Read Access** — Block malware introduction

---

## 7. FortiGate LDAP Integration

### LDAP Server Object (HQ)

```fortios
config user ldap
    edit "HQ_LDAP_SERVER"
        set server "10.1.20.x"          ! DC1-HQ IP
        set cnid "sAMAccountName"
        set dn "dc=hq,dc=zeropoint,dc=local"
        set type regular
        set username "CN=svc-fgt-ldap,OU=HQ-Users,DC=hq,DC=zeropoint,DC=local"
        set password ENC AK1...
        set secure disable
    next
    edit "HQ_LDAP_SERVER_2"
        set server "10.1.20.x"          ! DC2-HQ IP
        set cnid "sAMAccountName"
        set dn "dc=hq,dc=zeropoint,dc=local"
        set type regular
        set username "CN=svc-fgt-ldap,OU=HQ-Users,DC=hq,DC=zeropoint,DC=local"
        set password ENC AK1...
        set secure disable
    next
end
```

### LDAP User Group

```fortios
config user group
    edit "HQ_AD_Admins"
        set member "HQ_LDAP_SERVER" "HQ_LDAP_SERVER_2"
        config match
            edit 1
                set group-name "CN=Domain Admins,OU=HQ-Groups,DC=hq,DC=zeropoint,DC=local"
            next
        end
    next
end
```

### Remote Administrator Account

```fortios
config system admin
    edit "admin_ldap"
        set remote-auth enable
        set accprofile "super_admin"
        set vdom "root"
        set remote-group "HQ_AD_Admins"
    next
end
```

### BR1 LDAP Configuration

```fortios
config user ldap
    edit "BR1_LDAP_SERVER"
        set server "10.2.20.x"          ! DC1-BR1 IP
        set cnid "sAMAccountName"
        set dn "dc=br1,dc=hq,dc=zeropoint,dc=local"
        set type regular
        set username "CN=svc-fgt-ldap,OU=BR1-Users,DC=br1,DC=hq,DC=zeropoint,DC=local"
        set password ENC AK1...
    next
end
```

### BR2 LDAP Configuration

Same pattern as BR1, with DC1-BR2 IP and br2 domain DN.

---

## 8. FSSO (Single Sign-On)

### Collector Agent (HQ)

Installed on **DC1-HQ** and **DC2-HQ** for redundancy.

#### Collector Agent Settings

| Parameter | Value |
|-----------|-------|
| Listening Port | 8002 |
| FortiGate IP | 10.1.99.1 |
| Workgroup/Domain | hq.zeropoint.local |
| User Access | All Domain Users |

### FortiGate FSSO Configuration (HQ)

```fortios
config user fsso
    edit "HQ_FSSO"
        set server "10.1.20.x"          ! DC1-HQ IP
        set password ENC AK1...
        set port 8002
        set source-ip 10.1.99.1
    next
end
```

### FSSO User Group

```fortios
config user group
    edit "HQ_FSSO_Users"
        set member "HQ_FSSO"
        config match
            edit 1
                set group-name "CN=Domain Users,CN=Users,DC=hq,DC=zeropoint,DC=local"
            next
        end
    next
end
```

### Agentless FSSO (BR1)

BR1 uses **agentless polling** against DC1-BR1 as an alternative to Collector Agent deployment.

```fortios
config user fsso
    edit "BR1_FSSO_POLLING"
        set server "10.2.20.x"          ! DC1-BR1 IP
        set type polling
        set password ENC AK1...
        set port 8002
    next
end
```

---

## 9. Identity-Based Policies

### HQ Example: IT Admin Internet Access

```fortios
config firewall policy
    edit 20
        set name "IT_Admin_Internet"
        set srcintf "port5" "port4"
        set dstintf "virtual-wan-link"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set groups "HQ_FSSO_Users"
        set nat enable
        set logtraffic all
        set webfilter-profile "HQ_WEB_FILTER_BASELINE_V1"
        set application-list "HQ_APP_CONTROL_BASELINE_V1"
    next
end
```

### BR1 Example: Identity-Based Internet Access

```fortios
config firewall policy
    edit 30
        set name "BR1_Identity_Internet"
        set srcintf "LAN_TRUNK"
        set dstintf "virtual-wan-link"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set groups "BR1_FSSO_Users"
        set nat enable
        set logtraffic all
    next
end
```

---

## 10. Verification Commands

```bash
! Verify LDAP connectivity
diagnose test connect 10.1.20.x 389

! Verify LDAP user lookup
diagnose test auth ldap HQ_LDAP_SERVER username password

! Verify FSSO status
diagnose debug fsso server-status

! Verify FSSO user mapping
diagnose debug fsso user-list

! Verify identity-based policy match
diagnose firewall iprope show 100004 1

! Test administrator login (from AD account)
! Use web GUI or SSH with domain credentials
```

---

## Screenshots

Reference screenshots captured during the build, extracted from the original project log.

![Dedicated LDAP service account (svc-fgt-ldap) created in Active Directory for FortiGate integration.](images/01-hq-ad-ldap-service-account.png)
*Dedicated LDAP service account (svc-fgt-ldap) created in Active Directory for FortiGate integration.*

![ICMP reachability test confirming FortiGate can reach DC1-HQ and DC2-HQ.](images/02-hq-ad-dc-reachability.png)
*ICMP reachability test confirming FortiGate can reach DC1-HQ and DC2-HQ.*

![LDAP server object created on FortiGate targeting DC1-HQ.](images/03-hq-ad-ldap-server-object.png)
*LDAP server object created on FortiGate targeting DC1-HQ.*

![Secondary LDAP server object created targeting DC2-HQ for redundancy.](images/04-hq-ad-ldap-server-object-dc2.png)
*Secondary LDAP server object created targeting DC2-HQ for redundancy.*

![Active Directory groups imported into FortiGate as remote user groups.](images/05-hq-ad-ldap-user-group.png)
*Active Directory groups imported into FortiGate as remote user groups.*

![FortiGate administrator account configured using a remote AD group.](images/06-hq-ad-remote-admin-group.png)
*FortiGate administrator account configured using a remote AD group.*

![Successful administrator login test using a domain account.](images/07-hq-ad-admin-login-test-1.png)
*Successful administrator login test using a domain account.*

![Administrator login test — group membership enforcement confirmed.](images/08-hq-ad-admin-login-test-2.png)
*Administrator login test — group membership enforcement confirmed.*

![FSSO Collector Agent installation on DC1-HQ.](images/09-hq-fsso-agent-install-dc1-1.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/10-hq-fsso-agent-install-dc1-2.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/11-hq-fsso-agent-install-dc1-3.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/12-hq-fsso-agent-install-dc1-4.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/13-hq-fsso-agent-install-dc1-5.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/14-hq-fsso-agent-install-dc1-6.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/15-hq-fsso-agent-install-dc1-7.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/16-hq-fsso-agent-install-dc1-8.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/17-hq-fsso-agent-install-dc1-9.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installation on DC1-HQ.](images/18-hq-fsso-agent-install-dc1-10.png)
*FSSO Collector Agent installation on DC1-HQ.*

![FSSO Collector Agent installed on DC2-HQ for redundancy/failover.](images/19-hq-fsso-agent-install-dc2.png)
*FSSO Collector Agent installed on DC2-HQ for redundancy/failover.*

![Collector Agent connected to FortiGate to provide the SSO service.](images/20-hq-fsso-collector-fortigate-1.png)
*Collector Agent connected to FortiGate to provide the SSO service.*

![Collector Agent / FortiGate SSO connection.](images/21-hq-fsso-collector-fortigate-2.png)
*Collector Agent / FortiGate SSO connection.*

![BR1 FortiGate LDAP server settings.](images/22-br1-ldap-setting.png)
*BR1 FortiGate LDAP server settings.*

![BR1 LDAP user groups created.](images/23-br1-ldap-user-groups.png)
*BR1 LDAP user groups created.*

![BR1 admin profile for network security engineering, authenticated via BR1_LDAP_SERVER.](images/24-br1-ldap-admin-profile-1.png)
*BR1 admin profile for network security engineering, authenticated via BR1_LDAP_SERVER.*

![BR1 remote-authenticated administrator account configuration.](images/25-br1-ldap-admin-profile-2.png)
*BR1 remote-authenticated administrator account configuration.*

![BR1 remote-authenticated administrator account configuration.](images/26-br1-ldap-admin-profile-3.png)
*BR1 remote-authenticated administrator account configuration.*

![BR1 remote-authenticated administrator account configuration.](images/27-br1-ldap-admin-profile-4.png)
*BR1 remote-authenticated administrator account configuration.*

![BR1 LDAP logging verification.](images/28-br1-ldap-logging-check.png)
*BR1 LDAP logging verification.*

![BR1 agentless FSSO polling configured against the BR1 domain controller.](images/29-br1-fsso-agentless-polling.png)
*BR1 agentless FSSO polling configured against the BR1 domain controller.*

![BR1 agentless FSSO polling configuration.](images/30-br1-fsso-agentless-polling-2.png)
*BR1 agentless FSSO polling configuration.*

![BR1 FSSO user groups created.](images/31-br1-fsso-user-groups.png)
*BR1 FSSO user groups created.*

![BR1 identity user mapping test.](images/32-br1-fsso-identity-mapping-test.png)
*BR1 identity user mapping test.*

![BR1 firewall policy enhanced with identity-based (FSSO) matching.](images/33-br1-fsso-policy-enhanced.png)
*BR1 firewall policy enhanced with identity-based (FSSO) matching.*

![BR2 FortiGate queries the BR2 child domain controller over LDAP via a dedicated bind account.](images/34-br2-ldap-config.png)
*BR2 FortiGate queries the BR2 child domain controller over LDAP via a dedicated bind account.*

![BR2 FSSO polling mode configured against the BR2 Active Directory.](images/35-br2-fsso-polling-mode.png)
*BR2 FSSO polling mode configured against the BR2 Active Directory.*

![BR2 FSSO polling configuration.](images/36-br2-fsso-polling-mode-2.png)
*BR2 FSSO polling configuration.*

![BR2 FSSO and LDAP groups created.](images/37-br2-fsso-ldap-groups.png)
*BR2 FSSO and LDAP groups created.*

![BR2 remote-authenticated administrator with LDAP server.](images/38-br2-ldap-remote-admin-1.png)
*BR2 remote-authenticated administrator with LDAP server.*

![BR2 remote-authenticated administrator configuration.](images/39-br2-ldap-remote-admin-2.png)
*BR2 remote-authenticated administrator configuration.*

![BR2 FSSO successfully mapping authenticated users to IP addresses for real-time identity enforcement.](images/40-br2-fsso-identity-mapping.png)
*BR2 FSSO successfully mapping authenticated users to IP addresses for real-time identity enforcement.*
