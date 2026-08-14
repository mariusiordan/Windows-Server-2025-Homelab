# Windows Server 2025 — Active Directory Homelab

A build journal for a Windows Server 2025 Active Directory environment running on a
Proxmox VE homelab. Documents the full path from bare VM to a working domain with
Group Policy, a joined client, and PowerShell automation.

Built as a companion to the [SilverBank](https://github.com/mariusiordan/DevOps-final-project)
Linux/cloud platform — the goal is coverage of on-premises Windows infrastructure, which
is what most UK entry-level infrastructure roles actually run.

**Status:** in progress. Updated at the end of each session.

---

## Goal

Build and document a production-shaped Active Directory environment:

- Domain Controller with AD DS and integrated DNS
- DHCP with reservations and scope options
- Organisational Unit structure reflecting a real business
- Group Policy Objects for password policy, drive mapping, and desktop restrictions
- File server with NTFS and share permissions
- Windows 11 client joined to the domain
- PowerShell automation for bulk user provisioning and audit reporting

Certification target: **AZ-802 (Administering Windows Server)**, which replaces the
retiring AZ-800/AZ-801 pair. AD DS is the highest-weighted domain on that exam.

---

## Planned architecture

| Host | Role | OS | Address |
|---|---|---|---|
| DC01 | Domain Controller, DNS, DHCP | Windows Server 2025 Standard (Desktop Experience) | static, TBC |
| FS01 | File server, member server | Windows Server 2022 | static, TBC |
| WS01 | Domain-joined client | Windows 11 | DHCP |

Hypervisor: Proxmox VE. All hosts on a single lab bridge.

A deliberately mixed 2025/2022 environment — real estates are rarely single-version, and
it makes forest and domain functional levels a real consideration rather than a default.

---

## Design decisions

**Windows Server 2025 over 2022.**
The skills are effectively identical for AD DS, but AZ-802 targets the current release and
"Server 2025" reads better on a CV than a version already two generations back.

**Desktop Experience over Server Core.**
Core is the better production choice, but the graphical consoles (Server Manager, ADUC,
GPMC) are how the concepts are learned. This choice cannot be reversed after installation
without a full reinstall.

**virtio-win pinned to 0.1.271.**
Proxmox documents read errors and I/O performance problems when running Windows Server
2025 on virtio-win 0.1.285. Version 0.1.271 has no known issues and is the version used
here. Verified before VM creation rather than discovered afterwards.

**`--ostype win11` in Proxmox.**
Proxmox has no Server 2025 option. Server 2025 and Windows 11 share kernel build 26100
(visible in the ISO filename), so `win11` is the correct selection for KVM timer and
feature flags.

**UEFI (OVMF) with q35.**
Server 2025 expects UEFI. Secure Boot keys are pre-enrolled, including the 2023
Microsoft certificate.

**Static addressing on the Domain Controller.**
A DC is the DNS server for its domain. Clients receive its address via DHCP and use it for
every lookup. If that address moves, every client queries a dead address and the domain
appears to vanish. Same principle as reserving an Elastic IP for a bastion host.

---

## Build commands

### VM creation (run on the Proxmox node)

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
  --ide0 local:iso/<WINDOWS_SERVER_ISO>,media=cdrom \
  --ide1 local:iso/virtio-win.iso,media=cdrom \
  --net0 virtio,bridge=vmbr0 \
  --agent 1 \
  --boot order=ide0
```

Two CD-ROM drives are required: one for the installer, one for the VirtIO drivers.

### Verify the VirtIO driver version before building

```bash
blkid /var/lib/vz/template/iso/virtio-win.iso
# LABEL="virtio-win-0.1.271"
```

The ISO filename is identical across releases. Only the volume label reveals the version.

### Snapshots

```bash
qm snapshot 900 clean-install --description "Base OS installed, before AD DS"
qm rollback 900 clean-install
qm listsnapshot 900
```

### Boot order after installation completes

```bash
qm set 900 --boot "order=scsi0;ide0"
```

---

## Troubleshooting

**Windows installer shows no disks at the partition screen**
Not a disk fault — the disk is VirtIO SCSI and Windows ships no driver for it.
Load driver → browse the virtio-win CD → `vioscsi\2k25\amd64` (or `2k22` if absent).

**Installer restarts from the beginning after the first reboot**
`Press any key to boot from CD or DVD` is a question, not an instruction. Answer it once,
at the very start. On subsequent reboots, let it time out so the machine boots from disk.
Fix permanently by setting the boot order to disk first.

**rsync reports two "No such file or directory" errors for one file**
Path contains a space and was not quoted; the shell split it into two arguments. The
double error, splitting exactly at the space, is the signature of a quoting problem rather
than a missing file. Quote the whole path. Use `-P` (uppercase) for progress and resumable
partial transfers — `-p` only preserves permissions.

---

## Lessons learned

- **Verify driver versions before building, not after failing.** A wrong VirtIO version
  produces intermittent I/O errors, never a message naming the driver.
- **Some installation choices are irreversible.** Desktop Experience vs Core cannot be
  changed later. Read the edition list carefully.
- **A DC is infrastructure other machines depend on**, so it gets a fixed address — the
  same reasoning behind a static IP on a bastion or reverse proxy.
- **Restoring a Domain Controller from a hypervisor snapshot is unsafe in production.**
  AD replication uses Update Sequence Numbers; rolling a DC backwards causes USN rollback,
  after which other DCs silently ignore its changes. Harmless in a single-DC lab, but the
  reason Microsoft publishes dedicated DC recovery procedures.
- **Cloning a Windows VM without `sysprep /generalize` duplicates the machine SID**, which
  causes obscure failures once the clones are domain members.

---

## Session log

### Session 1 — base VM
- Downloaded Windows Server 2025 Evaluation ISO (build 26100) and virtio-win 0.1.271
- Transferred ISOs to the Proxmox node with `rsync -avP`
- Verified the VirtIO version by volume label before building
- Created VM 900 (`dc01`): q35, UEFI, 4 vCPU, 6 GB RAM, 60 GB VirtIO SCSI disk
- Installed Windows Server 2025 Standard (Desktop Experience), loading `vioscsi` at the
  partition screen
- Took a `clean-install` snapshot before any configuration

---

## Next steps

- [ ] Install full VirtIO driver package and QEMU Guest Agent
- [ ] Rename host to DC01
- [ ] Configure static IP address
- [ ] Install the AD DS role
- [ ] Promote to Domain Controller, creating a new forest
- [ ] Configure DHCP with a scope and reservations
- [ ] Build the OU structure and populate users with PowerShell
- [ ] Create and link Group Policy Objects
- [ ] Join a Windows 11 client to the domain
- [ ] Add FS01 as a member server with shared folders and NTFS permissions
- [ ] Architecture diagram

windows-server-2025-homelab
│
├── README.md
├── LICENSE
│
├── architecture
│   ├── infrastructure.md
│   └── images/
│
├── 01-active-directory
│   ├── installation.md
│   ├── organisational-units.md
│   ├── users-groups.md
│   ├── troubleshooting.md
│   └── images/
│
├── 02-dns
│   ├── configuration.md
│   ├── troubleshooting.md
│   └── images/
│
├── 03-dhcp
│   ├── configuration.md
│   ├── reservations.md
│   └── images/
│
├── 04-group-policy
│   ├── password-policy.md
│   ├── security-policy.md
│   ├── drive-mapping.md
│   ├── troubleshooting.md
│   └── images/
│
├── 05-file-server
│   ├── shares.md
│   ├── ntfs-permissions.md
│   └── images/
│
├── 06-client-join
│   ├── domain-join.md
│   └── troubleshooting.md
│
└── scripts
    ├── new-bulk-users.ps1
    ├── audit-inactive-accounts.ps1
    └── README.md