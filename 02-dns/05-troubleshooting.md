# 5. Troubleshooting DNS

> Common DNS helpdesk tickets, how to read `nslookup`, the cache, and a command toolbox.
> **GUI first** where there is one, then the command.

[← Maintenance](04-maintenance.md) · [Back to DNS](README.md)

---

## Contents

1. [The 4 questions](#the-4-questions)
2. [Common DNS tickets](#common-dns-tickets)
3. [Reading nslookup](#reading-nslookup)
4. [The cache](#the-cache)
5. [Real problems from this lab](#real-problems-from-this-lab)
6. [Command toolbox](#command-toolbox)
7. [Event Viewer](#event-viewer)

---

## The 4 questions

Ask these **in order**. Most DNS tickets are solved by question 2.

| # | Question | How to check |
|---|---|---|
| 1 | Does it work **by IP**? | `ping 192.168.7.81` — if no → not DNS (network, firewall, server off) |
| 2 | Which **DNS server** is this machine asking? | `ipconfig /all` — must be the DCs, not the router |
| 3 | Does the **record exist** and is it **correct**? | DNS Manager, or `nslookup name dc-ip` |
| 4 | Is an **old answer cached**? | `ipconfig /displaydns`, then `ipconfig /flushdns` |

---

## Common DNS tickets

| User says | Likely cause | What to check / fix |
|---|---|---|
| "I can open `\\192.168.7.81` but not `\\ln-fs01`" | DNS: missing record or wrong DNS server | `ipconfig /all` → DNS = DCs? `nslookup ln-fs01` → record exists? |
| "The internet stopped working, but the shared drives work" | **Forwarder** missing or the router is down | DNS Manager → server → Properties → **Forwarders**; `nslookup bbc.co.uk 192.168.7.80` |
| "The shared drives stopped working, but the internet works" | The PC uses the **router** or **8.8.8.8** as DNS | Set DNS to the DCs (or fix DHCP options) |
| "The server was moved/rebuilt and now I can't reach it" | **Old IP** in the record or in a cache | Update the A record → `ipconfig /flushdns` on the client |
| "`http://helpdesk` doesn't open" | Missing **CNAME**, or the target A record is wrong | `nslookup helpdesk` → follows the alias to an IP? |
| "Computer can't join the domain" / "domain controller could not be contacted" | Client DNS isn't the DCs, or SRV records missing | `nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal` |
| "A new Linux server isn't found by name" | Linux **doesn't register** in Windows DNS | Add an **A record** by hand ([guide 3](03-dns-records.md)) |
| "One PC reaches the wrong machine by name" | **Stale record**: two names on one IP | DNS Manager → find duplicates → delete the old one; enable [scavenging](04-maintenance.md) |
| "Works on one DC but not the other" | Replication problem | `repadmin /replsummary` |

---

## Reading nslookup

```
C:\> nslookup ln-fs01
Server:  ln-dc01.corp.silverbank.internal      ← WHICH DNS server answered
Address:  192.168.7.80

Name:    ln-fs01.corp.silverbank.internal      ← the full name it found
Address:  192.168.7.81                         ← the answer
```

| What you see | Meaning |
|---|---|
| `Server: UnKnown` | The DNS server has no **PTR** record → harmless; create the [reverse zone](02-reverse-lookup-zone.md) |
| `Server: 192.168.7.1` (router) | ⚠️ This machine asks the **router** → fix the DNS client settings |
| `*** can't find ln-xyz: Non-existent domain` | The record **doesn't exist** (or a typo) |
| `Non-authoritative answer:` | The answer came from a **cache/forwarder** (normal for internet names) |
| `DNS request timed out` | The DNS server isn't answering (off, firewall port 53, wrong IP) |
| `Aliases:` line | The name is a **CNAME** — it shows the alias and the real name |

### Ask a specific server

```powershell
nslookup ln-fs01 192.168.7.82        # ask LN-DC02 directly
```

Very useful: if LN-DC01 answers but LN-DC02 doesn't → replication or zone problem on LN-DC02.

### Interactive mode (more advanced)

```
nslookup
> server 192.168.7.82
> set type=SRV
> _ldap._tcp.dc._msdcs.corp.silverbank.internal
> exit
```

---

## The cache

Answers are **remembered** for a while (the TTL). After you fix a record, the client may still use the **old** answer.

| Where | See it | Clear it |
|---|---|---|
| **Client** (any Windows PC) | `ipconfig /displaydns` · `Get-DnsClientCache` | `ipconfig /flushdns` · `Clear-DnsClientCache` |
| **DNS server** | DNS Manager → **View → Advanced** → **Cached Lookups** | Right-click **Cached Lookups → Clear Cache** · `Clear-DnsServerCache` |
| **Browser** | — | Close and reopen the browser |
| **Linux** (systemd) | `resolvectl statistics` | `sudo resolvectl flush-caches` |

---

## Real problems from this lab

| Symptom | Real cause | Fix |
|---|---|---|
| Certificate request: "RPC server is unavailable" — works **by IP**, fails **by name** | **LN-FS01 had no A record**, and **LN-DC01 asked the router** for DNS (settings on a Hyper-V **vEthernet** adapter) | Created the A record; fixed DC01's DNS client to use the DCs |
| GLPI container couldn't find the DCs by name | The container used the **router** as DNS | Proxmox: `pct set 230 --nameserver 192.168.7.80 --searchdomain corp.silverbank.internal` |
| `nslookup` always says `Server: UnKnown` | No reverse lookup zone | Created `7.168.192.in-addr.arpa` |

---

## Command toolbox

Start with the ⭐ ones.

| Command | What it tells you |
|---|---|
| ⭐ `ipconfig /all` | Which DNS servers this machine uses |
| ⭐ `nslookup name` | Name → IP, and **which server answered** |
| ⭐ `nslookup name 192.168.7.82` | Ask one specific DNS server |
| ⭐ `ipconfig /flushdns` | Clear the client cache |
| `nslookup 192.168.7.81` | IP → name (reverse) |
| `nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.silverbank.internal` | List the domain controllers |
| `Resolve-DnsName name -Server 192.168.7.80` | PowerShell nslookup (more detail) |
| `ipconfig /displaydns` | What the client has cached |
| `ipconfig /registerdns` | Make a Windows machine re-register its own records |
| `ping -a 192.168.7.81` | Show the name for an IP |
| `Test-NetConnection 192.168.7.80 -Port 53` | Is the DNS port reachable? |
| `dcdiag /test:dns` | Full DNS health check of a DC |
| `Get-DnsServerZone` | All zones on this DNS server |
| `Get-DnsServerResourceRecord -ZoneName corp.silverbank.internal` | All records in a zone |
| `Get-DnsServerForwarder` | This server's forwarders |
| `getent hosts name` (Linux) | Name → IP, the way Linux applications see it |
| `resolvectl status` (Linux) | Which DNS servers Linux uses |

---

## Event Viewer

**DNS Manager → Global Logs → DNS Events**
(or Event Viewer → Applications and Services Logs → **DNS Server**)

| Event ID | Meaning |
|---|---|
| **4013** | DNS is waiting for AD to finish loading at startup — **normal right after a DC boots**; a problem only if it keeps repeating |
| **4000 / 4007** | DNS couldn't load a zone from Active Directory — check AD health (`dcdiag`) |
| **5501 / 5504** | Bad or invalid packets received — usually harmless in a lab |

> 💡 Use **Filter Current Log…** and show only **Errors** and **Warnings**.