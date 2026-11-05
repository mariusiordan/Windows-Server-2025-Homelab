# 1. Installation and Authorization

> Install the DHCP Server role on a Windows Server 2025 machine, finish the post-install setup, and **authorize** it in Active Directory.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

> ⚠️ Reference guide — example values. Not applied in my lab (see the [README](README.md)).

[← Back to DHCP](README.md) · Next: [Scopes and options →](02-scopes-and-options.md)

---

## Before you start

- [ ] The server has a **static IP** (a DHCP server can't get its own IP from DHCP)
- [ ] The server is **joined to the domain**
- [ ] You're logged in as a **Domain Admin** (needed to authorize in AD)
- [ ] **No other DHCP server** is active on the target network (check the router!)
- [ ] Snapshot taken

**Where to install it?** In small companies, DHCP often runs on a **domain controller**. In bigger ones, on a separate server. Both are fine.

---

## Step 1: Install the DHCP Server role

### GUI

1. **Server Manager → Manage → Add Roles and Features**.
2. **Installation Type:** Role-based or feature-based installation → Next.
3. **Server Selection:** choose the server → Next.
4. **Server Roles:** tick **DHCP Server** → **Add Features** → Next.
5. **Features** → Next · **DHCP Server** info page → Next.
6. **Install** → wait → **Close**.

### PowerShell (optional)

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

### Check

| GUI | PowerShell |
|---|---|
| Server Manager shows **DHCP** in the left menu and a yellow ⚠️ flag | `Get-WindowsFeature DHCP` → `[X] Installed` |

---

## Step 2: Complete the post-install configuration

The yellow flag means **two jobs are still open**: create the DHCP security groups, and authorize the server in AD.

### GUI

1. Click the **yellow flag** ⚠️ → **Complete DHCP configuration**.
2. **Description** page → Next.
3. **Authorization:** choose **Use the following user's credentials** → `SILVERBANK\Administrator` (a Domain Admin) → **Commit**.
4. Summary: "Security Groups creation: Done" and "Authorization: Done" → **Close**.

**What just happened:**

| Item | Meaning |
|---|---|
| **DHCP Administrators** group | Members can fully manage DHCP |
| **DHCP Users** group | Members can **view** DHCP (read-only) — good for helpdesk staff |
| **Authorization** | The server is now listed in AD as an **approved** DHCP server |

### PowerShell (optional)

```powershell
# 1. Create the DHCP Administrators and DHCP Users groups
Add-DhcpServerSecurityGroup
Restart-Service DHCPServer

# 2. Authorize the server in Active Directory
Add-DhcpServerInDC -DnsName "ln-dc01.corp.silverbank.internal" -IPAddress 192.168.7.80

# 3. Tell Server Manager the setup is finished (removes the yellow flag)
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\ServerManager\Roles\12" -Name ConfigurationState -Value 2
```

---

## Why authorization exists 🔐

If someone plugs a **home router** into the office network, it starts handing out **wrong** IPs and the wrong gateway — users lose the network. This is a **rogue DHCP server**.

**Authorization** protects against **Windows** rogue servers: a Windows DHCP server in a domain checks AD at startup, and **refuses to work** if it's not on the approved list.

> ⚠️ It does **not** stop a home router or a Linux DHCP server. On real networks, switches use **DHCP Snooping** to block those.

---

## Step 3: Check the installation

### GUI

1. **Server Manager → Tools → DHCP** (or **Win + R** → `dhcpmgmt.msc`).
2. Expand the server → **IPv4** and **IPv6**.
3. A **green tick** ✅ on IPv4 = authorized and running.
   A **red arrow** 🔻 = not authorized, or the service is stopped.
4. Right-click **DHCP** (top) → **Manage authorized servers…** → your server is listed.

### PowerShell (optional)

```powershell
Get-Service DHCPServer                 # Status: Running
Get-DhcpServerInDC                     # lists the authorized DHCP servers in AD
Get-ADGroup "DHCP Administrators"      # the security group exists
```

---

## Unauthorize / remove (for reference)

### GUI

DHCP console → right-click the server → **Unauthorize** · Then Server Manager → Manage → **Remove Roles and Features** → untick DHCP Server.

### PowerShell (optional)

```powershell
Remove-DhcpServerInDC -DnsName "ln-dc01.corp.silverbank.internal" -IPAddress 192.168.7.80
Uninstall-WindowsFeature -Name DHCP -IncludeManagementTools
```

---

## What I learned

- Installing the role is only half the job: the **post-install** creates the groups and **authorizes** the server.
- **Green tick** = working · **red arrow** = not authorized or stopped.
- Authorization protects against Windows rogue DHCP servers, not against home routers.

Next: [Scopes and options →](02-scopes-and-options.md)