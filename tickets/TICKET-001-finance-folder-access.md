# TICKET-001 — Can't open the Finance shared folder

| Field | Value |
|---|---|
| **GLPI ticket** | #___ |
| **Date** | 05/11/2026 |
| **Requester** | James Carter (Finance, previously Sales) |
| **Type** | Incident |
| **Category** | Access / File shares |
| **Urgency / Impact** | High (meeting at 3pm) / Low (one user) |
| **Difficulty** | 🟡 Medium |

---

## 1. The user's message

> Since this morning I get "Access denied" when I open `\\LN-FS01\Finance`.
> I need the monthly reports for a meeting at 3pm.

## 2. Understand

- **Who:** James Carter
- **What:** "Access denied" on the Finance shared folder
- **Since when:** this morning — the first time he tried
- **How urgent:** High — he needs the reports for a meeting at 3pm

## 3. Questions I asked the user

| Question | Answer |
|---|---|
| Which team do you work in? Could you open this folder before? | Moved from **Sales to Finance on Monday 02/11/2026**. Today was the first time he tried. His new manager, Sarah Mitchell, said he would have access. |

**Lesson:** users don't know what an AD group is — ask in their language ("Which team are you in?", "Did it work before?").

## 4. Investigation (GUI)

| Where I looked | What I found |
|---|---|
| AD Users and Computers → James Carter → **Member Of** | Not in the Finance access group |
| LN-FS01 → Finance share → **Share permissions** | `Administrator` – Full, `GG_Finance` – Change |
| LN-FS01 → Finance folder → **Security (NTFS)** | Access granted to `GG_Finance` |

## 5. Cause

James was a **mover** (Sales → Finance). His account was never added to `GG_Finance`, the group that gives access to the Finance share.

## 6. Fix

1. Asked his manager, **Sarah Mitchell**, to confirm the access — approved, and recorded as a follow-up in GLPI.
2. Added `james.carter` to **GG_Finance** in Active Directory Users and Computers.
3. Checked for old Sales groups to remove — none found (no privilege creep).
4. Checked both permission layers on the Finance folder:
   - Share: `GG_Finance` – Change
   - NTFS: `GG_Finance` – Modify

## 7. Verification

- [x] **Effective Access** (LN-FS01) for `james.carter`: Modify — read, write, delete files. No Full control, Change permissions or Take ownership (correct: least privilege).
- [ ] Tested as James on LN-CL01 (log off / log on → open `\\LN-FS01\Finance` → create a test file)
- [ ] Tested that a Sales user is denied
- [ ] User confirmed it works

## 8. Solution (as written in GLPI)

```
Cause:        James moved from Sales to Finance on Monday, but his account was
              not added to the Finance access group (GG_Finance).
Fix:          After approval from his manager Sarah Mitchell, I added
              james.carter to GG_Finance. Share (Change) and NTFS (Modify)
              permissions for GG_Finance were already correct.
Verification: Effective Access shows Modify for james.carter. James can open
              the Finance folder and create files on LN-CL01.
User note:    Please log off and log on again so the new access works.
```

## 9. What I learned

- **Two doors:** a share is protected by **share permissions** and **NTFS permissions** — the most restrictive one wins.
- **Effective Access** (Security → Advanced) shows what a user can *really* do, combining all their groups.
- **Modify, not Full control:** users can work with files but can't change permissions or take ownership.
- **Mover process:** add the new access **and remove** the old one.
- **Approval first:** never give access to data without the manager's confirmation, and write it in the ticket.
- Group membership loads **at login** — the user must log off and on.
- Interesting point: the user's own words ("I just moved to Finance") gave me the cause before I opened any tool.

## Improvements noted for later

- Share permission uses the **user** `Administrator` — better to use a **group** (e.g. `Domain Admins`).
- Create the same structure for HR and Sales (one share per department, one group each).