# Windows Server 2025 Cheat Sheet

> Home lab project. Examples use my fictional company SilverBank (domain `corp.silverbank.internal`, NetBIOS `SILVERBANK`); replace the names and IPs for other networks. No real company data or passwords are included.

Last updated: 2 November 2026

Everything from the Windows Server 2025 course in one place: for each topic, what it is, how to do it in the GUI, the PowerShell equivalent, and how to troubleshoot it.

## Contents

- [How to work like a sysadmin](#how-to-work-like-a-sysadmin)
- [Part 1: Build the server](#part-1-build-the-server)
- [Part 2: Management tools](#part-2-management-tools)
- [Part 3: Active Directory](#part-3-active-directory)
- [Part 4: Users, groups and computers](#part-4-users-groups-and-computers)
- [Part 5: DNS and DHCP](#part-5-dns-and-dhcp)
- [Part 6: Files, permissions and encryption](#part-6-files-permissions-and-encryption)
- [Part 7: Group Policy](#part-7-group-policy)
- [Part 8: Remote access and printing](#part-8-remote-access-and-printing)
- [Part 9: Backup and recovery](#part-9-backup-and-recovery)
- [Part 10: Monitoring and troubleshooting](#part-10-monitoring-and-troubleshooting)
- [Part 11: Hyper-V and containers](#part-11-hyper-v-and-containers)
- [Appendix: quick reference](#appendix-quick-reference)

## How to work like a sysadmin

Most problems are DNS, permissions, or something that changed recently. Check those first.

**The troubleshooting method**

1. **Identify the problem.** Ask: what exactly happens, when did it start, what changed, is it one user or many?
2. **Think of a cause.** Start with the simple ones: cable, IP, DNS, password, permissions.
3. **Test the theory** before changing anything.
4. **Fix it**, one change at a time.
5. **Verify** with the user that it works.
6. **Document** the problem and the fix in the ticket.

**Golden rules**

- Take a snapshot or backup before any big change.
- Never use a Domain Admin account for daily work: have a normal account and a separate admin account.
- Read the error message and the event log before guessing.
- Check `hostname` and `whoami` before running commands, so you know where you are and who you are.
- Escalate when you're stuck or the problem is outside your access; give the next person everything you've tried.
- Write down what you learn: the same ticket always comes back.

## Part 1: Build the server

### 1.1 Installation

Two editions of the same Windows: **Desktop Experience** has the full graphical desktop; **Server Core** has no desktop, uses less RAM and has fewer security updates, and is managed with PowerShell, `sconfig` and remote tools.

| | Desktop Experience | Server Core |
| --- | --- | --- |
| Interface | Full desktop, Server Manager | Command line and `sconfig` |
| Resources | More RAM and disk | Less |
| Best for | Learning, file/print servers | Domain controllers, DNS, DHCP |

**GUI steps**

1. Boot from the ISO and press a key when asked (UEFI shows this for only a few seconds).
2. Choose language and keyboard, then **Install now**.
3. Choose the edition, e.g. *Windows Server 2025 Standard Evaluation (Desktop Experience)*.
4. Choose **Custom: Install Windows only**.
5. No disk shown? Click **Load driver** and browse the VirtIO CD: `vioscsi\2k25\amd64` (Proxmox only).
6. Set the local Administrator password.

Evaluation licence: 180 days, starting at installation. Check it with `slmgr /dlv`.

On Proxmox, install `virtio-win-guest-tools.exe` from the VirtIO CD after setup: it adds the network driver and the QEMU guest agent.

**Terminal options:** on Server Core, run `sconfig` for a menu (computer name, network, updates, Remote Desktop). Fully automatic installs use an `autounattend.xml` answer file.

### 1.2 First configuration

Do these before anything else, and always before a server becomes a domain controller.

| Task | GUI | PowerShell |
| --- | --- | --- |
| Rename | Server Manager > Local Server > Computer name | `Rename-Computer -NewName "LN-DC01" -Restart` |
| Time zone | Settings > Time & language | `tzutil /s "GMT Standard Time"` |
| Updates | Settings > Windows Update | `sconfig` > Install updates |
| Remote Desktop | Server Manager > Local Server > Remote Desktop | see Part 8 |
| Check | Server Manager > Local Server | `hostname`, `ipconfig /all` |

### 1.3 Setting a static IP address

Servers get static IPs, so clients can always find them. PCs use DHCP.

**GUI:** `ncpa.cpl` > right-click the adapter > Properties > **Internet Protocol Version 4 (TCP/IPv4)** > Properties > *Use the following IP address* (IP, subnet mask, gateway) and *Use the following DNS server addresses*.

**PowerShell:**

```powershell
Get-NetAdapter                                     # find the adapter name
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.7.80 -PrefixLength 24 -DefaultGateway 192.168.7.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.82,127.0.0.1

# Go back to DHCP
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
```

`-PrefixLength 24` = subnet mask 255.255.255.0.

**Troubleshooting**

- *"The object already exists"*: the IP is already set. Remove it with `Remove-NetIPAddress -IPAddress <ip>` and the gateway with `Remove-NetRoute`.
- IP conflict: the address is inside the router's DHCP range. Keep static IPs outside it.
- Command changes nothing: wrong adapter name (check `Get-NetAdapter`) or PowerShell not run as Administrator.
- A `169.254.x.x` address means DHCP failed: check the cable/VLAN and the DHCP server.

### 1.4 Roles and features

- **Role**: a main job for the server (AD DS, DNS, DHCP, File Server, Print Server, Hyper-V).
- **Role service**: a part of a role (e.g. *File Server Resource Manager* inside File and Storage Services).
- **Feature**: a helper tool (e.g. Windows Server Backup, BitLocker, RSAT tools).

**GUI:** Server Manager > **Manage > Add Roles and Features** > Role-based installation > select the server > tick roles > tick features > Install. Remove with **Manage > Remove Roles and Features**.

**PowerShell:**

```powershell
Get-WindowsFeature                                   # everything; [X] = installed
Get-WindowsFeature | Where-Object Installed          # only installed
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Uninstall-WindowsFeature -Name DHCP -IncludeManagementTools
```

Common names: `AD-Domain-Services`, `DNS`, `DHCP`, `FS-FileServer`, `FS-Resource-Manager`, `Print-Server`, `Windows-Server-Backup`, `BitLocker`, `Hyper-V`, `Containers`.

**Troubleshooting:** `[ ] Available` means not installed on **this** server: check `hostname`. A yellow flag in Server Manager often means post-installation steps are waiting (e.g. promote to DC, complete DHCP configuration).

## Part 2: Management tools

### 2.1 Server Manager

The main dashboard on Desktop Experience servers. Each tile (role) shows Manageability, Events, Services, Performance and BPA results; red means one of them has a problem: click it to read the reason.

- **Add other servers:** Manage > Add Servers > search by name. Lets you manage several servers from one place.
- **Tools menu** (top right): opens every admin console.

**Troubleshooting:** a server shown in red with "WinRM Negotiate authentication error" is usually not in the domain yet (no trust). Join it, then press F5. A red Services tile is often harmless (an automatic service that stopped by itself): read the service name first.

### 2.2 MMC and Computer Management

**MMC (Microsoft Management Console)** is the frame that hosts the admin tools ("snap-ins"). Every `.msc` file is an MMC console.

- **Build your own console:** run `mmc` > File > **Add/Remove Snap-in** > add e.g. Active Directory Users and Computers, DNS, DHCP, Event Viewer > File > Save as `MyTools.msc`. One window for your daily tools.
- **Computer Management** (`compmgmt.msc`): Event Viewer, Shared Folders, Local Users and Groups, Device Manager, Disk Management and Services in one place.
- **Manage another computer:** in Computer Management, right-click the top item > **Connect to another computer**.

### 2.3 Windows Admin Center (WAC)

A website for managing servers from any browser, including a Mac. Install it on a member server, never on a domain controller (in my lab: LN-FS01).

- Open `https://LN-FS01` (port 443), log in with a domain admin account.
- **Add** > Servers > enter the server names.
- **Tools** per server: Overview, Roles & features, Services, Events, Files, Firewall, Updates, PowerShell, Remote Desktop.
- Extra tools such as DNS or DHCP are **extensions**: Settings (gear) > Extensions > Available > Install.

**Troubleshooting:** browser warning "not private" = self-signed certificate (normal in a lab; it expires after about 60 days). A WAC extension is only a remote control: the matching role must also be installed on the target server.

### 2.4 PowerShell essentials

Commands follow **Verb-Noun** with a hyphen: `Get-Service`, `Set-ADUser`, `New-SmbShare`. Run PowerShell **as Administrator** for system changes.

```powershell
Get-Help New-ADUser -Examples          # examples for any command
Get-Command *DnsServer*                # find commands by name
Get-Command -Module ActiveDirectory    # all commands in a module

Get-Service | Where-Object Status -eq 'Stopped'           # filter
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5   # sort and pick
Get-ADUser -Filter * | Export-Csv users.csv -NoTypeInformation      # export
```

**Remote PowerShell** (works between domain members):

```powershell
Enter-PSSession -ComputerName LN-FS01          # interactive, like SSH; exit with Exit-PSSession
Invoke-Command -ComputerName LN-FS01,LN-DC02 -ScriptBlock { Get-Service Spooler }
```

**Scripts:** save as `.ps1` and run with `.\script.ps1`. If scripts are blocked, check `Get-ExecutionPolicy` (servers default to RemoteSigned: local scripts run, downloaded ones must be signed or unblocked with `Unblock-File`).

**Troubleshooting:** "not recognized as the name of a cmdlet" = typo (missing hyphen) or the module isn't installed (e.g. AD commands need the RSAT AD tools). "Access denied" = not running as Administrator.

## Part 3: Active Directory

### 3.1 Concepts

Active Directory is the company's phone book (users, computers, groups) and security guard (logins and permissions).

| Term | Meaning | Lab example |
| --- | --- | --- |
| Forest | Top container; one or more domains that trust each other | corp.silverbank.internal |
| Domain | A security boundary with its own users and computers | corp.silverbank.internal |
| Domain Controller (DC) | Server that holds AD and checks logins | LN-DC01, LN-DC02 |
| OU | Folder for users/computers; GPOs link here | OU=Finance |
| Global Catalog (GC) | Searchable index of the forest; every DC should be one | both DCs |
| Site | A physical location (office) for replication and logins | Default-First-Site-Name |
| FSMO roles | 5 special jobs held by one DC each (e.g. PDC Emulator: time, lockouts, passwords) | `netdom query fsmo` |
| SYSVOL | Shared folder on every DC holding GPOs and logon scripts | `\\corp.silverbank.internal\SYSVOL` |

**Naming:** use a subdomain you control (e.g. `ad.company.co.uk`) or `.internal` in a lab. Avoid `.local` (conflicts with Apple Bonjour). Decide the server name **before** promoting: renaming a DC is difficult.

### 3.2 Create the forest (first DC)

**Before:** static IP, final name, correct time zone, updates done, snapshot taken.

**GUI:**

1. Add Roles and Features > **Active Directory Domain Services** > Install.
2. Yellow flag > **Promote this server to a domain controller**.
3. **Add a new forest** > root domain name.
4. Functional level Windows Server 2025; tick **DNS server** and **Global Catalog**; set the **DSRM password** (store it safely).
5. DNS delegation warning: normal, ignore. NetBIOS name: e.g. `SILVERBANK`.
6. Review Options > **View script** (save it). Prerequisites: warnings OK, errors not. Install; the server restarts.
7. Log in as `SILVERBANK\Administrator` (same password as the old local Administrator).

**PowerShell:**

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "corp.silverbank.internal" -DomainNetbiosName "SILVERBANK" `
  -InstallDns -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

**After:** set the DC's DNS to its partner DC first and `127.0.0.1` second, and add a forwarder for internet names: `Add-DnsServerForwarder -IPAddress 192.168.7.1`.

### 3.3 Add a second domain controller

Two DCs = if one fails, logins and DNS still work. They replicate every change to each other.

1. Prepare the server (static IP, name, time zone) with **DNS pointing to the first DC**.
2. Join the domain first, using the domain account: `Add-Computer -DomainName corp.silverbank.internal -Credential SILVERBANK\Administrator -Restart`.
3. Log in as `SILVERBANK\Administrator`, install AD DS, promote: **Add a domain controller to an existing domain**, tick DNS and GC, replicate from the first DC.

```powershell
Install-ADDSDomainController -DomainName "corp.silverbank.internal" -InstallDns `
  -Credential (Get-Credential SILVERBANK\Administrator) -ReplicationSourceDC "LN-DC01.corp.silverbank.internal" `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

Check: `Get-ADDomainController -Filter * | Select Name, IPv4Address` and `repadmin /replsummary` (0 fails).

### 3.4 AD management tools

| Tool | Run | Use for |
| --- | --- | --- |
| Active Directory Users and Computers | `dsa.msc` | Users, groups, computers, OUs, password resets |
| Active Directory Administrative Center | `dsac.exe` | Same, plus Recycle Bin and password policies; shows the PowerShell it runs |
| Active Directory Sites and Services | `dssite.msc` | Sites, subnets, replication |
| Active Directory Domains and Trusts | `domain.msc` | Trusts, UPN suffixes |
| Group Policy Management | `gpmc.msc` | GPOs (Part 7) |
| DNS Manager | `dnsmgmt.msc` | DNS (Part 5) |
| ADSI Edit | `adsiedit.msc` | Low-level editing: experts only |

In `dsa.msc`, turn on **View > Advanced Features** to see the Attribute Editor and more containers.

### 3.5 Health checks

```powershell
dcdiag /q                         # DC health; no output = healthy
repadmin /replsummary             # replication; 0 fails = good
repadmin /syncall /AdeP           # force replication now
netdom query fsmo                 # which DC holds each FSMO role
Get-Service NTDS, DNS, Netlogon   # core AD services must be Running
w32tm /query /status              # time source and sync
```

**Troubleshooting**

- "The server is not operational": DC off, AD services stopped, or the DC's network profile shows Public (`Get-NetConnectionProfile`; fix with `Restart-Service NlaSvc -Force`).
- Promotion "user name or password is incorrect": the wizard is using a local account. Use `SILVERBANK\Administrator`.
- Logins fail with a time error: clocks differ by more than 5 minutes (Kerberos). Check `w32tm /query /status`.

## Part 4: Users, groups and computers

### 4.1 OUs

OUs organise objects and are where Group Policies are linked. Typical layout: one OU per department for users, plus Workstations and Servers OUs for computers.

**GUI:** `dsa.msc` > right-click the domain > **New > Organizational Unit**. OUs are protected from accidental deletion (untick it in the object's Properties > Object tab, with Advanced Features on).

```powershell
$dc = "DC=corp,DC=silverbank,DC=internal"
New-ADOrganizationalUnit -Name "SilverBank" -Path $dc
"Finance","IT","Workstations","Servers" | ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=SilverBank,$dc" }
```

### 4.2 Users

**GUI:** `dsa.msc` > right-click the OU > **New > User** > first name, last name, logon name (e.g. `maria.popescu`) > password > tick **User must change password at next logon**.

```powershell
New-ADUser -Name "Maria Popescu" -GivenName Maria -Surname Popescu -SamAccountName maria.popescu `
  -UserPrincipalName maria.popescu@corp.silverbank.internal -Path "OU=Finance,OU=SilverBank,$dc" `
  -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true -ChangePasswordAtLogon $true
```

Bulk creation: `Import-Csv users.csv | ForEach-Object { New-ADUser ... }` (see `AD-Users-Computers-CheatSheet.md`).

**Daily helpdesk tasks**

| Task | GUI (`dsa.msc`) | PowerShell |
| --- | --- | --- |
| Reset password | Right-click user > Reset Password | `Set-ADAccountPassword maria.popescu -Reset -NewPassword (Read-Host -AsSecureString)` |
| Unlock | Properties > Account > Unlock account | `Unlock-ADAccount maria.popescu` |
| Find locked users | Saved query, or ADAC | `Search-ADAccount -LockedOut` |
| Disable a leaver | Right-click > Disable Account | `Disable-ADAccount maria.popescu` |
| Move to another OU | Right-click > Move | `Move-ADObject (Get-ADUser maria.popescu).DistinguishedName -TargetPath "OU=IT,OU=SilverBank,$dc"` |
| Account details | Properties | `Get-ADUser maria.popescu -Properties LockedOut, LastLogonDate, PasswordLastSet` |

Password rules (default): at least 7 characters (8+ recommended) and 3 of 4 types: capitals, small letters, numbers, symbols.

### 4.3 Groups and the AGDLP rule

| Scope | Contains | Used for |
| --- | --- | --- |
| **Global** (GG_) | Users from the same domain | Grouping people by role: GG_Finance |
| **Domain Local** (DL_) | Global groups | Giving permissions to a resource: DL_FinanceShare_RW |
| **Universal** | Groups from any domain in the forest | Multi-domain forests |

**Security** groups give permissions; **Distribution** groups are only for email.

**AGDLP:** **A**ccounts go into **G**lobal groups, Global groups go into **D**omain **L**ocal groups, and **P**ermissions are given to Domain Local groups. When someone joins Finance, add them to one group and they get every Finance permission.

**GUI:** right-click OU > **New > Group** > name, scope, type. Add members: group Properties > Members > Add.

```powershell
New-ADGroup -Name GG_Finance -GroupScope Global -GroupCategory Security -Path "OU=Finance,OU=SilverBank,$dc"
New-ADGroup -Name DL_FinanceShare_RW -GroupScope DomainLocal -GroupCategory Security -Path "OU=Finance,OU=SilverBank,$dc"
Add-ADGroupMember GG_Finance -Members maria.popescu
Add-ADGroupMember DL_FinanceShare_RW -Members GG_Finance
Get-ADPrincipalGroupMembership maria.popescu | Select Name   # a user's groups
```

Group changes apply at the user's **next logon** (the login ticket contains the groups).

### 4.4 Joining computers to the domain

**First, always:** set the computer's DNS to the domain controllers. The IP can stay DHCP.

**GUI (Windows 11 or Server):** Settings > System > About > **Domain or workgroup** > Change > Member of: **Domain** > `corp.silverbank.internal` > enter `SILVERBANK\Administrator` > "Welcome to the domain" > restart. Log in with **Other user** > `SILVERBANK\username`.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,192.168.7.82
Add-Computer -DomainName corp.silverbank.internal -Credential SILVERBANK\Administrator -Restart
Add-Computer -DomainName corp.silverbank.internal -NewName LN-CL02 -Credential SILVERBANK\Administrator -Restart  # rename + join
```

Windows 11 **Home** cannot join a domain: Pro, Enterprise or Education only.

**Organise:** move new computers to the right OU, or send them there automatically with `redircmp "OU=Workstations,OU=SilverBank,DC=corp,DC=silverbank,DC=internal"`.

**Troubleshooting**

| Error | Cause | Fix |
| --- | --- | --- |
| "An AD domain controller could not be contacted" | Computer DNS points to the router | Set DNS to the DCs; `nslookup corp.silverbank.internal` |
| "Trust relationship ... failed" | Computer password out of sync (old snapshot, cloned PC) | Log in locally, run `Test-ComputerSecureChannel -Repair -Credential SILVERBANK\Administrator` |
| Cloned PCs fight over one identity | Cloned without Sysprep | Sysprep the template (`sysprep /generalize /oobe /shutdown`) |
| Login says the account doesn't exist | Local account used by mistake | Type `SILVERBANK\username`; `.\name` = local account |

## Part 5: DNS and DHCP

### 5.1 DNS

DNS translates names to IP addresses. Active Directory depends on it: clients find domain controllers through special DNS records (SRV). If DNS is wrong, almost everything fails.

| Record | Does | Example |
| --- | --- | --- |
| A | Name > IPv4 | LN-FS01 > 192.168.7.81 |
| PTR | IP > name (reverse zone) | 192.168.7.81 > LN-FS01 |
| CNAME | Alias > another name | files > LN-FS01 |
| SRV | Where a service lives | `_ldap._tcp.dc._msdcs...` lists the DCs |
| MX | Mail server for the domain | |

**Zones:** a *forward lookup zone* holds name > IP; a *reverse lookup zone* holds IP > name. AD-integrated zones replicate automatically to every DC.

**GUI:** `dnsmgmt.msc`

- New record: Forward Lookup Zones > your zone > right-click > **New Host (A or AAAA)** (tick *Create associated PTR record*).
- Reverse zone: right-click Reverse Lookup Zones > **New Zone** > Primary, AD-integrated > IPv4 > network ID `192.168.7`.
- Forwarders (where to send internet names): right-click the server > Properties > **Forwarders** tab.
- Scavenging (delete old records): right-click the server > **Set Aging/Scavenging for All Zones**.

```powershell
Add-DnsServerResourceRecordA -ZoneName corp.silverbank.internal -Name web01 -IPv4Address 192.168.7.95 -CreatePtr
Add-DnsServerResourceRecordCName -ZoneName corp.silverbank.internal -Name files -HostNameAlias LN-FS01.corp.silverbank.internal
Add-DnsServerPrimaryZone -NetworkId "192.168.7.0/24" -ReplicationScope Domain     # reverse zone
Add-DnsServerForwarder -IPAddress 192.168.7.1
Get-DnsServerResourceRecord -ZoneName corp.silverbank.internal
Clear-DnsServerCache                 # on the DNS server
```

**Client-side checks:**

```powershell
nslookup LN-FS01                                           # what DNS answers
Resolve-DnsName LN-FS01.corp.silverbank.internal
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal   # can I find the DCs?
ipconfig /flushdns                                         # clear the client cache
ipconfig /registerdns                                      # re-register this computer's record
```

**Troubleshooting**

- Ping by IP works, by name fails: DNS. Check the client's DNS servers (`ipconfig /all`), then the record exists, then flush the cache.
- "Server: UnKnown" in nslookup: no reverse zone for the DNS server's IP. Harmless; create a reverse zone to fix it.
- Internet names fail but domain names work: missing or wrong forwarder.
- Old IP returned: stale record. Delete it or enable scavenging; `ipconfig /flushdns` on the client.

### 5.2 DHCP

DHCP gives computers an IP address, mask, gateway and DNS servers automatically. Only **one** DHCP server should answer on a network (or two in failover): two independent DHCP servers on the same LAN cause random wrong settings.

| Term | Meaning |
| --- | --- |
| Scope | The range of addresses for one subnet |
| Exclusion | Addresses inside the scope that DHCP must not give out (static devices) |
| Reservation | Always the same IP for one device (by MAC address): printers |
| Lease | How long a client keeps its address |
| Options | Extra settings: 003 Router (gateway), 006 DNS servers, 015 DNS domain name |
| Authorize | AD must approve a DHCP server before it hands out addresses |

**GUI:**

1. Add Roles and Features > **DHCP Server** > Install.
2. Yellow flag > **Complete DHCP configuration** (creates security groups and authorizes the server in AD).
3. `dhcpmgmt.msc` > IPv4 > right-click > **New Scope**: name, range, mask, exclusions, lease time, options (router, DNS, domain name) > activate.
4. Reservations: scope > Reservations > right-click > New Reservation (IP + MAC).
5. Failover: right-click the scope > **Configure Failover** > partner DC > Load balance 50/50.

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName LN-DC01.corp.silverbank.internal -IPAddress 192.168.7.80
Add-DhcpServerv4Scope -Name "Staff" -StartRange 10.10.20.100 -EndRange 10.10.20.200 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 10.10.20.0 -Router 10.10.20.1 -DnsServer 192.168.7.80,192.168.7.82 -DnsDomain corp.silverbank.internal
Add-DhcpServerv4ExclusionRange -ScopeId 10.10.20.0 -StartRange 10.10.20.100 -EndRange 10.10.20.109
Add-DhcpServerv4Reservation -ScopeId 10.10.20.0 -IPAddress 10.10.20.50 -ClientId "00-11-22-33-44-55" -Name "Printer01"
Get-DhcpServerv4Lease -ScopeId 10.10.20.0       # who has which address
```

**Client side:** `ipconfig /release` then `ipconfig /renew`; `ipconfig /all` shows the DHCP server and lease.

**Troubleshooting**

- Client gets `169.254.x.x`: no DHCP answer. Check the server is authorized and the scope active, the network/VLAN, and (across VLANs) the DHCP relay (`ip helper-address` on the router/switch).
- Clients get the wrong gateway or DNS: another DHCP server on the network (often the home router), or wrong scope options.
- Scope full: check leases, shorten the lease time or enlarge the range.
- DHCP icon red/down arrow: server not authorized (`Get-DhcpServerInDC`) or service stopped.

## Part 6: Files, permissions and encryption

### 6.1 File server and shares

Keep shared data on a separate data disk (e.g. `D:\Shares`), never on the system drive. Install the **File Server** role service (File and Storage Services).

**GUI:** Server Manager > File and Storage Services > **Shares** > Tasks > **New Share** > *SMB Share - Quick* > choose the folder > name > tick **Enable access-based enumeration** > permissions > Create. Quick alternative: folder Properties > Sharing > Advanced Sharing.

```powershell
New-Item -Path D:\Shares\Finance -ItemType Directory
New-SmbShare -Name Finance -Path D:\Shares\Finance -FullAccess "SILVERBANK\Domain Admins" -ChangeAccess "SILVERBANK\Domain Users" -FolderEnumerationMode AccessBased
Get-SmbShare                                   # list shares
Get-SmbOpenFile | Where-Object Path -like "*Budget*"   # who has a file open
Close-SmbOpenFile -FileId <id>                 # release a locked file
```

**Access-based enumeration (ABE):** users only see the folders they're allowed to open.

### 6.2 Share vs NTFS permissions

Two layers of permissions apply over the network, and the **most restrictive wins**.

| | Share permissions | NTFS permissions |
| --- | --- | --- |
| Applies | Only over the network | Network and local |
| Detail | Read, Change, Full Control | Fine-grained: Read, Write, Modify, Full Control, special |
| Best practice | Keep simple (e.g. Authenticated Users: Change) | Do the real control here, with groups |

**NTFS GUI:** folder Properties > **Security** > Edit > add the Domain Local group > choose Modify or Read. **Advanced** shows inheritance and **Effective Access** (test what a specific user can really do).

```powershell
icacls D:\Shares\Finance                                           # show permissions
icacls D:\Shares\Finance /grant "SILVERBANK\DL_FinanceShare_RW:(OI)(CI)M"   # Modify, inherited by subfolders/files
```

`(OI)(CI)` = applies to files and subfolders. `M` = Modify, `RX` = Read & execute, `F` = Full control.

**Rules:** give permissions to groups, never to individual users. Avoid "Full Control" for users (they could change the permissions themselves). A **Deny** beats an Allow.

### 6.3 Mapping drives

| Method | When |
| --- | --- |
| **Group Policy Drive Maps** (best) | Company-wide: the drive appears automatically for the right users (Part 7) |
| File Explorer > This PC > **Map network drive** | One user, quick fix |
| `net use S: \\LN-FS01\Finance /persistent:yes` | Command line |
| `New-PSDrive -Name S -PSProvider FileSystem -Root \\LN-FS01\Finance -Persist` | PowerShell |

Remove: `net use S: /delete`. List: `net use`.

**Troubleshooting "I can't open the S: drive"**

1. Can the PC reach the server? `Test-NetConnection LN-FS01 -Port 445`
2. Does the name resolve? `nslookup LN-FS01`
3. Is the user in the right group? `Get-ADPrincipalGroupMembership user`. If the group was just changed, the user must **log off and on**.
4. Check **Effective Access** on the folder for that user.
5. Red X on a mapped drive: often just disconnected; open it, or check the server is up.

### 6.4 File Server Resource Manager (FSRM)

Controls what goes on the file server: quotas (space limits), file screens (block file types) and storage reports.

**GUI:** install role service **File Server Resource Manager** > open `fsrm.msc`.

- **Quotas:** Quota Management > Quotas > Create Quota > path > template (e.g. 5 GB limit). *Hard* quota blocks writing; *soft* quota only warns.
- **File screens:** File Screening Management > File Screens > Create > path > template (e.g. block Audio and Video Files or Executable Files).
- **Reports:** Storage Reports Management > Generate Reports Now (large files, duplicate files, files by owner).

```powershell
Install-WindowsFeature FS-Resource-Manager -IncludeManagementTools
New-FsrmQuota -Path D:\Shares\Finance -Size 5GB
New-FsrmFileScreen -Path D:\Shares -IncludeGroup "Audio and Video Files" -Active
Get-FsrmQuota | Select Path, Size, Usage
```

**Troubleshooting:** "there isn't enough space" while the disk has space = the user hit a quota. "Access denied" when saving one file type = a file screen.

### 6.5 EFS and BitLocker

| | EFS (Encrypting File System) | BitLocker |
| --- | --- | --- |
| Encrypts | Individual files and folders, per user | A whole drive |
| Protects against | Other users on the same computer | A stolen disk or laptop |
| Key | The user's certificate | TPM chip + recovery key |
| Risk | Lost user profile/certificate = data lost | Lost recovery key = data lost |

**EFS GUI:** folder Properties > General > **Advanced** > tick *Encrypt contents to secure data*. Files show a padlock. **Back up the certificate** (certmgr.msc > Personal > Certificates > Export with private key). Commands: `cipher /e D:\Private` (encrypt), `cipher /d` (decrypt), `cipher /c file` (who can open it).

**BitLocker:**

1. Install the feature: `Install-WindowsFeature BitLocker -IncludeAllSubFeature -IncludeManagementTools -Restart`.
2. GUI: Control Panel > **BitLocker Drive Encryption** > Turn on BitLocker > save the **recovery key** > encrypt.
3. In companies: a GPO stores recovery keys in AD (*Store BitLocker recovery information in Active Directory Domain Services*); with the BitLocker Recovery Password Viewer feature, the key appears on the computer object in `dsa.msc`.

```powershell
manage-bde -status                                  # state of every drive
Enable-BitLocker -MountPoint "D:" -RecoveryPasswordProtector
(Get-BitLockerVolume -MountPoint C:).KeyProtector   # show protectors and recovery password
```

**Helpdesk classic:** "My laptop asks for a BitLocker recovery key" (after a BIOS update or hardware change). Find the 48-digit key in AD (computer object > BitLocker Recovery tab) using the Key ID shown on the screen.

## Part 7: Group Policy

A Group Policy Object (GPO) is a set of settings that applies automatically to users or computers in a site, domain or OU: passwords, drive maps, printers, security, software, desktop rules.

**Key ideas**

- **Two halves:** *Computer Configuration* applies to computers in the OU (at startup); *User Configuration* applies to users in the OU (at logon). A user setting in a GPO linked to an OU that only holds computers does nothing: the most common GPO mistake.
- **Order (LSDOU):** Local > Site > Domain > OU. The last one applied wins, so an OU GPO beats a domain GPO.
- **Enforced:** a GPO that lower GPOs can't override. **Block Inheritance:** an OU ignores GPOs from above (Enforced still wins).
- **Security filtering:** by default a GPO applies to Authenticated Users; you can limit it to a group.
- **Refresh:** every 90 minutes (plus a random offset), at startup and logon; or force it with `gpupdate /force`.
- **Default Domain Policy:** holds the domain password and account lockout policy. Don't put other settings in it: create new GPOs.

**GUI:** `gpmc.msc`

1. Right-click the target OU > **Create a GPO in this domain, and Link it here** > name it clearly (e.g. *Finance - Drive Map S*).
2. Right-click the GPO > **Edit** > change settings.
3. On a client: `gpupdate /force`, then log off and on for user settings.

**Useful GPOs for a small company**

| GPO | Where to find it |
| --- | --- |
| Password and lockout policy | Default Domain Policy > Computer > Policies > Windows Settings > Security Settings > Account Policies |
| Map a drive | User > Preferences > Windows Settings > **Drive Maps** > New > Mapped Drive (action Update; use Item-level targeting for a group) |
| Deploy a printer | User > Preferences > Control Panel Settings > Printers, or in Print Management > Deploy with Group Policy |
| Desktop wallpaper / lock screen | User > Policies > Administrative Templates > Desktop |
| Block USB storage | Computer > Policies > Administrative Templates > System > Removable Storage Access |
| Allow ping on servers | Computer > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security > Inbound Rules |
| Screen lock after inactivity | Computer > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Interactive logon: Machine inactivity limit |

```powershell
New-GPO -Name "Finance - Drive Map S" | New-GPLink -Target "OU=Finance,OU=SilverBank,DC=corp,DC=silverbank,DC=internal"
Get-GPO -All | Select DisplayName, GpoStatus
Backup-GPO -All -Path C:\GPOBackups
Invoke-GPUpdate -Computer LN-CL01 -Force        # remote refresh
```

**Troubleshooting "the GPO doesn't apply"**

1. On the client, `gpresult /r` (summary) or `gpresult /h C:\report.html` (full report). Is the GPO under *Applied* or *Denied* (with the reason)?
2. Is the user/computer in the OU where the GPO is linked? Computer settings need the **computer** in that OU, user settings need the **user** in it.
3. Is the link enabled, and does security filtering include the user/computer?
4. Is another GPO overriding it (order, Enforced, Block Inheritance)? GPMC > **Group Policy Results** wizard shows the winning GPO.
5. Run `gpupdate /force`; for some settings (drive maps, software) log off/on or restart.
6. Errors in Event Viewer: *Applications and Services Logs > Microsoft > Windows > GroupPolicy > Operational*.
7. Replication: a new GPO may not have reached the DC the client uses yet (`repadmin /replsummary`).

## Part 8: Remote access and printing

### 8.1 Remote Desktop (RDP)

Remote Desktop lets you control a server or PC over the network (port 3389). It's off by default.

**GUI (server):** Server Manager > Local Server > **Remote Desktop: Disabled** > Allow remote connections; keep **Network Level Authentication** ticked (login is checked before the session opens). On Windows 11: Settings > System > Remote Desktop > On.

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "SILVERBANK\maria.popescu"   # non-admins need this group
```

Connect: `mstsc` on Windows, **Windows App** on Mac. Username: `SILVERBANK\Administrator`.

**Remote Desktop Services (RDS)** is the bigger version: many users working on one server at once (Session Host), published apps, a web portal and a gateway. Server Manager > Add Roles and Features > **Remote Desktop Services installation** > Quick Start. It needs **RDS CALs** (licences) after a 120-day grace period. Standard admin RDP (2 sessions) doesn't.

**Troubleshooting**

| Problem | Check |
| --- | --- |
| Can't connect at all | RDP enabled? `Test-NetConnection server -Port 3389`; firewall rule; right IP/name |
| "The connection was denied because the user account is not authorized" | Add the user to Remote Desktop Users (admins are allowed by default) |
| Wrong username or password | Use `DOMAIN\user`; a saved old account in the client; keyboard layout |
| "An authentication error has occurred" / CredSSP | Missing updates on one side, or clock difference |
| Black screen | Wait, or press Ctrl+Alt+End; sign out the stuck session from Task Manager > Users on the server |

### 8.2 Network printing

A **print server** shares printers centrally: drivers are installed once, users connect to `\\LN-FS01\PrinterName`, and printers are deployed by Group Policy.

**GUI:**

1. Add Roles and Features > **Print and Document Services** > Print Server.
2. Open **Print Management** (`printmanagement.msc`) > Print Servers > your server > Printers > right-click **Add Printer** > *Add a new printer using an existing port* or *Add a TCP/IP or Web Services Printer by IP address* > enter the printer's IP > choose the driver > tick **Share this printer**.
3. Give the printer a fixed IP (DHCP reservation).
4. Deploy: right-click the printer > **Deploy with Group Policy** > choose a GPO > per user or per machine. Or GPO Preferences > Printers.

```powershell
Install-WindowsFeature Print-Server -IncludeManagementTools
Add-PrinterPort -Name "IP_10.10.20.50" -PrinterHostAddress 10.10.20.50
Add-Printer -Name "Finance-Printer" -DriverName "<exact driver name>" -PortName "IP_10.10.20.50" -Shared -ShareName "Finance-Printer"
Get-Printer; Get-PrintJob -PrinterName "Finance-Printer"
```

The driver must be installed on the server first (Print Management > Drivers > Add Driver).

**Troubleshooting "I can't print"**

1. Is it one user or everyone? Everyone = printer or server problem.
2. Printer on, paper, no error on its screen? Ping its IP.
3. Stuck queue: clear it.

```powershell
Get-PrintJob -PrinterName "Finance-Printer" | Remove-PrintJob
Restart-Service Spooler
# If jobs are still stuck:
Stop-Service Spooler; Remove-Item C:\Windows\System32\spool\PRINTERS\* -Force; Start-Service Spooler
```

4. Printer missing on the PC: check the GPO with `gpresult /r`, or add it manually from `\\LN-FS01`.
5. Driver install asks for admin rights: normal since the PrintNightmare security fixes. Deploy drivers centrally (print server/GPO) so users don't need admin rights.
6. Errors: Event Viewer > *Applications and Services Logs > Microsoft > Windows > PrintService*.

## Part 9: Backup and recovery

**Rules:** a backup you haven't restored isn't a backup. A backup on the same disk as the data is a single point of failure. Follow **3-2-1**: 3 copies, 2 types of storage, 1 off-site.

| Tool | Restores | Speed | Who |
| --- | --- | --- | --- |
| Volume Shadow Copy | Earlier versions of files | Seconds | The user |
| AD Recycle Bin | Deleted users, groups, OUs | Seconds | IT |
| Windows Server Backup | Files, volumes, system state, whole server | Minutes | IT |
| Proxmox snapshot/backup | The whole VM | Minutes to hours | IT |

### 9.1 Windows Server Backup

**GUI:** install the **Windows Server Backup** feature > open `wbadmin.msc` > **Backup Schedule** (daily, automatic) or **Backup Once**. Choose *Full server* or *Custom* (volumes, folders, **System state** for a DC). Destination: a dedicated disk (best), a volume, or a network share.

**Restore:** `wbadmin.msc` > **Recover** > choose the date > Files and folders / Volumes / System state > original or another location.

```powershell
Install-WindowsFeature Windows-Server-Backup
wbadmin get versions                                              # list backups
wbadmin start backup -backupTarget:E: -include:D:\Shares -quiet   # back up a folder to E:
wbadmin start systemstatebackup -backupTarget:E:                  # DC system state (AD, SYSVOL, registry)
```

### 9.2 Volume Shadow Copy (Previous Versions)

Snapshots of a volume at scheduled times (default 7:00 and 12:00 on weekdays). Users can restore files themselves.

**GUI (server):** right-click the volume in File Explorer > **Configure Shadow Copies** > select the volume > **Enable** > Settings (storage limit, schedule).

**User restore:** right-click the file or folder (also on a share) > **Restore previous versions** > pick a time > Open, Copy or Restore.

```powershell
vssadmin list shadows            # existing snapshots
vssadmin list shadowstorage      # space used
vssadmin create shadow /for=D:   # snapshot now (Windows Server)
```

Limits: stored on the same volume by default, max 64 snapshots, oldest deleted when space runs out. Not a replacement for backups.

### 9.3 AD Recycle Bin

Restores deleted AD objects with all their attributes and group memberships. Off by default, enable it once per forest (it can't be turned off again).

**GUI:** `dsac.exe` (AD Administrative Center) > click the domain > **Enable Recycle Bin** (right pane). Restore: open the **Deleted Objects** container > right-click the object > **Restore**.

```powershell
Enable-ADOptionalFeature -Identity 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target corp.silverbank.internal
Get-ADObject -Filter 'isDeleted -eq $true -and Name -like "*Maria*"' -IncludeDeletedObjects | Restore-ADObject
```

### 9.4 Snapshots (Proxmox / Hyper-V)

Take one before risky changes (updates, promotions, GPO experiments). A snapshot is **not** a long-term backup, and restoring an old snapshot of a DC can break replication in a multi-DC domain; in production, restore DCs from a system state backup instead.

**Troubleshooting**

- Backup fails: check free space on the target, the event log (*Applications and Services Logs > Microsoft > Windows > Backup*), and `vssadmin list writers` (every writer should be *Stable, No error*).
- No "Previous Versions" shown: shadow copies not enabled on that volume, or no snapshot yet since the file changed.
- Deleted user without Recycle Bin: restore from a system state backup (authoritative restore: advanced, ask a senior).

## Part 10: Monitoring and troubleshooting

### 10.1 Event Viewer

Every problem leaves a trace in the logs. Open `eventvwr.msc` (or Computer Management > Event Viewer). Start with **Critical** and **Error** events around the time the problem started.

| Log | Contains |
| --- | --- |
| Windows Logs > System | Windows: services, drivers, shutdowns, network |
| Windows Logs > Application | Program errors and crashes |
| Windows Logs > Security | Logins, failed passwords, lockouts, audits |
| Applications and Services Logs > Directory Service, DNS Server, DFS Replication | AD, DNS, SYSVOL replication (on DCs) |
| ...Microsoft > Windows > GroupPolicy / PrintService / Backup | GPO, printing, backup problems |

**Event IDs worth knowing**

| ID | Log | Meaning |
| --- | --- | --- |
| 4624 / 4625 | Security (the computer used) | Successful / failed logon |
| 4771 | Security (DC) | Kerberos pre-authentication failed: wrong password |
| 4740 | Security (PDC Emulator DC) | Account locked out; *Caller Computer Name* shows the source |
| 4720 / 4726 | Security (DC) | User account created / deleted |
| 4728 / 4732 | Security (DC) | Member added to a global / local group |
| 6005 / 6006 | System | Event log started (boot) / stopped (clean shutdown) |
| 6008 / 41 | System | Unexpected shutdown / power loss |
| 1074 | System | Who or what restarted the computer, and why |
| 7031 / 7034 | System | A service crashed |
| 1000 | Application | An application crashed (shows the program name) |

**GUI skills**

- **Filter Current Log** (right pane): by level, time, Event ID (e.g. `4740` or `6005,6006,1074`).
- **Create Custom View**: save a filter across several logs (e.g. "Lockouts and failed logons").
- **Attach Task to This Event**: run something or send an alert when an event happens.
- Look at the **Source** and **Event ID**, then search them: most errors are well documented.

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-1)}   # critical+errors, last 24h
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4740} -MaxEvents 10 | Select TimeCreated, Message
Get-WinEvent -ComputerName LN-DC01 -FilterHashtable @{LogName='System'; Id=1074} -MaxEvents 5
```

If expected Security events are missing, auditing may be off: `auditpol /get /category:*`.

### 10.2 Task Manager and Resource Monitor

**Task Manager** (Ctrl+Shift+Esc): the first look at a slow or frozen computer.

| Tab | Use |
| --- | --- |
| Processes | Which program uses CPU, memory, disk, network; End task on a frozen app |
| Performance | Live graphs; uptime; open **Resource Monitor** from here |
| Users | Who is logged on; sign out a stuck RDP session |
| Details | Process IDs, end process tree |
| Services | Start/stop services (Open Services for more) |
| Startup apps | Programs slowing down logon (Windows 11) |

**Resource Monitor** (`resmon`): which process uses which file, network connection or port; handy for "file in use" errors (CPU tab > Associated Handles > search the file name).

```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, Id, CPU, WS
Stop-Process -Name notepad
Get-Service | Where-Object {$_.StartType -eq 'Automatic' -and $_.Status -ne 'Running'}   # automatic services not running
```

### 10.3 Performance Monitor

`perfmon` measures performance over time, so you can prove what's slow instead of guessing.

- **Performance Monitor:** live graph; click **+** to add counters.
- **Data Collector Sets > User Defined > New:** record counters for hours or days, then view the report.
- **Data Collector Sets > System > System Performance:** a ready-made 60-second health report.

| Counter | Warning sign |
| --- | --- |
| `Processor(_Total)\% Processor Time` | Above 80% for a long time |
| `Memory\Available MBytes` | Below about 10% of total RAM |
| `Memory\Pages/sec` | Constantly high: not enough RAM |
| `PhysicalDisk\Avg. Disk sec/Read` and `/Write` | Above 0.020 (20 ms): slow disk |
| `LogicalDisk\% Free Space` | Below 10-15% |
| `Network Interface\Bytes Total/sec` | Close to the link speed |

```powershell
Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 5
Get-Counter '\Memory\Available MBytes'
```

### 10.4 Network troubleshooting commands

Work from the bottom up: link > IP > gateway > DNS > service port.

| Command | Answers |
| --- | --- |
| `ipconfig /all` | My IP, gateway, DNS servers, DHCP server, MAC |
| `ping 192.168.7.1` | Can I reach the gateway? (Windows blocks ping by default on many servers) |
| `ping LN-FS01` vs `ping 192.168.7.81` | Name fails but IP works = DNS problem |
| `tracert 8.8.8.8` | Where does the path break? |
| `nslookup name` / `Resolve-DnsName name` | What does DNS answer? |
| `Test-NetConnection server -Port 445` | Is a specific service reachable (445 SMB, 3389 RDP, 389 LDAP)? |
| `netstat -ano` / `Get-NetTCPConnection` | Open connections and listening ports, with process IDs |
| `arp -a` | IP-to-MAC table: IP conflicts |
| `w32tm /query /status` | Time source; Kerberos needs clocks within 5 minutes |
| `nltest /dsgetdc:corp.silverbank.internal` | Which DC this computer uses |

### 10.5 Common tickets and first checks

| Ticket | First checks | Usual fix |
| --- | --- | --- |
| "I can't log in" | Caps Lock, keyboard layout, `Get-ADUser -Properties LockedOut, Enabled, PasswordExpired` | Unlock, reset password, enable account |
| "My account keeps locking" | Event 4740 > Caller Computer Name | Old password saved on a phone, mapped drive or another PC |
| "No internet" | `ipconfig /all`: 169.254 = no DHCP; ping gateway; nslookup | Renew IP, fix DNS, check cable/Wi-Fi/VLAN |
| "I can't open the shared drive" | `Test-NetConnection server -Port 445`, group membership, Effective Access | Add to group (then log off/on), fix NTFS |
| "I deleted a file" | Previous Versions on the folder | Restore from shadow copy or backup |
| "I can't print" | One user or all? Queue, printer IP | Clear queue, restart Spooler, re-add printer |
| "My mapped drive / printer is missing" | `gpresult /r` | Fix GPO link/filter, `gpupdate /force`, log off/on |
| "Computer is slow" | Task Manager, free disk space, uptime | Close/remove heavy apps, restart, free space, check updates |
| "Trust relationship failed" | Logged in locally? | `Test-ComputerSecureChannel -Repair` or rejoin |
| "Server restarted by itself" | Events 1074, 6008, 41 | Updates schedule, power, hardware |
| "BitLocker asks for a key" | Key ID on screen | Recovery key from AD computer object |

## Part 11: Hyper-V and containers

### 11.1 Hyper-V

Microsoft's hypervisor: runs virtual machines on Windows Server, like Proxmox does on Linux. The ideas are the same: VMs, virtual switches, virtual disks, checkpoints (= snapshots).

**Inside Proxmox (nested virtualization):** the Windows VM needs CPU type `host` and nested virtualization enabled on the Proxmox host, otherwise the Hyper-V role refuses to install.

**GUI:** Add Roles and Features > **Hyper-V** > choose the network adapter for a virtual switch > restart. Open **Hyper-V Manager** (`virtmgmt.msc`) > New > Virtual Machine (Generation 2, memory, switch, new VHDX, ISO).

| Virtual switch | Connects VMs to |
| --- | --- |
| External | The physical network (like Proxmox `vmbr0`) |
| Internal | Each other and the host only |
| Private | Each other only (isolated lab) |

```powershell
Install-WindowsFeature Hyper-V -IncludeManagementTools -Restart
New-VMSwitch -Name External -NetAdapterName "Ethernet" -AllowManagementOS $true
New-VM -Name Test01 -Generation 2 -MemoryStartupBytes 2GB -NewVHDPath D:\VMs\Test01.vhdx -NewVHDSizeBytes 40GB -SwitchName External
Start-VM Test01; Get-VM
Checkpoint-VM -Name Test01 -SnapshotName "before-update"
```

### 11.2 Containers

A container packages an app with everything it needs and shares the host's kernel: lighter than a VM. Windows containers run Windows apps; Linux containers (Docker on Linux) are far more common in DevOps.

- **Process isolation:** shares the host kernel (fast, image version must match the host).
- **Hyper-V isolation:** each container gets a tiny VM (more secure, version-flexible).

**Setup outline:** install the **Containers** feature (`Install-WindowsFeature Containers -Restart`), install a container runtime following Microsoft's current documentation (e.g. Docker CE for Windows), then run an image such as `mcr.microsoft.com/windows/servercore:ltsc2025`.

```powershell
docker pull mcr.microsoft.com/windows/servercore:ltsc2025
docker run -it mcr.microsoft.com/windows/servercore:ltsc2025 cmd
docker ps -a
```

For helpdesk/sysadmin roles, knowing what containers are is usually enough; the Docker skills from Linux transfer directly.

## Appendix: quick reference

### Tool shortcuts (Win + R)

| Command | Opens |
| --- | --- |
| `servermanager` | Server Manager |
| `compmgmt.msc` | Computer Management |
| `dsa.msc` | AD Users and Computers |
| `dsac.exe` | AD Administrative Center |
| `dssite.msc` | AD Sites and Services |
| `gpmc.msc` | Group Policy Management |
| `dnsmgmt.msc` | DNS Manager |
| `dhcpmgmt.msc` | DHCP |
| `eventvwr.msc` | Event Viewer |
| `services.msc` | Services |
| `diskmgmt.msc` | Disk Management |
| `fsrm.msc` | File Server Resource Manager |
| `printmanagement.msc` | Print Management |
| `wbadmin.msc` | Windows Server Backup |
| `perfmon` / `resmon` | Performance / Resource Monitor |
| `ncpa.cpl` | Network connections |
| `sysdm.cpl` | System properties (name, domain, Remote Desktop) |
| `mstsc` | Remote Desktop client |
| `virtmgmt.msc` | Hyper-V Manager |

### Ports worth knowing

| Port | Service |
| --- | --- |
| 53 | DNS |
| 67/68 | DHCP |
| 88 | Kerberos (logins) |
| 123 | NTP (time) |
| 135 | RPC |
| 389 / 636 | LDAP / LDAPS (Active Directory) |
| 445 | SMB (file shares) |
| 3268 | Global Catalog |
| 3389 | Remote Desktop |
| 5985 / 5986 | WinRM (PowerShell remoting, Server Manager) |
| 443 | HTTPS (Windows Admin Center) |

### My lab values

| Item | Value |
| --- | --- |
| Domain / NetBIOS | corp.silverbank.internal / SILVERBANK |
| LN-DC01 | 192.168.7.80, first DC, DNS |
| LN-DC02 | 192.168.7.82, second DC, DNS, GC |
| LN-FS01 | 192.168.7.81, file server, Windows Admin Center |
| LN-CL01 | Windows 11 Pro, DHCP |
| Gateway | 192.168.7.1 |
| Admin account | `SILVERBANK\Administrator` |
