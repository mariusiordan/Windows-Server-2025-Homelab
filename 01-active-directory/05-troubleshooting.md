# 5. Troubleshooting Active Directory

> Real problems from this lab, the AD error codes I learned, and a **command toolbox** with simple explanations.
> In each section: **GUI first**, then the command that gives the same (or more) information.

[← Clients](04-clients.md) · [Back to Active Directory](README.md)

---

## Contents

1. [The method](#the-method)
2. [Real problems from this lab](#real-problems-from-this-lab)
3. [Common errors](#common-errors)
4. [AD login error codes](#ad-login-error-codes)
5. [Command toolbox](#command-toolbox)
6. [Event Viewer: where to look](#event-viewer-where-to-look)

---

## The method

1. **Understand** — who, what, since when, what changed?
2. **Test by IP, then by name.** IP works but name fails → **DNS**. Both fail → **network / firewall / server off**.
3. **Work from the bottom up:** cable/VM on → IP → DNS → time → service running → permissions.
4. **Change one thing at a time**, then test again.
5. **Verify with proof** (a command or a real user test), not "I think it's fixed".
6. **Document** the cause, fix and verification.

> **It's always DNS** — until you've proved it isn't. 😄

---

## Real problems from this lab

| # | Symptom | Real cause | Fix | How I found it |
|---|---|---|---|---|
| 1 | LDAP bind failed: *"Strong(er) authentication required"* | Windows Server 2025 DCs **require LDAP signing** — plain LDAP on port 389 is refused | Built an **Enterprise Root CA** on LN-FS01 and used **LDAPS (port 636)** — instead of weakening security | The error text named the policy |
| 2 | Certificate request: *"The RPC server is unavailable"* | **LN-FS01 had no DNS A record**, and **LN-DC01's DNS pointed to the router** (settings landed on a Hyper-V **vEthernet** adapter) | Added the A record, fixed DC01's DNS client | `certutil -ping` worked **by IP**, failed **by name** → DNS |
| 3 | *"Certificate is not yet valid"* | Servers were on **Pacific Time** (Windows default) — about 7 hours wrong | `tzutil /s "GMT Standard Time"` on every server + time sync on the PDC Emulator | Compared certificate **NotBefore** with the real time |
| 4 | User can't log in to GLPI, error `data 52e` | Password **typed wrongly** because of the **keyboard layout** (US keyboard, UK layout) | Matched keyboard layouts; reset the password with **letters and numbers only** | `ldapsearch` against the DC returned the error code |
| 5 | Password reset "didn't work" | The reset **was never saved** — `PasswordLastSet` was empty | Reset again and **verified** | Checked **PasswordLastSet** |
| 6 | *"Access denied"* on `\\LN-FS01\Finance` | User **moved department**, not added to `GG_Finance` | Manager approval → added to the group → user logs off/on | Member Of tab + Effective Access ([TICKET-001](../tickets/TICKET-001-finance-folder-access.md)) |

---

## Common errors

| Error | Likely cause | What to check / fix |
|---|---|---|
| *"An Active Directory domain controller could not be contacted"* | The machine's DNS points to the **router** | Set DNS to `192.168.7.80` / `.82`; test `nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal` |
| *"The server is not operational"* | DC off, AD services stopped, or network profile **Public** | Services: NTDS, DNS, Netlogon running? Network shows the domain name? |
| *"The trust relationship between this workstation and the primary domain failed"* | The PC's **secure channel** with the domain is broken (e.g. restored an old snapshot) | Log in locally → `Test-ComputerSecureChannel -Repair -Credential SILVERBANK\Administrator` — or remove and re-join the domain |
| Wizard: *"user name or password is incorrect"* when adding a DC | Wizard uses the **local** account | Join the domain first, or **Change…** credentials to `SILVERBANK\Administrator` |
| Login: *"user name or password is incorrect"* | Wrong password, **keyboard layout**, **Caps Lock**, local vs domain account | Type `SILVERBANK\user`; test the password somewhere visible |
| *"The referenced account is currently locked out"* | Too many wrong passwords | Unlock the account (see toolbox), then find **where** the bad attempts come from |
| *"There are currently no logon servers available"* | The PC can't reach any DC (network/DNS) | `nltest /dsgetdc:corp.silverbank.internal` |
| Ping **by name** fails, **by IP** works | No DNS record, or old cache | `ipconfig /registerdns` on the target; `ipconfig /flushdns` on the client |
| Ping fails but services work | Windows Firewall blocks ping (ICMP) | Normal — or enable *"File and Printer Sharing (Echo Request – ICMPv4-In)"* |
| `nslookup` shows *"Server: UnKnown"* | No **reverse lookup zone** | Harmless; create a reverse zone (see `02-dns`) |
| New group access doesn't work | Group membership loads **at login** | User **logs off and on** (or `klist purge` + reconnect) |

---

## AD login error codes

When AD refuses a login (in `ldapsearch`, GLPI, application logs), it gives a **`data` code**:

| Code | Meaning | Fix in the GUI |
|---|---|---|
| `525` | User **not found** | Check the username / UPN / search base |
| `52e` | **Wrong password** (correct user) | Reset Password (watch the keyboard layout!) |
| `530` | Not allowed to log in **at this time** | Account → **Logon Hours** |
| `531` | Not allowed to log in **from this computer** | Account → **Log On To…** |
| `532` | **Password expired** | Reset Password |
| `533` | Account **disabled** | Right-click the user → **Enable Account** |
| `701` | Account **expired** | Account tab → **Account expires: Never** (or a new date) |
| `773` | User **must change password** at next logon | Untick the box, or let the user change it at Windows login |
| `775` | Account **locked out** | Account tab → ✅ **Unlock account** |

---

## Command toolbox

You don't need to remember all of these. Start with the ⭐ ones. Each command has a GUI equivalent where one exists.

### ⭐ DC health

| Command | What it tells you | GUI equivalent |
|---|---|---|
| `dcdiag /q` | Runs dozens of DC health tests; **prints nothing = healthy** | — |
| `dcdiag /test:dns` | Checks the DC's DNS setup in detail | DNS Manager |
| `repadmin /replsummary` | Replication between DCs; the **fails** column must be `0` | Sites and Services → Replicate Now |
| `repadmin /showrepl` | Replication details per partner | — |
| `netdom query fsmo` | Which DC holds the 5 **FSMO roles** (e.g. PDC Emulator) | AD Users and Computers → right-click domain → **Operations Masters** |
| `Get-ADDomainController -Filter * \| Select Name, IPv4Address` | Lists all DCs | AD Users and Computers → **Domain Controllers** |

### ⭐ DNS

| Command | What it tells you |
|---|---|
| `ipconfig /all` | IP, gateway and **which DNS servers** this machine uses — check this first! |
| `nslookup ln-dc01.corp.silverbank.internal` | Does this name resolve, and **which server answered**? |
| `nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal` | Lists the DCs — this is how computers **find** a DC |
| `Resolve-DnsName ln-fs01.corp.silverbank.internal` | PowerShell version of nslookup |
| `ipconfig /flushdns` | Clears the client's DNS cache (old answers) |
| `ipconfig /registerdns` | Makes the machine **re-create its own DNS record** |

GUI: **Server Manager → Tools → DNS** (`dnsmgmt.msc`) → Forward Lookup Zones.

### ⭐ Network

| Command | What it tells you |
|---|---|
| `ping 192.168.7.80` then `ping ln-dc01` | **By IP vs by name** — splits network problems from DNS problems |
| `Test-NetConnection ln-dc01 -Port 389` | Is a **port** open? (better than ping, which the firewall may block) |
| `tracert 192.168.7.1` | The path packets take (useful once VLANs exist) |

**Ports to know:** `53` DNS · `88` Kerberos · `135` RPC · `389` LDAP · `636` LDAPS · `445` SMB (file shares) · `3389` RDP

### ⭐ Time

| Command | What it tells you |
|---|---|
| `tzutil /g` | The current **time zone** (should be GMT Standard Time) |
| `w32tm /query /status` | Where this machine gets its time and when it last synced |
| `w32tm /resync /force` | Sync the time now |

GUI: **Settings → Time & language → Date & time**, or Server Manager → Local Server → **Time zone**.

### Client and domain membership

| Command | What it tells you |
|---|---|
| `nltest /dsgetdc:corp.silverbank.internal` | Which DC this computer found |
| `nltest /sc_verify:corp.silverbank.internal` | Is the **secure channel** (PC ↔ domain trust) OK? |
| `Test-ComputerSecureChannel` | PowerShell version — `True` = OK (`-Repair` fixes it) |
| `whoami /groups` | Which groups the logged-in user **really** has right now |
| `gpresult /r` | Which Group Policies applied to the user and the computer |
| `gpupdate /force` | Apply Group Policy now instead of waiting |
| `klist` / `klist purge` | Show / clear the user's Kerberos tickets (refreshes group membership) |

### ⭐ User accounts

| Command | What it tells you | GUI equivalent |
|---|---|---|
| `Get-ADUser james.carter -Properties Enabled, LockedOut, PasswordLastSet, LastLogonDate` | Account health at a glance | User → **Account** tab |
| `Search-ADAccount -LockedOut` | All locked accounts | — |
| `Unlock-ADAccount james.carter` | Unlocks the account | Account tab → **Unlock account** |
| `Get-ADPrincipalGroupMembership james.carter \| Select Name` | The user's groups | User → **Member Of** tab |
| `net user james.carter /domain` | Quick account summary (works on any domain PC) | — |

### Certificates (LDAPS)

| Command | What it tells you |
|---|---|
| `certutil -ping` | Can this machine reach the Certificate Authority? |
| `Get-ChildItem Cert:\LocalMachine\My` | Certificates on this computer (check **NotBefore / NotAfter**) |
| `openssl s_client -connect ln-dc01.corp.silverbank.internal:636` (from Linux) | Is LDAPS working? Look for `Verify return code: 0 (ok)` |

GUI: **Win + R** → `certlm.msc` → Personal → Certificates.

---

## Event Viewer: where to look

**Win + R** → `eventvwr.msc`

| Log | Event ID | Meaning |
|---|---|---|
| **Windows Logs → Security** (on the PDC Emulator DC) | **4740** | An account was **locked out** — shows the computer it came from |
| Windows Logs → Security | **4625** | A login **failed** (wrong password, etc.) |
| Windows Logs → Security | **4771** | **Kerberos** pre-authentication failed (often a wrong password) |
| Windows Logs → Security | **4624** | Successful login |
| Applications and Services Logs → **Directory Service** | — | AD database and replication problems |
| Applications and Services Logs → **DNS Server** | — | DNS server problems |
| Applications and Services Logs → **DFS Replication** | — | SYSVOL (Group Policy files) replication between DCs |
| Windows Logs → **System** (source: Time-Service) | — | Time sync problems |

> 💡 Use **Filter Current Log…** on the right and type the Event ID, so you don't scroll through thousands of events.