# NIM-9 — Install Proxmox VE on HYP01

## Overview

This task installed **Proxmox VE** onto **HYP01**, establishing the physical host as the virtualisation platform for the Northwind v1 environment.

The installation was performed using the Proxmox VE 9.1 installation media validated during NIM-8. Particular care was taken to positively identify the intended installation disk and preserve the existing Windows installation.

Following installation, initial management connectivity and access to the Proxmox web interface were validated.

---

## Target Component

**HYP01 — Northwind v1 Virtualisation Host**

---

## Objectives

- Install Proxmox VE onto the designated HYP01 system disk.
- Protect the existing Windows installation from modification.
- Configure initial HYP01 system identity.
- Establish temporary/bootstrap management connectivity.
- Confirm access to the Proxmox management interface.
- Verify that Windows remained bootable following installation.

---

## Installation Media

The installation used:

**Proxmox VE 9.1-1 x86_64**

The installer was booted using the `nomodeset` kernel parameter established during NIM-8.

---

## Installation Target Validation

HYP01 contains three similar Samsung NVMe SSDs.

The target disk was therefore identified using its persistent hardware serial number rather than relying solely on Linux device names such as:

```text
/dev/nvme0n1
/dev/nvme1n1
/dev/nvme2n1
```

The installer console was accessed using:

```text
Ctrl + Alt + F3
```

and the available disks were inspected before proceeding.

The designated Proxmox system disk was positively identified and selected.

The NVMe device containing Windows was not selected or modified.

---

## Filesystem Selection

**XFS** was selected for the Proxmox installation filesystem.

The Proxmox installer created the required system storage configuration on the designated HYP01 system disk.

Detailed workload storage configuration was intentionally deferred to NIM-10.

---

## Initial System Identity

The host was configured as:

```text
Hostname: hyp01
FQDN:     hyp01.northwind.internal
```

This establishes HYP01 as the first Northwind virtualisation host.

---

## Bootstrap Management Network

The original design proposed a dedicated Northwind management network.

However, that network did not yet exist at this stage of implementation.

HYP01 was therefore temporarily connected to the existing home/bootstrap network:

```text
Network:     192.168.178.0/24
HYP01:       192.168.178.10/24
Gateway:     192.168.178.1
DNS:         192.168.178.1
```

The home router DHCP pool was:

```text
192.168.178.20 - 192.168.178.200
```

The static HYP01 address `192.168.178.10` therefore sits outside the DHCP allocation range.

This configuration is a **bootstrap dependency**, not the final Northwind management network.

The final management addressing will be introduced when the relevant Northwind network design is implemented.

---

## Initial Connectivity Issue

Following installation, the Proxmox web interface was initially unreachable.

Host networking was inspected and the physical Ethernet interface reported:

```text
NO-CARRIER
```

The issue was traced to the Ethernet cable not being connected.

After connecting the cable:

- Physical link became available.
- HYP01 networking became operational.
- The management IP became reachable.
- The Proxmox web interface became accessible.

No Proxmox network reconfiguration was required to resolve the issue.

---

## Proxmox Management Interface

Following restoration of physical network connectivity, the Proxmox management interface was successfully accessed at:

```text
https://192.168.178.10:8006
```

This confirmed that HYP01 was operational and remotely manageable over the bootstrap network.

---

## Dual-Boot Validation

HYP01 also serves as the user's Windows workstation when the Northwind lab is not in use.

Following the Proxmox installation, Windows was booted and confirmed operational.

The system's boot configuration was subsequently arranged so that:

- Windows remains the default boot target.
- Proxmox can be selected manually when a Northwind lab session is required.

This preserves the agreed dual-boot operating model for HYP01.

---

## Validation

The following checks were completed:

- [x] Proxmox VE installer successfully started.
- [x] Intended installation disk positively identified.
- [x] Windows system disk protected from modification.
- [x] Proxmox VE installed successfully.
- [x] HYP01 system identity configured.
- [x] Bootstrap management IP configured.
- [x] Default gateway configured.
- [x] Initial DNS configured.
- [x] Physical Ethernet connectivity validated.
- [x] Proxmox management interface accessible.
- [x] Windows remained bootable following installation.
- [x] Dual-boot operating model validated.

---

## Findings / Lessons Learned

### Verify destructive targets using persistent identifiers

When several disks have the same model and capacity, runtime device names alone are insufficient for confidently identifying a destructive installation target.

The serial-number inventory established during NIM-7 provided the authoritative cross-reference.

---

### Separate bootstrap configuration from final design

The `192.168.178.10/24` management address allows HYP01 to be administered while the Northwind infrastructure is being constructed.

It should not be confused with the final Northwind management-zone configuration.

Temporary dependencies should be documented so that they can later be removed or replaced.

---

### Start troubleshooting at the lowest relevant layer

The initial management-connectivity problem was caused by the absence of physical Ethernet link rather than an incorrect Proxmox configuration.

The `NO-CARRIER` state provided an important clue.

Before changing IP addressing, routing or services, verify that the underlying interface has physical connectivity.

---

### Installation should not include unnecessary configuration

NIM-9 established the hypervisor and bootstrap connectivity only.

Workload storage, platform baselining and VM validation were handled through separate implementation tasks so that each change could be independently understood and validated.

---

## Result

**PASS**

Proxmox VE was successfully installed onto the designated HYP01 system disk while preserving the existing Windows installation.

HYP01 was configured with its initial Northwind identity and bootstrap management connectivity, the Proxmox management interface was successfully accessed, and the agreed Windows/Proxmox dual-boot model was validated.

HYP01 was ready to proceed with configuration of the Northwind workload storage architecture.

---

## Next Task

**NIM-10 — Configure Local Storage on HYP01**

NIM-10 configures the dedicated Northwind workload NVMe device using LVM Thin Provisioning and integrates the resulting storage backend with Proxmox.

---

## Related Documentation

- NIM-7 — Pre-installation Host Readiness
- NIM-8 — Prepare Proxmox VE Installation Media
- Northwind Low-Level Design
- HYP01 hardware inventory
- Sprint 1 implementation records
