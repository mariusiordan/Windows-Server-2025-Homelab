# 4. Adding Clients (LN-CL01 — Windows 11 Pro)

> Join a Windows 11 PC to the domain, put it in the right OU, and prove that a domain user can log in.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← File server](03-file-server.md) · [Back to Active Directory](README.md) · Next: [Troubleshooting →](05-troubleshooting.md)

---

## Contents

1. [Before you start](#before-you-start)
2. [Step 1: Point the client's DNS to the DCs](#step-1-point-the-clients-dns-to-the-dcs)
3. [Step 2: Join the domain](#step-2-join-the-domain)
4. [Step 3: Move the computer to the right OU](#step-3-move-the-computer-to-the-right-ou)
5. [Step 4: Log in as a domain user](#step-4-log-in-as-a-domain-user)
6. [Step 5: Check the client is healthy](#step-5-check-the-client-is-healthy)

---

## Before you start

- [ ] **Windows 11 Pro, Enterprise or Education.** ⚠️ **Windows 11 Home cannot join a domain.**
- [ ] Time zone: **GMT Standard Time**
- [ ] A clear computer name, e.g. `LN-CL01`
- [ ] Both DCs running

---

## Step 1: Point the client's DNS to the DCs

The client gets its IP from the **home router's DHCP**, and the router gives **itself** as DNS. The router knows nothing about `corp.silverbank.internal`, so the join would fail.

**For now:** set DNS **manually** to the DCs.
**Later:** DHCP on the DCs will give clients the right DNS automatically (see `03-dhcp`).

### GUI

1. **Settings → Network & internet → Ethernet**.
2. **DNS server assignment → Edit** → **Manual** → turn on **IPv4**.
3. Preferred DNS: `192.168.7.80` · Alternate DNS: `192.168.7.82` → **Save**.

*(Or the classic way: **Win + R** → `ncpa.cpl` → adapter → **Properties → IPv4 → Properties**.)*

### PowerShell (optional)

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,192.168.7.82
```

### Check

```powershell
nslookup corp.silverbank.internal     # answers with the DCs' IPs
```

---

## Step 2: Join the domain

### GUI

1. **Win + R** → `sysdm.cpl` → **Enter**.
2. **Computer Name** tab → **Change…**
3. Computer name: `LN-CL01` (if not set yet).
4. Member of: **Domain** → `corp.silverbank.internal` → **OK**.
5. Credentials: `SILVERBANK\Administrator` → **OK**.
6. "Welcome to the corp.silverbank.internal domain" → **OK** → **restart**.

*(Windows 11 alternative: **Settings → System → About → Domain or workgroup**, which opens the same window.)*

### PowerShell (optional)

```powershell
Add-Computer -DomainName corp.silverbank.internal -Credential SILVERBANK\Administrator -Restart
```

---

## Step 3: Move the computer to the right OU

New computers land in the default **Computers** container, where your Group Policies don't apply.

### GUI

On LN-DC01: **Active Directory Users and Computers** → **Computers** → right-click `LN-CL01` → **Move…** → `SilverBank > Computers > Workstations` → **OK**.

Then restart the client, so it picks up the policies for its new OU.

### PowerShell (optional)

```powershell
Get-ADComputer LN-CL01 |
  Move-ADObject -TargetPath "OU=Workstations,OU=Computers,OU=SilverBank,DC=corp,DC=silverbank,DC=internal"
```

---

## Step 4: Log in as a domain user

### GUI

1. At the login screen: **Other user**.
2. Type `SILVERBANK\james.carter` (or `james.carter@corp.silverbank.internal`) + password.
3. The first login takes a minute (Windows creates the user profile).

> ⚠️ **Login fails?** Check:
> - **Keyboard layout** (UK ↔ US swaps `@` and `"`) — real problem in this lab!
> - **Caps Lock**
> - "User must change password at next logon" is ticked → Windows will ask for a new password (that's normal)
> - The account is **disabled** or **locked** (see [Troubleshooting](05-troubleshooting.md))

---

## Step 5: Check the client is healthy

### GUI

| Where | What you should see |
|---|---|
| **Settings → System → About** | "Domain: corp.silverbank.internal" |
| Network icon → network name | `corp.silverbank.internal` |
| File Explorer → `\\LN-FS01\Finance` (as a Finance user) | The folder opens |
| On LN-DC01: AD Users and Computers → **Workstations** | `LN-CL01` is listed |

### Commands (optional — run them on the client)

```powershell
whoami                                    # silverbank\james.carter
whoami /groups                            # the user's groups (GG_Finance, ...)
gpresult /r                               # which Group Policies applied, and from which DC
nltest /sc_verify:corp.silverbank.internal   # the secure channel between PC and domain is OK
echo %LOGONSERVER%                        # which DC logged the user in (cmd) — or $env:LOGONSERVER in PowerShell
```

📸 Snapshot: `after-CL01-joined`.

---

## What I learned

- **Windows Home can't join a domain** — check the edition first.
- The client must use the **DCs as DNS** — the router can't find the domain.
- Move computers into the right **OU**, or Group Policy won't apply.
- `whoami /groups` and `gpresult /r` show what the user and computer really received.

Next: [Troubleshooting →](05-troubleshooting.md)