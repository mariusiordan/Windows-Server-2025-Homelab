# Helpdesk Patterns — My Playbook

> Most helpdesk tickets are the **same problems again and again**. After each ticket, I add or improve one line here.
> Before a new ticket, I look here first: *"Have I seen this pattern before?"*

Last updated: 7 November 2026

---

## How to use this page

1. **Read the user's message** → find the closest "User says" line.
2. Do the **first checks** in order — stop when you find the cause.
3. After the ticket, **update the line** with what I learned.
4. Once a week: re-read this page. Before an interview: read it twice. 😉

**Status:** ✅ practised in my lab · 🔲 not practised yet

---

## Accounts and logins

| User says | First checks (in order) | Usual causes | Fix | Status |
|---|---|---|---|---|
| "My account is locked" | **Verify identity** (call back on the AD number, or the manager) · `LockedOut` on the Account tab (it may have **unlocked by itself** after 15 min) · if it keeps locking: Event **4740** on the PDC (source computer) | Wrong password typed many times · **old password saved** on a phone, mapped drive or second PC | Unlock (Service Desk, delegated) · if it keeps locking, fix the saved password on the source device | ✅ [TICKET-003](TICKET-003-account-locked.md) |
| "I forgot my password" | **Verify identity first** (in person: **staff badge**; phone: call back on the AD number or the manager) · **locked?** (the user often doesn't know!) · enabled? | Forgotten (often after a holiday) · expired | **Reset Password** window: temporary password + ✅ "must change at next logon" + ✅ **Unlock** in the same step · user logs in straight away | ✅ [TICKET-004](TICKET-004-password-reset.md) |
| "My password doesn't work" (but they're sure it's right) | Caps Lock · **keyboard layout** (UK ↔ US) · local vs domain account (`SILVERBANK\user`) · password expired | Keyboard layout · typing the old password · expired password | Type it somewhere visible · reset if needed | ✅ (lab: `52e` error, keyboard trap) |
| "I can't log in at all" | Account **Enabled**? **Expired** (contractor)? Locked? · One PC or every PC? | Disabled leaver · contractor past expiry · PC lost its domain trust | Enable / extend expiry (with manager approval) · `Test-ComputerSecureChannel -Repair` | 🔲 |
| "Trust relationship between this workstation and the domain failed" | Log in with a **local** admin account | PC restored from an old snapshot/backup · computer password out of sync | `Test-ComputerSecureChannel -Repair` · or remove and re-join the domain | 🔲 |

## Access and permissions

| User says | First checks (in order) | Usual causes | Fix | Status |
|---|---|---|---|---|
| "Access denied on a shared folder" | Which **team**? Did it ever work? · **Member Of** · share + NTFS permissions · **Effective Access** | User **moved department** (mover) · never added to the group · added but didn't log off/on | **Manager approval** → add to the group (remove old groups) → user **logs off and on** | ✅ [TICKET-001](TICKET-001-finance-folder-access.md) |
| "Give her the same access as her colleague" | **Compare** the two users' groups · ask the manager what is really needed | Colleague has extra access from old jobs | Add **only** the approved groups | 🔲 |
| "My mapped drive is missing" | `gpresult /r` · user in the right OU/group? · can the PC reach the server (`Test-NetConnection -Port 445`)? | GPO not applied · group change without logoff · server unreachable | Fix GPO/group · `gpupdate /force` · log off/on | 🔲 |
| "The file is locked by another user" | Who has it open? (`Get-SmbOpenFile` on the file server) | Someone left it open · a crashed session | Ask the user to close it · close the open file (warn: unsaved changes lost) | 🔲 |
| "I deleted a file by mistake" | Which folder? When? | Accidental delete | **Previous Versions** (Shadow Copy) · or restore from backup | ✅ (lab: Volume Shadow Copy restore) |

## Joiners, movers, leavers

| User says | First checks (in order) | Usual causes | Fix | Status |
|---|---|---|---|---|
| "New starter on Monday" | Ask the manager: **folders/access**, **PC**, **start date**, **end date** (contractor), **password handover** | The request is missing information | Create in the right OU · manager · expiry for contractors · **disabled until the start day** · groups · test · never email passwords | ✅ [TICKET-002](TICKET-002-new-contractor-account.md) |
| "X is moving to another team" | Manager approval · current groups | — | Move OU · update Department/Title/Manager · **add new groups AND remove old ones** | ✅ (part of TICKET-001) |
| "X has left the company" | **Confirm with the manager** · **when** exactly (end of day?) · **why** (friendly → end of day; dismissed → now) · who needs the files? · equipment? | — | **In AD, not just GLPI:** record groups → remove groups → **disable** (don't delete) → Description with date + ticket → move to **Disabled** OU → then GLPI inactive · **Pending** until equipment is back · L1 may need to **escalate** | ✅ [TICKET-005](TICKET-005-leaver.md) |

## Network

| User says | First checks (in order) | Usual causes | Fix | Status |
|---|---|---|---|---|
| "No internet" | One user or many? · `ipconfig /all` · ping gateway · ping 8.8.8.8 · `nslookup` | Cable/Wi-Fi · `169.254` (no DHCP) · DNS | See [network cheat sheet](../cheat-sheets/network-troubleshooting.md#1-no-internet--no-network-at-all) | 🔲 |
| "Works by IP, not by name" | `ipconfig /all` (DNS = the DCs?) · `nslookup` · flush cache | Wrong DNS server · missing record · old cache | Fix DNS settings · add the record · `ipconfig /flushdns` | ✅ (lab: LN-FS01 had no A record) |
| "It's slow" | What exactly? Wired or Wi-Fi? · `Get-NetAdapter` LinkSpeed · switch port errors | Bad cable (100 Mbps) · Wi-Fi signal · busy link | Replace cable · move closer to the AP · escalate | 🔲 |
| "PC on desk 12 has no network" | Link lights · `show interfaces status` · VLAN of the port | Unplugged · wrong VLAN · port err-disabled | Fix cable/VLAN · find the port by MAC | 🔲 |

## Printing

| User says | First checks (in order) | Usual causes | Fix | Status |
|---|---|---|---|---|
| "I can't print" | One user or everyone? · printer on, paper, errors on its screen? · ping the printer's IP · stuck queue? | Stuck queue · printer offline/out of paper · printer changed IP | Clear the queue · restart **Spooler** · DHCP **reservation** for the printer | 🔲 |
| "The printer is missing from my PC" | `gpresult /r` · right OU/group? | GPO not applied | Fix the GPO · `gpupdate /force` · add it from `\\LN-FS01` | 🔲 |

---

## Patterns I keep seeing (lessons across tickets)

- **Ask in the user's language** — "Which team are you in?", not "Are you in GG_Finance?"
- **Approval before access** — and write it in the ticket.
- **Group changes need a log off / log on.**
- **Verify every change** — `PasswordLastSet`, Effective Access, a real user test.
- **Test that the wrong people are denied**, not only that the right people get in.
- **It's always DNS** — test by IP, then by name.
- **Time matters** — wrong time zone broke certificates; an old PowerShell window showed old times.
- **Never send passwords by email.**