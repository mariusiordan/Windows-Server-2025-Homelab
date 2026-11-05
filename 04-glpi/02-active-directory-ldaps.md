# 2. Connecting GLPI to Active Directory (LDAPS)

> Let SilverBank employees log in to GLPI with their **Windows domain account**. GLPI asks a domain controller over **LDAPS** (encrypted LDAP, port 636) whether the password is correct.
> **GUI first**, then the **terminal** tests, then **Check**.

[← Installation](01-installation.md) · [Back to GLPI](README.md) · Next: [Using GLPI →](03-using-glpi.md)

---

## Contents

1. [Why LDAPS and not LDAP](#why-ldaps-and-not-ldap)
2. [Step 1: Create the service account](#step-1-create-the-service-account)
3. [Step 2: Certificates on the domain controllers](#step-2-certificates-on-the-domain-controllers)
4. [Step 3: Trust the SilverBank CA on the GLPI server](#step-3-trust-the-silverbank-ca-on-the-glpi-server)
5. [Step 4: Test from the GLPI server](#step-4-test-from-the-glpi-server)
6. [Step 5: Add the LDAP directory in GLPI](#step-5-add-the-ldap-directory-in-glpi)
7. [Step 6: Import the users](#step-6-import-the-users)
8. [Step 7: Test an employee login](#step-7-test-an-employee-login)
9. [Step 8: Use your own AD account as admin](#step-8-use-your-own-ad-account-as-admin)

---

## Why LDAPS and not LDAP

Windows Server 2025 domain controllers **require LDAP signing**. A plain LDAP login on port **389** is refused with:

```
Strong(er) authentication required
```

Two options:

| Option | Result |
|---|---|
| Weaken the DC's security policy | ❌ Works, but makes the whole domain less secure |
| Use **LDAPS** (LDAP inside TLS, port **636**) with a certificate from our own CA | ✅ Secure — **what I did** |

**What's needed:** a certificate on each DC, issued by the SilverBank CA (`corp-LN-FS01-CA`), and the GLPI server must **trust** that CA.

---

## Step 1: Create the service account

GLPI needs an AD account to **search** the directory. It never logs in interactively.

### GUI (LN-DC01)

**Active Directory Users and Computers** → `SilverBank > Service Accounts` → right-click → **New → User**:

1. First name: `svc_glpi` · User logon name: `svc_glpi` → Next.
2. Set a **long password** (save it in your password manager).
3. ❌ User must change password at next logon
4. ✅ User cannot change password
5. ✅ Password never expires
6. **Next → Finish**.

> 💡 It's a normal **Domain User** — reading the directory needs no admin rights. **Least privilege.**

### PowerShell (optional)

```powershell
New-ADUser -Name "svc_glpi" -SamAccountName "svc_glpi" `
  -UserPrincipalName "svc_glpi@corp.silverbank.internal" `
  -Path "OU=Service Accounts,OU=SilverBank,DC=corp,DC=silverbank,DC=internal" `
  -AccountPassword (Read-Host -AsSecureString "Password") `
  -Enabled $true -PasswordNeverExpires $true -CannotChangePassword $true
```

---

## Step 2: Certificates on the domain controllers

Each DC needs a certificate for its **own name** (`ln-dc01.corp.silverbank.internal`), with **Server Authentication**. Once it has one, the DC starts answering on port 636 automatically.

### Before you request — check these three things (they caused real problems in this lab)

| Check | GUI | Command |
|---|---|---|
| **Time zone** is GMT on every server | Server Manager → Local Server → Time zone | `tzutil /g` |
| The DC's **DNS client** points to the DCs, not the router | Adapter → IPv4 → DNS | `ipconfig /all` |
| The CA server has a **DNS record** | DNS Manager → `ln-fs01` | `nslookup ln-fs01` |

### GUI (on each DC)

1. **Win + R** → `certlm.msc` (Certificates – Local Computer).
2. **Personal → Certificates** → right-click → **All Tasks → Request New Certificate…**
3. Next → **Active Directory Enrollment Policy** → Next.
4. Tick **Kerberos Authentication** → **Enroll** → **Finish**.

### PowerShell (optional)

```powershell
Get-Certificate -Template "KerberosAuthentication" -CertStoreLocation Cert:\LocalMachine\My
```

### Check

| GUI | PowerShell |
|---|---|
| `certlm.msc` → Personal → Certificates: issued **to** the DC's name, issued **by** `corp-LN-FS01-CA`, valid **from** a date in the past | `Get-ChildItem Cert:\LocalMachine\My \| Format-List Subject, Issuer, NotBefore, NotAfter, EnhancedKeyUsageList` |

---

## Step 3: Trust the SilverBank CA on the GLPI server

The GLPI server must trust certificates issued by `corp-LN-FS01-CA`, or the TLS check fails.

### Export the CA certificate (GUI, on LN-FS01)

1. **Server Manager → Tools → Certification Authority**.
2. Right-click `corp-LN-FS01-CA` → **Properties → View Certificate**.
3. **Details → Copy to File…** → **Base-64 encoded X.509 (.CER)** → save as `silverbank-root-ca.crt`.
4. Copy the file to the GLPI server (e.g. with WinSCP, or `scp` from your computer).

*(PowerShell alternative: `certutil -ca.cert C:\silverbank-root-ca.cer`, then convert to Base-64 with `certutil -encode`.)*

### Install it on the GLPI server (terminal)

```bash
sudo cp silverbank-root-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates        # "1 added"
```

---

## Step 4: Test from the GLPI server

**Test from the bottom up.** If one step fails, fix it before going on.

### 4a. DNS — can GLPI find the DC by name?

```bash
getent hosts ln-dc01.corp.silverbank.internal     # → 192.168.7.80
```

If nothing comes back, the container uses the router for DNS. Fix it on the Proxmox host:

```bash
pct set 230 --nameserver 192.168.7.80 --searchdomain corp.silverbank.internal
pct reboot 230
```

### 4b. TLS — does the certificate check pass?

```bash
openssl s_client -connect ln-dc01.corp.silverbank.internal:636 -verify_return_error </dev/null 2>/dev/null \
  | grep -E "subject=|issuer=|Verify return"
```

**Expect:**

```
subject=CN=LN-DC01.corp.silverbank.internal
issuer=DC=internal, DC=silverbank, DC=corp, CN=corp-LN-FS01-CA
Verify return code: 0 (ok)
```

> 💡 Always connect by **name**, not IP: the certificate is issued to the name, so an IP would fail the check.

### 4c. LDAP — can the service account read AD?

```bash
sudo apt install -y ldap-utils
ldapsearch -x -H ldaps://ln-dc01.corp.silverbank.internal \
  -D "svc_glpi@corp.silverbank.internal" -W \
  -b "OU=SilverBank,DC=corp,DC=silverbank,DC=internal" \
  "(objectClass=user)" sAMAccountName | grep -E "sAMAccountName|result"
```

**Expect:** a list of usernames, then `result: 0 Success`.
`-W` asks for the password, so it isn't saved in the bash history.

---

## Step 5: Add the LDAP directory in GLPI

### GUI

**Setup → Authentication → LDAP directories → + Add**

First click the **Active Directory** preconfiguration link at the top (fills sensible defaults), then:

| Field | Value | Why |
|---|---|---|
| Name | `SilverBank AD` | |
| Default server | Yes | Used by default on the login page |
| Active | Yes | |
| Server | `ldaps://ln-dc01.corp.silverbank.internal` | `ldaps://` = encrypted |
| Port | `636` | LDAPS port |
| Connection filter | `(&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` | People only (no computers) and **no disabled accounts** |
| BaseDN | `OU=Users,OU=SilverBank,DC=corp,DC=silverbank,DC=internal` | Only staff OUs (no service accounts, no computers) |
| Use bind | Yes | Log in with the service account to search |
| RootDN (bind DN) | `svc_glpi@corp.silverbank.internal` | |
| Password | the svc_glpi password | |
| Login field | `samaccountname` | Staff type the same short name as at Windows login (`james.carter`) |
| Synchronization field | `objectguid` | A permanent ID — renaming a user in AD doesn't create a "new" GLPI user |

**Add** → open **SilverBank AD** → **Test** tab → **Test**.

### Check — every line green

```
TCP stream          Connection to ln-dc01.corp.silverbank.internal on port 636 succeeded
Base DN             configured
LDAP URI            check succeeded
Bind connection     Authentication succeeded
Search              Search succeeded (38 entries found)
```

> 💡 In this lab `ldapsearch` found 39 people but GLPI found 38 — one account was **disabled**, and the filter correctly left it out. Leavers must not log in to the helpdesk.

---

## Step 6: Import the users

### GUI

**Administration → Users → LDAP directory link → Import new users → Search** → tick all → **Action → Import**.
*(Direct address: `http://192.168.7.83/front/ldap.import.php`)*

### Terminal (what I used — the button was hard to find in GLPI 12)

```bash
sudo -u www-data php /var/www/glpi/bin/console ldap:synchronize_users --help
sudo -u www-data php /var/www/glpi/bin/console ldap:synchronize_users --only-create-new
```

> 💡 Users are also created **automatically at their first login**. Importing them first lets you assign tickets to them before they ever log in.

### Check

**Administration → Users**: the staff are listed, and `svc_glpi` and computer accounts (`LN-CL01$`) are **not**.

---

## Step 7: Test an employee login

### GUI

1. Open a **private/incognito window** → `http://helpdesk`.
2. Log in as `james.carter` with his **AD password** · Login source **SilverBank AD**.
3. **Expect:** the **Self-Service** page with **Create a ticket**. ✅

If it fails, test the password directly against AD — see [Troubleshooting](05-troubleshooting.md#login-problems).

---

## Step 8: Use your own AD account as admin

Daily work should use a **named** account, not the shared `glpi` admin.

### GUI

1. Log in as `glpi` (Login source **GLPI internal database**).
2. **Administration → Users → marius.iordan → Authorizations** tab → **Add**:
   - Profile: **Super-Admin** · Entity: **Root entity** · Recursive: **Yes**.
3. Log out → log in as `marius.iordan` · Login source **SilverBank AD**.

Keep `glpi` as the **break-glass** account for when AD is unavailable.

---

## What I learned

- WS2025 DCs refuse plain LDAP binds → **LDAPS with an internal CA** is the secure fix.
- **Time, DNS and certificates** all have to be right — each one broke the connection once.
- **Test outside the app first:** `getent` → `openssl` → `ldapsearch` → GLPI.
- The **filter + BaseDN** keep computers, service accounts and disabled users out.
- **Centralised authentication:** one password for Windows and the helpdesk.

Next: [Using GLPI →](03-using-glpi.md)