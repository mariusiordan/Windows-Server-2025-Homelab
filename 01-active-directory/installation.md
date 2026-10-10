# Active Directory Installation Guide

> Home lab project. SilverBank is a fictional company. This guide builds a two-DC Active Directory domain on Windows Server 2025: LN-DC01 creates the forest `corp.silverbank.internal`, then LN-DC02 joins as a second domain controller. Values are from my home lab; change them for your own network.

Last updated: 10 October 2026

## Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Prepare the server](#step-1-prepare-the-server)
3. [Step 2: Install the AD DS role](#step-2-install-the-ad-ds-role)
4. [Step 3: Promote LN-DC01 (new forest)](#step-3-promote-ln-dc01-new-forest)
5. [Step 4: Check LN-DC01 and set up DNS](#step-4-check-ln-dc01-and-set-up-dns)
6. [Step 5: Add LN-DC02](#step-5-add-ln-dc02)
7. [Step 6: Final DNS settings and replication check](#step-6-final-dns-settings-and-replication-check)
8. [Troubleshooting](#troubleshooting)

## Prerequisites

Finish every item before promoting. Renaming a server after it becomes a DC is difficult.

- [ ] Windows Server 2025 installed (Desktop Experience or Core), fully updated and restarted
- [ ] VirtIO guest tools installed; QEMU guest agent running (`Get-Service QEMU-GA`)
- [ ] Final computer name set (LN-DC01, LN-DC02)
- [ ] Static IP address outside the router's DHCP range
- [ ] Correct time zone (GMT Standard Time). Kerberos fails if clocks differ by more than 5 minutes
- [ ] Domain name chosen: a subdomain, never `.local` (conflicts with Bonjour on Macs)
- [ ] Proxmox snapshot taken (`before-AD`)
- [ ] Somewhere safe to store the DSRM password

| Setting | LN-DC01 | LN-DC02 |
| --- | --- | --- |
| IP address | 192.168.7.80/24 | 192.168.7.82/24 |
| Gateway | 192.168.7.1 | 192.168.7.1 |
| DNS before promotion | 192.168.7.1 (router) | 192.168.7.80 (LN-DC01) |
| Domain | corp.silverbank.internal | corp.silverbank.internal |
| NetBIOS name | SILVERBANK | SILVERBANK |

## Step 1: Prepare the server

Run these in PowerShell as Administrator on LN-DC01. The server restarts after the rename.

1. Check the adapter name:

   ```powershell
   Get-NetAdapter
   ```

2. Set the static IP and DNS (replace `"Ethernet"` with the real adapter name):

   ```powershell
   New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.7.80 -PrefixLength 24 -DefaultGateway 192.168.7.1
   Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.1
   ```

3. Set the time zone:

   ```powershell
   tzutil /s "GMT Standard Time"
   ```

4. Enable Remote Desktop (optional, for access from another computer):

   ```powershell
   Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
   Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
   ```

5. Rename and restart:

   ```powershell
   Rename-Computer -NewName "LN-DC01" -Restart
   ```

After the restart, confirm with `hostname` and `ipconfig /all`.

## Step 2: Install the AD DS role

This installs the software only. The server is not a domain controller until Step 3.

**PowerShell:**

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

**GUI:** Server Manager > Manage > Add Roles and Features > Role-based installation > select the server > tick **Active Directory Domain Services** > Add Features > Install.

Check:

```powershell
Get-WindowsFeature -Name AD-Domain-Services
```

`[X] ... Installed` means it worked. `[ ] ... Available` means it is not installed on this server: check `hostname`.

## Step 3: Promote LN-DC01 (new forest)

Server Manager > yellow flag > **Promote this server to a domain controller**. The server restarts automatically at the end.

| Wizard page | Choice | Note |
| --- | --- | --- |
| Deployment Configuration | Add a new forest; root domain `corp.silverbank.internal` | |
| Domain Controller Options | Functional level Windows Server 2025; DNS server and Global Catalog ticked; DSRM password | Store the DSRM password safely |
| DNS Options | Warning: "A delegation for this DNS server cannot be created" | Normal; ignore |
| Additional Options | NetBIOS name: change CORP to `SILVERBANK` | Logins become `SILVERBANK\user` |
| Paths | Keep defaults | |
| Review Options | Click **View script** and save it | The wizard is one PowerShell command |
| Prerequisites Check | Warnings are OK; errors are not | |

The same promotion in PowerShell (compare the mode values with the wizard's View script):

```powershell
Install-ADDSForest `
  -DomainName "corp.silverbank.internal" `
  -DomainNetbiosName "SILVERBANK" `
  -ForestMode "Win2025" `
  -DomainMode "Win2025" `
  -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

After the restart, log in as `SILVERBANK\Administrator`. The domain admin password is the old local Administrator password.

## Step 4: Check LN-DC01 and set up DNS

All three services must show Running, and `dcdiag /q` should print nothing.

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode
Get-Service NTDS, DNS, Netlogon
dcdiag /q
Get-NetConnectionProfile
```

`NetworkCategory` should be **DomainAuthenticated**. If it shows Public, run `Restart-Service NlaSvc -Force`.

LN-DC01 is now the DNS server, so it must ask itself, and forward internet names to the router:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,127.0.0.1
Add-DnsServerForwarder -IPAddress 192.168.7.1
```

Take a Proxmox snapshot now (`after-DC01`).

## Step 5: Add LN-DC02

Join the domain first, then promote. This avoids the wizard using the local account.

1. Prepare LN-DC02 as in Step 1, with IP **192.168.7.82** and DNS **192.168.7.80** (LN-DC01). Check that the domain is found:

   ```powershell
   nslookup corp.silverbank.internal
   ```

2. Install the AD DS role (Step 2).

3. Join the domain with the domain account:

   ```powershell
   Add-Computer -DomainName corp.silverbank.internal -Credential SILVERBANK\Administrator -Restart
   ```

4. Log in as `SILVERBANK\Administrator` (Other user).

5. Promote: Server Manager > yellow flag > Promote this server to a domain controller.

| Wizard page | Choice |
| --- | --- |
| Deployment Configuration | Add a domain controller to an existing domain; `corp.silverbank.internal`; credentials `SILVERBANK\Administrator` |
| Domain Controller Options | DNS server and Global Catalog ticked; site Default-First-Site-Name; DSRM password |
| Additional Options | Replicate from: LN-DC01 |
| Paths, Review, Prerequisites | Defaults; warnings OK |

The same promotion in PowerShell:

```powershell
Install-ADDSDomainController `
  -DomainName "corp.silverbank.internal" `
  -InstallDns `
  -Credential (Get-Credential SILVERBANK\Administrator) `
  -ReplicationSourceDC "LN-DC01.corp.silverbank.internal" `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

## Step 6: Final DNS settings and replication check

Each DC asks its partner first and itself second. The setup is finished when replication shows 0 fails.

| Server | Preferred DNS | Alternate DNS |
| --- | --- | --- |
| LN-DC01 | 192.168.7.82 | 127.0.0.1 |
| LN-DC02 | 192.168.7.80 | 127.0.0.1 |

```powershell
# On LN-DC01
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.82,127.0.0.1
# On LN-DC02
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,127.0.0.1
```

Check both DCs and replication:

```powershell
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address
repadmin /replsummary
```

Visible test: create a user on LN-DC01, wait a minute, and check that it appears on LN-DC02.

Take snapshots of both DCs.

## Troubleshooting

Most errors come from DNS: check that the machine uses a DC as DNS before anything else.

| Error | Likely cause | Fix |
| --- | --- | --- |
| "An AD domain controller could not be contacted" | DNS points to the router | Set DNS to the DCs; `nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal` must list the DCs |
| "The server is not operational" | DC off, AD services stopped, or network profile Public | `Get-Service NTDS, DNS, Netlogon`; `Get-NetConnectionProfile`; `Restart-Service NlaSvc -Force` |
| "Error getting the list of sites ... user name or password is incorrect" | Wizard uses the local account | Credentials > Change... > `SILVERBANK\Administrator`, or join the domain first |
| User name or password incorrect at login | Local account used, keyboard layout (UK/US swaps @ and "), Caps Lock, or clock difference | Type `SILVERBANK\Administrator`; check `Get-Date` on both machines |
| Ping by name fails, by IP works | No DNS record, or wrong DNS on the client | `ipconfig /registerdns` on the target; `ipconfig /flushdns` on the client |
| Ping fails but services work | Windows Firewall blocks ICMP | `Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"` |
| "Server: UnKnown" in nslookup | No reverse lookup zone | Harmless; create a reverse zone later |

Useful tools: `dcdiag /q`, `repadmin /replsummary`, `Test-NetConnection <DC> -Port 389`, `dsa.msc`, `dnsmgmt.msc`.
