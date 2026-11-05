# 02 — DNS

> Home lab project. **SilverBank** is a fictional company. This folder documents the **DNS** service in my Windows Server 2025 lab: how it works with Active Directory, how to manage zones and records, and how to troubleshoot it.

Every guide shows the **GUI steps first**, then **PowerShell (optional)**, then **how to check it worked**.

Last updated: 5 November 2026

---

## Guides in this folder

| # | Guide | What it covers | Lab status |
|---|---|---|---|
| 1 | [DNS Manager and zones](01-dns-manager-and-zones.md) | A tour of DNS Manager, the AD zones, forwarders | ✅ Built with AD |
| 2 | [Reverse lookup zone](02-reverse-lookup-zone.md) | IP → name lookups, PTR records, fixing "Server: UnKnown" | 🔲 To practise |
| 3 | [DNS records](03-dns-records.md) | A, CNAME and PTR records; adding Linux servers and network devices | 🔲 To practise |
| 4 | [Maintenance: aging and scavenging](04-maintenance.md) | Cleaning old records automatically, zone replication | 🔲 To practise |
| 5 | [Troubleshooting](05-troubleshooting.md) | Common DNS tickets, nslookup in depth, cache, command toolbox | 📖 Reference |

---

## What DNS does (in one minute)

DNS is the **phone book** of the network: it turns **names** into **IP addresses**.

```
You type:   \\ln-fs01\Finance
DNS answers: ln-fs01.corp.silverbank.internal = 192.168.7.81
Windows connects to 192.168.7.81
```

In an Active Directory domain, DNS does **more** than websites and file servers. Computers use DNS to **find a domain controller** for logins, Group Policy and Kerberos. **No working DNS = no working domain.** That's why most AD problems are really DNS problems.

---

## DNS in the SilverBank lab

```
                     Internet DNS
                          ▲
                          │  (forwarder: names the DCs don't know)
                  Home router 192.168.7.1
                          ▲
           ┌──────────────┴──────────────┐
     LN-DC01 (DNS)                  LN-DC02 (DNS)
     192.168.7.80   ◄── AD replication ──►   192.168.7.82
           ▲                              ▲
           └──────────┬───────────────────┘
                      │  every domain machine asks the DCs
        LN-FS01 · LN-CL01 · ln-glpi01
```

| Item | Value |
|---|---|
| DNS servers | LN-DC01 (192.168.7.80), LN-DC02 (192.168.7.82) |
| Forward zone | `corp.silverbank.internal` (Active Directory-integrated) |
| Forest zone | `_msdcs.corp.silverbank.internal` (DC locator records) |
| Reverse zone | `7.168.192.in-addr.arpa` (see guide 2) |
| Forwarder | 192.168.7.1 (home router) |
| Dynamic updates | Secure only (only domain members can register their own records) |

---

## Words to know

| Word | Meaning |
|---|---|
| **Zone** | A "page" of the phone book for one domain, e.g. `corp.silverbank.internal` |
| **Forward lookup** | Name → IP (the normal direction) |
| **Reverse lookup** | IP → name |
| **A record** | Name → IPv4 address (`ln-fs01` → `192.168.7.81`) |
| **CNAME** | An alias (nickname): `helpdesk` → `ln-glpi01` |
| **PTR record** | IP → name (lives in the reverse zone) |
| **SRV record** | "Which server offers this service?" — how PCs find DCs |
| **Forwarder** | Where the DC sends questions it can't answer (internet names) |
| **AD-integrated zone** | The zone is stored **inside Active Directory**, so it replicates to every DC automatically |
| **Dynamic update** | Windows machines create/update their **own** A record automatically |
| **TTL** | How long an answer may be cached (time to live) |
| **FQDN** | Fully qualified domain name: `ln-fs01.corp.silverbank.internal` |

---

## Golden rules

1. **Domain machines use only the DCs for DNS.** Never the router, never `8.8.8.8` — they can't find the domain.
2. **Test by IP, then by name.** IP works, name fails → it's DNS.
3. **Linux servers and network devices don't register themselves** — add their records manually.
4. **Check which server answered** (`nslookup` shows it) — you might be asking the wrong one.
5. **Old answers are cached.** After a change, flush the cache before testing again.