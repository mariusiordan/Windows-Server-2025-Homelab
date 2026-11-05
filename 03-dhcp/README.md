# 03 — DHCP

> Home lab project. **SilverBank** is a fictional company. This folder explains **DHCP on Windows Server 2025**: how it works, how to install and configure it, how to maintain it and how to troubleshoot it.

> ⚠️ **Reference only — not applied in my lab yet.** My home router already runs DHCP on `192.168.7.0/24`. Two DHCP servers on the same network would conflict, and home devices could lose internet when the lab is off. The values in these guides are **examples** for a future SilverBank VLAN (`10.20.0.0/24`).

Every guide shows the **GUI steps first**, then **PowerShell (optional)**, then **how to check it worked**.

Last updated: 5 November 2026

---

## Guides in this folder

| # | Guide | What it covers |
|---|---|---|
| 1 | [Installation and authorization](01-installation.md) | Install the DHCP role, finish the post-install setup, authorize it in AD |
| 2 | [Scopes and options](02-scopes-and-options.md) | Create a scope, exclusions, lease time, gateway/DNS options |
| 3 | [Reservations and leases](03-reservations-and-leases.md) | Fixed IPs for printers, reading the lease table |
| 4 | [Maintenance](04-maintenance.md) | Backup/restore, failover between two servers, statistics, logs, DHCP relay |
| 5 | [Troubleshooting](05-troubleshooting.md) | Common DHCP tickets, APIPA, client commands, event IDs |

---

## What DHCP does (in one minute)

**DHCP** (Dynamic Host Configuration Protocol) gives a device its **network settings automatically** when it connects:

| Setting | Example |
|---|---|
| IP address | 10.20.0.123 |
| Subnet mask | 255.255.255.0 |
| Default gateway (router) | 10.20.0.1 |
| DNS servers | 192.168.7.80, 192.168.7.82 |
| DNS domain name | corp.silverbank.internal |
| Lease time | 8 days |

Without DHCP, IT would type these settings by hand on every PC, phone and printer. 😅

---

## How it works: DORA

```
 Client                                   DHCP server
   │ ── 1. DISCOVER  (broadcast: "Any DHCP server out there?") ──►│
   │ ◄── 2. OFFER    ("You can have 10.20.0.123")             ───│
   │ ── 3. REQUEST   ("Yes please, I'll take 10.20.0.123")   ──►│
   │ ◄── 4. ACK      ("Confirmed — it's yours for 8 days")    ───│
```

Remember it as **D-O-R-A**: **D**iscover, **O**ffer, **R**equest, **A**cknowledge.

| Fact | Why it matters |
|---|---|
| DHCP uses **UDP 67** (server) and **UDP 68** (client) | Firewalls must allow them |
| The DISCOVER is a **broadcast** | Broadcasts **don't cross routers** → other networks need a **DHCP relay** |
| The client **renews** at 50% of the lease time | A short server outage doesn't break clients immediately |
| No answer → Windows gives itself **169.254.x.x** (APIPA) | `169.254` = "I couldn't find a DHCP server" |

---

## Words to know

| Word | Meaning |
|---|---|
| **Scope** | A range of IPs the server can hand out for one network |
| **Exclusion** | IPs inside the range that DHCP must **not** give out (servers, printers with static IPs) |
| **Reservation** | A **fixed** IP always given to one device, identified by its **MAC address** |
| **Lease** | An IP "loaned" to a device for a period of time |
| **Options** | Extra settings sent with the IP: gateway (003), DNS servers (006), domain name (015) |
| **Authorization** | In an AD domain, a Windows DHCP server must be **approved in AD** before it can work |
| **DHCP relay / IP helper** | A router or switch that forwards DHCP broadcasts to a server in another network |
| **Failover** | Two DHCP servers share a scope, so DHCP keeps working if one fails |
| **APIPA** | `169.254.x.x` — the address Windows picks when no DHCP server answers |
| **Rogue DHCP server** | An unauthorized DHCP server (e.g. a home router plugged into the office network) |

---

## Golden rules

1. **One DHCP server per network** — unless they're configured as **failover partners**.
2. **`169.254.x.x` = no DHCP answer.** Check the cable/Wi-Fi, the VLAN, the relay and the DHCP service.
3. **Servers and network devices get static IPs or reservations** — never a random DHCP address.
4. **Exclude** every static IP from the scope, or two devices will fight over one IP.
5. In a domain, the **DNS option** must point to the **DCs**, never to a public DNS.
6. **Watch the scope usage** — a full scope means new devices get no IP.