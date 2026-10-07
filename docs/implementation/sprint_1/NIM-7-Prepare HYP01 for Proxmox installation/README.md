# NIM-7 — Pre-installation Host Readiness

## Overview

This task validated that the physical host designated as **HYP01** was ready for the installation of Proxmox VE.

The objective was to identify the physical hardware and storage devices, verify virtualisation support, establish recovery options, and reduce the risk of accidentally overwriting the existing Windows installation during the Proxmox deployment.

This task was completed before any changes were made to the host storage configuration.

---

## Target Component

**HYP01 — Northwind v1 Virtualisation Host**

---

## Objectives

- Record the physical hardware configuration of HYP01.
- Identify all installed NVMe storage devices.
- Determine which physical disk contained the existing Windows installation.
- Establish persistent disk identifiers suitable for use during the Proxmox installation.
- Verify that hardware virtualisation support was enabled.
- Establish recovery options before modifying the system.
- Confirm that no known blocker prevented the Proxmox installation.

---

## Hardware Baseline

The following hardware was recorded during the readiness assessment:

| Component | Configuration |
|---|---|
| Manufacturer / Platform | Lenovo Legion T7 34IA7Z |
| CPU | Intel Core i9-12900KF |
| CPU Cores | 16 |
| Logical CPUs | 24 |
| Memory | 64 GB |
| Firmware | UEFI |
| Storage | 3 × Samsung NVMe SSDs (~2 TB each) |
| Existing OS | Windows 11 |

Hardware information was captured from Windows prior to installation.

---

## Storage Identification

HYP01 contains three NVMe SSDs of the same model and approximately the same capacity.

Because the devices are visually and technically similar, disk number or enumeration order was not considered a sufficiently reliable identifier for destructive installation operations.

Persistent physical serial numbers were therefore recorded and used to distinguish the devices.

The initial Windows layout identified:

- One NVMe device containing the Windows operating system, EFI partition and Windows recovery environment.
- Two additional NVMe devices available for the future Northwind platform.

The physical disk serial numbers were cross-referenced between Windows and the later Linux/Proxmox environment.

> **Important:** Full hardware serial numbers are retained only in internal project evidence and should be redacted from public documentation.

---

## Disk Identification Principle

A key implementation decision resulting from this task was:

> **Physical storage devices must be identified using persistent hardware identifiers rather than relying solely on runtime device names or disk numbering.**

This became particularly important later in the implementation when Linux NVMe device names changed between boots.

Examples of potentially non-persistent identifiers include:

```text
Disk 0
Disk 1
Disk 2

/dev/nvme0n1
/dev/nvme1n1
/dev/nvme2n1
