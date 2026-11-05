# 4. Maintenance

> Keep DHCP healthy: **back up and restore**, set up **failover** between two servers, watch **scope usage**, read the **logs**, and understand **DHCP relay** for other networks.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

> ⚠️ Reference guide — example values. Not applied in my lab (see the [README](README.md)).

[← Reservations and leases](03-reservations-and-leases.md) · [Back to DHCP](README.md) · Next: [Troubleshooting →](05-troubleshooting.md)

---

## Contents

1. [Backup and restore](#1-backup-and-restore)
2. [Failover (two DHCP servers)](#2-failover-two-dhcp-servers)
3. [Watching scope usage](#3-watching-scope-usage)
4. [Logs](#4-logs)
5. [DHCP relay (IP helper)](#5-dhcp-relay-ip-helper)
6. [Monthly checklist](#6-monthly-checklist)

---

## 1. Backup and restore

Windows backs up the DHCP database **automatically every 60 minutes** to:

```
C:\Windows\System32\dhcp\backup
```

> ⚠️ That folder is **on the same server**. If the server dies, the backup dies too. Copy it elsewhere (or include it in your server backups).

### GUI

| Task | How |
|---|---|
| Manual backup | DHCP console → right-click the server → **Backup…** → choose a folder |
| Restore | Right-click the server → **Restore…** → choose the backup folder → the service restarts |

### PowerShell (optional)

```powershell
# Backup (same format as the GUI)
Backup-DhcpServer -Path "D:\Backup\DHCP"

# Export everything to one file — great for moving DHCP to a NEW server
Export-DhcpServer -File "D:\Backup\dhcp-export.xml" -Leases

# Import on the new server
Import-DhcpServer -File "D:\Backup\dhcp-export.xml" -BackupPath "D:\Backup\DHCP-before-import" -Leases
```

---

## 2. Failover (two DHCP servers)

With only one DHCP server, if it fails, new devices get **no IP** (`169.254.x.x`). **Failover** lets two servers share a scope.

| Mode | How it works | When to use |
|---|---|---|
| **Load balance** | Both servers answer, e.g. 50% / 50% | Two servers in the same site |
| **Hot standby** | One server answers; the other only takes over if it fails | A main site + a backup site |

### GUI

1. Install and authorize DHCP on the **second** server (e.g. LN-DC02) — see [Installation](01-installation.md). Don't create scopes on it.
2. On the first server: right-click the scope (or **IPv4**) → **Configure Failover…**
3. Select the scopes → Next.
4. **Partner Server:** `ln-dc02.corp.silverbank.internal` → Next.
5. Settings:
   - Relationship name: `DC01-DC02-Failover`
   - **Mode:** Load balance · 50% / 50%
   - **Maximum Client Lead Time:** 1 hour (default)
   - ✅ **Enable Message Authentication** → type a **Shared Secret** (save it in your password manager)
6. **Finish.** The scope and its settings are **copied** to the partner automatically.

### PowerShell (optional)

```powershell
Add-DhcpServerv4Failover -Name "DC01-DC02-Failover" -PartnerServer "ln-dc02.corp.silverbank.internal" `
  -ScopeId 10.20.0.0 -LoadBalancePercent 50 -SharedSecret "YourSecretHere"
```

### Check

| GUI | PowerShell |
|---|---|
| Scope → Properties → **Failover** tab shows the partner and state **Normal** | `Get-DhcpServerv4Failover` |

> 💡 **After changing a scope** (new option, new reservation), replicate the change to the partner: right-click the scope → **Replicate Scope** · PowerShell: `Invoke-DhcpServerv4FailoverReplication`

---

## 3. Watching scope usage

A **full scope** = new devices get no IP. Check it regularly.

### GUI

Right-click **IPv4** (or a scope) → **Display Statistics** → look at **In Use** (percentage).

### PowerShell (optional)

```powershell
Get-DhcpServerv4ScopeStatistics |
  Select-Object ScopeId, Free, InUse, PercentageInUse
```

| In use | Action |
|---|---|
| Below 80% | OK ✅ |
| 80–95% | Plan: shorter lease time, delete dead leases, or a bigger scope |
| Above 95% | ⚠️ Act now — new devices will soon fail |

Windows also writes an **event** when a scope is nearly full (see [Troubleshooting](05-troubleshooting.md#event-viewer)).

---

## 4. Logs

| Log | Where | Good for |
|---|---|---|
| **Audit log** (text file, one per weekday) | `C:\Windows\System32\dhcp\DhcpSrvLog-Mon.log` … `-Sun.log` | Every lease given, renewed or released — "who had this IP on Tuesday?" |
| **Event log** | Event Viewer → Applications and Services Logs → **Microsoft → Windows → DHCP-Server** | Errors: not authorized, scope full, failover problems |

> 💡 The audit log is turned on by default: **IPv4 → Properties → General → Enable DHCP audit logging**. Each line has an **ID code** (e.g. `10` = new lease, `11` = renewed, `12` = released) — the meaning of each code is listed at the top of the file.

---

## 5. DHCP relay (IP helper)

DHCP **DISCOVER** messages are **broadcasts**, and broadcasts **don't cross routers**. So how does a PC in VLAN 20 reach a DHCP server in another network?

```
PC in VLAN 20 ──broadcast──► Switch/router (VLAN 20 interface 10.20.0.1)
                                    │  "ip helper-address 192.168.7.80"
                                    ▼  forwards it as a normal (unicast) packet
                              DHCP server LN-DC01 (192.168.7.80)
                                    │  picks the scope that matches 10.20.0.x
                                    ▼
                              Answer goes back to the PC
```

On a **Cisco** switch/router (the VLAN 20 interface):

```
interface Vlan20
 ip address 10.20.0.1 255.255.255.0
 ip helper-address 192.168.7.80
 ip helper-address 192.168.7.82
```

> 💡 The relay tells the DHCP server **which network** the request came from (the `10.20.0.1` address), so the server knows to use the **10.20.0.0** scope.
> **Typical ticket:** "New VLAN, nobody gets an IP" → the `ip helper-address` is missing.

---

## 6. Monthly checklist

- [ ] Scope usage below 80% (`Get-DhcpServerv4ScopeStatistics`)
- [ ] Failover state **Normal** on both servers
- [ ] Backup folder copied off the server
- [ ] DHCP-Server event log — no repeating errors
- [ ] Reservations still match real devices (old printers removed)

---

## What I learned

- DHCP backs itself up every hour, but **on the same server** — copy it elsewhere.
- **Failover** = two servers share a scope, so DHCP survives one server failing.
- Watch **scope usage** before it gets full.
- **DHCP relay** (`ip helper-address`) carries DHCP across VLANs.

Next: [Troubleshooting →](05-troubleshooting.md)