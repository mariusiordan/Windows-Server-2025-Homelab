# 4. Maintenance: Aging, Scavenging and Replication

> Keep DNS clean: remove **old records** automatically, and make sure both DCs have the same data.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← DNS records](03-dns-records.md) · [Back to DNS](README.md) · Next: [Troubleshooting →](05-troubleshooting.md)

---

## The problem: stale records

A laptop gets IP `.150` today and registers `laptop01 → .150`. Next week it gets `.162`. Sometimes the **old record stays**. Months later, another PC gets `.150`, and `laptop01` now points to the **wrong machine**. 😬

**Aging and scavenging** = DNS remembers when each dynamic record was last refreshed, and **deletes records that nobody has refreshed** for a long time.

> ✅ **Static records** (the ones you add by hand) are **never** scavenged.

---

## How the timers work

| Setting | Default | Meaning |
|---|---|---|
| **No-refresh interval** | 7 days | After a refresh, the record ignores further refreshes for 7 days (less replication traffic) |
| **Refresh interval** | 7 days | Then the machine has 7 more days to refresh it |
| **Scavenging period** | 7 days | How often the server runs the cleanup |

So a record that hasn't been refreshed for **more than 14 days** can be deleted on the next cleanup. Windows machines refresh their records every 24 hours while they're on, so **active machines are safe**.

---

## Step 1: Turn on aging for the zone

### GUI

1. **DNS Manager** → right-click **corp.silverbank.internal → Properties**.
2. **General** tab → **Aging…**
3. ✅ **Scavenge stale resource records**.
4. Leave **No-refresh** = 7 days and **Refresh** = 7 days → **OK** → **OK**.
5. Do the same for the **reverse zone** `7.168.192.in-addr.arpa`.

### PowerShell (optional)

```powershell
Set-DnsServerZoneAging -Name "corp.silverbank.internal" -Aging $true -NoRefreshInterval 7.00:00:00 -RefreshInterval 7.00:00:00
Set-DnsServerZoneAging -Name "7.168.192.in-addr.arpa" -Aging $true
```

---

## Step 2: Turn on scavenging on ONE server

Aging only **marks** old records. **Scavenging** actually **deletes** them, and it runs on a server.

> 💡 Enable it on **one DC only** (LN-DC01). It's enough, and it's easier to find in the logs if something gets deleted by mistake.

### GUI

1. **DNS Manager** → right-click the server **LN-DC01 → Properties → Advanced** tab.
2. ✅ **Enable automatic scavenging of stale records** → Period: **7 days** → **OK**.

### PowerShell (optional)

```powershell
Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval 7.00:00:00
```

### Check

```powershell
Get-DnsServerScavenging            # ScavengingState: True
Get-DnsServerZoneAging -Name "corp.silverbank.internal"   # AgingEnabled: True
```

**GUI:** **View → Advanced**, then open any dynamic record → you now see a **Record time stamp**.

> ⚠️ In a **real company**, wait at least **14 days** after enabling aging before turning on scavenging, and check which records would be removed. In the lab it's fine to enable both now.

---

## Step 3: Check replication between the DCs

Our zones are **AD-integrated**, so DNS changes travel with **AD replication**.

### GUI

1. Create a test record on **LN-DC01**, e.g. `test-repl → 192.168.7.250`.
2. In DNS Manager, connect to **LN-DC02** (right-click **DNS → Connect to DNS Server…**).
3. After about a minute, refresh (**F5**) → `test-repl` should appear on LN-DC02 too.
4. Delete the test record.

> If it doesn't appear: it's an **AD replication** problem, not a DNS problem → see [AD troubleshooting](../01-active-directory/05-troubleshooting.md).

### PowerShell (optional)

```powershell
repadmin /replsummary                     # fails = 0
repadmin /syncall /AdeP                   # force replication now (all partitions)
Get-DnsServerResourceRecord -ZoneName "corp.silverbank.internal" -Name "test-repl" -ComputerName LN-DC02
```

---

## Step 4: Clear the DNS server cache

The DNS **server** also caches answers (mostly internet names). Clearing it helps when an outside website changed IP.

### GUI

**View → Advanced** → right-click **Cached Lookups → Clear Cache**
(or right-click the server → **Clear Cache**).

### PowerShell (optional)

```powershell
Clear-DnsServerCache -Force
```

---

## Maintenance checklist (monthly)

- [ ] `dcdiag /test:dns` on each DC — no errors
- [ ] Both DCs listed under **Name Servers** for each zone
- [ ] Forwarders set on **both** DCs
- [ ] No obviously old records (old server names, wrong IPs)
- [ ] **DNS Server** event log — no repeating errors

---

## What I learned

- **Aging** = mark records with a timestamp · **Scavenging** = delete old ones.
- Static records are never scavenged.
- Scavenging on **one** server is enough.
- AD-integrated zones replicate with AD — a replication problem shows up as "missing records" on the other DC.

Next: [Troubleshooting →](05-troubleshooting.md)