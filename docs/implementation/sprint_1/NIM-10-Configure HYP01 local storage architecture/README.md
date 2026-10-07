# NIM-10 — Configure HYP01 Local Storage Architecture

## Overview

This task configured the local storage architecture for **HYP01**, establishing dedicated storage for Northwind virtual machine and container workloads.

HYP01 contains three approximately 2 TB NVMe SSDs with distinct roles:

- Windows workstation storage
- Proxmox system and VM OS storage
- Dedicated Northwind workload storage

The dedicated workload NVMe device was configured using **LVM Thin Provisioning** and subsequently registered with Proxmox as `northwind-workload`.

---

## Target Component

**HYP01 — Northwind v1 Virtualisation Host**

---

## Objectives

- Confirm the physical role of each HYP01 NVMe device.
- Positively identify the dedicated Northwind workload disk.
- Preserve the Windows and Proxmox system disks.
- Configure the workload disk using LVM.
- Create a dedicated LVM Thin Pool for Northwind workloads.
- Retain free capacity within the Volume Group.
- Register the thin pool with Proxmox.
- Validate the completed storage architecture.

---

## Storage Roles

The agreed HYP01 storage model is:

| Physical Storage | Role |
|---|---|
| NVMe 0 | Windows workstation |
| NVMe 1 | Proxmox system / VM OS |
| NVMe 2 | Northwind workload data |

The runtime Linux device names were not treated as authoritative because NVMe enumeration was observed to change between boots.

Physical disk serial numbers were therefore used to confirm disk identity before performing storage configuration.

> Full hardware serial numbers are retained in internal evidence and should not be published in the public repository.

---

## Storage Architecture

The dedicated Northwind workload disk was configured using the following LVM hierarchy:

```text
Physical NVMe SSD
        │
        ▼
LVM Physical Volume (PV)
        │
        ▼
Volume Group
vg_northwind
        │
        ▼
Thin Pool
tp_workload
        │
        ▼
Future VM / Container Thin Volumes
```

The thin pool was then exposed to Proxmox as:

```text
Storage ID: northwind-workload
```

---

## Technology Selection

**LVM Thin Provisioning** was selected for the Northwind workload storage.

This provides:

- Proxmox-native block storage
- Thin provisioning
- Efficient VM disk allocation
- Snapshot and cloning capability
- Direct exposure to Linux LVM administration concepts
- Strong alignment with RHCSA learning objectives

Alternative storage technologies such as ZFS were not required for the current single-host, single-workload-disk architecture and would introduce additional complexity without providing meaningful redundancy.

---

## Physical Volume

The dedicated workload NVMe device was initialised as an LVM **Physical Volume (PV)**.

The PV makes the physical storage device available for management by LVM.

Conceptually:

```text
Physical disk
     ↓
LVM-managed storage
```

The workload disk was verified using its persistent hardware identity before this destructive operation was performed.

---

## Volume Group

The Physical Volume was assigned to the Volume Group:

```text
vg_northwind
```

The Volume Group provides the storage pool from which Logical Volumes can be allocated.

An earlier naming attempt produced an unnecessarily duplicated storage name and was removed before the final configuration was created.

The final naming convention was deliberately kept simple:

```text
VG:        vg_northwind
Thin Pool: tp_workload
Proxmox:   northwind-workload
```

---

## Thin Pool

The Northwind thin pool was created as:

```text
tp_workload
```

with approximately:

```text
Data capacity:     1.75 TiB
Metadata:          4 GiB
Chunk size:        64 KiB
Metadata spare:    Enabled
```

The pool contains internal LVM components responsible for storing workload data and thin-provisioning metadata.

These include:

```text
tp_workload_tdata
tp_workload_tmeta
pmspare
```

### `_tdata`

Stores the physical data blocks allocated to thin volumes.

### `_tmeta`

Stores metadata describing which thin-volume logical blocks map to physical chunks within `_tdata`.

### `pmspare`

Provides spare metadata capacity that LVM can use during metadata repair or maintenance operations.

---

## Capacity Reserve

The entire Volume Group was deliberately **not** allocated to the thin pool.

Approximately:

```text
107 GiB
```

was retained as free Volume Group capacity.

This represents roughly 5% of the workload disk.

The reserve provides flexibility for future storage maintenance, recovery or architecture changes rather than committing all available physical capacity immediately.

---

## Thin Provisioning

Thin provisioning allows virtual disks to have a logical capacity without immediately consuming the same amount of physical storage.

For example, a future VM may receive:

```text
100 GiB virtual disk
```

without immediately consuming 100 GiB from the physical thin pool.

Physical capacity is allocated in chunks as the VM writes data.

This improves storage utilisation but introduces an operational requirement to monitor thin-pool consumption carefully.

Running a thin pool out of physical capacity can affect multiple workloads using that pool.

---

## Proxmox Integration

The completed thin pool was registered with Proxmox using:

```text
Storage ID: northwind-workload
Volume Group: vg_northwind
Thin Pool: tp_workload
Node: hyp01
```

The following content types were enabled:

```text
Disk image
Container
```

This allows future Northwind VM and container disks to be provisioned directly onto the dedicated workload storage.

---

## Proxmox Storage Configuration

The resulting storage configuration included:

```text
lvmthin: northwind-workload
        thinpool tp_workload
        vgname vg_northwind
        content images,rootdir
        nodes hyp01
```

This configuration maps the existing LVM Thin Pool into Proxmox.

Proxmox registration does not create the underlying LVM architecture; it tells Proxmox how to use the storage that has already been created.

---

## Validation

The completed architecture was validated using Linux LVM and Proxmox storage tools.

The following were confirmed:

- [x] Correct physical workload disk identified.
- [x] Windows disk remained untouched.
- [x] Proxmox system disk remained untouched.
- [x] Workload disk configured as an LVM Physical Volume.
- [x] `vg_northwind` created successfully.
- [x] `tp_workload` created successfully.
- [x] Thin-pool metadata created.
- [x] Metadata spare available.
- [x] Approximately 5% VG capacity retained as free space.
- [x] Thin pool healthy and initially empty.
- [x] `northwind-workload` registered with Proxmox.
- [x] VM disk and container content types enabled.
- [x] Proxmox recognised the storage backend.

---

## Findings / Lessons Learned

### Device names are not physical identities

NVMe device names changed between boots during implementation.

Names such as:

```text
/dev/nvme1n1
/dev/nvme2n1
```

therefore cannot safely be assumed to represent the same physical SSD indefinitely.

Persistent identifiers such as serial numbers or `/dev/disk/by-id/` should be preferred when physical identity matters.

---

### LVM separates physical storage from logical allocation

The task demonstrated the LVM hierarchy:

```text
Physical Disk
    ↓
Physical Volume
    ↓
Volume Group
    ↓
Logical Volume / Thin Pool
    ↓
Thin Volumes
```

This abstraction allows storage capacity to be managed independently of the underlying physical device layout.

---

### Thin provisioning is allocation, not additional capacity

Thin provisioning allows more efficient use of physical storage by allocating physical chunks as data is written.

It does not create additional physical storage.

Pool utilisation therefore needs to be monitored as Northwind workloads are introduced.

---

### Leave operational flexibility

Allocating 100% of the Volume Group to the thin pool would leave no unallocated capacity for future LVM operations.

Retaining approximately 5% of the VG provides operational flexibility.

---

### Storage naming matters

An initial configuration produced an unnecessarily duplicated storage name.

The configuration was removed and recreated using a clearer naming convention.

Correcting naming during initial implementation is preferable to carrying confusing identifiers into production operations.

---

## Result

**PASS**

The dedicated Northwind workload NVMe device was successfully configured using LVM Thin Provisioning.

The final architecture consists of:

```text
Workload NVMe
      ↓
LVM PV
      ↓
vg_northwind
      ↓
tp_workload
      ↓
northwind-workload
      ↓
Future Northwind VM / container disks
```

The storage backend was successfully registered with Proxmox and was ready for use by Northwind virtual workloads.

---

## Next Task

Proceed with the remaining HYP01 platform configuration and validation activities before deploying the first Northwind production workloads.

---

## Related Documentation

- NIM-7 — Pre-installation Host Readiness
- NIM-9 — Install Proxmox VE on HYP01
- Northwind Low-Level Design
- HYP01 hardware inventory
- Sprint 1 implementation records
