# PowerShell Cheat Sheet for Helpdesk

> Home lab project. Examples use my fictional company **SilverBank** (domain `corp.silverbank.internal`). Replace names, OUs and servers for other networks.
> Organised by **what you need to do**, not by command. Each task has the **GUI equivalent** where there is one — learn the GUI first, then use PowerShell to check and to work faster.

Last updated: 6 November 2026

---

## Contents

0. [Before you start](#0-before-you-start)
1. [Find a user](#1-find-a-user)
2. [Check a user's account health](#2-check-a-users-account-health)
3. ["I'm locked out" / "I forgot my password"](#3-im-locked-out--i-forgot-my-password)
4. [Departments and OUs: who is where?](#4-departments-and-ous-who-is-where)
5. [Groups and access](#5-groups-and-access)
6. [New starter (joiner)](#6-new-starter-joiner)
7. [Changes department (mover)](#7-changes-department-mover)
8. [Leaves the company (leaver)](#8-leaves-the-company-leaver)
9. [Audit and housekeeping reports](#9-audit-and-housekeeping-reports)
10. [Computers](#10-computers)
11. [Remote help on a PC or server](#11-remote-help-on-a-pc-or-server)
12. [File shares and permissions](#12-file-shares-and-permissions)
13. [Printers](#13-printers)
14. [Event logs](#14-event-logs)
15. [Network quick checks](#15-network-quick-checks)
16. [Safety rules](#16-safety-rules)

---

## 0. Before you start

```powershell
hostname                         # Which machine am I on?
whoami                           # Who am I logged in as? (adm- account for admin work)
Get-Module ActiveDirectory -ListAvailable   # Are the AD commands installed?
Import-Module ActiveDirectory    # Load them (usually automatic)
```

**Run PowerShell as Administrator** for changes: right-click → *Run as administrator*.
**Open a new window** after changing time zone, groups or region — old windows keep old settings.

### Finding your way

```powershell
Get-Help Get-ADUser -Examples    # Examples for any command
Get-Command *ADUser*             # Find commands by name
Get-Command -Module ActiveDirectory   # Every AD command
```

Press **Tab** to auto-complete command and parameter names. Press **↑** to repeat a command.

### Making output readable

| Add to the end of a command | What it does |
|---|---|
| `\| Format-List` | One property per line — best for **one** object |
| `\| Format-Table -AutoSize` | A table — best for **many** objects |
| `\| Select-Object Name, Department` | Only the columns you want |
| `\| Sort-Object Name` | Sort |
| `\| Measure-Object` | Count (`Count : 5`) |
| `\| Out-GridView` | Opens a window with a **sortable, searchable table** 👍 |
| `\| Export-Csv C:\Temp\report.csv -NoTypeInformation` | Save to a file you can open in Excel |

Shortcut used below so the commands stay short:

```powershell
$base = "OU=SilverBank,DC=corp,DC=silverbank,DC=internal"
```

---

## 1. Find a user

> GUI: AD Users and Computers → right-click the domain → **Find…**

```powershell
Get-ADUser james.carter                                    # By logon name (exact)
Get-ADUser -Filter "Name -like '*walsh*'"                  # Part of the name (* = anything)
Get-ADUser -Filter "Surname -eq 'Carter'"                  # By surname
Get-ADUser -Filter "EmailAddress -eq 'j.carter@silverbank.example'"   # By email
Get-ADUser -Filter "Name -like '*walsh*'" | Select-Object Name, SamAccountName, Enabled
```

> 💡 Real case from this lab: `tom.walsh` **and** `t.walsh` exist. Searching with `*walsh*` finds duplicates.

---

## 2. Check a user's account health

> GUI: user → **Properties** → Account, Organization and Member Of tabs

The **first command** for almost every user ticket:

```powershell
Get-ADUser james.carter -Properties * |
  Format-List Name, SamAccountName, Enabled, LockedOut, PasswordExpired,
              PasswordLastSet, LastLogonDate, BadLogonCount, AccountExpirationDate,
              Department, Title, Manager, DistinguishedName
```

| Property | What it tells you |
|---|---|
| `Enabled` | `False` = disabled (leaver? not started yet?) |
| `LockedOut` | `True` = too many wrong passwords |
| `PasswordExpired` | `True` = must set a new password |
| `PasswordLastSet` | **Empty** = "must change password at next logon" is ticked |
| `LastLogonDate` | When they last logged in (can be up to ~14 days behind) |
| `BadLogonCount` | Wrong passwords since the last good login (on the DC you ask) |
| `AccountExpirationDate` | Contractors: when the account stops working |
| `DistinguishedName` | Which **OU** the account is in |

Which groups is the user in?

```powershell
Get-ADPrincipalGroupMembership james.carter | Select-Object Name | Sort-Object Name
```

---

## 3. "I'm locked out" / "I forgot my password"

> ⚠️ **Verify the caller's identity first** (call back on their known number, ask their manager, use your company's process). Resetting a password for the wrong person is a security incident.

### Unlock

> GUI: user → **Account** tab → ✅ **Unlock account**

```powershell
Search-ADAccount -LockedOut -UsersOnly | Select-Object Name, SamAccountName   # Who is locked?
Unlock-ADAccount -Identity james.carter
Get-ADUser james.carter -Properties LockedOut | Select-Object Name, LockedOut  # Expect False
```

### Find **where** the lockout came from

If the account keeps locking, an old password is saved somewhere (phone, mapped drive, another PC). Event **4740** on the **PDC Emulator** shows the source computer:

```powershell
$pdc = (Get-ADDomain).PDCEmulator
Get-WinEvent -ComputerName $pdc -FilterHashtable @{LogName='Security'; Id=4740} -MaxEvents 10 |
  Select-Object TimeCreated,
    @{Name='User';   Expression={$_.Properties[0].Value}},
    @{Name='Source'; Expression={$_.Properties[1].Value}}
```

`Source` = the computer that sent the wrong password.

### Reset a password

> GUI: right-click the user → **Reset Password…**

```powershell
Set-ADAccountPassword -Identity james.carter -Reset -NewPassword (Read-Host -AsSecureString "New password")
Set-ADUser -Identity james.carter -ChangePasswordAtLogon $true    # User picks their own
Unlock-ADAccount -Identity james.carter                           # Often locked too
```

**Verify it really saved** (real lesson from this lab):

```powershell
Get-ADUser james.carter -Properties PasswordLastSet | Select-Object Name, PasswordLastSet
```

- Today's date → reset saved, "must change" **not** ticked
- **Empty** → reset saved **and** "must change at next logon" is ticked

> 💡 Login still fails after a reset? Check **keyboard layout** (UK ↔ US swaps `@` and `"`) and **Caps Lock**.

---

## 4. Departments and OUs: who is where?

Two **different** things (real lesson from this lab):

| | What it is | GUI |
|---|---|---|
| **OU** | The **folder** the account lives in | The tree on the left |
| **Department** | A **text field** on the account | Properties → **Organization** tab |

### Users in a department (by the Department field)

```powershell
Get-ADUser -Filter "Department -eq 'Finance'" -Properties Department, Title |
  Select-Object Name, SamAccountName, Title, Department
```

### Users in an OU (by folder)

```powershell
Get-ADUser -Filter * -SearchBase "OU=Finance,OU=Users,$base" -Properties Department |
  Select-Object Name, SamAccountName, Department
```

### How many people per department?

```powershell
Get-ADUser -Filter * -SearchBase $base -Properties Department |
  Group-Object Department | Select-Object Name, Count | Sort-Object Count -Descending
```

### Users with **no** department (missing information)

```powershell
Get-ADUser -Filter * -SearchBase "OU=Users,$base" -Properties Department |
  Where-Object { -not $_.Department } | Select-Object Name, SamAccountName
```

### Users whose OU and Department **don't match**

```powershell
Get-ADUser -Filter * -SearchBase "OU=Users,$base" -Properties Department |
  Select-Object Name, Department,
    @{Name='OU'; Expression={ ($_.DistinguishedName -split ',')[1] -replace '^OU=' }} |
  Where-Object { $_.Department -ne $_.OU }
```

> 💡 These two reports find the kind of mistake we found with `marius.iordan` (in the right OU, but Department empty).

### Who reports to a manager?

```powershell
Get-ADUser david.reid -Properties DirectReports |
  Select-Object -ExpandProperty DirectReports
```

---

## 5. Groups and access

> GUI: group → **Members** tab · user → **Member Of** tab

```powershell
Get-ADGroupMember GG_Finance | Select-Object Name, SamAccountName       # Who is in the group?
Get-ADGroupMember GG_Finance -Recursive | Select-Object Name            # Including groups inside groups
Get-ADPrincipalGroupMembership james.carter | Select-Object Name        # Which groups is the user in?
Get-ADGroup -Filter "Name -like 'GG_*'" | Select-Object Name            # Find groups by name
```

### Add / remove a member

```powershell
Add-ADGroupMember    -Identity GG_Finance -Members james.carter
Remove-ADGroupMember -Identity GG_Sales   -Members james.carter -Confirm:$false
```

> 💡 The user must **log off and on** to get new group access.

### "Give her the same access as her colleague"

Compare two users first:

```powershell
Compare-Object (Get-ADPrincipalGroupMembership tom.walsh).Name `
               (Get-ADPrincipalGroupMembership priya.shah).Name
```

`<=` = only Tom has it · `=>` = only Priya has it.

> ⚠️ "Same as a colleague" is a common request, but the colleague may have **extra access** from old jobs. **Ask the manager** which access is really needed.

### Who has the most powerful rights?

```powershell
Get-ADGroupMember "Domain Admins" | Select-Object Name, SamAccountName
```

---

## 6. New starter (joiner)

> GUI: right-click the OU → **New → User**, then fill in Properties

Create the account (example: a contractor with an expiry date):

```powershell
New-ADUser -Name "Kevin Brooks" -GivenName Kevin -Surname Brooks `
  -SamAccountName kevin.brooks -UserPrincipalName kevin.brooks@corp.silverbank.internal `
  -Path "OU=Contractors,OU=Users,$base" `
  -Description "Contractor - network cabling" -Company "Brooks Cabling Ltd" `
  -Department "IT" -Manager david.reid `
  -AccountExpirationDate "01/01/2027" `
  -AccountPassword (Read-Host -AsSecureString "Temporary password") `
  -ChangePasswordAtLogon $true -Enabled $false
```

| Parameter | Why |
|---|---|
| `-Path` | Put the account in the right **OU** straight away |
| `-Manager` | Someone to ask when access or the contract changes |
| `-AccountExpirationDate` | Contractors: stops working automatically (date format follows the server's region) |
| `-ChangePasswordAtLogon $true` | The user chooses their own password |
| `-Enabled $false` | Keep it disabled until the start date |

Add access and enable on the start date:

```powershell
Add-ADGroupMember -Identity GG_IT_Read -Members kevin.brooks
Enable-ADAccount  -Identity kevin.brooks          # On the start day
```

Verify — see [section 2](#2-check-a-users-account-health).

---

## 7. Changes department (mover)

Example: James moves from Sales to Finance. **Manager approval first.**

```powershell
# 1. Move to the new OU
Get-ADUser james.carter | Move-ADObject -TargetPath "OU=Finance,OU=Users,$base"

# 2. Update the information
Set-ADUser james.carter -Department "Finance" -Title "Finance Assistant" -Manager sarah.mitchell

# 3. Add the new access, REMOVE the old access (avoid privilege creep)
Add-ADGroupMember    -Identity GG_Finance -Members james.carter
Remove-ADGroupMember -Identity GG_Sales   -Members james.carter -Confirm:$false

# 4. Check
Get-ADUser james.carter -Properties Department, Manager | Format-List Name, Department, Manager, DistinguishedName
Get-ADPrincipalGroupMembership james.carter | Select-Object Name
```

---

## 8. Leaves the company (leaver)

> ⚠️ **Disable, don't delete.** The manager may need the files or mailbox, and deleting can't be undone easily.

```powershell
$user  = "ben.morris"
$today = Get-Date -Format "dd/MM/yyyy"

# 1. Disable and note why
Disable-ADAccount -Identity $user
Set-ADUser $user -Description "Leaver - disabled $today (ticket #___)"

# 2. Record the groups BEFORE removing them (in case they come back)
Get-ADPrincipalGroupMembership $user | Select-Object Name |
  Export-Csv "C:\Temp\$user-groups.csv" -NoTypeInformation

# 3. Remove all groups except Domain Users
Get-ADPrincipalGroupMembership $user | Where-Object Name -ne "Domain Users" |
  ForEach-Object { Remove-ADGroupMember -Identity $_ -Members $user -Confirm:$false }

# 4. Move to the Disabled OU
Get-ADUser $user | Move-ADObject -TargetPath "OU=Disabled,$base"

# 5. Check
Get-ADUser $user -Properties Description | Format-List Name, Enabled, Description, DistinguishedName
```

---

## 9. Audit and housekeeping reports

Good to run monthly — and good to mention in interviews.

```powershell
# Disabled accounts still sitting in normal OUs
Search-ADAccount -AccountDisabled -UsersOnly -SearchBase "OU=Users,$base" | Select-Object Name

# Users who haven't logged in for 90 days
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly | Select-Object Name, LastLogonDate

# Accounts expiring in the next 30 days (contractors — warn the manager)
Search-ADAccount -AccountExpiring -TimeSpan 30.00:00:00 -UsersOnly | Select-Object Name, AccountExpirationDate

# Passwords that never expire (should only be service accounts)
Get-ADUser -Filter 'PasswordNeverExpires -eq $true' | Select-Object Name, SamAccountName

# Locked accounts right now
Search-ADAccount -LockedOut -UsersOnly | Select-Object Name

# Everyone with Domain Admin rights
Get-ADGroupMember "Domain Admins" -Recursive | Select-Object Name

# Full user list to Excel
Get-ADUser -Filter * -SearchBase $base -Properties Department, Title, Manager, Enabled, LastLogonDate |
  Select-Object Name, SamAccountName, Department, Title, Enabled, LastLogonDate |
  Export-Csv C:\Temp\silverbank-users.csv -NoTypeInformation
```

---

## 10. Computers

> GUI: AD Users and Computers → Computers OUs

```powershell
Get-ADComputer LN-CL01 -Properties OperatingSystem, LastLogonDate, IPv4Address |
  Format-List Name, OperatingSystem, LastLogonDate, IPv4Address, DistinguishedName

Get-ADComputer -Filter * -Properties OperatingSystem |
  Select-Object Name, OperatingSystem | Sort-Object Name                    # All computers

Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -ComputersOnly |
  Select-Object Name, LastLogonDate                                         # Old / unused PCs

Get-ADComputer LN-CL02 | Move-ADObject -TargetPath "OU=Workstations,OU=Computers,$base"
```

### "The trust relationship between this workstation and the primary domain failed"

Run **on the broken PC**, logged in with a **local** admin account:

```powershell
Test-ComputerSecureChannel                                                  # False = broken
Test-ComputerSecureChannel -Repair -Credential SILVERBANK\adm-marius.iordan
```

---

## 11. Remote help on a PC or server

```powershell
Test-Connection LN-CL01 -Count 2                                    # Is it on?
quser /server:LN-CL01                                               # Who is logged in?
(Get-CimInstance Win32_OperatingSystem -ComputerName LN-CL01).LastBootUpTime   # Last restart
Get-CimInstance Win32_LogicalDisk -ComputerName LN-CL01 -Filter "DriveType=3" |
  Select-Object DeviceID, @{n='FreeGB';e={[math]::Round($_.FreeSpace/1GB,1)}}  # Free disk space

Get-Service -ComputerName LN-FS01 -Name Spooler                    # Is a service running?
Invoke-Command -ComputerName LN-FS01 -ScriptBlock { Restart-Service Spooler }

Invoke-GPUpdate -Computer LN-CL01 -Force                           # Apply Group Policy now
Enter-PSSession -ComputerName LN-FS01                              # Remote shell (Exit-PSSession to leave)
```

> ⚠️ `Restart-Computer -ComputerName X` restarts it **for whoever is using it**. Ask the user first.

On the user's own PC (no admin needed):

```powershell
whoami /groups        # Which groups did I get at login?
gpresult /r           # Which Group Policies applied?
```

---

## 12. File shares and permissions

> GUI: Server Manager → File and Storage Services → Shares · Folder → Properties → Security → Advanced → **Effective Access**

```powershell
Get-SmbShare                                             # All shares on this server
Get-SmbShareAccess -Name Finance                         # Door 1: share permissions
(Get-Acl (Get-SmbShare -Name Finance).Path).Access |
  Format-Table IdentityReference, FileSystemRights, AccessControlType -AutoSize   # Door 2: NTFS
```

### "The file is locked / in use by another user"

Run on the file server:

```powershell
Get-SmbOpenFile | Where-Object Path -like "*Budget*" |
  Select-Object FileId, ClientUserName, ClientComputerName, Path
Close-SmbOpenFile -FileId <id>          # ⚠️ The other user may lose unsaved changes
```

> 💡 No simple PowerShell command shows **Effective Access** — use the GUI for that.

---

## 13. Printers

```powershell
Get-Printer -ComputerName LN-FS01 | Select-Object Name, PrinterStatus, PortName
Get-PrintJob -ComputerName LN-FS01 -PrinterName "Finance-Printer"

# Clear a stuck queue
Get-PrintJob -ComputerName LN-FS01 -PrinterName "Finance-Printer" | Remove-PrintJob
Invoke-Command -ComputerName LN-FS01 -ScriptBlock { Restart-Service Spooler }
```

---

## 14. Event logs

> GUI: `eventvwr.msc` → **Filter Current Log…** → type the Event ID

```powershell
# Errors and critical events in the last 24 hours
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-1)} |
  Select-Object TimeCreated, Id, ProviderName, Message -First 20

# Failed logins (run on a DC)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10 | Select-Object TimeCreated, Message

# Who or what restarted the server?
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1074} -MaxEvents 5 | Select-Object TimeCreated, Message
```

| Event ID | Meaning |
|---|---|
| 4740 | Account locked out (on the PDC Emulator) |
| 4625 / 4624 | Failed / successful login |
| 4771 | Kerberos pre-authentication failed (wrong password) |
| 4720 / 4726 | User created / deleted |
| 4728 / 4732 | Member added to a global / local group |
| 1074 | Who restarted the computer, and why |
| 6008 | Unexpected shutdown |

---

## 15. Network quick checks

```powershell
ipconfig /all                                    # IP, gateway, DNS servers, DHCP server
Test-Connection 192.168.7.81 -Count 2            # Ping by IP
Test-Connection LN-FS01 -Count 2                 # Ping by name — IP works but name fails = DNS
Resolve-DnsName LN-FS01                          # What does DNS answer?
Test-NetConnection LN-FS01 -Port 445             # Is a port open? (445 = file shares)
ipconfig /flushdns                               # Clear the DNS cache
```

| Port | Service |
|---|---|
| 53 | DNS |
| 88 | Kerberos (logins) |
| 389 / 636 | LDAP / LDAPS |
| 445 | File shares (SMB) |
| 3389 | Remote Desktop |
| 5985 | PowerShell remoting (WinRM) |

---

## 16. Safety rules

1. **Look before you change.** Run the `Get-` command first, check the result, then the `Set-` / `Remove-` command.
2. **Add `-WhatIf`** to a change to see what it *would* do without doing it:
   ```powershell
   Remove-ADGroupMember -Identity GG_Sales -Members james.carter -WhatIf
   ```
3. **Test on one user** before running a command on many.
4. **Use your `adm-` account** for changes, and your normal account for everything else.
5. **Never type passwords in plain text** in a command — use `Read-Host -AsSecureString`.
6. **Disable, don't delete** users.
7. **Approval first** for any access to data — and write it in the ticket.
8. **Verify every change** with a `Get-` command, and write what you did in the ticket.