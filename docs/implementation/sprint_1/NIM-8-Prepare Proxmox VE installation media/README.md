# NIM-8 — Prepare Proxmox VE Installation Media

## Overview

This task prepared and validated the installation media required to deploy **Proxmox VE** onto **HYP01**, the Northwind v1 virtualisation host.

The objective was to obtain trusted Proxmox installation media, verify its integrity, create bootable USB media, and confirm that HYP01 could successfully boot the Proxmox installer.

During validation, the initial Proxmox VE 9.2 installation media encountered a boot issue on HYP01. Troubleshooting was performed and Proxmox VE 9.1 installation media was subsequently used successfully with the `nomodeset` kernel parameter.

---

## Target Component

**HYP01 — Northwind v1 Virtualisation Host**

---

## Objectives

- Obtain Proxmox VE installation media from an official source.
- Verify the integrity of the downloaded ISO.
- Create bootable installation media.
- Confirm HYP01 can boot from the installation media.
- Identify and investigate any installer compatibility issues.
- Establish a working installation path before modifying HYP01 storage.

---

## Initial Installation Media

The initial installation media selected was:

**Proxmox VE 9.2-1 x86_64**

The ISO was obtained from the official Proxmox distribution source.

Before creating the bootable USB, the downloaded ISO checksum was compared with the vendor-published checksum.

The checksum matched successfully.

This confirmed that the downloaded installation image matched the expected vendor-provided image and had not been corrupted during download.

---

## Bootable USB Creation

The verified Proxmox ISO was written to a USB device using **balenaEtcher**.

The USB device was then used to boot HYP01 through the system's boot menu.

HYP01 successfully detected and attempted to boot the Proxmox installation environment.

---

## Initial Proxmox VE 9.2 Boot Issue

The Proxmox VE 9.2 installer did not successfully reach the graphical installation environment on HYP01.

During startup, the installer displayed messages including:

```text
modprobe: FATAL: Module shpchp not found in directory /lib/modules/7.0.2-6-pve
```

The boot process subsequently stalled.

HYP01 contains an NVIDIA RTX-series graphics adapter, so graphics/kernel compatibility was considered during troubleshooting.

---

## Troubleshooting

### Test — `nomodeset`

The Linux kernel parameter:

```text
nomodeset
```

was added to the installer boot parameters.

`nomodeset` prevents the Linux kernel from enabling normal kernel mode-setting for the graphics hardware during early boot.

This can be useful when the installer encounters compatibility problems while initialising a graphics adapter.

Using `nomodeset` allowed the Proxmox VE 9.2 installer to progress further than before.

However, the installer subsequently stalled at approximately:

```text
Waiting for /dev to be fully populated
```

Additional output included:

```text
nvidiafb: No such device
```

The workaround therefore improved the boot process but did not result in a usable Proxmox VE 9.2 installer.

---

## Alternative Installation Path

Rather than repeatedly modifying the same installer environment without a clear diagnosis, an alternative supported Proxmox installation image was tested.

The official Proxmox archive was used to obtain:

**Proxmox VE 9.1-1 x86_64**

The installation USB was rewritten with the Proxmox VE 9.1 image.

HYP01 was then booted using the new installation media.

The `nomodeset` kernel parameter was again applied.

---

## Successful Installer Boot

The combination of:

```text
Proxmox VE 9.1-1
+
nomodeset
```

successfully reached the graphical Proxmox installation environment.

This established a working installation path for HYP01.

At this stage no Proxmox installation was performed; the objective of NIM-8 was to prepare and validate usable installation media.

Actual installation was deferred to the subsequent implementation task.

---

## Installer Console Validation

During installer testing, a Linux console was accessed using:

```text
Ctrl + Alt + F3
```

This allowed the available storage devices to be inspected independently of the graphical installer.

This was particularly important because HYP01 contains three similar NVMe SSDs and the Windows installation had to be protected from accidental overwrite.

The console provided an additional method of validating disk identity before progressing with installation.

The graphical installer could be returned to using:

```text
Ctrl + Alt + F1
```

---

## Key Risk — Storage Selection

HYP01 contains multiple NVMe devices of the same model and approximately the same capacity.

Selecting an installation target based only on:

```text
/dev/nvme0n1
/dev/nvme1n1
/dev/nvme2n1
```

would therefore create unnecessary risk.

The persistent disk-identification information gathered during **NIM-7** remained authoritative.

Before any destructive installation action, the target disk would be cross-referenced against its physical hardware serial number.

---

## Validation

The following checks were completed:

- [x] Official Proxmox VE installation media obtained.
- [x] Downloaded installation image integrity verified.
- [x] Bootable USB installation media created.
- [x] HYP01 successfully booted from USB.
- [x] Proxmox VE 9.2 installer behaviour tested.
- [x] `nomodeset` workaround tested.
- [x] Proxmox VE 9.2 installer issue documented.
- [x] Alternative Proxmox VE 9.1 installation media obtained.
- [x] Proxmox VE 9.1 boot media created.
- [x] Proxmox VE 9.1 installer successfully reached using `nomodeset`.
- [x] Installer console access validated.
- [x] No changes made to the existing Windows installation during media testing.
- [x] Working installation path established.

---

## Findings / Lessons Learned

### Installation media should be verified

Vendor-provided checksums should be used to validate downloaded installation media before it is trusted for infrastructure deployment.

A successful checksum comparison provides evidence that the downloaded file matches the vendor-published image.

---

### Installer compatibility can differ between releases

Proxmox VE 9.2 installation media did not successfully initialise on HYP01 during this task, while Proxmox VE 9.1 with `nomodeset` successfully reached the graphical installer.

This demonstrates that installation-media compatibility and the compatibility of the subsequently installed/upgraded operating system are not necessarily identical.

---

### Change one variable at a time

Troubleshooting was performed incrementally.

The use of `nomodeset` demonstrated that changing a single boot parameter altered installer behaviour, even though it did not completely resolve the Proxmox VE 9.2 boot problem.

Controlled testing makes troubleshooting results more meaningful than changing several parameters simultaneously.

---

### Installer consoles are valuable troubleshooting tools

The graphical installer is not the only source of information during installation.

Access to a Linux console allowed storage devices and other system information to be inspected before performing destructive operations.

---

### Runtime disk names are not authoritative identifiers

Linux device names such as:

```text
/dev/nvme0n1
```

are useful for interacting with the current running system but should not be assumed to permanently identify a particular physical device.

Persistent hardware identifiers remain preferable when identifying destructive installation targets.

---

## Deviation / Issue

### Proxmox VE 9.2 Installer Compatibility

**Observed behaviour:**  
Proxmox VE 9.2-1 installation media failed to reach a usable installation environment on HYP01.

**Troubleshooting performed:**

- Installer boot behaviour observed.
- `nomodeset` tested.
- Installer progressed further but subsequently stalled.
- Proxmox VE 9.1 installation media tested as an alternative.

**Workaround:**

Proxmox VE 9.1-1 with `nomodeset` successfully reached the graphical installer.

**Status:**

Working installation path established.

The exact root cause of the Proxmox VE 9.2 installer failure was not conclusively established during this task.

---

## Result

**PASS**

Usable Proxmox VE installation media was successfully prepared and validated.

Although the initial Proxmox VE 9.2 installation media encountered a boot compatibility issue on HYP01, troubleshooting established a successful installation path using **Proxmox VE 9.1-1 with `nomodeset`**.

HYP01 was therefore ready to proceed to the controlled Proxmox installation task.

---

## Next Task

**NIM-9 — Install Proxmox VE on HYP01**

NIM-9 performs the actual bare-metal Proxmox deployment while preserving the existing Windows installation and validating initial host operation.

---

## Related Documentation

- NIM-7 — Pre-installation Host Readiness
- HYP01 hardware inventory
- Northwind Low-Level Design
- Sprint 1 implementation records
