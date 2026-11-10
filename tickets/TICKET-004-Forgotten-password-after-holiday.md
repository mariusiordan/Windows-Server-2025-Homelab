# TICKET-004 — Forgotten password after holiday (Grace Lee)

| Field | Value |
|---|---|
| **GLPI ticket** | #004 |
| **Date** | 10/11/2026 |
| **Requester** | Grace Lee (Sales) |
| **Type** | Incident |
| **Category** | Accounts / Password reset |
| **Urgency / Impact** | High (customer quote this morning) / Low (one user) |
| **Difficulty** | 🟢 Easy |
| **Worked by** | Tom Walsh (Service Desk L1) — role-played by me |
| **Time to solve** | 5 min |

---

## 1. The user's message

> 🚶‍♀️ At the IT desk: *"Morning! I've just come back from two weeks' holiday and I can't remember my password at all. I tried a few times but I don't think it's locked yet… Can you reset it? I need to send a quote to a customer this morning."*

Grace couldn't log in, so I logged the ticket in GLPI for her.

## 2. Understand

- **Who:** Grace Lee, Sales
- **What:** forgotten password (and possibly locked, after several attempts)
- **Since when:** this morning, first day back from holiday
- **How urgent:** High — a customer is waiting for a quote

## 3. Identity check 🔐

| Contact | How I verified | Result |
|---|---|---|
| **In person** | Asked for her **staff ID badge** — name and photo match the AD account | ✅ Verified |

**Lesson:** in person, identity is **easier** than on the phone — ask for a badge or photo ID instead of calling the manager.

## 4. Investigation (GUI)

| Where I looked | What I found |
|---|---|
| AD Users and Computers → Grace → **Account** tab | Account **locked out** |
| **Reset Password** window → *Account Lockout Status on this Domain Controller* | **Locked** |

**Grace thought she wasn't locked — she was.** Users often don't know. Always check.

## 5. Cause

Forgotten password after two weeks' holiday. Several wrong attempts → the lockout policy (5 attempts) locked the account.

## 6. Fix

On LN-CL01, logged in as `tom.walsh`, AD Users and Computers opened with **Run as different user** → `adm-tom.walsh`:

1. Right-click **Grace Lee** → **Reset Password…**
2. Temporary password
3. ✅ **User must change password at next logon** — only Grace will know her real password
4. ✅ **Unlock the user's account** — in the same window
5. Gave Grace the temporary password **in person**, and she logged in straight away

> 💡 If I had reset the password **without** unlocking, her login would still have failed — and she would have thought the new password was wrong.

## 7. Verification

- [x] Lockout status checked before the reset (Locked) and unlocked in the same step
- [x] Grace logged in on LN-CL01 with the temporary password
- [x] Windows forced her to choose her own password ✅
- [x] Temporary password not left written down

### PowerShell checks (optional)

```powershell
# Before: is she locked? When was her password last set?
Get-ADUser grace.lee -Properties LockedOut, PasswordLastSet, BadLogonCount |
  Format-List Name, LockedOut, PasswordLastSet, BadLogonCount

# After the reset, before she logs in
Get-ADUser grace.lee -Properties LockedOut, PasswordLastSet |
  Format-List Name, LockedOut, PasswordLastSet
```

| After the reset | Meaning |
|---|---|
| `LockedOut: False` | Unlocked ✅ |
| `PasswordLastSet` **empty** | "Must change password at next logon" is ticked ✅ |
| `PasswordLastSet` = today, after she logs in | She set her own password ✅ |

## 8. Solution (as written in GLPI)

```
Cause:        Grace forgot her password after two weeks of holiday. Several wrong attempts
              had also locked the account.
Fix:          Identity verified in person (staff ID badge). Password reset by the Service
              Desk with "must change password at next logon", and the account unlocked in
              the same step.
Verification: Grace logged in on LN-CL01 and was prompted to set her own password. Login
              successful.
User note:    Choose a password you can remember after a holiday - a passphrase of three
              random words is easier to remember and harder to guess.
```

## 9. What I learned

- **In person = check the badge.** Faster than calling the manager.
- **Always check the lockout status** — the user may not know. The **Reset Password** window shows it and can unlock at the same time.
- **"Must change password at next logon"** — the technician should never know the user's real password.
- Give the temporary password **in person** and let the user log in **straight away**. No sticky notes left on the desk.
- A **kind user note** (passphrase tip) prevents the next ticket.