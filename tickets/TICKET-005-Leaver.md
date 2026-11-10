# TICKET-005 — Leaver: Ben Morris

| Field | Value |
|---|---|
| **GLPI ticket** | #005 |
| **Date** | 10/11/2026 |
| **Requester** | HR team · confirmed by Laura Green (HR Manager) |
| **Type** | Request |
| **Category** | Accounts / Leaver |
| **Urgency / Impact** | Medium (must be done after 17:00 today) / Low |
| **Difficulty** | 🟢 Easy |
| **Worked by** | Tom Walsh (Service Desk L1) → escalated to Infrastructure (L2) |
| **Status** | ⏸️ Pending — waiting for the laptop to be returned |

---

## 1. The user's message

> 📧 *"Hi IT, Ben Morris is leaving SilverBank. His last day is today. Please remove his access. Thanks, HR"*

## 2. Understand

- **Who:** Ben Morris, HR department, manager Laura Green
- **What:** a leaver — remove his access (a **Request**)
- **When:** today, 10/11/2026

## 3. Questions I asked the manager

| Question | Answer (Laura Green) |
|---|---|
| Do you confirm Ben is leaving and his access should be removed? | Yes |
| **When** exactly? | End of his last day — **after 17:00**. He is finishing his handover until then |
| Why is he leaving? *(friendly or not — changes the timing)* | Friendly — moving to another company |
| Who needs his files? | His work is on the **HR share**, so the HR team keeps it |
| Company equipment? | **Laptop and ID badge** — returned to Laura at 17:00, laptop brought to IT the next morning |

**Lessons:**
- **Confirm the request with the manager** — a fake or mistaken "remove access" email could cause damage.
- **Timing:** friendly leaver → disable at the **end** of the last day. Dismissal → disable **immediately**.

## 4. What I did

### Escalation

The Service Desk has delegated rights for **reset password** and **unlock** only — not to disable or move accounts. Escalated to **Infrastructure (L2)**.

### Active Directory (after 17:00)

1. **Member Of** tab: recorded his groups **before** removing anything → **Domain Users only** (no department groups to remove)
2. **Disabled** the account
3. **Description:** `Leaver - disabled 10/11/2026 (ticket #005)`
4. **Moved** to `SilverBank → Disabled`

### GLPI

5. Administration → Users → Ben Morris → untick **Active**

> ⚠️ **Not deleted.** Disable first, keep the account for a while: files, history and audit logs stay linked to it.

## 5. Verification

- [x] `Enabled: False`
- [x] Description set
- [x] In `OU=Disabled`
- [x] Groups: Domain Users only
- [x] GLPI user inactive
- [ ] **Laptop returned** to IT
- [ ] Negative test: Tom (`adm-tom.walsh`) gets **Access denied** when trying to disable or move a user

```powershell
Get-ADUser ben.morris -Properties Description |
  Format-List Name, Enabled, Description, DistinguishedName
Get-ADPrincipalGroupMembership ben.morris | Select-Object Name
```

**Result:**

```
Name              : Ben Morris
Enabled           : False
Description       : Leaver - disabled 10/11/2026 (ticket #005)
DistinguishedName : CN=Ben Morris,OU=Disabled,OU=SilverBank,DC=corp,DC=silverbank,DC=internal
```

## 6. Problems along the way

| Problem | Cause | Fix |
|---|---|---|
| "Cannot move — the object doesn't exist" | I was working in **GLPI**, not AD. GLPI has no OUs — it only holds a **copy** of the user | Did the account work in **AD Users and Computers**. Rule: **AD first, GLPI last** |
| Disabling Ben in GLPI didn't really stop him | The GLPI user only controls the helpdesk login — Windows and file shares still worked | Disabled the **AD** account |
| AD Users and Computers showed an error | An **old console window** lost its connection (the DC itself was healthy: services running, `dcdiag /q` clean) | Closed and reopened the console |

## 7. Solution (to add in GLPI when the laptop is back)

```
Cause:        Ben Morris (HR) left SilverBank on 10/11/2026 - friendly leaver,
              confirmed by his manager Laura Green.
Fix:          After 17:00 on his last day, the AD account was disabled, its description
              updated with the leave date and ticket number, and it was moved to the
              Disabled OU. Groups recorded before removal (Domain Users only). GLPI
              user set to inactive. Laptop returned to IT.
Verification: Enabled = False, account in OU=Disabled, no group memberships except
              Domain Users. Laptop received.
User note:    Ben's work files stay on the HR share for the HR team.
```

## 8. What I learned

- **Leaver = disable, don't delete.** Record the groups first, add a description, move to Disabled.
- **AD is the source of truth.** Disabling a user in GLPI only blocks the helpdesk — the real account is in AD.
- **Ask about timing and the reason.** It decides whether you act now or at the end of the day.
- **L1 can't do everything** — escalating to L2 is part of the process, not a failure.
- **Use your named admin account** (`adm-marius.iordan`), not `Administrator` — the logs then show **who** did the work.
- **Local vs domain accounts:** `LN-CL01\Administrator` can't read AD; `SILVERBANK\Administrator` can.
- **Equipment is part of a leaver** — the ticket isn't closed until the laptop is back.