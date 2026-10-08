# HYP01 — System Record

**Northwind Outdoor Ltd. | Project Redstone**

| Document Information | Value |
|---|---|
| System | HYP01 |
| Document Type | Infrastructure System Record (As-Built) |
| Version | 0.2 — Review Draft |
| Implementation Phase | Initial Infrastructure Build |
| Last Verified | 7 October 2026 (Sprint 1 evidence) |
| Documentation Task | NIM-15 |
| Status | Draft — Administrative access verification and design reconciliation pending |

## Server Quick Reference

| Attribute | Current Configuration |
|---|---|
| **Server Role** | Primary Proxmox virtualisation host |
| **Hostname** | `hyp01` |
| **FQDN** | `hyp01.northwind.internal` |
| **Management IP** | `192.168.178.10/24` — temporary |
| **Management URL** | `https://192.168.178.10:8006` |
| **Operating Platform** | Proxmox VE 9.2 / Debian 13 |
| **Physical Hardware** | Lenovo Legion |
| **Compute** | Intel Core i9-12900KF, 64 GiB RAM |
| **Storage** | Dedicated system and workload NVMe devices |
| **Network Bridge** | `vmbr0` |
| **Availability Model** | Single host, on-demand dual-boot |
| **Current Workloads** | No permanent virtual machines deployed |
| **Backup Status** | Not implemented |
| **Administrative Access** | Proxmox web interface and local console |

**Operational note:** HYP01 shares hardware with a Windows workstation. Windows is the default boot target; Proxmox must be selected manually using the firmware boot menu (F12).

---

## 1. Overview

HYP01 is the initial virtualisation platform for Northwind Outdoor Ltd.

It provides compute, memory, storage and virtual networking resources for the company's planned Linux server environment.

The host runs Proxmox VE on dedicated physical storage, with a separate NVMe device allocated to Northwind virtual machine workloads.

HYP01 follows the project's recovery-based resilience approach. It is a single-host environment without clustering or high availability.

The platform was successfully validated during Sprint 1 using a temporary RHEL 10.2 virtual machine (`TEST01`, VM ID `100`).

TEST01 was removed following validation. No permanent Northwind virtual machines had been deployed at the end of Sprint 1.

**Current operational state:** Initial platform build validated; additional network, administrative access and recovery capabilities remain outstanding.

## 2. Hardware & Platform

### 2.1 Physical Hardware

| Component | Configuration |
|---|---|
| Manufacturer / Model | Lenovo Legion T7 34IA7Z |
| Product Number | 90S2003GGE |
| Processor | Intel Core i9-12900KF |
| CPU Resources | 16 physical cores / 24 logical processors |
| Memory | 64 GiB |
| Storage Devices | 3 × Samsung NVMe SSDs, approximately 2 TB each |
| Firmware | UEFI |
| Boot Arrangement | Windows default; Proxmox manually selected |

### 2.2 Operating Platform

| Component | Verified Configuration |
|---|---|
| Hypervisor | Proxmox VE 9.2 |
| Base Operating System | Debian GNU/Linux 13 (Trixie) |
| `proxmox-ve` Package | 9.2.0 |
| `pve-manager` Package | 9.2.21 |
| Kernel | `7.0.14-20-pve` |
| Architecture | x86_64 |
| Root Filesystem | ext4 |
| Package Repositories | Debian Trixie and Proxmox No-Subscription |

Proxmox VE was installed on a dedicated NVMe device and upgraded to version 9.2.

The original installation challenges and upgrade procedure are retained in the NIM-9 and NIM-12 implementation records.

### 2.3 Platform Services

The following services were confirmed operational during Sprint 1:

| Service | Purpose | Validated State |
|---|---|---|
| `pveproxy` | Proxmox web management | Active |
| `pvedaemon` | Proxmox management operations | Active |
| `pvestatd` | Platform status collection | Active |
| `chrony` | Time synchronisation | Active / Enabled |

The final post-upgrade health check reported zero failed systemd units.

## 3. Network & Identity

### 3.1 Network Configuration

| Attribute | Current Configuration |
|---|---|
| Hostname | `hyp01` |
| FQDN | `hyp01.northwind.internal` |
| Domain | `northwind.internal` |
| Physical Interface | `nic0` |
| Linux Bridge | `vmbr0` |
| Management Address | `192.168.178.10/24` |
| Default Gateway | `192.168.178.1` |
| DNS Resolver | `192.168.178.1` |
| DNS Search Domain | `northwind.internal` |
| Network | `192.168.178.0/24` |

The host management address is assigned to `vmbr0`, which is connected to the physical interface `nic0`.

This configuration provides access to the existing home network.

**Temporary configuration:** The final Northwind management network and network-security zones have not yet been implemented.

### 3.2 Hostname Resolution

HYP01 maintains the following local hostname mapping in `/etc/hosts`:

```text
192.168.178.10 hyp01.northwind.internal hyp01
```

External DNS resolution is provided through the home router at `192.168.178.1`.

The home router is not assumed to provide authoritative DNS services for the future Northwind internal domain.

### 3.3 Time Configuration

| Attribute | Configuration |
|---|---|
| Timezone | Europe/Berlin |
| Hardware Clock | UTC |
| Time Service | Chrony |
| Time Sources | External NTP |
| Validation | Synchronised source confirmed |

The final time-source arrangements will be reviewed as the Northwind network develops.

## 4. Storage

### 4.1 Physical Storage Allocation

| Asset ID | Purpose | Device | Serial Suffix |
|---|---|---|---|
| HYP01-NVME01 | Windows workstation | Samsung NVMe, ~2 TB | `…0204` |
| HYP01-NVME02 | Proxmox system | Samsung NVMe, ~2 TB | `…0230` |
| HYP01-NVME03 | Northwind workloads | Samsung NVMe, ~2 TB | `…0206` |

The Windows device is excluded from Northwind storage configuration.

Linux `/dev/nvme*` device names may change between boots. Persistent device identifiers must be used when identifying physical disks for administrative operations.

Full hardware serial numbers are retained outside the public repository.

### 4.2 Proxmox System Storage

| Component | Configuration |
|---|---|
| Physical Device | HYP01-NVME02 |
| Volume Group | `pve` |
| Root Logical Volume | `pve/root` — 96 GiB |
| Swap Logical Volume | `pve/swap` — 8 GiB |
| Thin Pool | `pve/data` — approximately 1.71 TiB |
| Root Filesystem | ext4 |

Proxmox storage definitions:

| Storage ID | Type | Purpose |
|---|---|---|
| `local` | Directory | ISO images, templates, backups and imports |
| `local-lvm` | LVM Thin | VM disks and container root filesystems |

### 4.3 Northwind Workload Storage

| Component | Configuration |
|---|---|
| Physical Device | HYP01-NVME03 |
| Volume Group | `vg_northwind` |
| Thin Pool | `tp_workload` |
| Thin Pool Data | 1.75 TiB |
| Thin Pool Metadata | 4 GiB |
| Metadata Spare | Enabled |
| Chunk Size | 64 KiB |
| Zeroing | Enabled |
| Unallocated VG Capacity | Approximately 107.73 GiB |
| Proxmox Storage ID | `northwind-workload` |
| Permitted Content | VM disks and container root filesystems |

The unallocated volume group capacity provides flexibility for future storage administration. It does not provide disk redundancy.

### 4.4 Storage Architecture

```text
HYP01
├── NVME01 — Windows (Protected)
├── NVME02 — Proxmox System
│   └── VG: pve
│       ├── root
│       ├── swap
│       └── data (thin pool)
│           └── Proxmox: local-lvm
└── NVME03 — Northwind Workloads
    └── VG: vg_northwind
        └── tp_workload (thin pool)
            └── Proxmox: northwind-workload
```

The complete Proxmox storage configuration is maintained in `/etc/pve/storage.cfg`.

The storage architecture and validation evidence are documented under NIM-10 and NIM-14.

## 5. Administrative Access

### 5.1 Management Interfaces

- **Proxmox Web UI:** https://192.168.178.10:8006
- **SSH:** TCP/22, active and enabled
- **Physical console:** Available
- **Administrative access model:** Temporary direct bootstrap access

### 5.2 Authentication & Permissions

- **Proxmox administrator:** root@pam
- **Authentication realms:** PAM and PVE
- **Explicit Proxmox ACL assignments:** None
- **MFA:** Not configured for root@pam

### 5.3 SSH Configuration

- **Listening interfaces:** All IPv4 and IPv6 interfaces
- **Root login:** Permitted
- **Password authentication:** Enabled
- **Public-key authentication:** Enabled

### 5.4 Firewall & Access Restrictions

- **Legacy Proxmox firewall:** Disabled
- **nftables-based Proxmox firewall:** Active and enabled
- **Active nftables rules:** None
- **Management access restrictions:** No nftables restrictions currently enforced

### 5.5 Outstanding Actions

- Establish the approved administrative access path through JUMP01.
- Review and harden SSH authentication.
- Implement appropriate management firewall restrictions.
- Introduce MFA where required by the approved security design.
- Restrict or remove temporary direct administration once
  replacement and recovery access have been validated.

## 6. Dependencies & Limitations

### 6.1 Operational Dependencies

| Dependency | Operational Relevance |
|---|---|
| Physical workstation | Provides all host compute and memory resources |
| Electrical power | Required for host availability |
| System NVMe | Required for Proxmox boot and platform operation |
| Workload NVMe | Required for Northwind workload storage |
| Physical network | Required for normal remote management and guest connectivity |
| Home router | Provides bootstrap gateway and DNS |
| External NTP | Provides time synchronisation references |
| Debian / Proxmox repositories | Required for package installation and updates |
| Manual boot selection | Required to start Proxmox under the dual-boot arrangement |

### 6.2 Accepted Limitations

- HYP01 is a single physical hypervisor without high availability.
- The physical storage devices do not provide disk redundancy.
- HYP01 is not continuously available because it shares hardware with Windows.
- The current management network is a temporary bootstrap arrangement.
- No permanent Northwind workloads have been deployed.
- Backup and recovery capabilities have not yet been implemented or validated.
- Centralised monitoring and logging are not yet available.

These constraints reflect the initial Northwind v1 implementation scope and must be considered before deploying persistent business-critical workloads.

## 7. Known Issues & Outstanding Actions

### 7.1 Active Findings

| ID | Finding | Impact / Next Action |
|---|---|---|
| HYP01-KI-002 | RHEL 10.2 installation failed using `x86-64-v2-AES`; succeeded using `host` | Root cause unconfirmed. Review CPU model when establishing VM build standards |
| HYP01-KI-011 | Administrative access configuration not fully verified | Complete read-only inspection and update Section 5 |

The RHEL CPU finding does not currently prevent operation of the validated configuration.

Resolved installation and platform-service findings are retained in the relevant Sprint 1 implementation records rather than duplicated here.

### 7.2 Outstanding Actions

| Action | Tracking |
|---|---|
| Verify administrative access configuration | NIM-15 |
| Complete LLD-01 design reconciliation | NIM-15 |
| Update asset register and RTM | NIM-15 |
| Complete HYP01 Build / Rebuild Guide | NIM-15 |
| Implement dedicated administrative access environment | Future implementation |
| Implement final management network and security zones | Future implementation |
| Define standard VM configuration | Future implementation |
| Implement backup and recovery | Future implementation |
| Implement monitoring and logging | Future implementation |

Future implementation items must be linked to the appropriate Jira backlog records when planned.

### 7.3 Design Reconciliation

The current management network (`192.168.178.0/24`) is a temporary configuration.

No formal design deviation is declared solely on the basis of this temporary bootstrap configuration.

The full design reconciliation and implementation decision history should be maintained in the relevant design and implementation records.

## 8. References

### 8.1 Design and Requirements

| Reference | Purpose |
|---|---|
| Business Requirements Document | Business requirements supported by HYP01 |
| Requirements Traceability Matrix | Requirements and implementation traceability |
| High-Level Design | Overall Northwind infrastructure architecture |
| LLD-01 | Initial infrastructure design baseline |
| Design Decision Register | Rationale for approved architecture decisions |

### 8.2 Implementation Evidence

| Task | Subject |
|---|---|
| NIM-7 | HYP01 physical readiness |
| NIM-8 | Proxmox installation media |
| NIM-9 | Proxmox installation |
| NIM-10 | Local storage architecture |
| NIM-12 | Platform configuration and updates |
| NIM-14 | Virtualisation validation |
| NIM-15 | HYP01 documentation and acceptance |

Implementation evidence is retained under `implementation/sprint-01/`.

### 8.3 Related Operational Documents

| Document | Intended Location |
|---|---|
| HYP01 Build / Rebuild Guide | `build-guides/HYP01-build-guide.md` |
| HYP01 Asset Register Entry | Project asset register |
| HYP01 Documentation Record | `implementation/sprint-01/NIM-015-document-hyp01/README.md` |

Future operational procedures should be linked here as they are introduced.

---

## Document Maintenance

This System Record must be updated following material changes to HYP01's hardware, operating platform, networking, storage, administrative access or operational dependencies.

Historical troubleshooting, implementation procedures and validation results should remain in their respective records and be referenced rather than duplicated.

**Document Status:** Review Draft — Pending NIM-15 verification and acceptance.
