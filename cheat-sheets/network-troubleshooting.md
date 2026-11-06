# Network Troubleshooting Cheat Sheet for Helpdesk

> Home lab project. Examples use my fictional company **SilverBank** (`corp.silverbank.internal`, network `192.168.7.0/24`) and my **Cisco Catalyst 3560-CX** switch. Replace names, IPs and port numbers for other networks.
> Organised by **what the user says**. Windows **GUI first** where there is one, then the command. DNS and DHCP problems have their own guides — linked, not repeated.

Last updated: 6 November 2026

---

## Contents

0. [The method](#0-the-method)
1. ["No internet" / "No network at all"](#1-no-internet--no-network-at-all)
2. ["It works by IP but not by name"](#2-it-works-by-ip-but-not-by-name)
3. ["I can't reach one server / one website"](#3-i-cant-reach-one-server--one-website)
4. ["The network is slow"](#4-the-network-is-slow)
5. ["Wi-Fi keeps dropping"](#5-wi-fi-keeps-dropping)
6. ["Windows says there's an IP address conflict"](#6-windows-says-theres-an-ip-address-conflict)
7. ["I can't reach the shared drives from home" (VPN)](#7-i-cant-reach-the-shared-drives-from-home-vpn)
8. ["Nothing works after an update / clean-up"](#8-nothing-works-after-an-update--clean-up)
9. [The switch side (Cisco)](#9-the-switch-side-cisco)
10. [Find which switch port a PC is plugged into](#10-find-which-switch-port-a-pc-is-plugged-into)
11. [Windows command toolbox](#11-windows-command-toolbox)
12. [Cisco command toolbox](#12-cisco-command-toolbox)
13. [Ports and addresses to know](#13-ports-and-addresses-to-know)

---

## 0. The method

### First, ask the user

| Question | Why |
|---|---|
| **One person or many?** | Many = network or server problem. One = that PC, cable or account |
| **Wired or Wi-Fi?** | Different checks |
| **Everything, or one site / one server?** | Everything = local problem. One = DNS, firewall or that server |
| **Did it ever work? What changed?** | Moved desk, new cable, update, password change, VPN… |
| **Exact error message?** | Ask for a screenshot |

### Then check from the bottom up

Stop at the **first** step that fails — that's your problem.

| # | Layer | Question | Quick check |
|---|---|---|---|
| 1 | **Physical** | Is it plugged in / connected? | Cable, port lights, Wi-Fi icon · `Get-NetAdapter` |
| 2 | **IP address** | Does it have a correct IP? | `ipconfig /all` — `169.254.x.x` = no DHCP |
| 3 | **Gateway** | Can it reach the router? | `ping 192.168.7.1` |
| 4 | **Beyond** | Can it reach a known IP outside? | `ping 8.8.8.8` |
| 5 | **DNS** | Do names work? | `nslookup bbc.co.uk` · `nslookup ln-fs01` |
| 6 | **Service** | Is the right **port** open? | `Test-NetConnection ln-fs01 -Port 445` |
| 7 | **Application** | Login, permissions, the app itself | Not a network problem any more |

> 💡 **IP works but name fails → DNS.** **Ping fails but the port works → only ping is blocked** (normal on many servers).

---

## 1. "No internet" / "No network at all"

### GUI

**Settings → Network & internet**: connected? Which network? → **Properties** shows the IP and DNS.
Or **Win + R** → `ncpa.cpl` → is the adapter **enabled**? Red cross = no cable / no link.

### Commands

```powershell
Get-NetAdapter | Select-Object Name, Status, LinkSpeed      # Up / Disconnected / Disabled
ipconfig /all                                              # IP, gateway, DNS, DHCP server
ping 192.168.7.1                                           # Gateway
ping 8.8.8.8                                               # Internet by IP
nslookup bbc.co.uk                                         # Internet by name
```

| Result | Meaning | Next step |
|---|---|---|
| Adapter **Disconnected** | No link: cable, switch port, Wi-Fi off | Check the cable and port lights → [section 9](#9-the-switch-side-cisco) |
| Adapter **Disabled** | Turned off in Windows | `ncpa.cpl` → right-click → **Enable** |
| IP **169.254.x.x** | No DHCP answer | [DHCP troubleshooting](../03-dhcp/05-troubleshooting.md) · wrong VLAN on the switch port? |
| IP OK, gateway ping fails | Wrong network/VLAN, gateway down, or wrong gateway | Check the gateway in `ipconfig /all`; other users affected? |
| `8.8.8.8` works, names fail | DNS problem | [Section 2](#2-it-works-by-ip-but-not-by-name) |
| Everything works for IT, not for the user | Proxy, browser, firewall, VPN on the user's PC | [Section 7](#7-i-cant-reach-the-shared-drives-from-home-vpn) · `netsh winhttp show proxy` |

> 💡 **Many users at once?** Stop checking PCs — check the switch, router, DHCP and DNS servers. Raise it as a **major incident** and tell your team leader.

---

## 2. "It works by IP but not by name"

This is **always DNS**. Full guide: [DNS troubleshooting](../02-dns/05-troubleshooting.md).

```powershell
ipconfig /all                          # DNS servers = the DCs (192.168.7.80 / .82)? Not the router?
nslookup ln-fs01                       # Which server answered? Does the name exist?
nslookup ln-fs01 192.168.7.82          # Ask the other DC directly
ipconfig /flushdns                     # Clear old cached answers, then test again
```

---

## 3. "I can't reach one server / one website"

```powershell
Test-NetConnection ln-fs01 -Port 445                         # Name, IP and port in one test
Test-NetConnection ln-fs01 -Port 445 -InformationLevel Detailed
tracert -d 192.168.7.81                                     # The path; -d = faster (no names)
```

| `Test-NetConnection` shows | Meaning |
|---|---|
| `PingSucceeded: False`, `TcpTestSucceeded: True` | Server is fine — only ping is blocked |
| `TcpTestSucceeded: False` | Service down, wrong port, or a **firewall** blocks it |
| `Name resolution failed` | DNS → [section 2](#2-it-works-by-ip-but-not-by-name) |

### Is the Windows Firewall blocking it? (on the server)

```powershell
Get-NetFirewallProfile | Select-Object Name, Enabled                   # Domain / Private / Public
Get-NetConnectionProfile | Select-Object Name, NetworkCategory         # Should be DomainAuthenticated
Get-NetTCPConnection -State Listen -LocalPort 445                      # Is anything listening on the port?
```

> ⚠️ A server showing network profile **Public** uses the strictest firewall rules. Fix the cause (usually DNS / reaching a DC), don't just turn the firewall off.

**One website only:** try another browser, a private window, another PC. If only one PC fails → that PC (proxy, cache, extension). If everyone fails → the website or the company's web filter.

---

## 4. "The network is slow"

First: **what** is slow? Everything, one website, the shared drive, only on Wi-Fi?

```powershell
Get-NetAdapter | Select-Object Name, Status, LinkSpeed      # Wired should be "1 Gbps"
pathping -n 192.168.7.81                                   # Packet loss on each hop (takes ~1 min)
ping -n 20 192.168.7.1                                     # Look at the times and "Lost"
```

| Clue | Likely cause |
|---|---|
| Wired **LinkSpeed 100 Mbps** (or 10) on a gigabit network | **Bad or damaged cable**, or a cable crimped wrongly (only 2 pairs connected) |
| `ping` times jump around, some **lost** | Bad cable, busy link, Wi-Fi interference |
| Slow only on Wi-Fi | Signal / interference → [section 5](#5-wi-fi-keeps-dropping) |
| Slow for everyone | Internet link full, a server overloaded, switch problem |
| Slow only for one app | The app or its server, not the network |

### GUI on the PC

**Task Manager → Performance → Ethernet/Wi-Fi** (is the link full?) · **Resource Monitor** (`resmon`) → **Network** tab: which program uses the network?

### On the switch

```
show interfaces GigabitEthernet1/0/5          ! speed, duplex, input/output errors, CRC
show interfaces counters errors               ! errors on every port
```

Rising **CRC / input errors** = bad cable or bad port. **Half-duplex** = speed/duplex mismatch.

> 💡 Real lab skill: I crimp my own Cat6 cables (T568B). A cable that passes the tester but links at **100 Mbps** usually has a bad pair.

---

## 5. "Wi-Fi keeps dropping"

```powershell
netsh wlan show interfaces          # SSID, Signal %, Receive/Transmit rate, channel, band
netsh wlan show wlanreport          # Builds a report of the last 3 days of disconnects
```

The report is saved as `C:\ProgramData\Microsoft\Windows\WlanReport\wlan-report-latest.html` — open it in a browser and look for the red disconnect events.

| Clue | Likely cause / fix |
|---|---|
| **Signal below ~60%** | Too far from the access point, walls → move closer, 5 GHz vs 2.4 GHz |
| Drops at the same times | Interference, power-saving, access point restarting |
| Only one laptop | Wi-Fi driver → update it; **Device Manager → adapter → Power Management** → untick "Allow the computer to turn off this device" |
| Everyone in one area | The access point → escalate to the network team |

**Forget and rejoin** the network: **Settings → Network & internet → Wi-Fi → Manage known networks → Forget**.

---

## 6. "Windows says there's an IP address conflict"

Two devices are using the **same IP**. Usually a **static IP inside the DHCP range**.

```powershell
ipconfig /all            # This PC's IP and MAC (Physical Address)
arp -a                   # IP ↔ MAC table — which MAC is using that IP?
getmac /v                # This PC's MAC addresses
```

Then find the **other** device:
- DHCP console → scope → **Address Leases** — who has the IP? → [DHCP guide](../03-dhcp/03-reservations-and-leases.md)
- Switch: find the port by MAC → [section 10](#10-find-which-switch-port-a-pc-is-plugged-into)

**Fix:** the static device gets an IP **outside** the DHCP range (or a **reservation**), then on the PC: `ipconfig /release` + `ipconfig /renew`.

---

## 7. "I can't reach the shared drives from home" (VPN)

| Check | How |
|---|---|
| Does the home internet work? | Open a public website |
| Is the VPN **connected**? | VPN app icon / **Settings → Network & internet → VPN** |
| Is the user's **password** expired or the account locked? | VPN often uses the AD account → [PowerShell sheet, section 2](powershell-helpdesk.md#2-check-a-users-account-health) |
| Do company names resolve over the VPN? | `nslookup ln-fs01` — the answer should come from a company DNS server |
| Is the server reachable? | `Test-NetConnection ln-fs01 -Port 445` |
| Home network uses the **same range** as the office (e.g. both `192.168.1.x`)? | Common cause of "VPN connected but nothing works" → escalate |

> 💡 **Proxy check:** `netsh winhttp show proxy` — a leftover proxy setting breaks browsing outside the office.

---

## 8. "Nothing works after an update / clean-up"

Reset the Windows network stack. Run **as Administrator**, then **restart** the PC:

```powershell
ipconfig /flushdns
netsh winsock reset          # Resets network sockets (fixes many "after malware/VPN removal" problems)
netsh int ip reset           # Resets TCP/IP settings
Restart-Computer             # ⚠️ Tell the user first
```

> ⚠️ `netsh int ip reset` removes **static IP** settings. On a server or a PC with a static IP, **write the settings down first**.

GUI alternative on Windows 11: **Settings → Network & internet → Advanced network settings → Network reset**.

---

## 9. The switch side (Cisco)

When the PC looks fine but has **no link** or the **wrong network**, check the switch port.

```
show interfaces status                         ! every port: connected / notconnect / err-disabled, VLAN, speed
show interfaces GigabitEthernet1/0/5           ! one port in detail: up/down, errors, speed/duplex
show vlan brief                                ! which ports are in which VLAN
show running-config interface Gi1/0/5          ! the port's configuration
show logging                                   ! recent events (port up/down, errors)
show cdp neighbors                             ! which Cisco devices are connected where
```

| `show interfaces status` | Meaning | Next step |
|---|---|---|
| `connected` | Link is up ✅ | Check the **VLAN** column is the right one |
| `notconnect` | No device / bad cable / PC off | Check the cable, the wall socket, try another port |
| `disabled` | Someone ran `shutdown` on the port | Ask why first, then `no shutdown` |
| `err-disabled` | The switch **shut the port down itself** (port security, BPDU guard, loop) | Find the reason: `show logging` · `show interfaces status err-disabled` |

### Bring a port back (after you know **why** it went down)

```
configure terminal
 interface GigabitEthernet1/0/5
  shutdown
  no shutdown
 end
```

> ⚠️ **Don't change switch config without permission** — on a helpdesk you usually **report** what you find to the network team. In my lab it's fine to practise.
> 💡 My lab today: the 3560-CX runs one flat network (VLAN 1, management IP 192.168.7.90). VLANs come later.

---

## 10. Find which switch port a PC is plugged into

Very useful when a desk socket has no label.

**1. Get the PC's MAC address** (on the PC):

```powershell
getmac /v                  # or: ipconfig /all → "Physical Address"
```

Example: `00-15-5D-01-02-03`

**2. Convert it to Cisco format:** groups of 4, dots, lowercase → `0015.5d01.0203`

**3. Find it on the switch:**

```
show mac address-table address 0015.5d01.0203
```

The **Ports** column shows the port (e.g. `Gi1/0/5`) and the **Vlan** column shows its VLAN.

> 💡 If the port shown is the **uplink** to another switch, run the same command on that switch to keep following the cable.

---

## 11. Windows command toolbox

Start with the ⭐ ones.

| Command | What it tells you / does |
|---|---|
| ⭐ `ipconfig /all` | IP, mask, gateway, DNS servers, DHCP server, MAC |
| ⭐ `ping <ip or name>` | Can I reach it? (Servers may block ping) |
| ⭐ `Test-NetConnection <host> -Port <n>` | Is a specific **service** reachable? |
| ⭐ `nslookup <name>` | What does DNS answer, and which server answered? |
| `Get-NetAdapter` | Adapters: Up/Down, **LinkSpeed** |
| `Get-NetIPConfiguration` | Short, clean view of IP, gateway and DNS per adapter |
| `tracert -d <ip>` | The path, hop by hop |
| `pathping -n <ip>` | Path + **packet loss** per hop (slow but detailed) |
| `Test-NetConnection <host> -TraceRoute` | PowerShell version of tracert |
| `arp -a` | IP ↔ MAC table (IP conflicts) |
| `getmac /v` | This PC's MAC addresses |
| `route print` | The routing table — which gateway is used |
| `netstat -ano` | Connections and listening ports, with process IDs |
| `Get-NetTCPConnection -State Listen` | PowerShell version — what is listening? |
| `Get-Process -Id <PID>` | Which program owns that connection |
| `ipconfig /release` · `/renew` | Get a new DHCP address |
| `ipconfig /flushdns` · `/registerdns` | Clear DNS cache · re-register this PC in DNS |
| `netsh wlan show interfaces` | Wi-Fi signal, speed, channel |
| `netsh wlan show wlanreport` | Wi-Fi history report |
| `netsh winhttp show proxy` | System proxy setting |
| `Get-NetFirewallProfile` | Is the firewall on for each profile? |
| `Get-NetConnectionProfile` | Domain / Private / Public network |
| `netsh winsock reset` · `netsh int ip reset` | Reset the network stack (restart after) |

**GUI tools:** `ncpa.cpl` (adapters) · **Settings → Network & internet** · **Task Manager → Performance** · `resmon` → Network tab · **Settings → System → Troubleshoot → Other troubleshooters → Network and Internet**.

---

## 12. Cisco command toolbox

Read-only `show` commands are safe. Start with the ⭐ ones.

| Command | What it tells you |
|---|---|
| ⭐ `show interfaces status` | All ports: connected / notconnect / err-disabled, VLAN, speed |
| ⭐ `show vlan brief` | VLANs and their ports |
| ⭐ `show mac address-table address <mac>` | Which port a device is on |
| `show ip interface brief` | IP interfaces (VLANs/SVIs) and whether they're up |
| `show interfaces <port>` | Details: errors, CRC, speed, duplex |
| `show interfaces counters errors` | Error counters for every port |
| `show running-config interface <port>` | One port's configuration |
| `show logging` | Recent events |
| `show cdp neighbors` | Connected Cisco devices |
| `show spanning-tree interface <port>` | Is the port blocked by spanning tree? |
| `show port-security` | Port security limits and violations |
| `show version` | Model, IOS version, uptime |

**Navigation:** `enable` (admin mode, prompt `#`) · `configure terminal` (config mode) · `end` (back) · `?` (help on any command) · **Tab** (auto-complete) · `show ... | include <word>` (filter output)

---

## 13. Ports and addresses to know

| Port | Service |
|---|---|
| 53 | DNS |
| 67 / 68 | DHCP |
| 80 / 443 | HTTP / HTTPS |
| 88 | Kerberos (domain logins) |
| 389 / 636 | LDAP / LDAPS |
| 445 | SMB (file shares) |
| 3389 | Remote Desktop |
| 5985 | PowerShell remoting (WinRM) |

| Address | Meaning |
|---|---|
| `169.254.x.x` | No DHCP answer (APIPA) |
| `127.0.0.1` | This computer itself (loopback) |
| `10.x`, `172.16–31.x`, `192.168.x` | Private addresses (inside a network) |
| `0.0.0.0` | "Any" / no address |
| `255.255.255.0` (`/24`) | The usual office subnet mask: 254 usable addresses |