# 16 - Active Directory Lab

## Objective

Build an Active Directory lab environment on Windows Server and perform practical exercises involving domain configuration, DNS verification, user and group management, and security event analysis.

The main objectives were:

- Install Windows Server and configure a static IP address
- Install Active Directory Domain Services (AD DS) and DNS
- Create a new Active Directory forest
- Verify domain controller discovery and DNS records
- Create organizational units, users, and security groups
- Assign users to appropriate groups
- Review account creation and group membership audit events
- Generate and investigate a failed logon event
- Verify network connectivity between lab machines

---

## Lab Environment

| Component | Details |
|---|---|
| Domain Controller | Windows Server 2025 |
| Hostname | `AD-DC01` |
| Domain | `adlab.test` |
| NetBIOS Domain Name | `ADLAB` |
| Domain Controller IP | `192.168.153.10` |
| Default Gateway | `192.168.153.2` |
| Ubuntu Server | `192.168.153.129` |
| Kali Linux | `192.168.153.130` |
| Directory Service | Active Directory Domain Services |
| DNS Service | Windows DNS Server |
| Management Tools | Server Manager, PowerShell |
| Audit Log | Windows Security event log |

---

## 1. Windows Server Deployment

Windows Server 2025 Evaluation was installed in VMware Workstation.

The virtual machine was configured with:

- 4 GB RAM
- 2 virtual CPU cores
- 60 GB virtual disk
- NAT networking

The server hostname was changed to:

```text
AD-DC01
```

A static IPv4 address was configured:

```text
IP Address:      192.168.153.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.153.2
```

Network connectivity was tested against the gateway and Ubuntu Server.

After configuring a restricted inbound ICMP firewall rule for the lab subnet, Kali Linux successfully reached the Windows Server:

```text
4 packets transmitted, 4 received, 0% packet loss
```

---

## 2. Active Directory Domain Services Installation

The Active Directory Domain Services role was installed through Server Manager, together with the required management tools.

The server was promoted to a domain controller by creating a new forest:

```text
adlab.test
```

The resulting NetBIOS domain name was:

```text
ADLAB
```

The domain functional level reported by PowerShell was:

```text
Windows2025Domain
```

The following services were verified as running:

```text
DNS
Netlogon
NTDS
```

This confirmed that the core domain controller services were operational.

---

## 3. DNS and Domain Controller Verification

The domain DNS record was verified using:

```powershell
Resolve-DnsName adlab.test -Server 127.0.0.1
```

The query resolved to:

```text
adlab.test → 192.168.153.10
```

The domain controller LDAP service record was queried using:

```powershell
Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.adlab.test" -Type SRV -Server 127.0.0.1
```

The SRV record identified:

```text
ad-dc01.adlab.test
```

Domain controller discovery was verified using:

```powershell
nltest /dsgetdc:adlab.test
```

The command completed successfully and identified:

```text
DC:      \\AD-DC01.adlab.test
Address: \\192.168.153.10
Domain:  adlab.test
Forest:  adlab.test
```

These tests confirmed that domain DNS records and domain controller discovery were functioning correctly.

---

## 4. Organizational Units and Security Groups

Two organizational units were created to separate test users and security groups.

```text
adlab.test
├── LabUsers
└── LabGroups
```

The following global security groups were created under `LabGroups`:

```text
IT-Support
Security-Analysts
```

These groups were used to demonstrate basic role-based organization and group membership management.

---

## 5. User Account Management

Two test accounts were created under the `LabUsers` organizational unit.

| Display Name | SamAccountName | User Principal Name |
|---|---|---|
| Lab Support User | `lab.support` | `lab.support@adlab.test` |
| Lab Security Analyst | `lab.analyst` | `lab.analyst@adlab.test` |

The accounts were configured as enabled users.

During account creation, the domain password policy rejected an initial password because it did not satisfy the configured password requirements.

The existing account was inspected rather than recreated. A new password was then assigned, and the account was enabled.

The analyst account was also inspected and its password was reset successfully.

This exercise demonstrated the importance of understanding domain password policy and recovering from account provisioning errors.

---

## 6. Group Membership Assignment

The test accounts were assigned to their corresponding security groups:

```powershell
Add-ADGroupMember -Identity "IT-Support" -Members "lab.support"
Add-ADGroupMember -Identity "Security-Analysts" -Members "lab.analyst"
```

Group membership was verified using:

```powershell
Get-ADGroupMember -Identity "IT-Support"
Get-ADGroupMember -Identity "Security-Analysts"
```

The results confirmed:

```text
IT-Support
└── lab.support

Security-Analysts
└── lab.analyst
```

The users were assigned only to their respective test groups. No Domain Admin privileges were granted to these accounts.

---

## 7. Windows Security Event Log Analysis

Windows Security events were inspected using PowerShell and `Get-WinEvent`.

### Event ID 4720 — User Account Created

The following query was used:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4720
    StartTime = (Get-Date).AddDays(-1)
} -MaxEvents 5
```

The event log contained:

```text
Event ID: 4720
A user account was created.
```

The displayed event identified the creation of the `lab.analyst` account.

This demonstrated that user account creation could be investigated through Windows Security auditing.

### Event ID 4728 — Member Added to Security-Enabled Global Group

The Security log contained:

```text
Event ID: 4728
A member was added to a security-enabled global group.
```

The event included the `Security-Analysts` group and the `lab.analyst` account.

This provided audit evidence for the group membership assignment performed during the lab.

---

## 8. Failed Authentication Testing

A controlled failed logon attempt was generated using:

```powershell
runas /user:ADLAB\lab.support cmd
```

An incorrect password was entered once for the test account.

The Windows Security event log was queried for failed logon attempts:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4625
    StartTime = (Get-Date).AddMinutes(-10)
} -MaxEvents 5
```

The resulting event contained:

```text
Event ID: 4625
An account failed to log on.
```

Relevant fields included:

```text
Account Name: lab.support
Account Domain: ADLAB
Logon Type: 2
Failure Reason: Unknown user name or bad password
Workstation Name: AD-DC01
Source Network Address: ::1
```

### Analysis

The `4625` event confirmed that the incorrect password attempt was recorded.

The source address `::1` is the IPv6 loopback address. Combined with the workstation name and logon type, the event indicated a local logon attempt on the domain controller rather than a remote network authentication attempt.

This distinction is important when investigating security events: the event ID, account, logon type, source information, and failure reason should be considered together.

---

## Results

| Assessment Area | Result |
|---|---|
| Windows Server deployment | Completed |
| Static IP configuration | Completed |
| Lab network connectivity | Verified |
| AD DS installation | Successful |
| New forest creation | Successful |
| DNS A record verification | Successful |
| LDAP SRV record verification | Successful |
| Domain controller discovery | Successful |
| Organizational units | Created |
| Test users | Created and enabled |
| Security groups | Created |
| Group membership assignment | Verified |
| Event ID 4720 investigation | Verified |
| Event ID 4728 investigation | Verified |
| Failed logon test | Successful |
| Event ID 4625 investigation | Verified |

---

## Skills Demonstrated

### Active Directory Administration

- Windows Server deployment
- Active Directory Domain Services configuration
- Domain controller promotion
- Organizational unit creation
- User account management
- Global security group management
- Group membership assignment

### DNS and Networking

- Static IPv4 configuration
- DNS A record verification
- LDAP SRV record verification
- Domain controller discovery
- Lab network connectivity testing

### Security Monitoring

- Windows Security event log investigation
- User account creation auditing
- Global group membership auditing
- Failed logon analysis
- Event ID interpretation
- Logon type and source address analysis

### Troubleshooting

- Domain password policy validation
- Account provisioning recovery
- DNS troubleshooting
- Network connectivity verification
- PowerShell-based administration

---

## Conclusion

This lab established a functional Active Directory environment on Windows Server 2025.

A new forest, `adlab.test`, was created and verified through DNS queries and domain controller discovery. Test users and global security groups were configured, and group memberships were validated.

Windows Security event logs were then used to investigate user creation, group membership changes, and a controlled failed authentication attempt.

The lab demonstrated how Active Directory administration and Windows security auditing complement each other: directory changes create operational state, while audit events provide evidence that can be used for investigation and monitoring.

This environment provides a foundation for future exercises involving domain authentication, Group Policy, Windows event monitoring, and defensive security.
