# Network+ VirtualBox Small Business Lab

## Overview

This project documents a hands-on small-business network lab built in Oracle VirtualBox while studying for the CompTIA Network+ certification.

The lab uses Windows Server 2025 and Windows 11 to practice IPv4 addressing, DHCP, DNS, Active Directory, Group Policy, SMB file sharing, Windows Firewall, troubleshooting, and Wireshark packet analysis.

## Network Topology

```
<img width="1536" height="1024" alt="ChatGPT Image Sep 28, 2026, 12_05_18 PM" src="https://github.com/user-attachments/assets/d9c7291e-1d69-4deb-ab3b-3e3c898fe39e" />

```

### Core Systems

| System | Role | Address |
|---|---|---|
| SERVER01 | Windows Server 2025 / Domain Controller / DNS / DHCP | `192.168.58.3/27` |
| CLIENT01 | Windows 11 domain client | DHCP |
| Planned Gateway | Reserved for a future router | `192.168.58.21` |

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

This keeps internet connectivity separate from the internal practice network.

### Virtual Machine Setup

![SERVER01 VirtualBox configuration](02%20VirtualBox%20Setup/02-SERVER01-VM-Configuration.png)

## Windows Server Configuration

SERVER01 was configured with:

```text
IP Address:      192.168.58.3
Subnet Mask:     255.255.255.224
Preferred DNS:   192.168.58.3
```

SERVER01 was later promoted to the first domain controller for:

```text
corp.netplus.test
```

NetBIOS domain:

```text
CORP
```

### SERVER01 Static Address

![SERVER01 static IP](04%20Window%20Server/05-SERVER01-Static-IP.png)

## DHCP

Installed the DHCP Server role and created the `NETPLUS-LAN` scope.

```text
Start:       192.168.58.1
End:         192.168.58.30
Mask:        255.255.255.224
```

Excluded addresses:

```text
192.168.58.3   SERVER01
192.168.58.21  Planned gateway
```

CLIENT01 was changed from static addressing to DHCP and received its configuration from SERVER01.

### DHCP Scope

![NETPLUS DHCP scope](06%20DHCP/03-NETPLUS-DHCP-Scope.png)

### CLIENT01 DHCP Lease

![CLIENT01 DHCP lease](06%20DHCP/04-CLIENT01-DHCP-Lease.png)

### DHCP Troubleshooting

After SERVER01 was promoted into Active Directory, CLIENT01 fell back to an APIPA address:

```text
169.254.x.x
```

Troubleshooting showed that DHCP authorization was associated with SERVER01's VirtualBox NAT address instead of the internal lab address. DHCP authorization was corrected to `192.168.58.3`, the DHCP service was restarted, and CLIENT01 received a valid `192.168.58.x/27` lease again.

## DNS

Installed the DNS Server role and created a practice forward lookup zone:

```text
netplus.test
```

A record:

```text
server01.netplus.test -> 192.168.58.3
```

Created reverse lookup zone:

```text
58.168.192.in-addr.arpa
```

PTR record:

```text
192.168.58.3 -> server01.netplus.test
```

Active Directory later added:

```text
corp.netplus.test
_msdcs.corp.netplus.test
```

SRV records for services such as LDAP and Kerberos were verified.

### DNS A Record

![SERVER01 DNS A record](07%20DNS/02-SERVER01-A-Record.png)

### DNS Lookup from CLIENT01

![CLIENT01 DNS lookup](07%20DNS/03-CLIENT01-DNS-Lookup.png)

## Active Directory

Installed Active Directory Domain Services and promoted SERVER01 to the first domain controller.

Created organizational units:

```text
Lab-Users
Lab-Computers
Lab-Groups
```

Created:

```text
User:           CORP\labuser1
Security Group: CORP\Lab-Staff
```

Added `labuser1` to `Lab-Staff` and joined CLIENT01 to `corp.netplus.test`.

### Domain Verified

![Active Directory domain](Active%20Directory/03-AD-Domain-Verified.png.png)

### CLIENT01 Domain Join

![CLIENT01 joined to domain](Active%20Directory/06%20CLIENT01-Domain-Join-Success.png)

### OU Organization

![Active Directory OU organization](Active%20Directory/09-AD-OU-Organization.png)

### Lab-Staff Security Group

![Lab-Staff security group](Active%20Directory/10-Lab-Staff-Security-Group.png)

## SMB File Sharing

Created:

```text
C:\LabShare
```

Shared as:

```text
\\server01\LabShare
```

Access was assigned through `CORP\Lab-Staff` using SMB share permissions and NTFS permissions.

`CORP\labuser1` successfully opened the share from CLIENT01.

### SMB Access from CLIENT01

![CLIENT01 SMB share access](08%20Testing/04-CLIENT01-SMB-Share-Access.png)

### Permission Troubleshooting

Initial access failed. Troubleshooting included checking:

- Security-group naming
- User group membership
- SMB share permissions
- NTFS permissions
- The user's Windows logon token

After group membership and permissions were corrected, signing out and back in refreshed the user's security token and access succeeded.

## Group Policy

Created a GPO named:

```text
Lab-Computers-Firewall
```

Linked it to the `Lab-Computers` OU.

The GPO created an inbound Windows Firewall rule:

```text
NETPLUS Allow ICMPv4
```

Rule settings:

- Direction: Inbound
- Protocol: ICMPv4
- Type: Echo Request
- Remote subnet: `192.168.58.0/27`
- Action: Allow
- Profile: Domain

CLIENT01 refreshed policy with:

```cmd
gpupdate /force
```

Policy application was verified with:

```cmd
gpresult /scope computer /r
```

### GPO Linked to Lab-Computers

![GPO linked to Lab-Computers](Active%20Directory/12-GPO-Linked-to-Lab-Computers.png)

### GPO Applied on CLIENT01

![GPO applied on CLIENT01](Active%20Directory/14-CLIENT01-GPO-Applied.png)

## Connectivity Testing and Troubleshooting

An early ping test failed because Windows Firewall blocked inbound ICMP traffic. After the firewall configuration was corrected, connectivity succeeded.

### Failed Ping

![Failed ping test](08%20Testing/01-CLIENT01-to-SERVER01-Ping-FAILED.png)

### Successful Ping

![Successful ping test](08%20Testing/02-CLIENT01-to-SERVER01-Ping-SUCCESS.png)

Other issues diagnosed during the lab included:

- PowerShell commands failing without administrator privileges
- CLIENT01 receiving an APIPA address
- DNS tests using the wrong network interface
- DHCP authorization using the NAT address
- SMB access failing because of permissions and group-token refresh

## Wireshark Packet Analysis

Wireshark was installed on CLIENT01 and captures were taken on `NETPLUS-LAN`.

### ICMP

Display filter:

```text
icmp
```

Observed ICMP Echo Request and Echo Reply traffic.

![ICMP packet capture](09%20Wireshark/02-ICMP-Ping-Capture.png)

### DNS

Display filter:

```text
dns
```

Observed the query for `server01.netplus.test` and the DNS response returning `192.168.58.3`.

![DNS query and response](09%20Wireshark/03-DNS-Query-Response.png)

### DHCP DORA

Display filter:

```text
dhcp
```

Observed the four DHCP messages:

```text
Discover
Offer
Request
Acknowledge
```

![DHCP DORA capture](09%20Wireshark/04-DHCP-DORA-Capture.png)

### ARP

Cleared the ARP cache, generated local traffic, and observed the ARP request and reply used to map SERVER01's IPv4 address to its MAC address.

### TCP Three-Way Handshake

Generated an SMB connection test:

```powershell
Test-NetConnection 192.168.58.3 -Port 445
```

Display filter:

```text
tcp.port == 445
```

Observed:

```text
SYN
SYN-ACK
ACK
```

![TCP three-way handshake](09%20Wireshark/06-TCP-Three-Way-Handshake.png)

## Repository Evidence Folders

- [VirtualBox Setup](02%20VirtualBox%20Setup/)
- [Windows Server](04%20Window%20Server/)
- [Windows Client](05%20Window%20Clients/)
- [DHCP](06%20DHCP/)
- [DNS](07%20DNS/)
- [Testing](08%20Testing/)
- [Wireshark](09%20Wireshark/)
- [Troubleshooting](10%20Troubleshooting/)
- [Active Directory and Group Policy](Active%20Directory/)

## Skills Demonstrated

- Oracle VirtualBox
- Virtual networking
- NAT and internal networks
- IPv4 addressing
- CIDR and `/27` subnetting
- Static addressing
- DHCP scopes, exclusions, leases, options, and authorization
- APIPA troubleshooting
- DNS A, PTR, and SRV records
- Active Directory Domain Services
- Domain controllers
- Organizational Units
- Domain users and security groups
- Windows domain joining
- Group Policy
- Windows Defender Firewall
- SMB file sharing
- NTFS permissions
- ICMP
- ARP
- TCP
- DHCP DORA
- Wireshark packet analysis
- PowerShell
- Network troubleshooting

## Resume Project Entry

**Virtual Small Business Network Lab | Oracle VirtualBox, Windows Server 2025, Windows 11**

- Built a virtual client/server network using a `/27` IPv4 subnet with separate NAT and internal network interfaces.
- Configured Windows Server DHCP, DNS, Active Directory Domain Services, Group Policy, SMB sharing, NTFS permissions, and domain authentication.
- Joined a Windows 11 client to Active Directory and managed access through domain security groups.
- Diagnosed DHCP authorization, APIPA, DNS interface selection, Windows Firewall, and SMB permission issues using PowerShell and Windows networking tools.
- Used Wireshark to analyze ICMP, DNS, DHCP DORA, ARP, and TCP three-way-handshake traffic.

## Project Status

Core lab build and documentation complete. The repository now contains the README and supporting screenshots from the lab.
