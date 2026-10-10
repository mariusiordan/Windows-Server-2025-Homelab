# SilverBank Lab Documentation

> Home lab project, SilverBank is a fictional company, built on my own Proxmox server to practise Windows Server 2025 administration. No real company data, passwords or public IP addresses are included.

Last updated: 10 October 2026

## Overview

SilverBank is a fictional small bank with about 35 employees. Its IT network runs on my Proxmox homelab, a Dell OptiPlex 5060 (i7-8700, 48 GB RAM).

| Item | Value |
| --- | --- |
| Domain (forest root) | corp.silverbank.internal |
| NetBIOS name | SILVERBANK |
| Functional level | Windows Server 2025 |
| Head office site code | LN (London) |
| Naming pattern | LOCATION-ROLE## (e.g. LN-DC01) |
| Course followed | Windows Server 2025 Essential Training (LinkedIn Learning, Robert McMillen) |

The lab is a practice environment for entry-level support and sysadmin roles.

## Server inventory

The domain has four Windows machines: two domain controllers, one file server and one client PC.

| VM ID | Name | Role | OS | IP | RAM | Preferred DNS | Alternate DNS |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 200 | LN-DC01 | First DC, DNS (created the forest) | Server 2025 Desktop Experience | 192.168.7.80 (static) | 4 GB | 192.168.7.82 | 127.0.0.1 |
| 202 | LN-DC02 | Second DC, DNS, Global Catalog | Server 2025 Desktop Experience | 192.168.7.82 (static) | 4 GB | 192.168.7.80 | 127.0.0.1 |
| 201 | LN-FS01 | File server + Windows Admin Center | Server 2025 Desktop Experience | 192.168.7.81 (static) | 4 GB | 192.168.7.80 | 192.168.7.82 |
| 220 | LN-CL01 | Employee PC | Windows 11 Pro 25H2 | DHCP (router) | 4 GB | 192.168.7.80 | 192.168.7.82 |

Common VM settings: q35, OVMF (UEFI) with Secure Boot keys, TPM 2.0, VirtIO SCSI single, disks on ZFS (NVMe) with Discard, CPU type host, QEMU guest agent on.

## IP address plan

The home LAN is 192.168.7.0/24. Static devices use addresses below .110; the router's DHCP hands out .110-.199.

| Range | Use |
| --- | --- |
| .1 | Home router (gateway) |
| .2 - .109 | Static: servers, VMs, cameras |
| .110 - .199 | DHCP: home devices, lab clients |
| .200 - .254 | Free for the future |

| IP | Device | Type |
| --- | --- | --- |
| 192.168.7.1 | Home router | Gateway |
| 192.168.7.12 | Proxmox host (pve) | Static |
| 192.168.7.50 | edge-nginx (SilverBank app) | Static |
| 192.168.7.60 | db-postgresql (SilverBank app) | Static |
| 192.168.7.70 | monitoring-staging (SilverBank app) | Static |
| 192.168.7.80 | LN-DC01 | Static |
| 192.168.7.81 | LN-FS01 | Static |
| 192.168.7.82 | LN-DC02 | Static |
| 192.168.7.101 | prod-vm1-BLUE (SilverBank app) | Static |
| 192.168.7.102 | prod-vm2-GREEN (SilverBank app) | Static |

The guest Wi-Fi is a separate, isolated network. These addresses are temporary: the plan moves SilverBank to its own VLANs behind OPNsense (servers in 10.10.10.0/24).

## How each machine joined the domain

Every machine joined only after its DNS pointed to the domain controllers. That was the cause of most errors.

| Machine | Method | Problem met | Fix |
| --- | --- | --- | --- |
| LN-DC01 | Created the domain: promotion wizard, Add a new forest | None | - |
| LN-DC02 | Joined first with PowerShell, then promoted: Add a domain controller to an existing domain, replicate from LN-DC01 | Wizard error: "error getting the list of sites ... user name or password is incorrect" | Wizard was using the local account `LN-DC02\Administrator`. Joined with the domain account first, logged in as `SILVERBANK\Administrator`, then promoted |
| LN-FS01 | GUI: Settings > System > About > Domain or workgroup > Change | Red in Server Manager before joining (workgroup, no trust) | Joined the domain; turned green |
| LN-CL01 | GUI, same as LN-FS01 | "An AD domain controller could not be contacted" | DNS was the router. Set DNS to .80 and .82 in `ncpa.cpl`, IP stays DHCP |

Command used to join LN-DC02:

```powershell
Add-Computer -DomainName corp.silverbank.internal -Credential SILVERBANK\Administrator -Restart
```

Command to set DNS on a member machine (check the adapter name with `Get-NetAdapter` first):

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.7.80,192.168.7.82
```

The UI would also have worked for LN-DC02: click **Change...** next to the credentials and type `SILVERBANK\Administrator`.

## Accounts and logins

Use `SILVERBANK\Administrator` on all four machines. Passwords are kept outside this document.

| Account | Type as | Exists on | Use for |
| --- | --- | --- | --- |
| Domain admin | `SILVERBANK\Administrator` | Active Directory (all machines) | Normal admin work |
| Local admin | `.\Administrator` | LN-FS01 only | Emergencies (domain broken) |
| Local setup account | `.\<setup name>` | LN-CL01 only | Emergencies |
| DSRM | (set at promotion) | Each DC | Repairing Active Directory |

- Domain controllers have no local accounts after promotion.
- In the Windows App (Mac), each PC entry must use the domain account, or "Ask when required".
- Later: create personal accounts (a daily account and a separate admin account) instead of the built-in Administrator.

## Proxmox operations

Start the domain controllers first and shut them down last.

| Order | VM | Startup delay | Shutdown timeout |
| --- | --- | --- | --- |
| 1 | 200 LN-DC01 | 180 s | 180 s |
| 2 | 202 LN-DC02 | 120 s | 180 s |
| 3 | 201 LN-FS01 | 30 s | 180 s |
| 4 | 220 LN-CL01 | - | 180 s |

The order applies only to host boot, host shutdown and Bulk Start / Bulk Shutdown, not to single VMs.

Set the order:

```bash
qm set 200 --startup order=1,up=180,down=180
qm set 202 --startup order=2,up=120,down=180
qm set 201 --startup order=3,up=30,down=180
qm set 220 --startup order=4,down=180
```

Safe manual shutdown:

```bash
qm shutdown 220 --timeout 180   # LN-CL01
qm shutdown 201 --timeout 180   # LN-FS01
qm shutdown 202 --timeout 180   # LN-DC02
qm shutdown 200 --timeout 180   # LN-DC01 last
qm list
```

Check the QEMU guest agent on all running VMs:

```bash
for id in $(qm list | awk '$3=="running"{print $1}'); do
  echo -n "VM $id: "
  qm agent $id ping 2>/dev/null && echo "agent OK" || echo "no agent"
done
```

RAM: 48 GB since October 2026 (2 x 16 GB + 2 x 8 GB, DDR4-2666). memtester passed on 36 GB.

## Troubleshooting log

DNS caused most problems: check that the machine uses the DCs as DNS before anything else.

| Date | Problem | Cause | Fix |
| --- | --- | --- | --- |
| Oct 2026 | QEMU agent missing on LN-DC02 and LN-CL01 | Guest tools not installed in Windows | Ran `virtio-win-guest-tools.exe` from the virtio CD |
| Oct 2026 | LN-CL01: "AD domain controller could not be contacted" | Client DNS was the router | Set DNS to .80 and .82 |
| Oct 2026 | Ping by IP worked, by name failed | LN-FS01 was not joined yet, so no DNS record | Joined the domain (`ipconfig /registerdns` if a record is missing) |
| Oct 2026 | Ping failed but services worked | Windows Firewall blocks ICMP by default | `Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"` |
| Sep 2026 | LN-DC02 promotion: wrong user or password | Wizard used the local account | Used `SILVERBANK\Administrator` |
| Sep 2026 | "The server is not operational" | DC not reachable (DNS or network profile) | Checked DNS, services and `Get-NetConnectionProfile` |
| Sep 2026 | DNS role not shown | Only the WAC DNS extension was installed, not the role | Role installed with the domain controller |
| Sep 2026 | Proxmox root disk at 75% | 7 old kernels (8.4 GB) | `apt purge` of old kernels, `apt clean`, journal limit 200 MB: down to 62% |
| Sep 2026 | scp to Proxmox failed | IP typo: 192.169 instead of 192.168 | Used the private address 192.168.7.12 |

Useful checks: `ipconfig /all`, `nslookup`, `Test-NetConnection <host> -Port 389`, `Get-Service NTDS, DNS, Netlogon`, `dcdiag /q`, `repadmin /replsummary`.

## Next steps

- [ ] Create SilverBank OUs, groups and users (PowerShell script from a CSV)
- [ ] Group Policy: password policy, mapped drives, first GPO to allow ping on servers
- [ ] File shares on LN-FS01 with NTFS permissions per department
- [ ] Reverse DNS lookup zone
- [ ] Windows 11 template with Sysprep, then LN-CL02
- [ ] OPNsense and VLANs, then DHCP on the DCs
- [ ] Backups (Proxmox Backup Server or USB disk) and a UPS
- [ ] GLPI ticketing system in a Debian container for helpdesk practice
- [ ] Later: second PC as a Manchester site (AD Sites and replication)
