# 1. DNS Manager and Zones

> A guided tour of the DNS server that was installed with Active Directory: the zones, the important records, forwarders and the DNS client settings.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

[← Back to DNS](README.md) · Next: [Reverse lookup zone →](02-reverse-lookup-zone.md)

---

## Contents

1. [Open DNS Manager](#step-1-open-dns-manager)
2. [Explore the forward zone](#step-2-explore-the-forward-zone)
3. [Explore the _msdcs zone](#step-3-explore-the-_msdcs-zone)
4. [Check the zone settings](#step-4-check-the-zone-settings)
5. [Forwarders](#step-5-forwarders)
6. [DNS client settings on every machine](#step-6-dns-client-settings-on-every-machine)

---

## Step 1: Open DNS Manager

### GUI

**Server Manager → Tools → DNS** (or **Win + R** → `dnsmgmt.msc`).

You see the server `LN-DC01` with these folders:

| Folder | What's inside |
|---|---|
| **Forward Lookup Zones** | Name → IP zones (`corp.silverbank.internal`, `_msdcs...`) |
| **Reverse Lookup Zones** | IP → name zones (empty until [guide 2](02-reverse-lookup-zone.md)) |
| **Conditional Forwarders** | Rules like "send questions about *partner.com* to this server" |
| **Global Logs → DNS Events** | The DNS Server event log |

> 💡 To manage LN-DC02 from the same window: right-click **DNS → Connect to DNS Server…** → `LN-DC02`.
> 💡 **View → Advanced** shows extra things, like the **Cached Lookups** folder.

### PowerShell (optional)

```powershell
Get-DnsServerZone
```

---

## Step 2: Explore the forward zone

### GUI

Click **Forward Lookup Zones → corp.silverbank.internal**.

| You'll see | Meaning |
|---|---|
| **(same as parent folder) – Start of Authority (SOA)** | Which server is "in charge" of the zone |
| **(same as parent folder) – Name Server (NS)** | The DNS servers for this zone — both DCs should be listed |
| **(same as parent folder) – Host (A)** | The domain name itself → the DCs' IPs |
| **ln-dc01, ln-dc02, ln-fs01, ln-cl01 – Host (A)** | Each machine's name → IP |
| Folders **_msdcs, _sites, _tcp, _udp** | **SRV records** — how computers find DCs, Kerberos and LDAP |
| **DomainDnsZones, ForestDnsZones** | Records for the AD partitions that store DNS |

**Exercise:** open **_tcp** — find `_ldap` and `_kerberos`. Double-click one: it points to a DC on port **389** (LDAP) or **88** (Kerberos).

### PowerShell (optional)

```powershell
Get-DnsServerResourceRecord -ZoneName "corp.silverbank.internal" -RRType A
Get-DnsServerResourceRecord -ZoneName "corp.silverbank.internal" -Name "_ldap._tcp" -RRType SRV
```

---

## Step 3: Explore the _msdcs zone

`_msdcs.corp.silverbank.internal` is the **DC locator** zone. When a PC starts, it asks: *"Where is a domain controller?"* — the answer comes from here.

### GUI

**Forward Lookup Zones → _msdcs.corp.silverbank.internal**:
- **dc → _tcp → _ldap** → one SRV record **per DC**
- A list of **long GUID names** (CNAME) → one per DC, used for **replication**

If a DC is missing here, other machines can't find it. ⚠️

### Check (from any domain machine)

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal
```

**Expect:** both `ln-dc01` and `ln-dc02` on port 389.

---

## Step 4: Check the zone settings

### GUI

Right-click **corp.silverbank.internal → Properties**:

| Tab / setting | Expected value | Why |
|---|---|---|
| General → **Type** | Active Directory-Integrated Primary | Stored in AD, replicates with AD |
| General → **Replication** | All DNS servers running on domain controllers in this domain | Both DCs get the zone |
| General → **Dynamic updates** | **Secure only** | Only domain members can register records — protects against fake records |
| **Name Servers** tab | `ln-dc01` and `ln-dc02` | Both DCs answer for the zone |

### PowerShell (optional)

```powershell
Get-DnsServerZone -Name "corp.silverbank.internal" |
  Select-Object ZoneName, ZoneType, IsDsIntegrated, ReplicationScope, DynamicUpdate
```

---

## Step 5: Forwarders

The DCs only know the **SilverBank** names. For everything else (`bbc.co.uk`, Windows Update…), they **forward** the question to the router.

### GUI

1. Right-click the server **LN-DC01 → Properties → Forwarders** tab.
2. You should see `192.168.7.1`. If not: **Edit…** → type `192.168.7.1` → **OK**.
3. Leave ✅ **Use root hints if no forwarders are available**.
4. **Repeat on LN-DC02** — forwarders are a **per-server** setting, they don't replicate.

### PowerShell (optional)

```powershell
Get-DnsServerForwarder
Add-DnsServerForwarder -IPAddress 192.168.7.1      # only if it's missing
```

### Check

```powershell
nslookup bbc.co.uk 192.168.7.80     # internet name through LN-DC01
nslookup bbc.co.uk 192.168.7.82     # and through LN-DC02
```

**Expect:** both return IP addresses. If one fails → check that DC's forwarder.

---

## Step 6: DNS client settings on every machine

The **DNS server** can be perfect, but each machine also has **DNS client** settings: *"which server do I ask?"*

| Machine | Preferred DNS | Alternate DNS |
|---|---|---|
| LN-DC01 | 192.168.7.82 | 127.0.0.1 |
| LN-DC02 | 192.168.7.80 | 127.0.0.1 |
| LN-FS01 | 192.168.7.80 | 192.168.7.82 |
| LN-CL01 | 192.168.7.80 | 192.168.7.82 |
| ln-glpi01 (Linux) | 192.168.7.80 | — (set in Proxmox) |

### GUI (Windows)

Adapter → **Properties → IPv4 → Properties** → Preferred / Alternate DNS.

> ⚠️ **Real problem in this lab:** LN-DC01's DNS client pointed to the **router** because the settings were on a **Hyper-V vEthernet** adapter. Certificate requests failed with "RPC server unavailable". **Always check `ipconfig /all` on the server itself.**

### Check

```powershell
ipconfig /all              # look at "DNS Servers" on the adapter with the real IP
Get-DnsClientServerAddress -AddressFamily IPv4
```

On Linux (`ln-glpi01`):

```bash
resolvectl status          # or: cat /etc/resolv.conf
getent hosts ln-dc01.corp.silverbank.internal
```

---

## What I learned

- DNS was installed **automatically** with AD, and the zone lives **inside AD**.
- The **_msdcs** zone and **SRV records** are how computers find DCs.
- **Forwarders** are per server — set them on **both** DCs.
- The **DNS client** settings on each machine matter as much as the server.

Next: [Reverse lookup zone →](02-reverse-lookup-zone.md)