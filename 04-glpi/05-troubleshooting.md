# 5. Troubleshooting GLPI

> Real problems from this lab (installation, the AD connection, logins), the method I use, a command toolbox and where to find the logs.

[← Maintenance](04-maintenance.md) · [Back to GLPI](README.md)

---

## Contents

1. [The method: test each layer](#the-method-test-each-layer)
2. [Login problems](#login-problems)
3. [Real problems from this lab](#real-problems-from-this-lab)
4. [Command toolbox](#command-toolbox)
5. [Logs](#logs)

---

## The method: test each layer

GLPI depends on several layers. **Test each one outside GLPI**, from the bottom up — the first one that fails is your problem.

| # | Layer | Test | Expected |
|---|---|---|---|
| 1 | Container running, network | `ping 192.168.7.83` | Replies |
| 2 | Web server | `systemctl status apache2` | active (running) |
| 3 | Database | `mysql -u glpi -p'<pw>' -e "SELECT 1;" glpi` | `1` |
| 4 | DNS to the DC | `getent hosts ln-dc01.corp.silverbank.internal` | 192.168.7.80 |
| 5 | TLS to the DC | `openssl s_client -connect ln-dc01.corp.silverbank.internal:636 …` | `Verify return code: 0 (ok)` |
| 6 | AD login (service account) | `ldapsearch … -D svc_glpi@…` | `result: 0 Success` |
| 7 | AD login (the user) | `ldapsearch … -D james.carter@…` | `result: 0 Success` |
| 8 | GLPI itself | **Setup → Authentication → LDAP directories → Test** | All green |

If layers 1–7 work and GLPI still fails → the problem is in the **GLPI settings**.

---

## Login problems

### Step 1 — which account, which source?

| Account type | Must use Login source |
|---|---|
| AD user (`james.carter`, `marius.iordan`) | **SilverBank AD** |
| Local GLPI account (`glpi`) | **GLPI internal database** |

### Step 2 — test the user's password directly against AD

```bash
ldapsearch -x -H ldaps://ln-dc01.corp.silverbank.internal \
  -D "james.carter@corp.silverbank.internal" -W -b "" -s base | grep -E "result|data"
```

| Result | Meaning | Fix (GUI, on a DC) |
|---|---|---|
| `result: 0 Success` | Password is fine → the problem is in GLPI | Check the LDAP directory settings, the user is **Active** in GLPI |
| `data 52e` | **Wrong password** | Reset Password — watch the **keyboard layout** |
| `data 525` | User **not found** | Check the username / UPN |
| `data 773` | **Must change password** at next logon | Untick the box, or the user changes it at Windows login first |
| `data 775` | Account **locked** | Account tab → **Unlock account** |
| `data 533` | Account **disabled** | Right-click → **Enable Account** |

### Step 3 — did the password reset really save?

```powershell
Get-ADUser james.carter -Properties PasswordLastSet | Format-List PasswordLastSet
```

Empty, or an old date → the reset didn't happen. Do it again and **verify**.

### Step 4 — is the GLPI account local or AD?

```bash
sudo mariadb glpi -e "SELECT id, name, authtype, is_active, is_deleted FROM glpi_users WHERE name='james.carter';"
```

`authtype`: `1` = GLPI database · `3` = LDAP/AD · `is_active` must be `1` · `is_deleted` must be `0`.

---

## Real problems from this lab

### Installation

| Problem | Cause | Fix |
|---|---|---|
| `nslookup` printed nothing | Minimal container — tool missing | Tested with `getent hosts` (works for every program); installed `dnsutils` |
| Installer: "Security configuration for sessions" not OK | Apache still running with the old PHP settings | Restarted Apache; verified the live values with a temporary PHP test page |
| `mysql -u glpi -p` "Access denied" with the right password | Proxmox console **keyboard layout** changed the symbols typed at the hidden prompt | Tested with `-p'<password>'`; used SSH instead of the web console |
| "The connection to the SQL server could not be established" | `config_db.php` held an **old password** from a first install attempt | Found repeated "Access denied" in `journalctl -u mariadb` (an app retrying), compared `config_db.php` with the real password |
| Login "Incorrect username or password" after fixing the config | Unclear state after several partial attempts | **Clean reinstall:** removed `config_db.php` and `glpicrypt.key`, recreated the database, installed with `bin/console database:install` |

### Connecting to Active Directory

| Problem | Cause | Fix |
|---|---|---|
| LDAP bind refused: *"Strong(er) authentication required"* | WS2025 DCs **require LDAP signing** on port 389 | Built an internal CA and switched to **LDAPS (636)** |
| Certificate request: *"RPC server is unavailable"* | No **DNS record** for LN-FS01 (the CA), and LN-DC01's DNS pointed to the **router** | Added the record; fixed DC01's DNS client |
| *"Certificate is not yet valid"* | Servers on **Pacific Time** — about 7 hours wrong | Set **GMT Standard Time** on all servers, resynced time |

### Logins and access

| Problem | Cause | Fix |
|---|---|---|
| `ssh root@192.168.7.83` → "Permission denied" with the correct password | Ubuntu **blocks root password login over SSH** | Created a `marius` user with sudo + SSH key |
| Employee login fails, `ldapsearch` shows `data 52e` | Wrong password — **US keyboard, UK layout** | Reset with letters and numbers only; matched keyboard layouts |
| Password "reset" but still `52e` | The reset **never saved** — `PasswordLastSet` was empty | Reset again and verified `PasswordLastSet` |
| Admin can't log in after connecting AD | AD set as **default** Login source; `glpi` is a **local** account | Chose **GLPI internal database** on the login page |
| Forgot the `glpi` admin password (twice) | No password manager 😄 | Wrote a new bcrypt hash into the database ([Maintenance](04-maintenance.md#5-recovering-the-admin-password)); started using a password manager |
| "Import new users" button not found | Moved in GLPI 12 | `bin/console ldap:synchronize_users --only-create-new` |

### Lessons

- **Test outside the application** to find which layer is broken.
- Changing a password in one place doesn't update the **apps that use it** — check their config files.
- A burst of automatic "Access denied" lines in a log = a **service retrying** with saved credentials.
- When an install has been half-done several times, a **clean reinstall** to a known state is faster than guessing.
- **Look at the prompt** (`root@pve` vs `marius@LN-GLPI01`) before every command.
- **Verify every change** — `PasswordLastSet`, `Verify return code`, `result: 0 Success` are proof.

---

## Command toolbox

Start with the ⭐ ones. Commands that change things need `sudo`.

### Services

| Command | What it tells you |
|---|---|
| ⭐ `systemctl status apache2 --no-pager` | Is the web server running? |
| ⭐ `systemctl status mariadb --no-pager` | Is the database running? |
| `sudo systemctl restart apache2` | Restart the web server (after PHP/Apache changes) |
| `apache2ctl configtest` | Is the Apache config valid? ("Syntax OK") |
| `php -m \| grep -i ldap` | Is the PHP LDAP module loaded? |

### Network and AD

| Command | What it tells you |
|---|---|
| ⭐ `getent hosts ln-dc01.corp.silverbank.internal` | Does DNS work for applications? |
| `resolvectl status` | Which DNS servers the container uses |
| ⭐ `openssl s_client -connect ln-dc01.corp.silverbank.internal:636 -verify_return_error </dev/null` | Does LDAPS + the certificate check work? |
| ⭐ `ldapsearch -x -H ldaps://… -D user@corp.silverbank.internal -W -b "" -s base` | Can this account log in to AD? (error `data` codes) |

### Database

| Command | What it tells you |
|---|---|
| `sudo mariadb glpi -e "SELECT id, name, authtype, is_active FROM glpi_users;"` | GLPI users, their login type and status |
| `mysql -u glpi -p'<pw>' -e "SELECT 1;" glpi` | Does the GLPI database login work? |

### GLPI console

| Command | What it does |
|---|---|
| `sudo -u www-data php /var/www/glpi/bin/console list` | All available console commands |
| `sudo -u www-data php /var/www/glpi/bin/console ldap:synchronize_users` | Import/update users from AD |
| `sudo -u www-data php /var/www/glpi/bin/console database:update` | Upgrade the database after a GLPI update |

### Proxmox host

| Command | What it does |
|---|---|
| `pct enter 230` | Root shell inside the container (no password) |
| `pct snapshot 230 <name>` · `pct listsnapshot 230` | Take / list snapshots |
| `pct set 230 --nameserver 192.168.7.80` | Change the container's DNS server |

---

## Logs

| Log | Where | Good for |
|---|---|---|
| GLPI events | **GUI:** Administration → **Logs** | Logins, changes, who did what |
| GLPI application logs | `/var/www/glpi/files/_log/` (e.g. `php-errors.log`, `sql-errors.log`) — the path can differ by version | PHP and database errors |
| Apache | `/var/log/apache2/error.log` · `access.log` | Web errors, who opened which page |
| MariaDB | `journalctl -u mariadb` | Database errors, repeated "Access denied" |
| System | `journalctl -xe` | Everything else |
| AD side (on the DC) | Event Viewer → **Security** → 4625 / 4771 | Failed logins as the DC sees them |

```bash
sudo tail -f /var/log/apache2/error.log     # watch live — Ctrl+C to stop
```