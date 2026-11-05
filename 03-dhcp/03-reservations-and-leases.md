# 3. Reservations and Leases

> Give devices like **printers** a fixed IP through DHCP, and read the **lease table** to see who has which IP.
> **GUI first**, then **PowerShell (optional)**, then **Check**.

> ⚠️ Reference guide — example values. Not applied in my lab (see the [README](README.md)).

[← Scopes and options](02-scopes-and-options.md) · [Back to DHCP](README.md) · Next: [Maintenance →](04-maintenance.md)

---

## Reservation or static IP?

| | Static IP (typed on the device) | DHCP reservation |
|---|---|---|
| Where it's set | On the device itself | Centrally, on the DHCP server |
| If the DNS server changes | Visit every device 😩 | Change the option once ✅ |
| Documentation | Easy to forget | Listed in the DHCP console |
| Best for | Servers, domain controllers, network switches | **Printers**, scanners, IP phones, cameras |

> ⚠️ **Servers and DCs keep static IPs.** If DHCP is down, they must still work.

---

## Step 1: Find the device's MAC address

A reservation is linked to the **MAC address** (the network card's hardware ID, e.g. `00-15-5D-01-02-03`).

| Device | How to find the MAC |
|---|---|
| Windows PC | `ipconfig /all` → **Physical Address** |
| Printer | Print a **network configuration page** from the printer menu |
| Already has a lease | DHCP console → scope → **Address Leases** → **Unique ID** column |

---

## Step 2: Create a reservation

### GUI — new reservation

1. **DHCP console** → scope → right-click **Reservations → New Reservation…**
2. Fill in:
   - Reservation name: `PRN-FINANCE-01`
   - IP address: `10.20.0.20`
   - MAC address: `00-15-5D-01-02-03`
   - Description: `Finance printer, 1st floor`
   - Supported types: **Both**
3. **Add → Close**.

### GUI — the quick way (from an existing lease)

**Address Leases** → right-click the device → **Add to Reservation**. The device keeps the IP it already has.

### PowerShell (optional)

```powershell
Add-DhcpServerv4Reservation -ScopeId 10.20.0.0 -IPAddress 10.20.0.20 `
  -ClientId "00-15-5D-01-02-03" -Name "PRN-FINANCE-01" -Description "Finance printer, 1st floor"
```

> 💡 A reservation can use an IP **inside the exclusion range** (`.1–.49`). That's common practice: keep all fixed devices in one block.

### Check

| GUI | PowerShell |
|---|---|
| Scope → **Reservations** lists `[10.20.0.20] PRN-FINANCE-01` | `Get-DhcpServerv4Reservation -ScopeId 10.20.0.0` |

The device gets the reserved IP at its **next renewal**. To force it on a Windows device: `ipconfig /release` then `ipconfig /renew`. On a printer: restart it.

---

## Step 3: Read the lease table

### GUI

Scope → **Address Leases**:

| Column | Meaning |
|---|---|
| **Client IP Address** | The IP given out |
| **Name** | The device name (e.g. `LN-CL01.corp.silverbank.internal`) |
| **Lease Expiration** | When the lease ends (`Reservation (active)` for reservations) |
| **Type** | DHCP or reservation |
| **Unique ID** | The device's **MAC address** |

**Helpdesk use:** *"Which PC has 10.20.0.123?"* → find the IP here → read the **Name**.

### PowerShell (optional)

```powershell
Get-DhcpServerv4Lease -ScopeId 10.20.0.0 |
  Select-Object IPAddress, HostName, ClientId, LeaseExpiryTime, AddressState

# Find one device
Get-DhcpServerv4Lease -ScopeId 10.20.0.0 | Where-Object HostName -like "*CL01*"
```

---

## Step 4: Delete a lease

Useful when an old device still holds an IP, or a scope is full of dead leases.

### GUI

**Address Leases** → right-click → **Delete**. The device simply asks again next time.

### PowerShell (optional)

```powershell
Remove-DhcpServerv4Lease -IPAddress 10.20.0.123
```

---

## DHCP and DNS working together

When DHCP gives a PC an IP, the **DNS record** must be updated too, or `ping LN-CL01` goes to the old IP.

### GUI

Right-click **IPv4 → Properties → DNS** tab:
- ✅ **Enable DNS dynamic updates** → **Dynamically update DNS records only if requested by the DHCP clients** (default — Windows PCs update their own records)
- ✅ **Discard A and PTR records when lease is deleted**

> 💡 For devices that can't update DNS themselves (printers, Linux), choose **Always dynamically update DNS records**, and set a dedicated **DNS update account** under **IPv4 → Properties → Advanced → Credentials**.

---

## What I learned

- **Reservations** = fixed IPs managed centrally, linked to a **MAC address**. Great for printers.
- **Servers and DCs** keep static IPs.
- The **lease table** answers "who has this IP?".
- DHCP can keep **DNS records** up to date.

Next: [Maintenance →](04-maintenance.md)