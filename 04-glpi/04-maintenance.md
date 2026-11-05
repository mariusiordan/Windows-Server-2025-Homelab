# 4. Maintenance

> Keep GLPI safe and healthy: snapshots and backups, updates, automatic user sync, admin password recovery, and regular security checks.
> **GUI** where there is one (Proxmox, GLPI), **terminal (bash)** for the server.

[← Using GLPI](03-using-glpi.md) · [Back to GLPI](README.md) · Next: [Troubleshooting →](05-troubleshooting.md)

---

## Contents

1. [Snapshots and backups](#1-snapshots-and-backups)
2. [Updating the server](#2-updating-the-server)
3. [Updating GLPI](#3-updating-glpi)
4. [Keeping users in sync with AD](#4-keeping-users-in-sync-with-ad)
5. [Recovering the admin password](#5-recovering-the-admin-password)
6. [Security checks](#6-security-checks)
7. [Monthly checklist](#7-monthly-checklist)

---

## 1. Snapshots and backups

| Type | What it is | Good for |
|---|---|---|
| **Snapshot** | A "save point" of the container, on the same disk | Quick undo before a change |
| **Backup** (vzdump) | A full copy of the container in a backup file | Disk failure, or moving to another host |
| **Database dump** | Just the GLPI data (tickets, users, settings) | Restoring data, or moving GLPI to a new server |

> ⚠️ A snapshot is **not** a backup: if the disk dies, the snapshots die with it.

### Snapshot

**GUI:** Proxmox → CT 230 → **Snapshots → Take Snapshot** → name (e.g. `before-update`).

**Terminal (Proxmox host):**

```bash
pct snapshot 230 before-update
pct listsnapshot 230
pct rollback 230 before-update      # undo (the container must be stopped)
```

### Full backup

**GUI:** Proxmox → CT 230 → **Backup → Backup now** → choose the storage → Mode **Snapshot** → **Backup**.
For automatic backups: **Datacenter → Backup → Add** (schedule, e.g. daily at 02:00).

**Terminal (Proxmox host):**

```bash
vzdump 230 --mode snapshot --storage local --compress zstd
```

### Database dump (inside the container)

```bash
sudo mariadb-dump glpi > ~/glpi-db-$(date +%F).sql
ls -lh ~/glpi-db-*.sql
```

Also keep a copy of `/var/www/glpi/config/` (it holds the **encryption key** `glpicrypt.key` — without it, saved passwords in GLPI can't be read after a restore).

---

## 2. Updating the server

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
sudo systemctl restart apache2
```

> 💡 Kernel updates happen on the **Proxmox host**, not in the container.
> 💡 Before a big upgrade (e.g. a new PHP version), check that your GLPI version supports it.

📸 Snapshot first: `before-apt-upgrade`.

---

## 3. Updating GLPI

GLPI updates replace the program files and then upgrade the database.

**Overview of the process:**

1. Read the **release notes** for the new version (supported PHP, breaking changes).
2. 📸 **Snapshot + database dump** (section 1).
3. Download the new `.tgz` from the GLPI GitHub releases page.
4. Replace the program files, **keeping** `config/`, `files/`, `marketplace/` (and `plugins/` if used).
5. Run the database update:

   ```bash
   sudo -u www-data php /var/www/glpi/bin/console database:update
   ```

6. Log in and check: version shown in **Setup → General**, tickets still there, AD login still works.

> ⚠️ Follow the **official upgrade documentation** for the exact steps of your version. If anything breaks, roll back the snapshot.

---

## 4. Keeping users in sync with AD

New starters in AD should appear in GLPI, and changed details (name, email, department) should update.

### GUI

**Administration → Users → LDAP directory link**:
- **Import new users** — adds new AD users
- **Synchronize already imported users** — updates existing ones

### Terminal (can be automated)

```bash
sudo -u www-data php /var/www/glpi/bin/console ldap:synchronize_users
```

To run it **every night at 02:30**, add a cron job for `www-data`:

```bash
sudo crontab -u www-data -e
```

```
30 2 * * * php /var/www/glpi/bin/console ldap:synchronize_users --no-interaction
```

> 💡 Real sysadmin automation: new starters appear in the helpdesk without anyone clicking anything.

### GLPI's own background jobs

GLPI runs **automatic actions** (sending emails, closing old tickets, cleaning logs). Check them in **Setup → Automatic actions**. They run when users browse GLPI, or from a system cron job — see the GLPI documentation for your version.

---

## 5. Recovering the admin password

Lost the `glpi` admin password? (Happened twice in this lab. 😄 Use a password manager!)

### Option 1 — use another admin

Log in as `marius.iordan` (AD, Super-Admin) → **Administration → Users → glpi** → set a new password.

### Option 2 — terminal (if no admin can log in)

1. 📸 Snapshot: `pct snapshot 230 before-admin-reset`
2. Check the account in the database:

   ```bash
   sudo mariadb glpi -e "SELECT id, name, authtype, is_active FROM glpi_users WHERE name='glpi';"
   ```

   `authtype 1` = GLPI internal database · `3` = LDAP/AD · `is_active` must be `1`.

3. Look for a built-in command first:

   ```bash
   sudo -u www-data php /var/www/glpi/bin/console list | grep -i password
   ```

4. If there isn't one, write a new **bcrypt hash** into the database (letters and numbers only, to avoid keyboard problems):

   ```bash
   HASH=$(php -r 'echo password_hash("NewPassword2026", PASSWORD_DEFAULT);')
   sudo mariadb glpi -e "UPDATE glpi_users SET password='$HASH' WHERE name='glpi';"
   sudo mariadb glpi -e "SELECT name, LEFT(password,4) FROM glpi_users WHERE name='glpi';"   # $2y$
   ```

5. Log in: `glpi` · Login source **GLPI internal database** → save the password in your password manager.

> 💡 GLPI stores passwords as a **hash** (a one-way fingerprint), never as plain text. That's why we can't "read" the old password — only replace it.

### Lost the Linux password?

From the **Proxmox host**: `pct enter 230` gives a root shell without a password, then `passwd marius`.
Whoever controls the Proxmox host controls every container — protect it well.

---

## 6. Security checks

| Check | How | Expected |
|---|---|---|
| Default accounts disabled | `sudo mariadb glpi -e "SELECT name, is_active FROM glpi_users WHERE name IN ('tech','normal','post-only');"` | `is_active = 0` for all three |
| Web installer removed | `ls /var/www/glpi/install/install.php` | "No such file" |
| DB config protected | `ls -l /var/www/glpi/config/config_db.php` | `-rw-r-----` (640) |
| Root SSH login blocked | `sudo sshd -T \| grep permitrootlogin` | `without-password` / `prohibit-password` |
| No warning banners | GLPI home page as admin | No red/orange security warnings |
| Admins are named accounts | **Administration → Users** → filter by profile Super-Admin | Only `glpi` (break-glass) + named IT staff |

---

## 7. Monthly checklist

- [ ] Snapshot and backup taken; old snapshots deleted
- [ ] `apt update && apt full-upgrade` done
- [ ] New GLPI release? Read the notes, plan the update
- [ ] **LDAP directory → Test** all green
- [ ] Users synchronized with AD (leavers no longer active)
- [ ] Security checks (section 6) still OK
- [ ] Certificates on the DCs not close to expiry (`certlm.msc` on each DC)

---

Next: [Troubleshooting →](05-troubleshooting.md)