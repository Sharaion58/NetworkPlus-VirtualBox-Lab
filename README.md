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
<img width="887" height="571" alt="02-SERVER01-VM-Configuration" src="https://github.com/user-attachments/assets/923dc60d-ea4b-439b-af20-22217d0d80d9" />
<img width="890" height="555" alt="01-CLIENT01-VM-Configuration" src="https://github.com/user-attachments/assets/e783e451-2d62-4980-8abf-37f5004f58ef" />

├── server/
<img width="1012" height="847" alt="03-SERVER01-Hostname" src="https://github.com/user-attachments/assets/6c8d55a8-4d71-4d41-8a18-373224d3cc08" />
<img width="980" height="587" alt="05-SERVER01-Static-IP" src="https://github.com/user-attachments/assets/11bea23c-7625-4904-b33f-262d3471c43c" />

├── client/
├── dhcp/
<img width="1048" height="653" alt="05-SERVER01-DHCP-Lease" src="https://github.com/user-attachments/assets/33dcf7d8-f291-43d7-ba89-ab02e9c3fb7e" />
<img width="1011" height="383" alt="04-CLIENT01-DHCP-Lease" src="https://github.com/user-attachments/assets/c736b54c-8cc6-4dd4-9a33-a32b4fb5dde3" />
<img width="1006" height="603" alt="03-NETPLUS-DHCP-Scope" src="https://github.com/user-attachments/assets/c3617f64-8ec1-40ee-a360-4d0c582991c6" />

├── dns/
<img width="848" height="567" alt="02-SERVER01-A-Record" src="https://github.com/user-attachments/assets/5570bbbc-543d-4654-8538-8022561a2cef" />
<img width="688" height="156" alt="04-DNS-Reverse issue" src="https://github.com/user-attachments/assets/5567d234-22b6-4dfe-8e2d-9ece1286404f" />

├── active-directory/
<img width="873" height="543" alt="03-AD-Domain-Verified png" src="https://github.com/user-attachments/assets/e17d4395-05f8-4d53-964b-9953b8182114" />
<img width="968" height="366" alt="07-CLIENT01-Domain-User-Login" src="https://github.com/user-attachments/assets/2adadd66-a5b4-4ec1-9c2b-5ad163ae9414" />
<img width="916" height="589" alt="04-First-Domain-User" src="https://github.com/user-attachments/assets/78c4621a-81d0-4e52-91d8-7227b5bcbe65" />

├── group-policy/
<img width="892" height="646" alt="12-GPO-Linked-to-Lab-Computers" src="https://github.com/user-attachments/assets/54e51eae-34d9-41b5-902c-d962d339c0fe" />
<img width="963" height="476" alt="14-CLIENT01-GPO-Applied" src="https://github.com/user-attachments/assets/dcdd0185-a79c-45b7-a2e6-c0987eaa3d29" />

├── smb/
<img width="891" height="610" alt="04-CLIENT01-SMB-Share-Access" src="https://github.com/user-attachments/assets/63fdb55d-3c3e-47e3-be9c-985b85563fec" />

├── troubleshooting/
<img width="983" height="256" alt="03-SERVER01-to-CLIENT01-Ping-Failed" src="https://github.com/user-attachments/assets/39625382-eca7-4c75-969a-0dccaa539c3b" />
<img width="994" height="282" alt="03-SERVER01-to-CLIENT01-Ping-SUCCESS" src="https://github.com/user-attachments/assets/49721974-deda-445e-95e1-ae8150223f8b" />


└── wireshark/
<img width="991" height="671" alt="02-ICMP-Ping-Capture" src="https://github.com/user-attachments/assets/25b84c7e-1690-4f96-8e3b-b70d96c5bda5" />
<img width="1021" height="789" alt="03-DNS-Query-Response" src="https://github.com/user-attachments/assets/f3a118a0-5983-4ff4-9428-0d4c255432d0" />
<img width="1049" height="822" alt="04-DHCP-DORA-Capture" src="https://github.com/user-attachments/assets/5251fad7-2bf6-46c9-b2e9-fe6e1765fbf1" />
<img width="1015" height="772" alt="06-TCP-Three-Way-Handshake" src="https://github.com/user-attachments/assets/fcc05278-619b-44f8-a54e-28db6d572403" />

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
