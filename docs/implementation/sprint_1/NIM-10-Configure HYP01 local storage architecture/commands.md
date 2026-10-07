# NIM-10 — Command Reference

Commands encountered while configuring and validating the HYP01 local storage architecture.

---

## `lsblk`

```bash
lsblk
```

**Purpose:**  
Lists block devices, partitions and their relationships.

**Why we used it:**  
To inspect HYP01's disks and confirm the current storage layout before modifying the dedicated workload device.

---

## `pvs`

```bash
pvs
```

**Purpose:**  
Displays LVM Physical Volumes and the Volume Groups to which they belong.

**Why we used it:**  
To verify that the workload NVMe device had been successfully initialised as an LVM Physical Volume and assigned to the correct Volume Group.

---

## `vgs`

```bash
vgs
```

**Purpose:**  
Displays LVM Volume Groups, including their total and available capacity.

**Why we used it:**  
To validate `vg_northwind` and confirm that approximately 5% of its capacity remained unallocated.

---

## `lvs`

```bash
lvs
```

**Purpose:**  
Displays LVM Logical Volumes and thin-pool information.

**Why we used it:**  
To verify `tp_workload`, its data and metadata utilisation, and the presence of the thin-pool metadata spare.

---

## `pvcreate`

```bash
pvcreate <device>
```

**Purpose:**  
Initialises a block device as an LVM Physical Volume.

**Why we used it:**  
To make the dedicated Northwind workload NVMe device available for management by LVM.

> This is a destructive storage operation and the target device must be positively identified before execution.

---

## `vgcreate`

```bash
vgcreate vg_northwind <physical-volume>
```

**Purpose:**  
Creates an LVM Volume Group using one or more Physical Volumes.

**Why we used it:**  
To create `vg_northwind` as the managed storage pool for Northwind workload storage.

---

## `lvcreate`

```bash
lvcreate --type thin-pool \
  -n tp_workload \
  -L 1.75T \
  --poolmetadatasize 4G \
  --poolmetadataspare y \
  -c 64K \
  vg_northwind
```

**Purpose:**  
Creates an LVM Logical Volume. In this case, it creates an LVM Thin Pool.

**Key options:**

- `--type thin-pool` — create a thin-provisioning pool.
- `-n tp_workload` — name the thin pool.
- `-L 1.75T` — allocate 1.75 TiB to the pool.
- `--poolmetadatasize 4G` — allocate 4 GiB for thin metadata.
- `--poolmetadataspare y` — create spare metadata capacity.
- `-c 64K` — use a 64 KiB thin-pool chunk size.
- `vg_northwind` — create the pool within this Volume Group.

**Why we used it:**  
To create the storage pool from which Proxmox can thin-provision future Northwind VM and container disks.

---

## `lvremove`

```bash
lvremove <logical-volume>
```

**Purpose:**  
Removes an LVM Logical Volume.

**Why we used it:**  
During the initial storage configuration, an incorrectly named/created storage object was removed before creating the final clean architecture.

> Removing an LV can permanently destroy data and must be performed only after verifying the target.

---

## `vgremove`

```bash
vgremove <volume-group>
```

**Purpose:**  
Removes an LVM Volume Group.

**Why we used it:**  
As part of cleaning up the initial storage configuration before recreating it with the final `vg_northwind` naming convention.

---

## `wipefs`

```bash
wipefs <device>
```

**Purpose:**  
Examines or removes filesystem and storage signatures from a block device.

**Why we used it:**  
During storage cleanup to remove stale signatures before recreating the intended LVM configuration.

> `wipefs` can make existing storage structures inaccessible. The target must be verified carefully before removing signatures.

---

## `blkid`

```bash
blkid
```

**Purpose:**  
Displays block-device attributes such as UUIDs and filesystem/storage signatures.

**Why we used it:**  
To inspect storage devices and help verify existing signatures and identifiers during configuration and cleanup.

---

## `/dev/disk/by-id/`

```bash
ls -l /dev/disk/by-id/
```

**Purpose:**  
Displays persistent device links based on hardware identifiers rather than runtime enumeration order.

**Why we used it:**  
To help identify the physical workload NVMe reliably after observing that `/dev/nvmeXn1` numbering could change between boots.

---

## `cat /etc/pve/storage.cfg`

```bash
cat /etc/pve/storage.cfg
```

**Purpose:**  
Displays the Proxmox storage configuration file.

**Why we used it:**  
To verify that `northwind-workload` had been correctly registered with Proxmox and mapped to `vg_northwind/tp_workload`.

---

## `reboot`

```bash
reboot
```

**Purpose:**  
Restarts the Linux system.

**Why we used it:**  
To test storage persistence across HYP01 reboots and observe the behaviour of NVMe runtime device enumeration.
