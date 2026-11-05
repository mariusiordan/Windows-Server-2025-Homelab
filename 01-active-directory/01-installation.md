# 1. Installation — First Domain Controller (LN-DC01)

> Build the first domain controller and create the forest `corp.silverbank.internal`.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← Back to Active Directory](README.md) · Next: [Adding a second DC →](02-second-domain-controller.md)

---

## Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Prepare the server](#step-1-prepare-the-server)
3. [Step 2: Install the AD DS role](#step-2-install-the-ad-ds-role)
4. [Step 3: Promote LN-DC01 (new forest)](#step-3-promote-ln-dc01-new-forest)
5. [Step 4: Check the new domain](#step-4-check-the-new-domain)
6. [Step 5: Set DNS on LN-DC01](#step-5-set-dns-on-ln-dc01)
7. [Step 6: Create the OU structure](#step-6-create-the-ou-structure)

---

## Prerequisites

Finish every item **before** promoting. Renaming a server after it becomes a DC is difficult.

- [ ] Windows Server 2025 installed (**Desktop Experience** = with GUI), fully updated and restarted
- [ ] VirtIO guest tools installed; QEMU guest agent running
- [ ] Final computer name chosen (`LN-DC01`)
- [ ] Static IP address **outside** the router's DHCP range (`.110–.199`)
- [ ] Time zone set to **GMT Standard Time**
- [ ] Domain name chosen: a subdomain you control, never `.local` (conflicts with Bonjour on Macs)
- [ ] Proxmox snapshot taken (`before-AD`)
- [ ] A safe place for the **DSRM password** (password manager)

| Setting | LN-DC01 |
|---|---|
| IP address | 192.168.7.80 / 24 |
| Gateway | 192.168.7.1 |
| DNS **before** promotion | 192.168.7.1 (router) |
| Domain | corp.silverbank.internal |
| NetBIOS name | SILVERBANK |

---

## Step 1: Prepare the server

### GUI

Open **Server Manager → Local Server**. Everything below is on this page. Click the blue text next to each item.

**1. Static IP address**
1. Click the link next to **Ethernet** (it says "IPv4 address assigned by DHCP").
2. Right-click the adapter → **Properties**.
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
4. Choose **Use the following IP address**:
   - IP address: `192.168.7.80`
   - Subnet mask: `255.255.255.0`
   - Default gateway: `192.168.7.1`
   - Preferred DNS: `192.168.7.1`
5. **OK → Close**.

> ⚠️ **Hyper-V trap (real problem in this lab):** if Hyper-V is installed on a server, Windows creates a **vEthernet** virtual adapter, and the IP and DNS settings can end up on the wrong adapter. Check that your settings are on the adapter that is really connected. Better still: **don't install Hyper-V on a domain controller.**

**2. Time zone**
1. Click the link next to **Time zone**.
2. **Change time zone…** → `(UTC+00:00) Dublin, Edinburgh, Lisbon, London` → **OK**.

> ⚠️ **Real problem in this lab:** new servers default to **Pacific Time**. It caused certificates to be "not yet valid" for 7 hours. Always check this first.

**3. Remote Desktop (optional)**
1. Click the link next to **Remote Desktop** → **Allow remote connections to this computer** → **OK**.

**4. Computer name**
1. Click the link next to **Computer name** → **Change…**
2. Computer name: `LN-DC01` → **OK** → restart when asked.

### PowerShell (optional)

```powershell
# Find the adapter name
Get-NetAdapter

# Static IP and DNS (replace "Ethernet" with the real adapter name)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.7.80 -PrefixLength 24 -DefaultGateway 192.168.7.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.1

# Time zone
tzutil /s "GMT Standard Time"

# Remote Desktop
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Rename and restart
Rename-Computer -NewName "LN-DC01" -Restart
```

### Check

| GUI | PowerShell / command |
|---|---|
| Server Manager → Local Server shows the new name, IP and time zone | `hostname` · `ipconfig /all` · `tzutil /g` |

---

## Step 2: Install the AD DS role

This installs the **software only**. The server is **not** a domain controller until Step 3.

### GUI

1. **Server Manager → Manage → Add Roles and Features**.
2. **Before You Begin** → Next.
3. **Installation Type:** Role-based or feature-based installation → Next.
4. **Server Selection:** select `LN-DC01` → Next.
5. **Server Roles:** tick **Active Directory Domain Services** → **Add Features** in the pop-up → Next.
6. **Features** → Next. **AD DS** page → Next.
7. **Confirmation** → **Install**.
8. Wait for "Installation succeeded" → **Close**.

### PowerShell (optional)

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### Check

| GUI | PowerShell |
|---|---|
| Server Manager shows **AD DS** in the left menu and a yellow ⚠️ flag | `Get-WindowsFeature -Name AD-Domain-Services` → `[X] Installed` |

---

## Step 3: Promote LN-DC01 (new forest)

### GUI

Server Manager → click the **yellow flag** ⚠️ → **Promote this server to a domain controller**.

| Wizard page | What to choose | Why / note |
|---|---|---|
| **Deployment Configuration** | **Add a new forest** · Root domain name: `corp.silverbank.internal` | First DC = new forest |
| **Domain Controller Options** | Forest + domain functional level: **Windows Server 2025** · ✅ DNS server · ✅ Global Catalog · type the **DSRM password** | Save the DSRM password in your password manager |
| **DNS Options** | Warning: *"A delegation for this DNS server cannot be created"* | Normal in a lab — ignore |
| **Additional Options** | NetBIOS name: change `CORP` to **`SILVERBANK`** | Logins become `SILVERBANK\user` |
| **Paths** | Keep the defaults | Database, logs, SYSVOL |
| **Review Options** | Click **View script** and save it | The wizard is really one PowerShell command |
| **Prerequisites Check** | **Warnings** are OK · **Errors** are not | Then click **Install** |

The server **restarts automatically**. Log in as **`SILVERBANK\Administrator`** — the password is the old local Administrator password.

### PowerShell (optional)

The same promotion as one command (compare it with the wizard's **View script**):

```powershell
Install-ADDSForest `
  -DomainName "corp.silverbank.internal" `
  -DomainNetbiosName "SILVERBANK" `
  -ForestMode "Win2025" `
  -DomainMode "Win2025" `
  -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

---

## Step 4: Check the new domain

### GUI

| Where | What you should see |
|---|---|
| **Server Manager → Tools → Active Directory Users and Computers** | The domain `corp.silverbank.internal` with Users, Computers, Domain Controllers |
| **Domain Controllers** container | `LN-DC01` listed |
| **Server Manager → Tools → DNS** | Forward Lookup Zones → `corp.silverbank.internal` and `_msdcs.corp.silverbank.internal` |
| **Services** (`services.msc`) | **Active Directory Domain Services**, **DNS Server**, **Netlogon** = Running |
| Network icon in the taskbar | Network shows as **corp.silverbank.internal** (Domain network), not "Public" |

### PowerShell (optional)

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode
Get-Service NTDS, DNS, Netlogon
dcdiag /q                    # prints nothing = no errors
Get-NetConnectionProfile     # NetworkCategory should be DomainAuthenticated
```

If the network shows **Public**: `Restart-Service NlaSvc -Force`

---

## Step 5: Set DNS on LN-DC01

LN-DC01 is now a **DNS server**, so it must ask **itself** for domain names, and send internet names to the router (a **forwarder**).

### GUI

**1. DNS client settings**
1. Adapter → **Properties → IPv4 → Properties**.
2. Preferred DNS: `127.0.0.1` (for now — this changes when LN-DC02 exists).
3. **OK**.

**2. Forwarder**
1. **Server Manager → Tools → DNS**.
2. Right-click **LN-DC01** → **Properties → Forwarders** tab → **Edit…**
3. Add `192.168.7.1` → **OK → OK**.

### PowerShell (optional)

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
Add-DnsServerForwarder -IPAddress 192.168.7.1
```

### Check

```powershell
nslookup corp.silverbank.internal     # answers with 192.168.7.80
nslookup bbc.co.uk                     # internet names still work (through the forwarder)
```

📸 Take a Proxmox snapshot: `after-DC01`.

---

## Step 6: Create the OU structure

**OUs** (Organizational Units) are folders for users, computers and groups. They make management and Group Policy easier.

### GUI

1. **Active Directory Users and Computers**.
2. Right-click the domain → **New → Organizational Unit** → `SilverBank` → **OK**.
   - Leave **Protect container from accidental deletion** ticked.
3. Inside `SilverBank`, create: `Users`, `Computers`, `Servers`, `Groups`, `Service Accounts`, `Disabled`.
4. Inside `Users`, create one OU per department: `Management`, `IT`, `Finance`, `HR`, `Sales`, `CustomerService`, `Compliance`, `Repair`.
5. Inside `Computers`, create `Workstations` and `Laptops`.

### PowerShell (optional)

```powershell
$base = "DC=corp,DC=silverbank,DC=internal"
New-ADOrganizationalUnit -Name "SilverBank" -Path $base
$sb = "OU=SilverBank,$base"
"Users","Computers","Servers","Groups","Service Accounts","Disabled" |
  ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path $sb }
"Management","IT","Finance","HR","Sales","CustomerService","Compliance","Repair" |
  ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=Users,$sb" }
"Workstations","Laptops" |
  ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=Computers,$sb" }
```

### Check

| GUI | PowerShell |
|---|---|
| The tree in Active Directory Users and Computers matches the [OU structure](README.md#ou-structure) | `Get-ADOrganizationalUnit -Filter * \| Select-Object Name, DistinguishedName` |

---

Next: [Adding a second domain controller →](02-second-domain-controller.md)