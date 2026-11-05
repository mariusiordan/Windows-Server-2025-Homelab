# 2. Scopes and Options

> Create a **scope** (the range of IPs to hand out), exclude static IPs, choose the lease time, and send the **gateway, DNS and domain name** to clients.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

> ⚠️ Reference guide — example values. Not applied in my lab (see the [README](README.md)).

[← Installation](01-installation.md) · [Back to DHCP](README.md) · Next: [Reservations and leases →](03-reservations-and-leases.md)

---

## Example plan (SilverBank staff VLAN)

| Setting | Value | Why |
|---|---|---|
| Scope name | `SilverBank Staff` | Clear name for the network |
| Network | 10.20.0.0 / 24 | VLAN 20 — Staff |
| Range | 10.20.0.1 – 10.20.0.254 | The whole network… |
| Exclusion | 10.20.0.1 – 10.20.0.49 | …except the first 49 IPs, kept for the gateway, printers and static devices |
| Lease time | 8 days | Windows default — fine for office PCs |
| Router (option 003) | 10.20.0.1 | The VLAN 20 interface on the switch/router |
| DNS servers (option 006) | 192.168.7.80, 192.168.7.82 | The domain controllers |
| DNS domain name (option 015) | corp.silverbank.internal | Lets users type short names (`ln-fs01`) |

---

## Step 1: Create a scope

### GUI

**DHCP console** → expand the server → right-click **IPv4 → New Scope…**

| Wizard page | What to enter | Note |
|---|---|---|
| **Scope Name** | Name: `SilverBank Staff` · Description: `VLAN 20 - Staff PCs` | |
| **IP Address Range** | Start `10.20.0.1` · End `10.20.0.254` · Length `24` (mask `255.255.255.0`) | |
| **Add Exclusions and Delay** | Start `10.20.0.1` · End `10.20.0.49` → **Add** | Delay = 0 (only used with two servers without failover) |
| **Lease Duration** | 8 days | Shorter for guest Wi-Fi (e.g. 4 hours) |
| **Configure DHCP Options** | **Yes, I want to configure these options now** | |
| **Router (Default Gateway)** | `10.20.0.1` → **Add** | |
| **Domain Name and DNS Servers** | Parent domain: `corp.silverbank.internal` · DNS: `192.168.7.80`, `192.168.7.82` → **Add** each | The wizard checks each DNS server answers |
| **WINS Servers** | Leave empty → Next | WINS is old technology |
| **Activate Scope** | **Yes, I want to activate this scope now** | |

**Finish.** The scope appears as `Scope [10.20.0.0] SilverBank Staff` with a green tick.

### PowerShell (optional)

```powershell
# Create the scope
Add-DhcpServerv4Scope -Name "SilverBank Staff" -StartRange 10.20.0.1 -EndRange 10.20.0.254 `
  -SubnetMask 255.255.255.0 -LeaseDuration 8.00:00:00 -State Active

# Exclude the first 49 addresses
Add-DhcpServerv4ExclusionRange -ScopeId 10.20.0.0 -StartRange 10.20.0.1 -EndRange 10.20.0.49

# Options: gateway, DNS servers, domain name
Set-DhcpServerv4OptionValue -ScopeId 10.20.0.0 -Router 10.20.0.1 `
  -DnsServer 192.168.7.80,192.168.7.82 -DnsDomain "corp.silverbank.internal"
```

> 💡 `Set-DhcpServerv4OptionValue` **tests** the DNS servers. If they can't be reached, add `-Force`.

### Check

| GUI | PowerShell |
|---|---|
| Scope → **Address Pool** shows the range and the exclusion | `Get-DhcpServerv4Scope` |
| Scope → **Scope Options** shows 003, 006, 015 | `Get-DhcpServerv4OptionValue -ScopeId 10.20.0.0` |
| | `Get-DhcpServerv4ExclusionRange -ScopeId 10.20.0.0` |

---

## Step 2: Server options vs scope options

Options can be set at **three levels**. The most **specific** one wins:

```
Server options      → apply to ALL scopes on this server
   ▼ overridden by
Scope options       → apply to ONE scope
   ▼ overridden by
Reservation options → apply to ONE device
```

**Good practice:**
- **Server options:** things that are the same everywhere → **DNS servers (006)** and **domain name (015)**
- **Scope options:** things that differ per network → **Router (003)**

### GUI

- Server level: **IPv4 → Server Options** → right-click → **Configure Options…**
- Scope level: **Scope → Scope Options** → right-click → **Configure Options…**

### PowerShell (optional)

```powershell
# Server-level (all scopes)
Set-DhcpServerv4OptionValue -DnsServer 192.168.7.80,192.168.7.82 -DnsDomain "corp.silverbank.internal"
# Scope-level (one scope)
Set-DhcpServerv4OptionValue -ScopeId 10.20.0.0 -Router 10.20.0.1
```

---

## Step 3: Change, deactivate or delete a scope

### GUI

| Task | How |
|---|---|
| Change the range or lease time | Right-click the scope → **Properties** → General tab |
| Add an exclusion | Scope → right-click **Address Pool → New Exclusion Range…** |
| Stop giving out IPs (keep the settings) | Right-click the scope → **Deactivate** |
| Delete | Right-click the scope → **Delete** (⚠️ existing leases are lost) |

### PowerShell (optional)

```powershell
Set-DhcpServerv4Scope -ScopeId 10.20.0.0 -LeaseDuration 1.00:00:00   # 1 day
Set-DhcpServerv4Scope -ScopeId 10.20.0.0 -State InActive             # deactivate
Remove-DhcpServerv4Scope -ScopeId 10.20.0.0 -Force                   # delete
```

---

## Common option numbers

| Option | Name | Example |
|---|---|---|
| **003** | Router (default gateway) | 10.20.0.1 |
| **006** | DNS Servers | 192.168.7.80, 192.168.7.82 |
| **015** | DNS Domain Name | corp.silverbank.internal |
| 042 | NTP Servers | Time servers |
| 066 / 067 | Boot server / boot file | Network boot (PXE) for installing Windows |

---

## Choosing the lease time

| Network | Lease time | Why |
|---|---|---|
| Office PCs (wired) | 8 days | Devices stay for a long time |
| Laptops / staff Wi-Fi | 1 day | People come and go |
| Guest Wi-Fi | 1–4 hours | Many short visitors — avoid a full scope |

---

## What I learned

- A **scope** = one network's range of IPs + its settings.
- **Exclude** static IPs so DHCP never hands them out twice.
- **Options** give clients the gateway (003), DNS (006) and domain name (015).
- Server options → scope options → reservation options: **most specific wins**.

Next: [Reservations and leases →](03-reservations-and-leases.md)