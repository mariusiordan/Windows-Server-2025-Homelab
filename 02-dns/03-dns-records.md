# 3. DNS Records

> Add and manage the records helpdesk staff use most: **A** (host), **CNAME** (alias) and **PTR** (reverse).
> Practise on real machines from this lab that **don't** register themselves: the GLPI server, the Proxmox host and the Cisco switch.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← Reverse lookup zone](02-reverse-lookup-zone.md) · [Back to DNS](README.md) · Next: [Maintenance →](04-maintenance.md)

---

## Records to create in this lab

Windows machines register their own records. **Linux servers and network devices don't**, so we add them by hand.

| Name | Type | Points to | Why |
|---|---|---|---|
| `ln-glpi01` | A (+ PTR) | 192.168.7.83 | GLPI helpdesk server (Linux) |
| `helpdesk` | CNAME | `ln-glpi01.corp.silverbank.internal` | Easy name for staff: `http://helpdesk` |
| `pve` | A (+ PTR) | 192.168.7.12 | Proxmox host |
| `sw-core01` | A (+ PTR) | 192.168.7.90 | Cisco Catalyst 3560-CX management IP |

### First, check what already exists

```powershell
nslookup ln-glpi01.corp.silverbank.internal
```

If you get **"Non-existent domain"**, the record is missing → create it below.

---

## Step 1: Create an A record (host)

### GUI

1. **DNS Manager → Forward Lookup Zones → corp.silverbank.internal**.
2. Right-click → **New Host (A or AAAA)…**
3. Name: `ln-glpi01` → the FQDN fills in automatically.
4. IP address: `192.168.7.83`
5. ✅ **Create associated pointer (PTR) record** (needs the [reverse zone](02-reverse-lookup-zone.md)).
6. **Add Host** → "The host record was successfully created" → **OK** → **Done**.

Repeat for `pve` (192.168.7.12) and `sw-core01` (192.168.7.90).

### PowerShell (optional)

```powershell
$zone = "corp.silverbank.internal"
Add-DnsServerResourceRecordA -ZoneName $zone -Name "ln-glpi01" -IPv4Address 192.168.7.83 -CreatePtr
Add-DnsServerResourceRecordA -ZoneName $zone -Name "pve"       -IPv4Address 192.168.7.12 -CreatePtr
Add-DnsServerResourceRecordA -ZoneName $zone -Name "sw-core01" -IPv4Address 192.168.7.90 -CreatePtr
```

### Check

```powershell
nslookup ln-glpi01.corp.silverbank.internal     # → 192.168.7.83
nslookup 192.168.7.83                           # → ln-glpi01 (reverse)
ping sw-core01                                  # short name works from domain PCs
```

---

## Step 2: Create a CNAME (alias)

A **CNAME** is a nickname. Staff type `helpdesk`; DNS sends them to `ln-glpi01`.

**Why it's useful:** if GLPI moves to a new server one day, you change **one** record — staff keep using `http://helpdesk`.

### GUI

1. Right-click **corp.silverbank.internal → New Alias (CNAME)…**
2. Alias name: `helpdesk`
3. Fully qualified domain name for target host: **Browse…** → `ln-glpi01` (or type `ln-glpi01.corp.silverbank.internal`) → **OK**.

### PowerShell (optional)

```powershell
Add-DnsServerResourceRecordCName -ZoneName "corp.silverbank.internal" -Name "helpdesk" -HostNameAlias "ln-glpi01.corp.silverbank.internal"
```

### Check

```powershell
nslookup helpdesk.corp.silverbank.internal
```

**Expect:** `Aliases: helpdesk.corp.silverbank.internal` and the address `192.168.7.83`.

Then on **LN-CL01**, open a browser → `http://helpdesk` → the GLPI login page. 🎉

---

## Step 3: Change a record (a server got a new IP)

### GUI

Double-click the record → change the **IP address** → ✅ **Update associated pointer (PTR) record** → **OK**.

Then, on the clients, **clear the cache** (old answers are remembered for the TTL):

```powershell
ipconfig /flushdns
```

### PowerShell (optional)

```powershell
$old = Get-DnsServerResourceRecord -ZoneName "corp.silverbank.internal" -Name "pve" -RRType A
$new = $old.Clone()
$new.RecordData.IPv4Address = [ipaddress]"192.168.7.13"
Set-DnsServerResourceRecord -ZoneName "corp.silverbank.internal" -OldInputObject $old -NewInputObject $new
```

> This one is long in PowerShell — the **GUI is faster** for single changes.

---

## Step 4: Delete a record

### GUI

Right-click the record → **Delete** → **Yes**. If asked about the matching PTR record, choose **Yes** too.

### PowerShell (optional)

```powershell
Remove-DnsServerResourceRecord -ZoneName "corp.silverbank.internal" -Name "old-server" -RRType A -Force
```

> ⚠️ **Never delete** records in **_msdcs, _tcp, _udp, _sites**, or the DCs' own records. AD needs them.

---

## Static vs dynamic records

| | Dynamic | Static |
|---|---|---|
| **Created by** | The machine itself (Windows) | You, by hand |
| **Timestamp** in record properties* | A date | `static` |
| **Cleaned by scavenging** | Yes, if old | **Never** |
| **Examples** | ln-fs01, ln-cl01 | ln-glpi01, pve, sw-core01, helpdesk |

\* Turn on **View → Advanced** in DNS Manager to see the timestamp.

---

## What I learned

- **A** = name → IP · **CNAME** = alias → name · **PTR** = IP → name.
- Linux servers and network devices **don't register themselves** — add them manually.
- A **CNAME** like `helpdesk` hides the real server name from users.
- After changing a record, clients may remember the **old** answer — `ipconfig /flushdns`.

Next: [Maintenance: aging and scavenging →](04-maintenance.md)