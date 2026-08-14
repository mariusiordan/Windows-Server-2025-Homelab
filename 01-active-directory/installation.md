# Domain Controller Build — Full Runbook

Complete rebuild path for `CRW-DC01`, from a blank Proxmox node to a promoted Domain
Controller running Windows Server 2025 with integrated DNS.

Every command here has been run end to end. Placeholders in `<ANGLE_BRACKETS>` are the
only values that need changing for a different environment.

---

## Environment

| Item | Value |
|---|---|
| Hypervisor | Proxmox VE |
| Proxmox node address | `192.168.7.12` |
| VM ID | `900` |
| Hostname | `CRW-DC01` |
| IP address | `192.168.7.199/24` (static) |
| Default gateway | `192.168.7.1` |
| Domain | `silverbank.internal` |
| NetBIOS name | `SILVERBANK` |
| Forest / domain functional level | `WinThreshold` (Server 2016) |
| Guest OS | Windows Server 2025 Standard (Desktop Experience) |

### Naming convention

`<SITE>-<ROLE><NN>` — a three-letter site code, the role, and a two-digit index.

| Name | Role |
|---|---|
| `CRW-DC01` | Domain Controller, Crewe |
| `CRW-FS01` | File server, Crewe |
| `CRW-WS01` | Windows 11 client, Crewe |

Three letters rather than two, because two-letter codes collide quickly across a real
estate. The scheme leaves room for a second site (`LON-DC02`) without renaming anything,
which matters because renaming a Domain Controller is a dedicated procedure rather than a
single command.

NetBIOS names are capped at 15 characters. Longer names are silently truncated, leaving
the machine with two different identities.

---

## Stage 1 — Media preparation

### 1.1 Download the ISOs

**Windows Server 2025 Evaluation** (180 days) — from the Microsoft Evaluation Center.
Select ISO, English, 64-bit. Roughly 7.6 GB for build 26100.

**VirtIO drivers** — pinned to **0.1.271**, not the `stable-virtio` link.

```bash
cd /var/lib/vz/template/iso
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.271-1/virtio-win.iso
```

Proxmox documents read errors and I/O performance problems when running Windows Server
2025 on virtio-win 0.1.285. Version 0.1.271 has no known issues.

### 1.2 Transfer a locally downloaded ISO

```bash
rsync -avP "/path/with spaces/<WINDOWS_ISO>.iso" root@<PROXMOX_IP>:/var/lib/vz/template/iso/
```

Quote any path containing spaces. Use `-P` (uppercase) for progress and resumable partial
transfers — `-p` only preserves permissions.

### 1.3 Verify the VirtIO version before building

```bash
blkid /var/lib/vz/template/iso/virtio-win.iso
# LABEL="virtio-win-0.1.271"
```

Every release ships as `virtio-win.iso`. Only the volume label reveals the version, so
this check is the only way to know what is actually on disk.

---

## Stage 2 — VM creation

Run on the Proxmox node:

```bash
qm create 900 \
  --name dc01 \
  --ostype win11 \
  --machine q35 \
  --bios ovmf \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=1 \
  --cpu host \
  --cores 4 \
  --memory 6144 \
  --balloon 0 \
  --scsihw virtio-scsi-single \
  --scsi0 local-lvm:60,cache=writeback,discard=on,ssd=1 \
  --ide0 local:iso/<WINDOWS_ISO>.iso,media=cdrom \
  --ide1 local:iso/virtio-win.iso,media=cdrom \
  --net0 virtio,bridge=vmbr0 \
  --agent 1 \
  --boot order=ide0
```

| Parameter | Reason |
|---|---|
| `--ostype win11` | Proxmox has no Server 2025 option. Server 2025 and Windows 11 share kernel build 26100, visible in the ISO filename. |
| `--machine q35 --bios ovmf` | Server 2025 expects UEFI, not legacy BIOS. |
| `--efidisk0 ... pre-enrolled-keys=1` | Stores UEFI variables and enrols Microsoft Secure Boot keys. |
| `--balloon 0` | Disables dynamic memory. A Domain Controller wants stable RAM. |
| `--scsihw virtio-scsi-single` | Paravirtualised storage — faster than emulated, but needs a driver at install time. |
| `--ide1 virtio-win.iso` | The second CD-ROM. Without it the installer shows no disks. |
| `--agent 1` | Enables QEMU Guest Agent integration, so Proxmox can report the IP and shut down cleanly. |

Start the VM:

```bash
qm start 900
```

---

## Stage 3 — Windows installation

1. Press a key at `Press any key to boot from CD or DVD` — **once, at the first boot only**
2. Edition: **Windows Server 2025 Standard (Desktop Experience)**
3. Install type: **Custom: Install Windows only (advanced)**
4. At the partition screen, the disk list will be **empty** — this is expected
5. **Load driver** → Browse → virtio-win CD → `vioscsi\2k25\amd64` (fall back to `2k22`)
6. The 60 GB disk appears; continue normally
7. On reboot, **do not** press a key — let it boot from disk
8. Set the local Administrator password when prompted

Desktop Experience versus Core cannot be changed after installation without reinstalling.

Fix the boot order permanently once installation completes:

```bash
qm set 900 --boot "order=scsi0;ide0"
```

### Snapshot before configuration

```bash
qm snapshot 900 clean-install --description "Base OS installed, before AD DS"
```

Rollback if needed:

```bash
qm rollback 900 clean-install
```

---

## Stage 4 — Guest configuration

### 4.1 VirtIO drivers and Guest Agent

In the guest: open the virtio-win CD and run **`virtio-win-gt-x64`**. Reboot afterwards.

The install-time driver load covers storage only. This package adds networking (NetKVM),
balloon, and the QEMU Guest Agent.

### 4.2 Enable Remote Desktop

Run in PowerShell as Administrator:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

Two separate things: the first line starts the service, the second opens port 3389 in
Windows Firewall. Enabling only the service leaves the port closed, which presents as an
unreachable server rather than a firewall message.

### 4.3 Inspect the current network configuration

```powershell
(Get-NetIPConfiguration).IPv4Address.IPAddress
(Get-NetIPConfiguration).IPv4DefaultGateway.NextHop
(Get-NetIPConfiguration).DNSServer.ServerAddresses
Get-NetIPAddress -InterfaceAlias Ethernet -AddressFamily IPv4 | Select-Object IPAddress, PrefixLength
```

`Format-List` on `IPv4DefaultGateway` prints `MSFT_NetRoute` rather than an address,
because the property holds an object. Drilling into `.NextHop` returns the value. Any
`MSFT_...` or `System.Object[]` in output means one more level down, not an error.

### 4.4 Set a static address

Run the whole block at once. The RDP session drops partway through, and running the lines
individually leaves the server without a gateway or DNS.

```powershell
$IF = "Ethernet"
$IP = "192.168.7.199"
$GW = "192.168.7.1"

Set-NetIPInterface -InterfaceAlias $IF -Dhcp Disabled
Remove-NetIPAddress -InterfaceAlias $IF -AddressFamily IPv4 -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute -InterfaceAlias $IF -AddressFamily IPv4 -Confirm:$false -ErrorAction SilentlyContinue
New-NetIPAddress -InterfaceAlias $IF -IPAddress $IP -PrefixLength 24 -DefaultGateway $GW
Set-DnsClientServerAddress -InterfaceAlias $IF -ServerAddresses $GW
```

Addresses are not edited in Windows — the old one is removed and a new one added. The old
DHCP-supplied route must go too, or the machine ends up with two default gateways and
intermittent routing.

DNS deliberately points at the router at this stage, so name resolution keeps working
during setup. Promotion repoints it at the server itself.

**Verify:**

```powershell
Get-NetIPInterface -InterfaceAlias Ethernet -AddressFamily IPv4 | Select-Object Dhcp
(Get-NetIPConfiguration).IPv4DefaultGateway.NextHop
Test-NetConnection 8.8.8.8 -InformationLevel Quiet
Resolve-DnsName google.com
```

Expect `Disabled`, the gateway address, `True`, and a list of A records. `Dhcp: Disabled`
is the check that matters — without it, an unchanged address looks identical to a
successfully configured one.

Ping tests routing; name resolution tests DNS. They fail independently, so both are worth
running.

If the chosen address sits inside the router's DHCP pool, add a DHCP reservation for the
VM's MAC address so the router never hands it to another host.

### 4.5 Rename the host

```powershell
Rename-Computer -NewName "CRW-DC01" -Restart
```

Confirm after reboot:

```powershell
hostname
```

Do this **before** promotion. Afterwards the name is embedded in the domain identity and
in dozens of DNS records, and renaming becomes a dedicated procedure.

---

## Stage 5 — Active Directory Domain Services

### 5.1 Install the role

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

Expect `Success: True`. No reboot required.

This copies the binaries only — the server is not yet a Domain Controller. Comparable to
installing a database engine without creating a database.

`-IncludeManagementTools` brings ADUC, ADAC, Group Policy Management, and the
`ActiveDirectory` PowerShell module. Omitting it leaves the service running with no
tooling to administer it.

### 5.2 Promote to Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "silverbank.internal" `
  -DomainNetbiosName "SILVERBANK" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -NoRebootOnCompletion:$false `
  -Force:$true
```

Prompts for a **DSRM password** — Directory Services Restore Mode. Separate from the
Administrator password, used only when booting into AD database recovery. In production
this is the credential nobody can find on the day it is needed.

| Parameter | Reason |
|---|---|
| `Install-ADDSForest` | Creates a new forest rather than joining one. |
| `WinThreshold` | Server 2016 functional level — the highest generally available; no new levels shipped with 2019 or 2022. Sets the minimum OS version allowed to become a DC. Raisable later, never lowerable. |
| `-InstallDns:$true` | Installs DNS and creates an AD-integrated zone for the domain. |
| `-DatabasePath` / `-LogPath` | Location of `NTDS.dit` and its logs. Production deployments separate these onto different disks. |
| `-SysvolPath` | Replicated share holding logon scripts and Group Policy files. |

Runs for 5–10 minutes, then reboots automatically.

Yellow warnings about DNS delegation and default cryptography settings are expected on a
new forest and do not indicate a problem — there is no parent zone to delegate from.

### 5.3 Domain name choice

`.local` is reserved for mDNS/Bonjour and causes intermittent resolution failures on
networks containing Apple devices. Microsoft advises against it, though it appears
throughout older tutorials.

`.internal` is reserved by ICANN for private use and can never collide with a public
domain.

---

## Stage 6 — Post-promotion verification

Log in as **`SILVERBANK\Administrator`** — the former local account is now a domain
account.

```powershell
# Domain and forest identity
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode, InfrastructureMaster
Get-ADForest | Select-Object Name, ForestMode, SchemaMaster, DomainNamingMaster

# All five FSMO role holders
netdom query fsmo

# Domain Controller registration and services
dcdiag /test:services /test:replications

# DNS should now point at the server itself
(Get-NetIPConfiguration).DNSServer.ServerAddresses

# The SRV records clients use to locate the DC
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.silverbank.internal
```

The SRV lookup is the definitive test. Domain join and authentication both depend on
those records — if they resolve, the domain is genuinely working rather than merely
installed.

---

## Rebuild summary

From nothing, in order:

1. Download both ISOs, verify the VirtIO label
2. `qm create 900` with two CD-ROM drives
3. Install Windows, loading `vioscsi` at the partition screen
4. Snapshot
5. VirtIO guest tools, enable RDP
6. Static IP, rename to `CRW-DC01`
7. `Install-WindowsFeature AD-Domain-Services`
8. `Install-ADDSForest`
9. Verify with `dcdiag` and the SRV lookup

Roughly 45 minutes end to end, most of it waiting on the Windows installer.