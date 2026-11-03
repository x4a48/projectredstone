# NIM-21 — Deploy FW01 RHEL Virtual Machine

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Sprint:** Sprint 2 — Infrastructure Foundation  
**Jira:** NIM-21 (NIM-S2-05)  
**Component:** FW01  
**Priority:** High  
**Status:** Technical implementation complete — documentation and sign-off pending  
**Implementation date:** 2026-11-03  
**Design references:** LLD-01-004, LLD-02-001, LLD-02-002, LLD-06, NIM-19 Interface Schedule

---

## 1. Overview

NIM-21 establishes FW01, Northwind Outdoor Ltd.'s first Red Hat Enterprise Linux virtual machine, and creates the internal virtual networking foundation required for the company's future server infrastructure.

FW01 will eventually provide routing and firewall services between Northwind's internal network segments and the external network.

This implementation deliberately focuses on the **virtual hardware, base operating system, storage layout, and isolated Layer 2 networking foundation**.

Detailed VLAN gateway configuration, IP forwarding, routing, NAT and firewall policy implementation are outside the scope of this ticket and will be completed in subsequent NIMs.

## 2. Business Requirement

Northwind requires a secure, maintainable network infrastructure that supports controlled communication between business departments, internal services and external networks.

Before introducing business applications and services, the company needs a reliable foundation for network segmentation and controlled access.

This implementation contributes to that requirement by establishing a dedicated virtual server that can later provide network security and routing capabilities.

## 3. Objectives

The objectives of NIM-21 were to:

1. Provision FW01 on the existing Proxmox hypervisor, HYP01.
2. Create an isolated, VLAN-aware internal Linux bridge (`vmbr1`).
3. Preserve existing HYP01 management connectivity through `vmbr0`.
4. Attach FW01's external and internal virtual network interfaces to their approved bridges.
5. Install RHEL 10 Minimal using the approved virtual machine specification.
6. Configure the server's hostname and storage layout.
7. Establish secure local console administration.
8. Verify core operating-system services and security settings.
9. Demonstrate successful operation without configured network connectivity.
10. Validate that the bridge and virtual machine configuration survive their respective reboots.

## 4. Implementation Scope

### In scope

- Proxmox virtual bridge provisioning
- VLAN-aware Layer 2 networking foundation
- FW01 virtual machine creation
- Virtual CPU, memory, storage and NIC allocation
- RHEL 10.2 Minimal installation
- UEFI boot configuration
- LVM storage provisioning
- Hostname configuration
- Local administrator authentication
- NetworkManager connection-state configuration
- SELinux validation
- QEMU Guest Agent validation
- Controlled reboot and persistence testing

### Out of scope

- VLAN subinterface creation
- Static IPv4 addressing and VLAN gateways
- IP forwarding
- NAT configuration
- Inter-VLAN routing
- Firewalld policies
- Remote SSH administration
- Integration with JUMP01
- Production network connectivity testing
- High availability and clustering

These activities will be introduced through subsequent implementation tickets.

## 5. Implemented Architecture

### 5.1 Hypervisor

| Property | Configuration |
|---|---|
| Hypervisor | HYP01 |
| Platform | Proxmox VE 9.2.1 |
| Existing management bridge | `vmbr0` |
| Internal network bridge | `vmbr1` |
| Internal bridge VLAN-aware | Yes |
| Internal bridge physical uplink | None |
| Internal bridge host IP | None |

### 5.2 Virtual networking

The implementation uses two distinct Linux bridges.

**`vmbr0` — External/bootstrap network**

Connected to HYP01's existing physical home-LAN interface. Provides the external network attachment for FW01 and preserves HYP01 management connectivity.

**`vmbr1` — Northwind internal network**

An isolated, VLAN-aware Linux bridge with no physical network uplink and no HYP01 management IP address.

This bridge will provide Layer 2 connectivity between FW01 and future Northwind virtual machines.

The bridge is configured to support VLAN IDs 2–4094, although individual Northwind VLANs have not yet been provisioned.

### 5.3 FW01 virtual network interfaces

| Proxmox interface | Bridge | RHEL interface | Intended role |
|---|---|---|---|
| `net0` | `vmbr0` | `ens18` | External/upstream |
| `net1` | `vmbr1` | `ens19` | Internal VLAN connectivity |

Both interfaces are present in RHEL, but no active IPv4 connections or routes are configured at the completion of NIM-21.

This allows FW01 to operate independently of the future Northwind routing infrastructure.

## 6. Virtual Machine Specification

| Property | Implemented configuration |
|---|---|
| VM name | FW01 |
| VM ID | 101 |
| Hypervisor node | HYP01 |
| Operating system | Red Hat Enterprise Linux 10.2 |
| Installation type | Minimal Install |
| Firmware | OVMF (UEFI) |
| Machine type | q35 |
| Secure Boot | Deferred |
| CPU | 1 socket, 2 cores |
| CPU type | host |
| Memory | 4096 MiB |
| Memory ballooning | Disabled |
| Operating-system disk | 40 GiB |
| Disk storage | `local-lvm` |
| Disk interface | VirtIO SCSI |
| SCSI controller | VirtIO SCSI single |
| SSD emulation | Enabled |
| Discard | Enabled |
| IO thread | Enabled |
| EFI disk | `local-lvm` |
| Network adapter model | VirtIO |
| Proxmox NIC firewall option | Enabled on both NICs |
| QEMU Guest Agent | Enabled and operational |

Secure Boot was deliberately deferred from the initial installation and should be revisited through a separately validated security change.

The enabled Proxmox NIC firewall option does not, by itself, establish the future FW01 routing and firewall policies.

## 7. Operating-System Configuration

### 7.1 System identity

| Property | Value |
|---|---|
| Hostname | `fw01.internal.northwindoutdoor.xyz` |
| Distribution | RHEL 10.2 (Coughlan) |
| Architecture | x86_64 |
| Virtualisation | KVM |
| Local administrator | `nw-admin` |
| Administration method | Proxmox virtual console |
| SELinux | Enforcing |

Local administrator login and privileged command execution through `sudo` were successfully demonstrated.

The root recovery credential protection procedure must be confirmed in the restricted operational documentation before final sign-off. No credentials should be committed to GitHub.

### 7.2 Storage configuration

FW01 uses a 40 GiB virtual operating-system disk with separate boot partitions and LVM-managed root and swap volumes.

| Mount point / volume | Filesystem | Device | Observed size |
|---|---|---|---|
| `/boot/efi` | VFAT | `/dev/sda1` | Approximately 572 MiB |
| `/boot` | XFS | `/dev/sda2` | Approximately 953 MiB |
| `/` | XFS | `vg_os/lv_root` | Approximately 23.28 GiB |
| Swap | Linux swap | `vg_os/lv_swap` | Approximately 1.86 GiB |

**LVM configuration:**

- Physical volume: `/dev/sda3`
- Volume group: `vg_os`
- Logical volumes: `lv_root`, `lv_swap`
- Remaining unallocated volume-group capacity: approximately 13.36 GiB

The remaining capacity provides flexibility for future filesystem expansion and RHCSA storage administration exercises.

The installed sizes are recorded as observed, rather than represented as exact matches to the originally proposed binary-unit capacities.

## 8. Work Performed

### 8.1 Internal bridge provisioning

Created `vmbr1` through the Proxmox Web UI with:

- Autostart enabled
- VLAN awareness enabled
- No physical bridge ports
- No IPv4 or IPv6 host address
- No default gateway
- No changes to `vmbr0` management addressing

Applied the configuration and verified the resulting Linux bridge settings.

A controlled HYP01 reboot was completed successfully.

After reboot, `vmbr1` remained present and VLAN filtering remained enabled. HYP01 management connectivity through `vmbr0` was preserved.

### 8.2 Virtual machine creation

Created FW01 as VM ID 101 using the approved Proxmox hardware specification.

Attached the external virtual NIC to `vmbr0` and added the internal virtual NIC on `vmbr1`.

Verified the resulting virtual hardware configuration in the Proxmox Hardware view before starting installation.

### 8.3 RHEL installation

Installed RHEL 10.2 Minimal from the existing installation ISO using the Proxmox virtual console.

Configured:

- UEFI boot
- System hostname
- Local administrative account
- Standard EFI and `/boot` partitions
- LVM volume group and logical volumes
- XFS root filesystem
- Swap logical volume

No additional application software groups were selected.

### 8.4 Storage correction during installation

During initial manual partitioning, Anaconda proposed formatting `/boot/efi` using XFS.

This was identified during the storage-change review before installation began.

The EFI partition was removed and recreated. Anaconda then correctly recognised it as an EFI System Partition.

The final installed filesystem was verified as VFAT.

The logical-volume names were also standardised to `lv_root` and `lv_swap` within `vg_os`.

### 8.5 Initial network configuration

During installation, the external interface obtained a temporary DHCP address from the upstream home LAN.

Following installation, NetworkManager was found to have automatically activated the connection profile associated with `ens18`.

To align with the agreed console-first deployment baseline:

1. Automatic activation was disabled for the external connection profile.
2. The active `ens18` connection was disconnected.
3. Both virtual NICs were verified as disconnected in NetworkManager.
4. The IPv4 routing table was confirmed empty.
5. The absence of assigned IPv4 addresses was verified.

The connection profile was retained for future configuration.

### 8.6 Operating-system verification

Validated the operating-system release, hostname, filesystem mounts, LVM configuration, swap activation, network interfaces, SELinux status and QEMU Guest Agent.

### 8.7 Controlled reboot

Performed an orderly FW01 reboot through the operating system.

After reboot, confirmed:

- Successful console login
- Root, boot and EFI filesystems mounted
- Swap activated
- LVM volume group and free capacity retained
- Both network interfaces disconnected
- No configured IPv4 default route
- SELinux enforcing
- QEMU Guest Agent active

This demonstrated that the core configuration persists across an operating-system restart.

## 9. Acceptance Criteria and Validation

| Acceptance criterion | Evidence / result | Status |
|---|---|---|
| VM created using approved ID and resources | VM 101, hardware inventory | Pass |
| External and internal NICs attached correctly | `net0`/`vmbr0`, `net1`/`vmbr1` | Pass |
| RHEL Minimal installed | RHEL 10.2 console and release output | Pass |
| Disk layout follows build design | Filesystem and LVM inspection | Pass, actual sizes documented |
| Hostname configured | `hostnamectl` | Pass |
| Root recovery credentials protected | Restricted procedure to be confirmed | Pending |
| Console authentication demonstrated | `nw-admin` login | Pass |
| Boots without final network configuration | Successful boot without active connections | Pass |
| OS and virtual hardware inventory recorded | Proxmox and RHEL evidence | Pass |
| VM reboot successful | Post-reboot verification | Pass |
| `vmbr1` created | Proxmox network configuration | Pass |
| VLAN-aware bridge persists | Host reboot and VLAN-filtering verification | Pass |
| No physical uplink or HYP01 IP on `vmbr1` | Bridge configuration and address inspection | Pass |
| No unintended Layer 2 connection to home LAN | No physical uplink; configuration inspection | Configuration validated; active isolation test deferred |
| HYP01 management remains operational | Management access following host reboot | Pass |
| Bridge recovery procedure documented | Deployment/rollback guide | Pending |

**Assessment:** The principal technical implementation is complete. Final ticket closure remains dependent on documentation and confirmation of outstanding recovery and isolation evidence.

## 10. Issues, Findings and Lessons Learned

### Finding 1 — Virtual bridges operate as Layer 2 switches

A Linux bridge provides Ethernet connectivity between attached interfaces.

Unlike FW01's future gateway interfaces, `vmbr1` does not require an IP address to forward Layer 2 frames.

This reinforces the distinction between switching and routing, and connects the virtual network architecture to conventional CCNA networking concepts.

### Finding 2 — Installer-generated configurations require verification

The initial EFI partition was incorrectly proposed as XFS.

Reviewing Anaconda's pending storage operations prevented an unsuitable UEFI partition layout from being installed.

**Lesson:** Validate the intended filesystem type and partition purpose before approving destructive storage changes.

### Finding 3 — A network adapter and a connection profile are different objects

FW01's virtual NICs remained present even when their NetworkManager connections were disconnected.

NetworkManager profiles determine how and when interfaces are configured.

**Lesson:** Network hardware state, interface state, IP addressing and connection-profile activation must be understood separately.

### Finding 4 — Persistence is an operational requirement

Successful initial configuration does not guarantee that settings will survive a reboot.

Both the hypervisor bridge and FW01 operating system were tested across controlled restarts.

**Lesson:** Include persistence testing in acceptance criteria for infrastructure changes.

### Finding 5 — LVM provides storage flexibility

The operating-system disk contains a volume group with spare unallocated capacity.

This allows future logical-volume expansion without immediately allocating the entire available space to the root filesystem.

**Lesson:** Storage design should consider future operational changes, not merely initial installation.

### Finding 6 — Virtual hardware configuration and guest configuration are separate

Enabling the QEMU Guest Agent in Proxmox does not automatically prove the corresponding guest service is running.

The service was separately verified inside RHEL and confirmed active.

**Lesson:** Validate integrations from both sides of the hypervisor/guest boundary.

## 11. Security Considerations

The implementation establishes a minimal operating-system foundation with the following characteristics:

- SELinux enforcing
- Minimal software installation
- Named local administrative account
- Console-first administration
- No active IPv4 network connectivity at handover
- No operational inter-VLAN routing or NAT
- Secure Boot deferred for a later controlled change
- Recovery credentials excluded from public documentation

Further network security configuration, firewall policies and secure remote administration are deferred to subsequent NIMs.

**Important:** FW01 must not be treated as an operational security boundary until its routing and firewall policies have been implemented and validated.

## 12. Rollback and Recovery

The deployment was performed before any Northwind production workloads depended on FW01.

Accordingly, a failed guest installation can be recovered by correcting the VM configuration or rebuilding FW01 from the approved specification.

For HYP01 bridge failures, recovery must prioritise restoration of `vmbr0` management connectivity and removal or correction of the new `vmbr1` configuration.

Detailed recovery procedures, prerequisites, commands and validation steps should be maintained in `rollback.md`.

No rollback was required during this implementation.

## 13. Evidence Register

| Evidence | Description |
|---|---|
| EV-01 | HYP01 network configuration showing `vmbr0` and `vmbr1` |
| EV-02 | Bridge VLAN-filtering and interface validation |
| EV-03 | HYP01 post-reboot network configuration |
| EV-04 | FW01 Proxmox virtual hardware inventory |
| EV-05 | RHEL Anaconda partitioning configuration |
| EV-06 | Final EFI partition correction |
| EV-07 | RHEL version, hostname and initial console login |
| EV-08 | Installed filesystem and LVM inventory |
| EV-09 | NetworkManager profile and interface validation |
| EV-10 | SELinux and QEMU Guest Agent status |
| EV-11 | FW01 post-reboot validation |

Evidence screenshots and terminal outputs should be stored in the repository's designated evidence directory, with sensitive information reviewed before publication.

## 14. RHCSA Skills Practised

This implementation provided practical experience in:

- RHEL installation and initial configuration
- Local administrative access and `sudo`
- Linux filesystem types and mount points
- GPT and UEFI boot requirements
- LVM physical volumes, volume groups and logical volumes
- Swap configuration and validation
- NetworkManager connection management
- Linux interface and routing inspection
- SELinux enforcement verification
- Systemd service inspection
- System reboot and configuration persistence
- Linux virtualisation and virtual network interfaces

The implementation also reinforced the relationship between Linux administration and Layer 2/Layer 3 networking.

## 15. Deliverables

- [x] HYP01 internal VLAN-aware bridge
- [x] FW01 virtual machine
- [x] RHEL 10.2 Minimal installation
- [x] UEFI and LVM storage configuration
- [x] Two virtual NICs connected to the approved bridges
- [x] Local console administration
- [x] Network-independent operating-system baseline
- [x] SELinux and QEMU Guest Agent verification
- [x] Hypervisor bridge persistence test
- [x] FW01 reboot and persistence test
- [ ] Root recovery procedure confirmation
- [ ] Active Layer 2 isolation evidence or documented test deferral
- [ ] Final deployment and rollback documentation
- [ ] Jira acceptance review and closure

## 16. Next Steps

The next implementation task is **NIM-22 — FW01 Interfaces and VLAN Gateways**.

This will introduce the approved network addressing and VLAN interface configuration required for FW01 to begin operating as Northwind's internal network gateway.

Routing, NAT and firewall policy enforcement will be introduced and validated through their respective subsequent NIMs.

No additional network functionality should be considered operational solely because the base virtual machine has been deployed.

## 17. Implementation Outcome

**Outcome: Successful technical deployment, subject to final documentation and sign-off.**

FW01 is now an operational RHEL 10.2 virtual machine hosted on HYP01, with the approved virtual hardware, storage layout and network attachments.

The server has demonstrated successful console administration, storage activation, SELinux enforcement, guest-agent operation and reboot persistence.

Northwind now has the foundational platform needed to implement its future network routing and security services in a controlled, incremental manner.
