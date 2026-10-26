# NIM-18 — Finalise FW01 and JUMP01 VM Specifications

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Sprint:** 02 — Initial Infrastructure Foundation  
**Jira Reference:** NIM-18 / NIM-S2-02  
**Priority:** High  
**Story Points:** 2  
**Parent Epic:** Platform & Virtualisation Foundation  
**Design References:** LLD-01-001, LLD-01-004, LLD-02-001, LLD-06  
**Predecessor:** NIM-17 — Verify HYP01 Platform Readiness  
**Status:** Design decisions approved; documentation and acceptance review pending  
**Target Hypervisor:** HYP01 — Proxmox VE 9.2  
**Target Guest OS:** Red Hat Enterprise Linux 10 Minimal

---

## 1. Objective

Establish implementation-ready virtual machine specifications for FW01 and JUMP01, ensuring both systems receive appropriate computing, storage and networking resources without unnecessarily consuming HYP01 capacity.

The specifications must reflect the distinct responsibilities of the two systems:

- **FW01:** Provide routing, NAT and firewall enforcement between Northwind's internal VLANs and the external network.
- **JUMP01:** Provide a controlled administrative entry point for managing Northwind's internal Linux infrastructure.

The resulting specifications will serve as the approved baseline for subsequent deployment tasks and future operational documentation.

### 1.1 Scope

**In scope:**

- VM identifiers and hostnames.
- CPU and memory allocations.
- Virtual disk sizing, controllers and provisioning.
- Firmware, machine type and CPU compatibility.
- Virtual NIC models, bridge assignments and VLAN configuration.
- HYP01 capacity assessment.
- Security and recovery design considerations.
- VM specification and asset register records.

**Out of scope:**

- Creating or installing either VM.
- Configuring RHEL network interfaces or firewall policies.
- Creating the internal Proxmox bridge.
- Implementing SSH hardening or administrative accounts.
- Configuring backups or performing recovery tests.
- Finalising IP addressing.

These activities are owned by subsequent implementation tickets.

---

## 2. Existing Infrastructure and Design Context

NIM-17 verified that HYP01 is operational and has sufficient provisional capacity to host the initial Northwind infrastructure.

### 2.1 HYP01 verified baseline

| Component | Verified configuration |
|---|---|
| Hypervisor | Proxmox VE 9.2 |
| Host CPU | Intel Core i9-12900KF |
| Logical processors | 24 |
| Physical RAM | Approximately 62 GiB |
| Available RAM at verification | Approximately 60 GiB |
| VM OS storage | `local-lvm` — approximately 1.71 TiB |
| Workload storage | `northwind-workload` — approximately 1.75 TiB |
| Existing VMs | None at NIM-17 verification |
| External bridge | `vmbr0` |
| Internal bridge | `vmbr1` — not yet implemented |
| Management address | `192.168.178.10/24` |

**Important:** These figures are based on NIM-17 evidence and represent the platform at the time of verification. Capacity and configuration must be rechecked before deployment.

### 2.2 Existing architectural decisions

The following established Northwind decisions apply:

1. RHEL 10 is the standard guest operating system.
2. VM OS disks are hosted on `local-lvm`.
3. Business and workload data use `northwind-workload` where appropriate.
4. Internal networks are separated using VLANs and firewall policies.
5. FW01 provides inter-VLAN routing and controlled external connectivity.
6. JUMP01 provides a controlled administrative access point.
7. HYP01 management must remain accessible independently of FW01.
8. Initial resilience is recovery-based rather than high-availability-based.
9. Infrastructure changes must be documented, validated and recoverable.

---

## 3. Design Decision Register

| ID | Decision | Status |
|---|---|---|
| NIM-18-DD01 | Three-digit VM identifiers grouped by functional category | Approved |
| NIM-18-DD02 | Initial CPU, RAM and OS disk allocations | Approved |
| NIM-18-DD03 | VirtIO SCSI disk configuration and thin provisioning | Approved |
| NIM-18-DD04 | UEFI, q35, host CPU model and Secure Boot | Approved |
| NIM-18-DD05 | VirtIO networking and VLAN-to-bridge mapping | Approved |
| NIM-18-DD06 | Capacity assessment and dual-path recovery strategy | Approved |

---

## 4. DD01 — VM Identification Convention

**Status:** Approved

### 4.1 Decision

Adopt three-digit Proxmox VM identifiers grouped into functional categories.

| VM ID range | Functional category | Example |
|---|---|---|
| 100–199 | Network infrastructure | FW01 |
| 200–299 | Management and administration | JUMP01 |
| 300–399 | Core infrastructure services | CORE01 |
| 400–499 | Business applications and file services | FILE01 |
| 500–599 | Monitoring and operations | OPS01 |
| 600–699 | Backup and recovery | BACKUP01 |
| 700–899 | Future infrastructure and expansion | Reserved |
| 900–999 | Temporary and test systems | TEST01 |

### 4.2 Initial assignments

| VM ID | Hostname | Role |
|---|---|---|
| 101 | FW01 | Firewall / Router |
| 201 | JUMP01 | Administrative Jump Server |

### 4.3 Rationale

Functional numbering provides:

- Predictable identification of infrastructure components.
- Consistency across Proxmox, Jira and technical documentation.
- Capacity for future expansion within each functional category.
- Easier identification during administration and troubleshooting.

### 4.4 Constraints

- VM identifiers must be unique within the Proxmox environment.
- Existing VM identifiers should remain stable throughout their lifecycle.
- Functional ranges are a Northwind administrative convention, not a Proxmox technical requirement.
- VM IDs must not be treated as security boundaries.

---

## 5. DD02 — Initial Resource Allocation

**Status:** Approved

### 5.1 Decision

Adopt the following initial resource allocations.

| Resource | FW01 | JUMP01 |
|---|---|---|
| VM ID | 101 | 201 |
| Guest OS | RHEL 10 Minimal | RHEL 10 Minimal |
| CPU sockets | 1 | 1 |
| CPU cores | 2 | 2 |
| Total vCPUs | 2 | 2 |
| RAM | 4 GiB | 2 GiB |
| OS disk | 40 GiB | 30 GiB |
| Storage backend | `local-lvm` | `local-lvm` |

### 5.2 FW01 sizing rationale

FW01 will provide routing, NAT, connection tracking and firewall enforcement for Northwind's internal networks.

The initial allocation of two vCPUs and 4 GiB RAM provides a conservative starting point for a small simulated enterprise environment.

Additional memory provides headroom for system services, connection tracking and operational logging.

CPU resources may be increased if monitoring identifies sustained processing pressure.

The 40 GiB OS disk provides space for RHEL, updates, configuration, logs and maintenance.

### 5.3 JUMP01 sizing rationale

JUMP01 will primarily support SSH administration and essential management tooling.

Two vCPUs and 2 GiB RAM are considered appropriate initial allocations for RHEL 10 Minimal without a graphical desktop.

The 30 GiB OS disk provides space for the operating system, updates, tools and logs.

### 5.4 Scaling policy

Resource allocations will be reviewed when:

- Sustained CPU utilisation affects performance.
- Memory pressure or swapping becomes excessive.
- Disk utilisation approaches operational thresholds.
- Additional services are introduced.
- Monitoring demonstrates that initial allocations are insufficient.

Resources will not be increased solely because unused HYP01 capacity is available.

---

## 6. DD03 — Virtual Disk Configuration

**Status:** Approved

### 6.1 Decision

Both VMs will use the following virtual storage configuration.

| Setting | Configuration |
|---|---|
| Storage backend | `local-lvm` |
| Provisioning | Thin provisioning |
| Disk interface | SCSI |
| SCSI controller | VirtIO SCSI single |
| Primary OS disk | `scsi0` |
| Discard/TRIM | Enabled |
| I/O thread | Enabled |
| SSD emulation | Enabled |

### 6.2 Rationale

**VirtIO SCSI single**

Provides a modern, virtualisation-optimised storage interface suitable for RHEL 10.

It supports efficient disk I/O and useful features including discard and per-disk I/O processing.

**Thin provisioning**

Allows physical storage consumption to grow as the guest writes data rather than requiring the entire virtual disk allocation to be physically consumed immediately.

This improves storage efficiency but requires monitoring to prevent pool exhaustion.

**Discard/TRIM**

Allows supported guest filesystems to notify the storage stack when blocks are no longer required.

This may permit unused blocks to be reclaimed by the underlying thin pool.

**I/O thread**

Allows eligible disk I/O operations to be handled through a dedicated I/O processing thread, reducing contention in some workloads.

**SSD emulation**

Exposes the virtual disk as non-rotational storage to the guest, reflecting the underlying NVMe-backed environment.

### 6.3 Risks and operational considerations

Thin provisioning introduces the possibility of overcommitting physical storage.

If physical or thin-pool metadata capacity is exhausted, guest writes may fail and services may be disrupted.

The following must therefore be monitored:

- Thin-pool data utilisation.
- Thin-pool metadata utilisation.
- Host filesystem capacity.
- Guest filesystem utilisation.
- Storage growth trends.

Discard support must be validated after deployment. Deleting a file inside RHEL does not necessarily release the corresponding physical allocation immediately.

### 6.4 Implementation validation

During deployment:

1. Confirm the disk is attached using the approved SCSI interface.
2. Verify that RHEL detects the expected disk capacity.
3. Confirm the selected controller and virtual disk configuration.
4. Inspect discard capability.
5. Verify normal filesystem operation and persistence after reboot.

Relevant Linux commands may include:

```bash
lsblk
lsblk -D
findmnt
fstrim -v /
```

Discard operations should be tested on an appropriate filesystem after confirming the storage stack supports them.

---

## 7. DD04 — Firmware, Machine Type and CPU Model

**Status:** Approved

### 7.1 Decision

| Setting | FW01 | JUMP01 |
|---|---|---|
| Firmware | OVMF (UEFI) | OVMF (UEFI) |
| Machine type | q35 | q35 |
| CPU model | `host` | `host` |
| CPU topology | 1 socket × 2 cores | 1 socket × 2 cores |
| EFI disk | Required | Required |
| Secure Boot | Enabled | Enabled |
| Pre-enrolled EFI keys | Enabled | Enabled |
| Virtual TPM | Not required | Not required |

### 7.2 UEFI rationale

UEFI provides a modern firmware environment suitable for RHEL 10.

It supports EFI boot management and Secure Boot.

### 7.3 q35 rationale

The q35 machine type provides a modern virtual hardware platform with PCI Express support.

It is preferred over legacy i440fx emulation for these new RHEL virtual machines.

### 7.4 CPU model rationale

The `host` CPU model exposes the physical host's supported CPU features to the guest.

During Sprint 1, TEST01 successfully booted using this configuration after a kernel panic was observed with `x86-64-v2-AES`.

The precise cause of the earlier kernel panic was not conclusively established.

The `host` model is therefore selected as a previously successful configuration for HYP01, not as proof that alternative CPU models are inherently incompatible with RHEL.

**Trade-off:** The VM may be less portable to other physical hypervisors with different CPU capabilities.

CPU compatibility must be reconsidered if Northwind introduces additional hypervisors, clustering or live migration.

### 7.5 Secure Boot rationale

Secure Boot is enabled to strengthen the integrity of the operating system boot process.

UEFI verifies trusted signatures on supported boot components, helping prevent unauthorised boot software from executing.

Secure Boot does not replace:

- Operating system patching.
- SELinux.
- Host and guest firewall controls.
- Administrative access restrictions.
- Backup and recovery.
- Disk encryption.

### 7.6 Recovery implications

Secure Boot can prevent startup if required boot components fail signature verification.

Operational documentation must therefore include suitable boot troubleshooting and recovery procedures.

The selected firmware and pre-enrolled keys must be validated against the RHEL 10 installation media and signed boot components.

### 7.7 Implementation validation

During deployment:

1. Confirm the VM uses OVMF and q35.
2. Confirm the EFI disk exists and the appropriate keys are enrolled.
3. Verify successful RHEL installation.
4. Confirm Secure Boot is enabled inside RHEL.
5. Verify successful reboot.
6. Record any compatibility or boot issues.

Example verification:

```bash
mokutil --sb-state
ls /sys/firmware/efi
```

The `mokutil` utility must be installed if it is not present in the minimal guest.

---

## 8. DD05 — Virtual Network Interface Design

**Status:** Approved

### 8.1 Decision

Use VirtIO virtual network adapters for both VMs.

FW01 will have two NICs: one external and one internal VLAN trunk.

JUMP01 will have one NIC connected to the Management Zone.

### 8.2 Virtual NIC mapping

| VM | Proxmox NIC | Model | Bridge | VLAN configuration | Purpose |
|---|---|---|---|---|---|
| FW01 | `net0` | VirtIO | `vmbr0` | Untagged | External/upstream |
| FW01 | `net1` | VirtIO | `vmbr1` | 802.1Q trunk | Internal VLAN routing |
| JUMP01 | `net0` | VirtIO | `vmbr1` | Access VLAN 100 | Management |

### 8.3 Network segmentation

| VLAN | Zone | Subnet |
|---|---|---|
| 100 | Management | `10.20.10.0/24` |
| 200 | Server | `10.20.20.0/24` |
| 300 | User | `10.20.30.0/24` |
| 400 | Peripheral | `10.20.40.0/24` |

Final interface addresses and gateway assignments are controlled by NIM-19.

### 8.4 FW01 external interface

The external interface will connect to `vmbr0`.

This bridge provides access to the upstream home network.

The approved external addressing, gateway and DNS/bootstrap requirements will be documented in NIM-19.

HYP01's management connectivity must remain independent of FW01.

### 8.5 FW01 internal interface

The internal interface will connect to `vmbr1`, which must be:

- VLAN-aware.
- Isolated from physical home-network interfaces.
- Configured without a HYP01 management IP address.
- Able to carry the approved tagged VLANs.
- Protected against unintended connectivity to the external network.

FW01's internal virtual NIC must not be assigned a single Proxmox access VLAN.

Instead, RHEL will create VLAN interfaces for VLANs 100, 200, 300 and 400.

Illustrative Linux interface names:

```text
ens19.100
ens19.200
ens19.300
ens19.400
```

Actual device names must be confirmed after deployment.

### 8.6 JUMP01 management interface

JUMP01 will have one VirtIO NIC connected to `vmbr1` with Proxmox VLAN tag 100.

Proxmox will handle the relevant VLAN tagging and untagging.

JUMP01 will therefore use a normal network interface inside RHEL rather than requiring a guest VLAN subinterface.

Access to other internal networks must pass through FW01 and comply with the approved firewall policy.

### 8.7 Security considerations

- FW01 is the controlled routing boundary between internal VLANs.
- JUMP01 must not have direct attachment to other internal VLANs.
- No internal bridge should provide unintended connectivity to the home LAN.
- Inter-VLAN access must be governed by FW01 firewall policies.
- HYP01 management must not depend on FW01 being operational.

### 8.8 Implementation ownership

| Activity | Jira owner |
|---|---|
| Addressing and interface schedule | NIM-19 |
| Create and validate `vmbr1` | NIM-21 |
| Deploy FW01 virtual NICs | NIM-21 |
| Configure FW01 VLAN interfaces and gateways | NIM-22 |
| Deploy JUMP01 with its approved NIC | JUMP01 deployment task |

---

## 9. DD06 — Capacity and Recovery Readiness

**Status:** Approved

### 9.1 Capacity assessment

The combined proposed allocations are:

| Resource | FW01 | JUMP01 | Combined |
|---|---|---|---|
| vCPU | 2 | 2 | 4 |
| RAM | 4 GiB | 2 GiB | 6 GiB |
| Virtual OS disk | 40 GiB | 30 GiB | 70 GiB |

Comparison against NIM-17:

| Resource | HYP01 baseline | Proposed commitment | Assessment |
|---|---|---|---|
| Logical CPUs | 24 | 4 vCPUs | Pass |
| Physical RAM | ~62 GiB | 6 GiB | Pass |
| OS storage | ~1.71 TiB | 70 GiB | Pass |

The combined memory allocation represents approximately 9.7% of physical RAM.

The combined virtual disk allocation represents approximately 4% of available `local-lvm` capacity.

vCPU allocation is not equivalent to dedicating physical CPU cores. Thin-provisioned disk allocation does not necessarily represent immediate physical storage consumption.

**Conclusion:** The initial resource specifications are comfortably within HYP01's verified capacity.

This is a planning assessment. Actual resource utilisation and guest compatibility remain subject to deployment validation.

### 9.2 Recovery strategy

Adopt a dual-path recovery strategy.

**Primary method — Backup restoration**

Restore the affected VM from a verified backup and validate service functionality.

**Alternative method — Documented rebuilding**

Recreate the VM using approved specifications, reinstall RHEL, restore required configuration and validate the service.

### 9.3 Recovery documentation requirements

Future implementation and recovery tasks must provide:

- VM build guides.
- Operating system configuration records.
- Network configuration records.
- Firewall and administrative access configuration.
- Backup configuration and schedules.
- Restore procedures.
- Rebuild procedures.
- Post-recovery validation checklists.
- Timed recovery test evidence.

### 9.4 Recovery targets

Northwind's provisional recovery objective for FW01/JUMP01 is two hours.

This target has not yet been demonstrated.

Recovery validation must establish whether the agreed objectives can be achieved using the available infrastructure and documented procedures.

### 9.5 Recovery limitations

- Both VMs depend on the single HYP01 hypervisor.
- No redundant standby VMs are included in the initial design.
- Backup infrastructure and independent recovery storage are not yet fully implemented.
- Backup availability does not guarantee successful restoration.
- Rebuilding requires access to installation media, configuration records and necessary credentials.
- Hypervisor failure may require additional recovery infrastructure.

---

## 10. Approved VM Specification Records

### 10.1 FW01 — Firewall / Router

| Field | Approved specification |
|---|---|
| Asset name | FW01 |
| Proxmox VM ID | 101 |
| Hostname | `fw01` |
| Role | Firewall, router, NAT and inter-VLAN gateway |
| Hypervisor | HYP01 |
| Guest OS | RHEL 10 Minimal |
| CPU sockets | 1 |
| CPU cores | 2 |
| Total vCPUs | 2 |
| CPU model | `host` |
| RAM | 4 GiB |
| OS disk | 40 GiB |
| Disk identifier | `scsi0` |
| Storage backend | `local-lvm` |
| Provisioning | Thin |
| SCSI controller | VirtIO SCSI single |
| Discard | Enabled |
| I/O thread | Enabled |
| SSD emulation | Enabled |
| Firmware | OVMF (UEFI) |
| Machine type | q35 |
| EFI disk | Required |
| Secure Boot | Enabled, with pre-enrolled keys |
| Network adapter 1 | VirtIO, `net0`, `vmbr0`, external |
| Network adapter 2 | VirtIO, `net1`, `vmbr1`, VLAN trunk |
| Internal VLANs | 100, 200, 300, 400 |
| IP addressing | Pending NIM-19 |
| Recovery method | Backup restore or documented rebuild |
| Deployment status | Not deployed |

### 10.2 JUMP01 — Administrative Jump Server

| Field | Approved specification |
|---|---|
| Asset name | JUMP01 |
| Proxmox VM ID | 201 |
| Hostname | `jump01` |
| Role | Secure administrative access |
| Hypervisor | HYP01 |
| Guest OS | RHEL 10 Minimal |
| CPU sockets | 1 |
| CPU cores | 2 |
| Total vCPUs | 2 |
| CPU model | `host` |
| RAM | 2 GiB |
| OS disk | 30 GiB |
| Disk identifier | `scsi0` |
| Storage backend | `local-lvm` |
| Provisioning | Thin |
| SCSI controller | VirtIO SCSI single |
| Discard | Enabled |
| I/O thread | Enabled |
| SSD emulation | Enabled |
| Firmware | OVMF (UEFI) |
| Machine type | q35 |
| EFI disk | Required |
| Secure Boot | Enabled, with pre-enrolled keys |
| Network adapter | VirtIO, `net0`, `vmbr1`, VLAN 100 |
| Network zone | Management |
| IP addressing | Pending NIM-19 |
| Recovery method | Backup restore or documented rebuild |
| Deployment status | Not deployed |

---

## 11. VM Resource Allocation Register

| VM ID | Hostname | Role | vCPU | RAM | OS disk | Storage | Status |
|---|---|---|---|---|---|---|---|
| 101 | FW01 | Firewall / Router | 2 | 4 GiB | 40 GiB | `local-lvm` | Planned |
| 201 | JUMP01 | Jump Server | 2 | 2 GiB | 30 GiB | `local-lvm` | Planned |
| **Total** | | | **4** | **6 GiB** | **70 GiB** | | |

These records must be reflected in the Northwind asset register.

---

## 12. Implementation Dependencies

| Dependency | Description | Responsible task |
|---|---|---|
| DEP-01 | Finalise IP addressing and interface schedule | NIM-19 |
| DEP-02 | Create isolated, VLAN-aware `vmbr1` | NIM-21 |
| DEP-03 | Deploy FW01 using approved specifications | NIM-21 |
| DEP-04 | Configure FW01 VLAN interfaces and gateways | NIM-22 |
| DEP-05 | Deploy JUMP01 using approved specifications | JUMP01 deployment task |
| DEP-06 | Validate guest firmware, CPU and Secure Boot compatibility | VM deployment tasks |
| DEP-07 | Implement configuration protection and backup | Subsequent recovery tasks |
| DEP-08 | Test restoration and documented rebuilding | Subsequent recovery tasks |

The absence of `vmbr1` is an acknowledged implementation dependency from NIM-17, not an unresolved NIM-18 design question.

---

## 13. Risks and Mitigations

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R01 | Thin-pool exhaustion | Guest write failures and service disruption | Monitor data and metadata utilisation; maintain capacity headroom |
| R02 | CPU-model portability limitations | Difficulties moving VMs to different hardware | Reassess CPU compatibility before introducing additional hypervisors |
| R03 | Secure Boot incompatibility | Guest fails to boot | Validate signed boot components and document recovery procedures |
| R04 | Incorrect bridge or VLAN assignment | Network exposure or connectivity failure | Validate NIC mapping, bridge isolation and VLAN behaviour |
| R05 | FW01 failure | Loss of internal routing and external connectivity | Maintain tested backups and documented rebuild procedures |
| R06 | JUMP01 failure | Administrative access disruption | Maintain recovery procedures and approved emergency access |
| R07 | HYP01 failure | Multiple infrastructure services unavailable | Maintain independent recovery documentation and backup copies |
| R08 | Unverified resource assumptions | Performance or capacity issues | Monitor actual utilisation after deployment and adjust as necessary |

---

## 14. Validation and Acceptance Criteria

### 14.1 Jira acceptance checklist

- [x] Unique VM identifiers and hostnames assigned.
- [x] CPU and memory allocations documented.
- [x] OS disk sizes and storage locations confirmed.
- [x] RHEL 10 guest configuration and firmware settings selected.
- [x] Virtual NIC models and bridge assignments documented.
- [x] FW01 external and internal NIC requirements defined.
- [x] JUMP01 specified with one NIC on MGMT VLAN 100.
- [x] Resource allocations assessed against NIM-17 HYP01 capacity.
- [ ] Approved specifications entered into the asset register.
- [ ] Build documentation updated or cross-referenced.
- [ ] Final documentation reviewed and committed to GitHub.

The configuration decisions are complete. Actual guest compatibility and operational behaviour will be verified during deployment.

### 14.2 Future deployment validation

The following checks are deferred to implementation:

- Confirm the VMs boot successfully.
- Confirm RHEL detects the expected CPU and memory resources.
- Confirm virtual disk sizes and controller configuration.
- Confirm UEFI and Secure Boot operation.
- Confirm the intended network interfaces are present.
- Confirm VLAN connectivity and isolation.
- Confirm persistent network configuration after reboot.
- Confirm resource utilisation is within acceptable limits.
- Confirm backup and recovery procedures when implemented.

---

## 15. Evidence and Deliverables

| Deliverable | Status |
|---|---|
| FW01 VM specification | Complete — Section 10.1 |
| JUMP01 VM specification | Complete — Section 10.2 |
| VM ID and resource allocation register | Complete — Sections 4 and 11 |
| Virtual NIC and bridge mapping | Complete — Section 8 |
| Storage allocation record | Complete — Sections 6 and 11 |
| HYP01 capacity assessment | Complete — Section 9 |
| Design decision register | Complete — Section 3 |

**Evidence basis:** NIM-17 HYP01 readiness verification and NIM-18 approved design discussions.

No VM deployment or configuration changes were performed as part of NIM-18.

---

## 16. Findings and Lessons Learned

### 16.1 Findings

- HYP01 has sufficient verified capacity for the proposed initial VMs.
- FW01 requires a tagged internal VLAN trunk, whereas JUMP01 requires only a VLAN 100 access connection.
- Secure Boot introduces additional boot validation and recovery responsibilities.
- The `host` CPU model is suitable for the current single-host design but introduces portability considerations.
- Thin provisioning improves storage efficiency but requires active capacity monitoring.
- VM specifications alone are insufficient for disaster recovery; complete build records, protected configuration and tested backups are also required.

### 16.2 RHCSA learning relevance

This task establishes the virtual hardware foundation for future RHCSA-aligned work involving:

- Linux block devices and storage.
- Partitioning, filesystems and LVM.
- NetworkManager and VLAN interfaces.
- Linux routing and firewalling.
- UEFI and boot troubleshooting.
- System resource monitoring.
- Service administration and recovery.

### 16.3 Key lesson

Infrastructure sizing should be based on expected workload, operational requirements and observed performance rather than allocating resources simply because they are available.

Likewise, an approved design is not proof of successful implementation or recovery.

---

## 17. Final Assessment

**Design outcome:** Approved.

FW01 and JUMP01 now have defined virtual machine specifications covering identity, compute, storage, firmware, networking, security and recovery considerations.

The proposed allocations fit within HYP01's verified capacity and align with Northwind's existing infrastructure architecture.

**NIM-18 is ready for final documentation review and administrative closure once the asset register and associated build documentation have been updated.**

### Next steps

1. Update the Northwind asset register with both VM specification records.
2. Commit the approved NIM-18 documentation.
3. Review and satisfy the remaining Jira acceptance criteria.
4. Merge the NIM-18 feature branch into `main`.
5. Close NIM-18 in Jira.
6. Proceed to NIM-19 — Finalise Network Addressing and Interface Schedule.