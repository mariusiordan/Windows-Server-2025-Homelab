# 01 — Active Directory

> Home lab project. **SilverBank** is a fictional company. This folder documents how I built and run a two-DC Active Directory domain on **Windows Server 2025** in my Proxmox homelab: domain controllers, a file server and a Windows 11 client.

Every guide shows the **GUI steps first** (the way most helpdesk and 1st/2nd line work is done), then the same task in **PowerShell (optional)**, then **how to check it worked**.

Last updated: 5 November 2026

---

## Guides in this folder

| # | Guide | What it covers |
|---|---|---|
| 1 | [Installation — first domain controller](01-installation.md) | Prepare the server, install AD DS, create the forest on LN-DC01 |
| 2 | [Adding a second domain controller](02-second-domain-controller.md) | Join and promote LN-DC02, DNS between DCs, replication check |
| 3 | [Adding a file server](03-file-server.md) | Join LN-FS01, File Server role, department shares, groups, share + NTFS permissions |
| 4 | [Adding clients](04-clients.md) | Join a Windows 11 PC, move it to the right OU, test a user login |
| 5 | [Troubleshooting](05-troubleshooting.md) | Real problems from this lab, AD error codes, and a command toolbox |

**Suggested order:** 1 → 2 → 3 → 4. Keep guide 5 open while you work.

---

## Lab at a glance

```
Internet
   │
Home router 192.168.7.1
   │
Cisco Catalyst 3560-CX (192.168.7.90)
   │
Proxmox host 192.168.7.12
   ├── LN-DC01   192.168.7.80   Domain controller (first DC, DNS, PDC Emulator)
   ├── LN-DC02   192.168.7.82   Domain controller (second DC, DNS, Global Catalog)
   ├── LN-FS01   192.168.7.81   File server + Certificate Authority (corp-LN-FS01-CA)
   └── LN-CL01   DHCP           Windows 11 Pro client
```

| Setting | Value |
|---|---|
| Domain (FQDN) | `corp.silverbank.internal` |
| NetBIOS name | `SILVERBANK` (logins look like `SILVERBANK\james.carter`) |
| Functional level | Windows Server 2025 |
| Time zone | GMT Standard Time (all servers) |

---

## OU structure

```
SilverBank
├── Users
│   ├── Management, IT, Finance, HR, Sales,
│   └── CustomerService, Compliance, Repair
├── Computers
│   ├── Workstations
│   └── Laptops
├── Servers
├── Groups
├── Service Accounts
└── Disabled
```

---

## Startup order (Proxmox)

| Order | VM | Wait after start |
|---|---|---|
| 1 | LN-DC01 | 180 s |
| 2 | LN-DC02 | 120 s |
| 3 | LN-FS01 | 30 s |
| 4 | LN-CL01 | — |

Shutdown happens in the reverse order. The DCs start first so DNS and logins work when the other machines boot.

---

## Golden rules learned in this lab

1. **It's always DNS.** Every domain machine must use the **DCs** as DNS servers, never the home router.
2. **Time matters.** Kerberos fails if clocks differ by more than 5 minutes. Set the right **time zone** on every server.
3. **Snapshot before every big change.** Promotions, role installs and permission changes are easy to undo with a snapshot.
4. **Verify every change.** "I did it" is not proof. A command or a test is proof.
5. **GUI and PowerShell do the same thing.** The wizard's **View script** button shows the PowerShell behind it.