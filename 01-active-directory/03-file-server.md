# 3. Adding a File Server (LN-FS01)

> Join a member server to the domain, install the File Server role, and create **department shares** where each team can open only its own folder.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← Second DC](02-second-domain-controller.md) · [Back to Active Directory](README.md) · Next: [Adding clients →](04-clients.md)

---

## Contents

1. [Plan](#plan)
2. [Step 1: Prepare and join LN-FS01](#step-1-prepare-and-join-ln-fs01)
3. [Step 2: Install the File Server role](#step-2-install-the-file-server-role)
4. [Step 3: Create the department groups](#step-3-create-the-department-groups)
5. [Step 4: Create a department share](#step-4-create-a-department-share)
6. [Step 5: Set NTFS permissions](#step-5-set-ntfs-permissions)
7. [Step 6: Hide folders users can't open](#step-6-hide-folders-users-cant-open)
8. [Step 7: Check — Effective Access and a real user test](#step-7-check--effective-access-and-a-real-user-test)

---

## Plan

| Setting | LN-FS01 |
|---|---|
| IP address | 192.168.7.81 / 24 |
| Gateway | 192.168.7.1 |
| DNS | 192.168.7.80, 192.168.7.82 (the DCs) |
| Roles | File Server · (also: Windows Admin Center, AD Certificate Services) |

**Design used in this lab:** one share per department, one group per department.

| Share | Path example | Group with access |
|---|---|---|
| `\\LN-FS01\Finance` | `D:\Shares\Finance` | `GG_Finance` |
| `\\LN-FS01\HR` | `D:\Shares\HR` | `GG_HR` |
| `\\LN-FS01\Sales` | `D:\Shares\Sales` | `GG_Sales` |

> Use the real drive/path on your server. `GG_` = **G**lobal **G**roup.

### Two doors 🚪🚪

Every shared folder is protected by **two layers**. A user needs **both** to say yes, and the **most restrictive** one wins.

| Layer | Where | Used for |
|---|---|---|
| **Share permissions** | Folder → Properties → **Sharing → Advanced Sharing → Permissions** | Access **over the network** |
| **NTFS permissions** | Folder → Properties → **Security** | Access to the files, **always** (network and local) |

📸 Snapshot first: `before-FS01`.

---

## Step 1: Prepare and join LN-FS01

### GUI

1. Prepare as in [Installation — Step 1](01-installation.md#step-1-prepare-the-server): IP `192.168.7.81`, DNS **`192.168.7.80`** and **`192.168.7.82`**, time zone GMT, name `LN-FS01`.
2. Join the domain as in [Second DC — Step 2](02-second-domain-controller.md#step-2-join-the-domain) (`sysdm.cpl` → Change → Domain).
3. On LN-DC01: Active Directory Users and Computers → **Computers** → right-click `LN-FS01` → **Move…** → `SilverBank > Servers`.

### PowerShell (optional)

```powershell
# On LN-FS01
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.7.81 -PrefixLength 24 -DefaultGateway 192.168.7.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,192.168.7.82
tzutil /s "GMT Standard Time"
Add-Computer -DomainName corp.silverbank.internal -NewName LN-FS01 -Credential SILVERBANK\Administrator -Restart

# On LN-DC01 (after the restart)
Get-ADComputer LN-FS01 | Move-ADObject -TargetPath "OU=Servers,OU=SilverBank,DC=corp,DC=silverbank,DC=internal"
```

### Check — does DNS know the file server?

```powershell
nslookup ln-fs01.corp.silverbank.internal     # must answer 192.168.7.81
```

> ⚠️ **Real problem in this lab:** LN-FS01 had **no DNS record**, so everything worked **by IP** but failed **by name** ("RPC server unavailable"). Fix: on LN-FS01 run `ipconfig /registerdns`, or add the A record in **DNS Manager**.

---

## Step 2: Install the File Server role

### GUI

1. **Server Manager → Manage → Add Roles and Features**.
2. **Server Roles** → expand **File and Storage Services → File and iSCSI Services** → tick **File Server**.
3. Next → **Install** → **Close**.

### PowerShell (optional)

```powershell
Install-WindowsFeature -Name FS-FileServer -IncludeManagementTools
```

### Check

| GUI | PowerShell |
|---|---|
| Server Manager → **File and Storage Services → Shares** opens | `Get-WindowsFeature FS-FileServer` → `[X] Installed` |

---

## Step 3: Create the department groups

### GUI

On LN-DC01: **Active Directory Users and Computers** → `SilverBank > Groups`:
1. Right-click → **New → Group**.
2. Group name: `GG_Finance` · Scope: **Global** · Type: **Security** → **OK**.
3. Repeat for `GG_HR`, `GG_Sales`.
4. Add members: double-click the group → **Members** tab → **Add…** → type the user (e.g. `james.carter`) → **Check Names** → **OK**.

### PowerShell (optional)

```powershell
$groups = "OU=Groups,OU=SilverBank,DC=corp,DC=silverbank,DC=internal"
"GG_Finance","GG_HR","GG_Sales" | ForEach-Object {
  New-ADGroup -Name $_ -GroupScope Global -GroupCategory Security -Path $groups
}
Add-ADGroupMember -Identity GG_Finance -Members james.carter
```

### Check

| GUI | PowerShell |
|---|---|
| Group → **Members** tab · User → **Member Of** tab | `Get-ADGroupMember GG_Finance` · `Get-ADPrincipalGroupMembership james.carter \| Select Name` |

> 💡 **Movers:** when someone changes department, **add** the new group **and remove** the old one. Otherwise they collect access from every old job (**privilege creep**).
> 💡 New group membership loads **at login** — the user must **log off and on**.

---

## Step 4: Create a department share

### GUI (Server Manager wizard — recommended)

1. **Server Manager → File and Storage Services → Shares**.
2. **Tasks → New Share…**
3. Profile: **SMB Share – Quick** → Next.
4. Server: `LN-FS01` → **Type a custom path** → `D:\Shares\Finance` → Next (it creates the folder if needed).
5. Share name: `Finance` → Next.
6. **Other Settings:** ✅ **Enable access-based enumeration** → Next.
7. **Permissions:** click **Customize permissions…** (we do this properly in Step 5) → Next → **Create**.

**Share permissions** (Advanced Sharing → Permissions, or the wizard's **Share** tab):

| Account | Permission |
|---|---|
| `Administrators` (group) | Full Control |
| `GG_Finance` | Change |

> 💡 Use the **group** `Administrators` / `Domain Admins`, not the single **user** `Administrator`. People change; groups stay.

### PowerShell (optional)

```powershell
New-Item -Path "D:\Shares\Finance" -ItemType Directory
New-SmbShare -Name "Finance" -Path "D:\Shares\Finance" `
  -FullAccess "SILVERBANK\Domain Admins" `
  -ChangeAccess "SILVERBANK\GG_Finance" `
  -FolderEnumerationMode AccessBased
```

### Check

| GUI | PowerShell |
|---|---|
| Server Manager → Shares lists **Finance** | `Get-SmbShare` · `Get-SmbShareAccess -Name Finance` |

---

## Step 5: Set NTFS permissions

### GUI

1. File Explorer → `D:\Shares\Finance` → right-click → **Properties → Security → Advanced**.
2. **Disable inheritance** → **Convert inherited permissions into explicit permissions**.
3. **Remove** `Users`, `Domain Users`, `Everyone` if you see them.
4. **Keep** `SYSTEM` and `Administrators` → **Full control**.
5. **Add → Select a principal** → `GG_Finance` → **Check Names → OK** → tick **Modify** → **OK**.
6. **Apply → OK**.

> 💡 **Why disable inheritance?** A folder copies its parent's permissions. Disabling it lets Finance have its **own** rules.
> 💡 **Why Modify, not Full control?** Modify = read, write, edit, delete files. Full control also lets users **change permissions** and **take ownership** — users should never have that.

### PowerShell (optional)

```powershell
# Read the current NTFS permissions
(Get-Acl "D:\Shares\Finance").Access |
  Format-Table IdentityReference, FileSystemRights, AccessControlType, IsInherited -AutoSize
```

> Changing NTFS permissions in PowerShell is long and easy to get wrong — use the **GUI** (or `icacls`) for now.

---

## Step 6: Hide folders users can't open

**Access-based enumeration (ABE)** hides folders a user has no permission to open. Users see only their own department.

### GUI

**Server Manager → File and Storage Services → Shares** → right-click the share → **Properties → Settings** → ✅ **Enable access-based enumeration** → **OK**.

### PowerShell (optional)

```powershell
Set-SmbShare -Name "Finance" -FolderEnumerationMode AccessBased -Force
```

---

## Step 7: Check — Effective Access and a real user test

### GUI: Effective Access 🛠️

Folder → **Properties → Security → Advanced → Effective Access** tab → **Select a user** → `james.carter` → **View effective access**.

| User | Expected result |
|---|---|
| `james.carter` (in GG_Finance) | ✅ Read, Write, Delete · ❌ Full control, Change permissions, Take ownership, Delete subfolders and files |
| A Sales user (not in GG_Finance) | ❌ everything |

> 💡 **Delete subfolders and files** is a special permission that only comes with Full control. Users with **Modify** can still delete files normally.

### Real user test (on LN-CL01)

| Test | Expected |
|---|---|
| Log on as `james.carter` → `\\LN-FS01\Finance` → create `test.txt` | Opens ✅ and the file is created ✅ |
| Log on as a Sales user → `\\LN-FS01\Finance` | **Access denied** ❌ (correct!) |

> Always test that the **wrong** people are **denied**, not only that the right people get in.

📸 Snapshot: `after-FS01-shares`.

---

## What I learned

- **Share + NTFS** = two doors; the most restrictive wins.
- **Effective Access** shows what a user can really do, combining all their groups.
- **Least privilege:** users get **Modify**, never Full control.
- **Approval first:** get the manager's OK before giving access to data, and write it in the ticket.
- Real case: [TICKET-001 — Can't open the Finance shared folder](../tickets/TICKET-001-finance-folder-access.md).

Next: [Adding clients →](04-clients.md)
