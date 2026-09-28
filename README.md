# Network+ VirtualBox Small Business Lab

## Overview

This project documents a hands-on small-business network lab built in Oracle VirtualBox while studying for the CompTIA Network+ certification.

The lab uses Windows Server 2025 and Windows 11 to practice network configuration, DHCP, DNS, Active Directory, Group Policy, SMB file sharing, firewall rules, troubleshooting, and Wireshark packet analysis.

## Network Topology

```text
                         Internet
                            |
                      VirtualBox NAT
                       /          \
                  SERVER01       CLIENT01
                       \          /
                        \        /
                         NETPLUS-LAN
                       192.168.58.0/27
```

### Core Systems

| System | Role | Address |
|---|---|---|
| SERVER01 | Windows Server 2025 / DC / DNS / DHCP | `192.168.58.3/27` |
| CLIENT01 | Windows 11 domain client | DHCP |
| Planned Gateway | Future router/gateway | `192.168.58.21` |

### Subnet

```text
Network:        192.168.58.0/27
Subnet Mask:    255.255.255.224
Usable Hosts:   192.168.58.1 - 192.168.58.30
Broadcast:      192.168.58.31
```

## VirtualBox Networking

Each VM uses two virtual NICs:

- `INTERNET-NAT` — internet access through VirtualBox NAT
- `NETPLUS-LAN` — isolated internal lab network

This allowed internet access to remain separate from the private lab network.

## Windows Server Configuration

SERVER01 was configured with a static address:

```text
IP Address:      192.168.58.3
Subnet Mask:     255.255.255.224
Preferred DNS:   192.168.58.3
```

SERVER01 was later promoted to a domain controller for:

```text
corp.netplus.test
```

NetBIOS domain:

```text
CORP
```

## DHCP

Installed the DHCP Server role and created the following scope:

```text
Scope:       NETPLUS-LAN
Start:       192.168.58.1
End:         192.168.58.30
Mask:        255.255.255.224
```

Excluded:

```text
192.168.58.3   SERVER01
192.168.58.21  Planned gateway
```

CLIENT01 was changed from static addressing to DHCP and successfully received a lease from SERVER01.

### DHCP Troubleshooting

After SERVER01 was promoted into Active Directory, CLIENT01 fell back to an APIPA address:

```text
169.254.x.x
```

Investigation showed DHCP authorization was associated with SERVER01's VirtualBox NAT address instead of its internal lab address.

The DHCP authorization was corrected to:

```text
192.168.58.3
```

CLIENT01 then received a valid `192.168.58.x/27` lease again.

## DNS

Installed the DNS Server role.

### Forward DNS

Created:

```text
netplus.test
```

A record:

```text
server01.netplus.test -> 192.168.58.3
```

### Reverse DNS

Created:

```text
58.168.192.in-addr.arpa
```

PTR record:

```text
192.168.58.3 -> server01.netplus.test
```

Forward and reverse DNS resolution were tested successfully.

### Active Directory DNS

Active Directory created:

```text
corp.netplus.test
_msdcs.corp.netplus.test
```

SRV records such as `_ldap` and `_kerberos` were verified.

## Active Directory

Installed Active Directory Domain Services and promoted SERVER01 to the first domain controller.

Created:

```text
Lab-Users
Lab-Computers
Lab-Groups
```

Created the domain user:

```text
CORP\labuser1
```

Created the security group:

```text
CORP\Lab-Staff
```

Added `labuser1` to `Lab-Staff`.

CLIENT01 was successfully joined to:

```text
corp.netplus.test
```

Domain authentication was verified with:

```cmd
whoami
```

Result:

```text
corp\labuser1
```

## SMB File Sharing

Created:

```text
C:\LabShare
```

Shared as:

```text
\\server01\LabShare
```

Access was assigned through the domain security group:

```text
CORP\Lab-Staff
```

Configured:

- SMB share permissions
- NTFS permissions
- Group-based access control

`CORP\labuser1` successfully opened the share and created a file from CLIENT01.

### Permission Troubleshooting

Initial access failed.

The following were checked:

- Group naming
- Group membership
- SMB share permissions
- NTFS permissions
- User security token

After confirming `labuser1` was in `Lab-Staff`, signing out and back in refreshed the access token and the share worked.

## Group Policy

Created:

```text
Lab-Computers-Firewall
```

Linked the GPO to:

```text
Lab-Computers
```

Configured an inbound firewall rule:

```text
NETPLUS Allow ICMPv4
```

Settings:

- Direction: Inbound
- Protocol: ICMPv4
- Type: Echo Request
- Remote subnet: `192.168.58.0/27`
- Action: Allow
- Profile: Domain

Forced an update on CLIENT01:

```cmd
gpupdate /force
```

Verified application with:

```cmd
gpresult /scope computer /r
```

The result showed:

```text
Lab-Computers-Firewall
Default Domain Policy
```

SERVER01 then successfully pinged CLIENT01.

## Wireshark Packet Analysis

Wireshark was installed on CLIENT01 and captures were taken on `NETPLUS-LAN`.

### ICMP

Filter:

```text
icmp
```

Captured:

- Echo Request
- Echo Reply

### DNS

Filter:

```text
dns
```

Captured:

- DNS query for `server01.netplus.test`
- DNS response returning `192.168.58.3`

### DHCP DORA

Filter:

```text
dhcp
```

Captured:

```text
Discover
Offer
Request
Acknowledge
```

### ARP

Filter:

```text
arp
```

Captured:

- ARP Request for `192.168.58.3`
- ARP Reply containing SERVER01's MAC address

### TCP Three-Way Handshake

Generated an SMB connection test:

```powershell
Test-NetConnection 192.168.58.3 -Port 445
```

Filter:

```text
tcp.port == 445
```

Captured:

```text
SYN
SYN-ACK
ACK
```

## Troubleshooting Examples

### ICMP blocked by Windows Firewall

Symptom:

```text
Request timed out
```

Resolution:

- Verified addressing
- Verified VirtualBox network
- Created inbound ICMP rule
- Retested successfully

### PowerShell permission error

Symptom:

```text
Access is denied.
Windows System Error 5
```

Resolution:

- Reopened PowerShell with administrator privileges

### DNS query using wrong interface

A DNS test initially used the NAT connection instead of `NETPLUS-LAN`.

Investigation included:

```powershell
Test-NetConnection
ipconfig
route print
```

CLIENT01 was found using an APIPA address because DHCP was not servicing the internal network correctly.

### DHCP authorization

SERVER01 was initially authorized in Active Directory using its NAT address.

Corrected authorization:

```text
192.168.58.3
```

CLIENT01 then received a valid lease again.

### SMB permissions

Share access initially failed.

Verified:

- `Lab-Staff` group
- User group membership
- Share permissions
- NTFS permissions
- Windows logon token

Access succeeded after correcting group-based permissions and refreshing the user session.

## Skills Demonstrated

- Oracle VirtualBox
- Virtual networking
- NAT
- IPv4 addressing
- Private IPv4
- CIDR
- `/27` subnetting
- Static addressing
- DHCP scopes
- DHCP exclusions
- DHCP leases
- DHCP authorization
- APIPA
- DNS
- A records
- PTR records
- SRV records
- Active Directory Domain Services
- Domain controllers
- OUs
- Users and security groups
- Domain joining
- Group Policy
- Windows Defender Firewall
- SMB
- NTFS permissions
- ICMP
- ARP
- TCP
- DNS traffic analysis
- DHCP DORA
- Wireshark
- PowerShell
- Network troubleshooting

## Screenshot Structure

```text
screenshots/
├── virtualbox/
├── server/
├── client/
├── dhcp/
├── dns/
├── active-directory/
├── group-policy/
├── smb/
├── troubleshooting/
└── wireshark/
```

Recommended screenshots include:

- SERVER01 static IP
- CLIENT01 DHCP lease
- Successful ping
- DHCP scope
- DHCP authorization
- A and PTR records
- AD domain
- CLIENT01 domain membership
- `whoami` domain login
- `Lab-Staff` membership
- SMB permissions
- Successful SMB access
- GPO link
- `gpresult`
- ICMP capture
- DNS capture
- DHCP DORA capture
- ARP capture
- TCP handshake capture

## Resume Project Entry

**Virtual Small Business Network Lab | Oracle VirtualBox, Windows Server 2025, Windows 11**

- Built a virtual client/server network using a `/27` IPv4 subnet and separate NAT and internal network interfaces.
- Configured Windows Server DHCP, DNS, Active Directory Domain Services, Group Policy, SMB sharing, NTFS permissions, and domain authentication.
- Joined a Windows 11 client to Active Directory and managed access through domain security groups.
- Diagnosed DHCP authorization, APIPA, DNS routing, firewall, and SMB permission issues using PowerShell and Windows networking tools.
- Used Wireshark to analyze ICMP, DNS, DHCP DORA, ARP, and TCP three-way handshake traffic.

## Project Status

Core lab build complete.

Next steps:

- Upload screenshots
- Add a network diagram
- Publish the project to GitHub
