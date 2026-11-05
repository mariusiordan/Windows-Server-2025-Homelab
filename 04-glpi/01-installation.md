# 1. Installation

> Build GLPI in an Ubuntu container on Proxmox: create the container, install Apache, MariaDB and PHP, install GLPI, then secure it.
> **GUI** where there is one (Proxmox web UI, GLPI web interface), **terminal (bash)** for the Linux setup, then **Check**.

[← Back to GLPI](README.md) · Next: [Connecting to Active Directory →](02-active-directory-ldaps.md)

---

## Contents

1. [Create the container](#1-create-the-container)
2. [Check the network and update](#2-check-the-network-and-update)
3. [Install Apache, MariaDB and PHP](#3-install-apache-mariadb-and-php)
4. [Create the database](#4-create-the-database)
5. [Download GLPI](#5-download-glpi)
6. [Configure Apache and PHP](#6-configure-apache-and-php)
7. [Install GLPI](#7-install-glpi)
8. [Post-installation security](#8-post-installation-security)
9. [Create an admin user for SSH](#9-create-an-admin-user-for-ssh)

---

## 1. Create the container

**Why a container and not a VM?** GLPI is a small Linux web app. A container shares the Proxmox kernel, starts in seconds and uses only the RAM it needs.

### Download the template

**GUI:** Proxmox → **local** storage → **CT Templates → Templates** → search `ubuntu` → **Download**.

**Terminal (Proxmox host):**

```bash
pveam update
pveam available --section system | grep ubuntu
pveam download local <ubuntu-standard-template-name>
```

### Create the container (GUI)

Proxmox web UI → **Create CT** (top right):

| Tab | Setting |
|---|---|
| **General** | CT ID `230` · Hostname `ln-glpi01` · root password · ✅ Unprivileged container |
| **Template** | The Ubuntu standard template |
| **Disks** | Storage `zfs-nvmeT500-vm` · 20 GB |
| **CPU** | 2 cores |
| **Memory** | 2048 MB · swap 512 MB |
| **Network** | Bridge `vmbr0` · IPv4 **Static** `192.168.7.83/24` · Gateway `192.168.7.1` |
| **DNS** | Domain `corp.silverbank.internal` · Servers `192.168.7.80 192.168.7.82` |
| **Confirm** | ✅ Start after created → **Finish** |

Then: CT 230 → **Options → Start/Shutdown order** → `5` (after the Windows lab).

---

## 2. Check the network and update

Open a shell: **GUI** CT 230 → **Console**, or **terminal** on the Proxmox host: `pct enter 230`.

```bash
ip a                                  # 192.168.7.83 on eth0
ping -c 3 192.168.7.1                 # gateway
ping -c 3 google.com                  # internet + DNS forwarder on the DC
getent hosts corp.silverbank.internal # finds both domain controllers
timedatectl set-timezone Europe/London

apt update
apt full-upgrade -y
apt autoremove -y
```

> 💡 `/etc/resolv.conf` is managed by Proxmox (between `BEGIN PVE` and `END PVE`). Change DNS in the container's **Network/DNS** tab in Proxmox, not in the file.
> 💡 The container uses the host's kernel, so kernel updates happen on the **Proxmox host**.

📸 Snapshot: `clean-updated`.

---

## 3. Install Apache, MariaDB and PHP

```bash
apt install -y apache2 mariadb-server libapache2-mod-php php php-mysql php-curl php-gd \
  php-intl php-ldap php-xml php-mbstring php-zip php-bz2 php-bcmath wget tar
php -v            # check against GLPI's supported PHP versions
mariadb --version
php -m | grep -i ldap   # "ldap" — needed for the Active Directory connection
```

---

## 4. Create the database

```bash
mysql_secure_installation
mysql -u root
```

```sql
CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpi'@'localhost' IDENTIFIED BY '<db-password>';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpi'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Check — test the login before using it anywhere

```bash
mysql -u glpi -p'<db-password>' -e "SELECT 'login OK';" glpi
```

---

## 5. Download GLPI

Latest release from the GLPI GitHub releases page:

```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/12.0.1/glpi-12.0.1.tgz
tar -xzf glpi-12.0.1.tgz -C /var/www/
chown -R www-data:www-data /var/www/glpi
ls -ld /var/www/glpi       # owner: www-data
```

---

## 6. Configure Apache and PHP

Only GLPI's `public` folder is exposed to the web.

`/etc/apache2/sites-available/glpi.conf`:

```apache
<VirtualHost *:80>
    ServerName helpdesk.corp.silverbank.internal
    DocumentRoot /var/www/glpi/public

    <Directory /var/www/glpi/public>
        Require all granted
        RewriteEngine On
        RewriteCond %{HTTP:Authorization} ^(.+)$
        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>
</VirtualHost>
```

```bash
a2enmod rewrite
a2dissite 000-default
a2ensite glpi
apache2ctl configtest      # must say "Syntax OK"
```

PHP session security in `/etc/php/8.5/apache2/php.ini` (the **apache2** file, not the **cli** one):

```ini
session.cookie_httponly = On
session.cookie_samesite = Lax
```

`session.cookie_secure` stays **off** until GLPI runs on HTTPS (turning it on over HTTP breaks the login).

```bash
systemctl restart apache2
systemctl status apache2 --no-pager
```

### Check — what the running Apache PHP really sees

```bash
echo '<?php echo "httponly=" . ini_get("session.cookie_httponly") . " samesite=" . ini_get("session.cookie_samesite");' > /var/www/glpi/public/sesscheck.php
# open http://192.168.7.83/sesscheck.php  ->  httponly=1 samesite=Lax
rm /var/www/glpi/public/sesscheck.php
```

---

## 7. Install GLPI

### Option A — Web installer (GUI)

Open `http://192.168.7.83` in a browser:

1. Choose the language → accept the licence → **Install**.
2. **Requirements check:** every line must be green (orange = recommended, red = must fix).
3. **Database connection:** server `localhost` · user `glpi` · password `<db-password>`.
4. Select the database **glpi** → Continue.
5. Wait for the tables to be created → **Use GLPI**.

### Option B — Command line (what I used)

The command line gives **clear errors** and can be reused in scripts.

```bash
runuser -u www-data -- php /var/www/glpi/bin/console database:install --help

runuser -u www-data -- php /var/www/glpi/bin/console database:install \
  --db-host=localhost --db-name=glpi --db-user=glpi --db-password='<db-password>' \
  --default-language=en_GB --no-telemetry
```

Answers: port 3306 (default) · SSL **No** (the database is on the same machine) · continue **Yes**.

**Expected result:**

```
> Database structure created.
> Default data imported.
> Default forms created.
> Default rules initialized.
> Security keys generated.
> Configuration defaults defined.
> Installation done.
```

> 💡 Why `runuser -u www-data`? GLPI's files belong to the web server user. Running the console as `www-data` keeps file ownership correct.

---

## 8. Post-installation security

Log in at `http://192.168.7.83` with the default super-admin account, then secure it **immediately**. Every GLPI installation starts with the same default accounts, and attackers try them first.

### GUI (GLPI web interface)

| Task | Where |
|---|---|
| Change the `glpi` super-admin password | **Administration → Users → glpi** → Password |
| Change the passwords of `tech`, `normal`, `post-only` **and untick Active** | Each user → new password + untick **Active** → **Save** |
| Leave `glpi-system` | Internal account |
| Application URL | **Setup → General** → `http://helpdesk.corp.silverbank.internal` |

GLPI shows a **warning banner** until every default password is changed.

### Terminal

```bash
rm /var/www/glpi/install/install.php            # delete the web installer
chmod 640 /var/www/glpi/config/config_db.php    # protect the database password
```

### DNS (on LN-DC01)

Add the server and a friendly alias — see [DNS records](../02-dns/03-dns-records.md):
- A record `ln-glpi01` → 192.168.7.83
- CNAME `helpdesk` → `ln-glpi01.corp.silverbank.internal`

📸 Snapshot: `glpi-installed`.

---

## 9. Create an admin user for SSH

Ubuntu **blocks root login with a password over SSH** (`PermitRootLogin prohibit-password`). Use a named user with `sudo` instead.

```bash
adduser marius
usermod -aG sudo marius
```

From your own computer — log in with an **SSH key** instead of a password:

```bash
ssh-keygen -t ed25519                # only if you don't have a key yet
ssh-copy-id marius@192.168.7.83
ssh marius@192.168.7.83              # no password prompt
sudo -i                              # become root when needed
```

> 💡 Interview point: *"I disable root SSH login and use named accounts with sudo and SSH keys, so every action is logged under a real person."*

---

Next: [Connecting to Active Directory (LDAPS) →](02-active-directory-ldaps.md)