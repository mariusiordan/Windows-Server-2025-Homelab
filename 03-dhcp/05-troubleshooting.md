# 5. Troubleshooting DHCP

> Common DHCP helpdesk tickets, how to read a client's IP settings, the commands to use, and where to look in Event Viewer.
> **GUI first** where there is one, then the command.

> ⚠️ Reference guide — example values. Not applied in my lab (see the [README](README.md)).

[← Maintenance](04-maintenance.md) · [Back to DHCP](README.md)

---

## Contents

1. [Start with the client](#start-with-the-client)
2. [Reading the client's IP address](#reading-the-clients-ip-address)
3. [Common DHCP tickets](#common-dhcp-tickets)
4. [Checking the server](#checking-the-server)
5. [Command toolbox](#command-toolbox)
6. [Event Viewer](#event-viewer)

---

## Start with the client

Most DHCP tickets start with *"I have no internet"* or *"I can't reach the shared drives"*.

### GUI (Windows 11)

**Settings → Network & internet → Ethernet** (or Wi-Fi) → look at **IPv4 address**, **DNS servers** and **IPv4 DNS assignment**.

Or **Win + R** → `ncpa.cpl` → double-click the adapter → **Details…**

### Command

```powershell
ipconfig /all
```

Look at these lines:

| Line | What to check |
|---|---|
| **DHCP Enabled** | `Yes` — otherwise someone set a static IP |
| **IPv4 Address** | Which network is it in? (see the table below) |
| **Default Gateway** | Present? Correct for this network? |
| **DHCP Server** | Is it **our** server, or a stranger? |
| **DNS Servers** | The **DCs**? |
| **Lease Obtained / Expires** | Recent and valid? |

---

## Reading the client's IP address

| The client has… | Meaning | Next step |
|---|---|---|
| `10.20.0.x` + correct gateway + DNS = DCs | ✅ DHCP is working | The problem is somewhere else (DNS, permissions, the app) |
| **`169.254.x.x`** (APIPA) | ❌ **No DHCP server answered** | Cable/Wi-Fi? Correct VLAN on the switch port? DHCP relay? DHCP service running? Scope full? |
| `192.168.1.x` / `192.168.0.x` (unexpected network) | ⚠️ A **rogue DHCP server** answered (e.g. someone's home router) | Look at **DHCP Server** in `ipconfig /all` — find and unplug that device |
| Correct IP, but **DNS = 8.8.8.8** or the router | ⚠️ Wrong **option 006** | Fix the DNS servers option on the scope/server |
| Correct IP, **no gateway** | ⚠️ Missing **option 003** | Add the Router option to the scope |
| Static IP (`DHCP Enabled: No`) | Someone typed it by hand | Set the adapter back to **Obtain automatically** (unless it's a server) |

---

## Common DHCP tickets

| User says | Likely cause | What to check / fix |
|---|---|---|
| "No internet" + client has `169.254.x.x` | No DHCP answer | `ipconfig /release` + `ipconfig /renew`; check cable, switch port VLAN, DHCP service, scope usage |
| "Windows says there's an IP address conflict" | Two devices have the same IP — usually a **static IP inside the scope** | Find the static device; add an **exclusion** or a **reservation**; DHCP **conflict detection** can help (IPv4 → Properties → Advanced → Conflict detection attempts: `1`) |
| "Nobody in the new office/VLAN gets an IP" | **DHCP relay** missing, or no scope for that network | Check `ip helper-address` on the VLAN interface; check a scope exists for that network |
| "New laptops get no IP, old ones are fine" | **Scope is full** | Display Statistics; delete dead leases; shorten the lease time; widen the scope |
| "Everyone suddenly has the wrong network/IPs" | **Rogue DHCP server** | `ipconfig /all` → **DHCP Server** IP → find the device; on the switch, **DHCP Snooping** prevents this |
| "I can browse the internet but not the shared drives" | DNS option points to the router or a public DNS | Option **006** must be the DCs |
| "The printer keeps changing IP and people can't print" | No **reservation** | Create a reservation for the printer's MAC |
| "Nobody gets an IP since the server was rebuilt" | Server **not authorized**, or scope not **activated** | DHCP console: red arrow? → Authorize · Scope → **Activate** |

---

## Checking the server

### GUI

| Where | What to look for |
|---|---|
| **DHCP console** → IPv4 | Green tick ✅ (authorized and running) · red arrow 🔻 (not authorized / stopped) |
| Scope icon | A red ⬇️ arrow on the scope = **deactivated** |
| Right-click **IPv4 → Display Statistics** | Scope usage (In Use %) |
| Scope → **Address Leases** | Is the client listed? With which IP? |
| Scope → **Scope Options** | 003, 006, 015 present and correct? |
| **services.msc** → **DHCP Server** | Running, startup type Automatic |
| Scope → Properties → **Failover** | State: Normal |

### PowerShell (optional)

```powershell
Get-Service DHCPServer
Get-DhcpServerInDC
Get-DhcpServerv4Scope | Select-Object ScopeId, Name, State
Get-DhcpServerv4ScopeStatistics | Select-Object ScopeId, PercentageInUse
Get-DhcpServerv4OptionValue -ScopeId 10.20.0.0
Get-DhcpServerv4Failover
```

---

## Command toolbox

Start with the ⭐ ones.

### On the client (any Windows PC)

| Command | What it does |
|---|---|
| ⭐ `ipconfig /all` | Shows the IP, gateway, DNS, **DHCP server** and lease times |
| ⭐ `ipconfig /release` | Gives the IP back (the PC loses its connection) |
| ⭐ `ipconfig /renew` | Asks DHCP for an IP again (watch for errors) |
| `ipconfig /flushdns` | Clears DNS cache (often needed after an IP change) |
| `ping <gateway>` | Can the PC reach its router? |
| `arp -a` | IP ↔ MAC table — useful to spot an IP conflict |
| `getmac /v` | The PC's MAC addresses (for reservations) |

> ⚠️ If you're connected **remotely** (RDP), `ipconfig /release` disconnects you! In **Command Prompt**, run `ipconfig /release && ipconfig /renew` on **one line** (so the renew still runs), or do it on site.

### On the server

| Command | What it tells you |
|---|---|
| ⭐ `Get-DhcpServerv4ScopeStatistics` | How full each scope is |
| ⭐ `Get-DhcpServerv4Lease -ScopeId 10.20.0.0` | Who has which IP |
| `Get-DhcpServerInDC` | Authorized DHCP servers in AD |
| `Get-DhcpServerv4Scope` | All scopes and whether they're Active |
| `Get-DhcpServerv4OptionValue -ScopeId 10.20.0.0` | The options a scope sends |
| `Get-DhcpServerv4Reservation -ScopeId 10.20.0.0` | The reservations |
| `Get-DhcpServerv4Failover` | Failover relationships and state |

---

## Event Viewer

**Server:** Event Viewer → Applications and Services Logs → **Microsoft → Windows → DHCP-Server → Admin / Operational**
**Client:** Event Viewer → Applications and Services Logs → **Microsoft → Windows → DHCP-Client**

| Event ID (server) | Meaning |
|---|---|
| **1046** | The DHCP server is **not authorized** in AD and won't start giving IPs |
| **1020** | A scope is **nearly full** (e.g. 85% or more in use) |
| **1063** | A scope has **no free addresses** left |
| **20xxx** (various) | **Failover** problems — check the partner server and the shared secret |

> 💡 Also check the **audit log** (`C:\Windows\System32\dhcp\DhcpSrvLog-<Day>.log`) to see exactly when a device got or lost its IP.

---

## What I learned

- **Start at the client:** `ipconfig /all` tells you most of the story.
- **169.254.x.x = no DHCP answer**. An unexpected network = **rogue DHCP**.
- IP conflicts usually come from **static IPs inside the scope** → exclusions and reservations.
- New VLAN without IPs → check the **relay** (`ip helper-address`).