# TICKET-003 — Account locked (Emma Hughes)

| Field | Value |
|---|---|
| **GLPI ticket** | #003 |
| **Date** | 07/11/2026 |
| **Requester** | Emma Hughes (Compliance) |
| **Type** | Incident |
| **Category** | Accounts / Locked account |
| **Urgency / Impact** | High (meeting at 9:30) / Low (one user) |
| **Difficulty** | 🟢 Easy |
| **Worked by** | Tom Walsh (Service Desk L1) — role-played by me |
| **Time to solve** | 5 min (after identity check) |

---

## 0. Lab setup before the ticket

The lab couldn't lock accounts yet, so I set it up first.

### Account lockout policy

**Group Policy Management → Default Domain Policy → Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy**

| Setting | Before | After |
|---|---|---|
| Account lockout threshold | **0** (never locks!) | **5** invalid attempts |
| Account lockout duration | Not defined | 15 minutes |
| Reset account lockout counter after | Not defined | 15 minutes |

Then `gpupdate /force` on both DCs.

> 💡 A threshold of **0** means an attacker can try unlimited passwords. Not acceptable for a bank. The duration is 15 minutes (not "until IT unlocks") so an attacker can't lock out every employee on purpose.
> 💡 I did **not** change *Allow Administrator account lockout* — I'll review it once my `adm-` account is my daily admin account.

### Delegation to the Service Desk

On **SilverBank → Users** (not on `Admin Accounts`!), **Delegate Control…** to `GG_IT_ServiceDesk`:

1. Common task: **Reset user passwords and force password change at next logon**
2. Custom task: **User objects** → Property-specific → **Read lockoutTime** + **Write lockoutTime** (= unlock)

> 💡 The Service Desk can unlock and reset **staff** accounts, but not **admin** accounts — so nobody on L1 can take over a Domain Admin account.

### Admin tools on the Service Desk PC

Installed **RSAT: Active Directory Domain Services Tools** on LN-CL01 (as `adm-marius.iordan`). The Service Desk works **from their own PC**, never logged in to a domain controller.

---

## 1. The user's message

> 📞 Phone call: *"Hi, it's Emma from Compliance. I can't log in to my computer. It says my account is locked. I think I typed my password wrong a few times this morning… I have a meeting at 9:30, can you help?"*

Emma couldn't log in, so **I logged the ticket in GLPI on her behalf**.

## 2. Understand

- **Who:** Emma Hughes, Compliance
- **What:** account locked out
- **Since when:** this morning, after several wrong passwords
- **How urgent:** High — meeting at 9:30

## 3. Identity check 🔐

Anyone can phone and say *"I'm Emma"* (**social engineering**). Before touching the account:

| Method | Result |
|---|---|
| Call back on her number **from AD** (not a number the caller gives) | ❌ No telephone number in AD |
| Call her **manager** (number from AD) | ✅ Sarah Mitchell confirmed Emma was with her |

**Lesson:** unlocking her **own** account needs no manager **approval** (it restores existing access) — but it always needs **identity verification**.

## 4. Investigation (GUI)

| Where I looked | What I found |
|---|---|
| AD Users and Computers → Emma → **Account** tab | *"Unlock account. This account is currently locked out on this Active Directory Domain Controller."* |

**First attempt:** by the time I checked, the account had already **unlocked by itself** (15-minute lockout duration). I locked it again to practise the manual unlock.

## 5. Cause

Emma typed her password wrong 5 times → the lockout policy locked the account.
Not an "old password saved on another device" case — it didn't lock again afterwards.

## 6. Fix

1. On LN-CL01, logged in as **`tom.walsh`** (normal account)
2. Opened **AD Users and Computers** with **Run as different user** → `adm-tom.walsh`
3. **SilverBank → Users → Compliance → Emma Hughes → Account** tab → ✅ **Unlock account** → OK

## 7. Verification

- [x] Account tab: no "locked out" message
- [x] `LockedOut: False`, `lockoutTime: 0`
- [x] Emma logged in successfully on LN-CL01
- [ ] **Negative test:** Tom can't unlock or reset `adm-marius.iordan` (Admin Accounts OU)

### PowerShell checks (optional)

On any domain PC (reading AD needs no admin rights):

```powershell
# Is the account locked?
Get-ADUser emma.hughes -Properties LockedOut, lockoutTime, BadLogonCount |
  Format-List Name, LockedOut, lockoutTime, BadLogonCount

# When did it lock? (converts the timestamp to local time)
[DateTime]::FromFileTime((Get-ADUser emma.hughes -Properties lockoutTime).lockoutTime)

# All locked accounts right now
Search-ADAccount -LockedOut -UsersOnly | Select-Object Name, SamAccountName

# The lockout policy
Get-ADDefaultDomainPasswordPolicy | Format-List LockoutThreshold, LockoutDuration, LockoutObservationWindow
```

| Result | Meaning |
|---|---|
| `LockedOut: True` | Locked now → unlock |
| `LockedOut: False`, `lockoutTime` = a big number | It **unlocked by itself** — AD keeps the old lockout time |
| `LockedOut: False`, `lockoutTime: 0` | Unlocked **manually** (or the user logged in successfully since) |

If an account **keeps** locking, find the source computer with Event **4740** on the PDC Emulator (see the [PowerShell cheat sheet](../cheat-sheets/powershell-helpdesk.md#3-im-locked-out--i-forgot-my-password)).

## 8. Solution (as written in GLPI)

```
Cause:        Account locked after 5 wrong password attempts (lockout policy: 5 attempts,
              15 minutes). The user confirmed she mistyped her password.
Fix:          Caller identity verified through her manager, Sarah Mitchell. Account
              unlocked in AD by the Service Desk (adm-tom.walsh).
Verification: LockedOut = False, lockoutTime = 0. Emma logged in successfully on LN-CL01.
User note:    If the account locks again without wrong typing, call the Service Desk -
              we will check for a saved old password on another device (Event 4740).
```

## 9. What I learned

- **Verify identity before unlocking or resetting** — call back on a number from **AD**, or ask the manager.
- **Approval vs identity:** restoring existing access needs no approval, but always needs identity verification.
- **A user who can't log in can't create a ticket** — the Service Desk logs it for them.
- **Lockout duration:** the account may unlock by itself before you even look.
- **`LockedOut` is the truth.** `lockoutTime` keeps the old time after an automatic unlock, and only resets to 0 after a manual unlock or a good login.
- **Delegation:** the Service Desk can reset and unlock **staff** without being Domain Admins. Admin accounts are outside their OU.
- **Work from your own PC with RSAT**, using **Run as different user** for the admin tool only.
- A non-zero `lockoutTime` showed **9:37 AM** instead of **5:37 PM** — the **old PowerShell window** again. Close old windows after a time zone change.

## Improvements noted for later

- Add **telephone numbers** to staff accounts in AD, so the Service Desk can call users back.
- Check the time zone on **LN-CL01, LN-DC02, LN-FS01** (`tzutil /g`).
- Review **Allow Administrator account lockout** once `adm-marius.iordan` is the daily admin account.