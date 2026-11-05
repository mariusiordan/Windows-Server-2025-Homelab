# 2. Adding a Second Domain Controller (LN-DC02)

> A second DC means the domain keeps working if one DC is off: logins, DNS and Group Policy continue.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← Installation](01-installation.md) · [Back to Active Directory](README.md) · Next: [Adding a file server →](03-file-server.md)

---

## Contents

1. [Plan](#plan)
2. [Step 1: Prepare LN-DC02](#step-1-prepare-ln-dc02)
3. [Step 2: Join the domain](#step-2-join-the-domain)
4. [Step 3: Install AD DS and promote](#step-3-install-ad-ds-and-promote)
5. [Step 4: Final DNS settings on both DCs](#step-4-final-dns-settings-on-both-dcs)
6. [Step 5: Check replication](#step-5-check-replication)

---

## Plan

| Setting | LN-DC02 |
|---|---|
| IP address | 192.168.7.82 / 24 |
| Gateway | 192.168.7.1 |
| DNS | **192.168.7.80** (LN-DC01) — never the router |
| Role | Additional DC, DNS server, Global Catalog |

**Order:** prepare → **join the domain** → install AD DS → promote.
Joining first means the wizard uses the **domain** account, not the local one (avoids a common error).

📸 Snapshot first: `before-DC02`.

---

## Step 1: Prepare LN-DC02

### GUI

Same as [Installation — Step 1](01-installation.md#step-1-prepare-the-server), with these values:
- IP `192.168.7.82`, mask `255.255.255.0`, gateway `192.168.7.1`
- Preferred DNS **`192.168.7.80`** (LN-DC01)
- Time zone **GMT Standard Time**
- Computer name `LN-DC02` → restart

### PowerShell (optional)

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.7.82 -PrefixLength 24 -DefaultGateway 192.168.7.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80
tzutil /s "GMT Standard Time"
Rename-Computer -NewName "LN-DC02" -Restart
```

### Check — can it find the domain?

```powershell
nslookup corp.silverbank.internal                                  # answers with 192.168.7.80
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal   # lists LN-DC01
```

If this fails, **stop**: the join will fail too. Check the DNS setting.

---

## Step 2: Join the domain

### GUI

1. Press **Win + R** → type `sysdm.cpl` → **Enter**.
2. **Computer Name** tab → **Change…**
3. Member of: **Domain** → `corp.silverbank.internal` → **OK**.
4. Credentials: `SILVERBANK\Administrator` + password.
5. "Welcome to the corp.silverbank.internal domain" → **OK** → restart.
6. At the login screen: **Other user** → `SILVERBANK\Administrator`.

### PowerShell (optional)

```powershell
Add-Computer -DomainName corp.silverbank.internal -Credential SILVERBANK\Administrator -Restart
```

### Check

| GUI | PowerShell |
|---|---|
| On LN-DC01, Active Directory Users and Computers → **Computers** shows `LN-DC02` | `Get-ADComputer LN-DC02` (run on LN-DC01) |

---

## Step 3: Install AD DS and promote

### GUI

1. Install the role: **Server Manager → Manage → Add Roles and Features** → tick **Active Directory Domain Services** (same as [Installation — Step 2](01-installation.md#step-2-install-the-ad-ds-role)).
2. Click the **yellow flag** ⚠️ → **Promote this server to a domain controller**.

| Wizard page | What to choose |
|---|---|
| **Deployment Configuration** | **Add a domain controller to an existing domain** · Domain: `corp.silverbank.internal` · Credentials: `SILVERBANK\Administrator` |
| **Domain Controller Options** | ✅ DNS server · ✅ Global Catalog · Site: `Default-First-Site-Name` · DSRM password |
| **DNS Options** | Delegation warning — ignore |
| **Additional Options** | Replicate from: **LN-DC01** |
| **Paths / Review / Prerequisites** | Defaults · warnings OK · **Install** |

The server restarts automatically.

### PowerShell (optional)

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Install-ADDSDomainController `
  -DomainName "corp.silverbank.internal" `
  -InstallDns `
  -Credential (Get-Credential SILVERBANK\Administrator) `
  -ReplicationSourceDC "LN-DC01.corp.silverbank.internal" `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

---

## Step 4: Final DNS settings on both DCs

Each DC asks its **partner first** and **itself second**. If one DC is off, the other still answers.

| Server | Preferred DNS | Alternate DNS |
|---|---|---|
| LN-DC01 | 192.168.7.82 | 127.0.0.1 |
| LN-DC02 | 192.168.7.80 | 127.0.0.1 |

### GUI

On each DC: adapter → **Properties → IPv4 → Properties** → set Preferred and Alternate as in the table → **OK**.

Add the forwarder on LN-DC02 too: **Tools → DNS** → right-click **LN-DC02 → Properties → Forwarders → Edit…** → `192.168.7.1`.

### PowerShell (optional)

```powershell
# On LN-DC01
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.82,127.0.0.1
# On LN-DC02
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,127.0.0.1
Add-DnsServerForwarder -IPAddress 192.168.7.1
```

---

## Step 5: Check replication

**Replication** = the DCs copy changes to each other. A new user created on LN-DC01 must appear on LN-DC02.

### GUI

| Where | What to check |
|---|---|
| Active Directory Users and Computers → **Domain Controllers** | Both `LN-DC01` and `LN-DC02` listed |
| **Tools → Active Directory Sites and Services** → Sites → Default-First-Site-Name → Servers | Both DCs listed, each with **NTDS Settings** |
| Same place → right-click **NTDS Settings** of LN-DC02 → **Replicate Now** | "Active Directory Domain Services has replicated the connections" |
| **Visible test:** create a test user on LN-DC01 | After a minute it appears on LN-DC02 (right-click the domain → **Change Domain Controller** → LN-DC02) |
| **DNS Manager** on LN-DC02 | The `corp.silverbank.internal` zone is there (copied from LN-DC01) |

### PowerShell (optional)

```powershell
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog
repadmin /replsummary      # "fails" column must be 0
repadmin /showrepl         # details per partner; look for "was successful"
dcdiag /q                  # prints nothing = healthy
```

📸 Take snapshots of **both** DCs: `after-DC02`.

---

## What I learned

- A second DC = **redundancy**: the domain survives one DC being offline.
- DCs replicate automatically, but **DNS settings** decide whether they can find each other.
- Joining the domain **before** promoting avoids the "user name or password is incorrect" error in the wizard.

Next: [Adding a file server →](03-file-server.md)
