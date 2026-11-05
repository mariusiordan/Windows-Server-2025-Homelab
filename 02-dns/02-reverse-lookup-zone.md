# 2. Reverse Lookup Zone

> Create the **IP → name** zone, so tools can show names instead of only IPs, and `nslookup` stops saying **"Server: UnKnown"**.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← DNS Manager and zones](01-dns-manager-and-zones.md) · [Back to DNS](README.md) · Next: [DNS records →](03-dns-records.md)

---

## Why do I need it?

| Without a reverse zone | With a reverse zone |
|---|---|
| `nslookup` shows `Server: UnKnown` | Shows `Server: ln-dc01.corp.silverbank.internal` |
| Logs and tools show only `192.168.7.81` | They can show `ln-fs01` |
| Some applications complain or run slowly | They work normally |

The reverse zone name is the network **backwards**: `192.168.7.x` → **`7.168.192.in-addr.arpa`**.

📸 Optional snapshot of LN-DC01 first. (Low risk — you can delete a zone easily.)

---

## Step 1: Create the reverse zone

### GUI

On **LN-DC01**: **DNS Manager** → right-click **Reverse Lookup Zones → New Zone…**

| Wizard page | What to choose | Why |
|---|---|---|
| **Zone Type** | **Primary zone** · ✅ **Store the zone in Active Directory** | Replicates to LN-DC02 automatically |
| **AD Zone Replication Scope** | **To all DNS servers running on domain controllers in this domain** | Same as the forward zone |
| **Reverse Lookup Zone Name** | **IPv4 Reverse Lookup Zone** | Our network is IPv4 |
| **Network ID** | `192.168.7` | The wizard builds `7.168.192.in-addr.arpa` for you |
| **Dynamic Update** | **Allow only secure dynamic updates** | Same protection as the forward zone |
| **Completing** | **Finish** | |

### PowerShell (optional)

```powershell
Add-DnsServerPrimaryZone -NetworkId "192.168.7.0/24" -ReplicationScope Domain -DynamicUpdate Secure
```

### Check

| GUI | PowerShell |
|---|---|
| **Reverse Lookup Zones** shows `7.168.192.in-addr.arpa` — on **LN-DC02** too after a minute | `Get-DnsServerZone -Name "7.168.192.in-addr.arpa"` |

---

## Step 2: Fill the zone with PTR records

The new zone is **empty**. There are three ways to fill it.

### Way 1 — Windows machines register themselves (easiest)

On each **Windows** server/client, open **Command Prompt as admin**:

```powershell
ipconfig /registerdns
```

Wait a minute, then refresh the reverse zone (**F5**). You should see PTR records like `192.168.7.81 → ln-fs01.corp.silverbank.internal`.

### Way 2 — From an existing A record (GUI)

1. **Forward Lookup Zones → corp.silverbank.internal** → double-click a host, e.g. `ln-fs01`.
2. Tick ✅ **Update associated pointer (PTR) record** → **OK**.

### Way 3 — Add a PTR manually (Linux servers, network devices)

**GUI:** right-click **7.168.192.in-addr.arpa → New Pointer (PTR)…**
- Host IP address: `192.168.7.83`
- Host name: **Browse…** or type `ln-glpi01.corp.silverbank.internal` → **OK**

**PowerShell (optional):**

```powershell
Add-DnsServerResourceRecordPtr -ZoneName "7.168.192.in-addr.arpa" -Name "83" -PtrDomainName "ln-glpi01.corp.silverbank.internal"
```

> 💡 In the reverse zone, the record name is just the **last number** of the IP (`83` for `.83`).

---

## Step 3: Check it works

```powershell
nslookup 192.168.7.81
```

**Expect:**

```
Server:  ln-dc01.corp.silverbank.internal      ← no more "UnKnown"!
Address:  192.168.7.80

Name:    ln-fs01.corp.silverbank.internal
Address:  192.168.7.81
```

Also try:

```powershell
Resolve-DnsName 192.168.7.80
ping -a 192.168.7.82        # -a shows the name for an IP
```

---

## What I learned

- Reverse zones turn **IP → name** using **PTR** records.
- The zone name is the network **backwards** + `in-addr.arpa`.
- "Server: UnKnown" in nslookup meant **no PTR record for the DNS server itself** — harmless, but now fixed.
- Windows machines can register their own PTR records; Linux and network devices need them added by hand.

Next: [DNS records →](03-dns-records.md)