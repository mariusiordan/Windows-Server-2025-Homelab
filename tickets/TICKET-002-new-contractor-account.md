# TICKET-002 — New contractor starting Monday (Kevin Brooks)

| Field | Value |
|---|---|
| **GLPI ticket** | #002 |
| **Date** | 06/11/2026 |
| **Requester** | David Reid (IT Manager) |
| **Type** | Request |
| **Category** | Accounts / New starter |
| **Urgency / Impact** | Low / Low |
| **Difficulty** | 🟢 Easy |
| **Time to solve** | ___ min |

---

## 1. The user's message

> Hi IT, we have a contractor, Kevin Brooks from Brooks Cabling Ltd, starting Monday. He'll be with us until the end of the year for the network cabling project. Can you set him up please? Thanks, David

Received by email, so I logged the ticket in GLPI on David's behalf.

## 2. Understand

- **Who:** Kevin Brooks, external contractor (Brooks Cabling Ltd)
- **What:** a new starter account for a contractor (a **Request**, not an Incident)
- **When:** starts Monday 09/11/2026, leaves 31/12/2026
- **Manager:** David Reid

## 3. Questions I asked the manager

| Question | Answer (David Reid) |
|---|---|
| Which folders does he need? | The IT network diagrams — **read only**. Nothing else. |
| Does he need a computer? | Yes, an office PC (LN-CL01) |
| When should the account start working? | Monday 09/11/2026, 9:00 |
| How does he get his password? | David receives the temporary password and gives it to Kevin **in person** on Monday. Kevin chooses his own password at first login. |

**Lesson:** the manager's email was missing important details. Ask before creating anything — the technician proposes, the manager decides.

## 4. What I did

### Active Directory (LN-DC01)

1. Created `kevin.brooks` in **SilverBank → Users → Contractors**
2. Filled in: Description `Contractor - network cabling`, Company `Brooks Cabling Ltd`, Manager **David Reid**
3. **Account expires: end of 31/12/2026** — the account disables itself when the contract ends
4. Temporary password + ✅ **User must change password at next logon**
5. **Disabled** the account until the start date (an unused account is an easy target)
6. Created group **`GG_IT_Read`** (Global, Security) — "Read-only access to the IT share" — and added Kevin

### File server (LN-FS01)

Created the **IT** share and set both permission layers:

| Who | Share permissions | NTFS permissions |
|---|---|---|
| Administrators / SYSTEM | Full Control | Full control |
| `GG_IT` (IT staff, normal accounts) | Change | Modify |
| `GG_IT_Read` (Kevin) | Read | Read & execute |

Access-based enumeration enabled. Kevin's permissions come **only through the group**, never directly.

## 5. Verification

- [x] `Get-ADUser kevin.brooks`: correct OU, description, company, manager, expiry date
- [x] **Effective Access** on the IT folder for `kevin.brooks`: read ✅ — write / delete ❌
- [x] **Effective Access** for an IT user: Modify ✅
- [x] Account is **disabled** until Monday
- [ ] **Monday 09/11:** enable the account
- [ ] **Monday 09/11:** test as Kevin on LN-CL01 — forced to change password, opens `\\LN-FS01\IT`, cannot create or delete files
- [ ] David confirms Kevin can work

### PowerShell checks (optional)

The same checks as the GUI, in PowerShell. Run each block **as Administrator** on the server named in the heading. Open a **new** PowerShell window, because an old one can show old settings (see "What I learned").

#### On LN-DC01: Kevin's account

```powershell
Get-ADUser kevin.brooks -Properties Enabled, Description, Company, Manager, AccountExpirationDate, PasswordLastSet, DistinguishedName |
  Format-List Name, Enabled, Description, Company, Manager, AccountExpirationDate, PasswordLastSet, DistinguishedName
```

| Line | Expect | Proves |
|---|---|---|
| `Enabled` | `False` (today) · `True` (Monday) | Disabled until the start date |
| `Description` / `Company` | `Contractor - network cabling` / `Brooks Cabling Ltd` | Details filled in |
| `Manager` | `CN=David Reid,...` | Someone to ask when the contract ends |
| `AccountExpirationDate` | `1/1/2027 12:00:00 AM` | Works all of 31/12, stops on 1 January |
| `PasswordLastSet` | **empty** | "Must change password at next logon" is ticked |
| `DistinguishedName` | `CN=Kevin Brooks,OU=Contractors,...` | In the right OU |

#### On LN-DC01: the groups

```powershell
Get-ADGroupMember GG_IT_Read | Select-Object Name, SamAccountName
Get-ADGroupMember GG_IT | Select-Object Name, SamAccountName
Get-ADPrincipalGroupMembership kevin.brooks | Select-Object Name
```

| Command | Expect |
|---|---|
| `GG_IT_Read` members | Kevin Brooks only |
| `GG_IT` members | The IT staff's **normal** accounts (no `adm-` accounts, no Kevin) |
| Kevin's groups | `Domain Users`, `GG_IT_Read` |

#### On LN-FS01: the share (door 1) and NTFS (door 2)

```powershell
$path = (Get-SmbShare -Name IT).Path
$path                                                     # the folder behind the share

Get-SmbShareAccess -Name IT |
  Format-Table AccountName, AccessControlType, AccessRight -AutoSize

(Get-SmbShare -Name IT).FolderEnumerationMode             # AccessBased = hidden if no access

(Get-Acl $path).Access |
  Format-Table IdentityReference, FileSystemRights, AccessControlType, IsInherited -AutoSize
```

| Check | Expect |
|---|---|
| Share access | `Administrators` Full · `GG_IT` Change · `GG_IT_Read` Read · **no** Kevin Brooks |
| `FolderEnumerationMode` | `AccessBased` |
| NTFS | `GG_IT` → `Modify, Synchronize` · `GG_IT_Read` → `ReadAndExecute, Synchronize` · **no** Kevin Brooks, Users or Everyone |

> 💡 There's no simple built-in PowerShell command for **Effective Access**. For that one, the **GUI** (Security → Advanced → Effective Access) is the best tool.

#### Monday, on LN-DC01: enable the account

```powershell
Enable-ADAccount -Identity kevin.brooks
Get-ADUser kevin.brooks | Select-Object Name, Enabled          # Enabled: True
```

#### Monday, on LN-CL01: logged in as Kevin (normal PowerShell, not Administrator)

```powershell
whoami                                    # silverbank\kevin.brooks
whoami /groups | findstr GG_IT_Read       # the group is in his login ticket
Test-Path \\LN-FS01\IT                    # True = he can open the share
Get-ChildItem \\LN-FS01\IT                # he can see the files
New-Item \\LN-FS01\IT\test.txt            # EXPECT an error: Access is denied ✅
```

> 💡 The **error** on the last line is the result we **want**. It proves Kevin really is read-only. Testing that the wrong action **fails** is as important as testing that the right action works.

## 6. Ticket status

- **Friday 06/11:** status **Pending** ⏸️ — waiting for the start date
- **Monday 09/11:** enable, test, add the Solution, close

## 7. Solution (to add in GLPI on Monday)

```
Cause:        New contractor Kevin Brooks (Brooks Cabling Ltd) starting 09/11/2026,
              requested by his manager David Reid.
Fix:          Created kevin.brooks in the Contractors OU with manager David Reid and
              an account expiry date of 31/12/2026. Added him to GG_IT_Read for
              read-only access to the IT share (\\LN-FS01\IT). The account was kept
              disabled until his start date and enabled on Monday morning.
Verification: Effective Access shows read-only access for kevin.brooks. Tested a login
              on LN-CL01: password change was forced, the IT share opens, and he
              cannot create or delete files.
User note:    David gave Kevin his temporary password in person. Kevin must choose his
              own password at first login. The account stops working after 31/12/2026 —
              David should raise a ticket if the contract is extended.
```

## 8. What I learned

- **New starter = Request**, not Incident.
- A manager's request is often **incomplete** — ask before acting.
- **Contractors:** separate OU, **expiry date**, a **manager** to ask when the contract ends.
- **Disable until the start date** — don't leave unused accounts active.
- **Never email a password**, especially to an external address.
- Permissions go to **groups, not people** (`GG_IT_Read`). When Kevin leaves, the folder's permissions don't change.
- Staff get **Modify**, not Full control. Read-only users get **Read & execute**.
- **Effective Access** works even on a **disabled** account — useful for testing before the start date.
- Bonus: an old PowerShell window still showed the **old time zone** — programs read settings when they start.