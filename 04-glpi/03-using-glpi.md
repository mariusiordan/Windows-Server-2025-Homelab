# 3. Using GLPI — Technician Cheat Sheet

> Quick reference for working tickets in GLPI: logging in, the ticket life cycle, what each field means, and the 6-step troubleshooting method.
> All **GUI**.

[← Connecting to AD](02-active-directory-ldaps.md) · [Back to GLPI](README.md) · Next: [Maintenance →](04-maintenance.md)

---

## 1. Logging in

| Account | Login source | Use it for |
|---|---|---|
| `marius.iordan` (named admin) | **SilverBank AD** | Daily technician work |
| `glpi` (break-glass admin) | **GLPI internal database** | Emergencies only (e.g. AD is down) |
| Staff, e.g. `james.carter` | **SilverBank AD** | Self-Service: create and follow tickets |

> 💡 If a login fails, check the **Login source** dropdown first. Local accounts never work with "SilverBank AD", and AD accounts never work with "GLPI internal database".

---

## 2. Two interfaces

| Interface | Who sees it | What it shows |
|---|---|---|
| **Self-Service** | Normal employees | Create a ticket, see my tickets, FAQ |
| **Standard** | Technicians / admins | All tickets, assets, users, setup |

---

## 3. Ticket life cycle

```
New → Processing (assigned) → [Pending] → Solved → Closed
```

| Status | Meaning | When |
|---|---|---|
| **New** | Nobody has taken it yet | Automatic when created |
| **Processing (assigned)** | A technician is working on it | When you take / assign it |
| **Processing (planned)** | Work is scheduled for later | When you plan a task |
| **Pending** | Waiting for someone else (user, manager, supplier) | e.g. waiting for approval |
| **Solved** | Fix done, waiting for the user to accept | Automatic when you add a **Solution** |
| **Closed** | User accepted (or auto-closed after some days) | End of the ticket |

---

## 4. Ticket fields — what to choose

| Field | Meaning | Example |
|---|---|---|
| **Type** | **Incident** = something is broken · **Request** = asking for something new | "Access denied" → Incident · "New laptop" → Request |
| **Category** | Topic of the ticket (used for reports) | Access, Hardware, Network, Printing, Software |
| **Urgency** | How fast the **user** needs it | Meeting in 2 hours → High |
| **Impact** | How many **people** are affected | 1 person → Low · whole office → High |
| **Priority** | Calculated from Urgency + Impact | Automatic |
| **Requester** | Who reported the problem | The employee |
| **Observer** | Who should be kept informed | Their manager |
| **Assigned to** | The technician or group working on it | You |

---

## 5. The "Answer" button — 4 choices

| Choice | Use it for | Who sees it |
|---|---|---|
| **Follow-up** 💬 | Messages to the user, or notes | User (unless **Private**) |
| **Task** 🔧 | Work you did + time spent | Usually IT only |
| **Solution** ✅ | The final fix — ticket becomes **Solved** | User |
| **Document** 📎 | Attach a screenshot or file | Depends |

**Remember:** Talking → Follow-up · Working → Task · Finished → Solution

> **Private** switch ON = only IT staff can read it. Use it for internal notes.
> **Approval** (under Answer) = send a formal approval request to a manager inside GLPI.

---

## 6. The 6-step troubleshooting method

| Step | What to do |
|---|---|
| 1. **Understand** | Who? What exactly? Since when? How urgent? |
| 2. **Ask** | Questions in the **user's** language ("Which team are you in?", "Did it work before?") |
| 3. **Check** | Look in the GUI: AD Users and Computers, Server Manager, Event Viewer, Effective Access |
| 4. **Find the cause** | Say what you think and **why**. Be ready to be wrong. |
| 5. **Fix + verify** | Change one thing, then test **like the user** (and test that the wrong people are still denied) |
| 6. **Document** | Write the Solution: **Cause → Fix → Verification → User note** |

---

## 7. A typical ticket timeline

1. **Follow-up:** "Hi James, I'm looking into this now and will update you shortly."
2. **Status → Pending** while waiting for the manager (if approval is needed)
3. **Follow-up:** "Manager Sarah Mitchell confirmed the access request."
4. **Task:** "Added james.carter to GG_Finance — 5 min."
5. **Solution:** Cause → Fix → Verification → User note
6. User accepts → **Closed**

---

## 8. Solution template

```
Cause:        What was wrong, and why.
Fix:          What exactly I changed.
Verification: How I know it works now.
User note:    Anything the user needs to do (e.g. log off and on).
```

**Rule:** only write what you really did and tested.

---

## 9. Useful places in GLPI

| I want to... | Go to |
|---|---|
| See all tickets | **Assistance → Tickets** |
| Create a ticket for someone | **Assistance → Create ticket** |
| Find a user | **Administration → Users** |
| Give someone admin rights | **Administration → Users → user → Authorizations** |
| Check the AD connection | **Setup → Authentication → LDAP directories → SilverBank AD → Test** |
| Import new AD users | **Administration → Users → LDAP directory link → Import new users** |
| See login errors / events | **Administration → Logs** |
| Automatic background jobs | **Setup → Automatic actions** |

---

## 10. Helpdesk habits

- Reply to the user **quickly**, even before you fix anything.
- Get **manager approval** before giving access to data — and write it in the ticket.
- **Movers:** add the new access **and remove** the old access (avoid privilege creep).
- Group changes need a **log off / log on** to take effect.
- Never put **passwords** in tickets or on GitHub.

---

Next: [Maintenance →](04-maintenance.md)