# 04 — GLPI Helpdesk

> Home lab project. **SilverBank** is a fictional company. **GLPI** is the ticketing system for the SilverBank lab: employees log in with their **Active Directory** accounts and create tickets; I work them as the helpdesk technician.

GLPI runs on **Linux**, so its setup uses the **terminal (bash)**, not PowerShell. Daily work — tickets, users, settings — is done in the **web interface (GUI)**.

Passwords are never stored in this repository.

Last updated: 5 November 2026

---

## Guides in this folder

| # | Guide | What it covers |
|---|---|---|
| 1 | [Installation](01-installation.md) | Proxmox container, Apache, MariaDB, PHP, GLPI install, first security steps |
| 2 | [Connecting to Active Directory (LDAPS)](02-active-directory-ldaps.md) | Service account, certificates, LDAPS tests, LDAP directory in GLPI, importing users |
| 3 | [Using GLPI (technician cheat sheet)](03-using-glpi.md) | Ticket life cycle, fields, Follow-up / Task / Solution, the 6-step method |
| 4 | [Maintenance](04-maintenance.md) | Snapshots and backups, updates, user sync, admin password recovery, security checks |
| 5 | [Troubleshooting](05-troubleshooting.md) | Real problems from this lab, login problems, command toolbox, logs |

**Practice tickets** solved in GLPI are in [`../tickets/`](../tickets/README.md).

---

## Overview

| Item | Value |
|---|---|
| Container | Proxmox LXC, CT ID **230**, unprivileged |
| Hostname | `ln-glpi01` |
| OS | Ubuntu 26.04 LTS |
| IP | 192.168.7.83 / 24, gateway 192.168.7.1 |
| DNS | 192.168.7.80, 192.168.7.82 (the DCs), search domain `corp.silverbank.internal` |
| URL | `http://helpdesk.corp.silverbank.internal` (or `http://helpdesk` from domain PCs) |
| Resources | 2 vCPU, 2 GB RAM, 20 GB disk |
| Stack | Apache 2.4 · PHP 8.5 · MariaDB 11.8 · **GLPI 12.0.1** |
| Authentication | Active Directory over **LDAPS (port 636)**, service account `svc_glpi` |

---

## How it fits together

```
 Employee (LN-CL01)                         Technician
   │ http://helpdesk                           │
   ▼                                           ▼
 ┌──────────────────── ln-glpi01 (CT 230) ────────────────────┐
 │ Apache → GLPI (PHP) → MariaDB (tickets, users, settings)   │
 └───────────────────────────┬────────────────────────────────┘
                             │ LDAPS 636 (encrypted)
                             │ "Is james.carter's password correct?"
                             ▼
              LN-DC01 / LN-DC02 (Active Directory)
              certificate issued by corp-LN-FS01-CA
```

GLPI **never stores AD passwords**. It asks the domain controller every time someone logs in.

---

## Accounts

| Account | Where it lives | Login source | Used for |
|---|---|---|---|
| `marius.iordan` | Active Directory | **SilverBank AD** | Daily technician / admin work (Super-Admin profile) |
| `glpi` | GLPI database | **GLPI internal database** | **Break-glass** admin — only if AD is down |
| Staff (`james.carter`, …) | Active Directory | **SilverBank AD** | Self-Service: create and follow tickets |
| `svc_glpi` | Active Directory | — (never logs in) | Service account GLPI uses to read AD |
| `tech`, `normal`, `post-only` | GLPI database | — | Default demo accounts — **disabled** |
| `glpi-system` | GLPI database | — | Internal account — leave it alone |
| `marius` (Linux, sudo) | CT 230 | SSH | Server administration (root SSH login is blocked) |

---

## Golden rules

1. **Snapshot CT 230 before every change** (`pct snapshot 230 <name>`).
2. **Test outside the application first** — `mysql`, `openssl`, `ldapsearch` — to find which layer is broken.
3. **Check the Login source** when a login fails: AD accounts and local accounts use different sources.
4. **Use SSH, not the Proxmox web console** — the keyboard layout always matches and copy/paste works.
5. **Keep every password in a password manager**, never in GLPI tickets or on GitHub.