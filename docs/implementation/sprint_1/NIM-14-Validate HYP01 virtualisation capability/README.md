# NIM-14 — Validate HYP01 Virtualisation Capability

## Overview

This task validated that **HYP01** could function as the virtualisation platform for the Northwind v1 environment.

A temporary RHEL 10.2 virtual machine named `TEST01` was provisioned, installed and operated on HYP01. Compute, memory, storage, networking, VM lifecycle operations, platform monitoring and host health were validated.

The temporary workload was removed after testing and its allocated storage was successfully reclaimed.

---

## Target Component

**HYP01 — Northwind v1 Virtualisation Host**

---

## Objectives

- Create a temporary Linux virtual machine.
- Allocate virtual CPU and memory resources.
- Provision the VM disk on `northwind-workload`.
- Attach the VM to the approved bootstrap network.
- Successfully boot and install a Linux guest OS.
- Validate guest compute, memory, storage and networking.
- Test VM start, shutdown and reboot operations.
- Confirm HYP01 can report VM and storage utilisation.
- Confirm HYP01 remains healthy while hosting the workload.
- Remove all temporary VM resources after validation.

---

## Test Workload

The temporary validation VM was configured as:

| Setting | Configuration |
|---|---|
| VM ID | 100 |
| Name | TEST01 |
| Guest OS | RHEL 10.2 x86_64 |
| vCPU | 2 |
| Memory | 2 GiB final |
| Virtual Disk | 20 GiB final |
| Storage | `northwind-workload` |
| Network | `vmbr0` |
| NIC Model | VirtIO |
| Disk Controller | VirtIO SCSI |
| CPU Type | `host` final |

TEST01 was a disposable validation workload and did not form part of the final Northwind server estate.

---

## Installation Media

RHEL 10.2 x86_64 installation media was downloaded and its SHA-256 checksum verified against the vendor-published checksum.

The ISO was uploaded to Proxmox `local` storage.

The initial RHEL Boot ISO required an external package source / Red Hat CDN registration during installation.

The full RHEL 10.2 DVD ISO was therefore subsequently downloaded, verified and uploaded. This provided the required installation packages locally.

This demonstrated the distinction between:

```text
local
    → ISO installation media

northwind-workload
    → VM virtual disks
```

---

## Initial VM Provisioning

TEST01 was initially provisioned with:

```text
2 vCPU
1536 MiB RAM
10 GiB disk
```

The virtual disk was created on:

```text
vg_northwind
    ↓
tp_workload
    ↓
vm-100-disk-0
```

Before the VM wrote data, the thin volume consumed effectively no physical data capacity.

This provided the first practical validation of the LVM Thin architecture created during NIM-10.

---

## RHEL Installer Compatibility Issue

The RHEL 10.2 boot menu loaded successfully.

However, selecting the installation option resulted in an early kernel panic:

```text
Kernel panic - not syncing: Attempted to kill init!
```

The failure was repeatable.

---

## Controlled Troubleshooting

The initial VM configuration used:

```text
1536 MiB RAM
CPU type: x86-64-v2-AES
```

Because memory was initially suspected, RAM was increased to:

```text
2048 MiB
```

with the CPU configuration unchanged.

The same kernel panic occurred.

The CPU model was then changed from:

```text
x86-64-v2-AES
```

to:

```text
host
```

while retaining 2048 MiB RAM.

The RHEL installer subsequently loaded successfully.

The observed test results were therefore:

```text
1536 MiB + x86-64-v2-AES → FAIL
2048 MiB + x86-64-v2-AES → FAIL
2048 MiB + host           → PASS
```

This was recorded as a compatibility finding.

The exact underlying cause of the RHEL installer failure was not conclusively established.

---

## CPU Model Consideration

Using CPU type `host` exposes the physical host's CPU capabilities more directly to the virtual machine.

This provided the required compatibility for TEST01 and was appropriate for validation on the standalone HYP01 host.

However, CPU type selection should be revisited when establishing the standard Northwind VM specification because `host` can reduce portability between virtualisation hosts with different processors.

---

## Virtual Disk Expansion

The initial 10 GiB virtual disk met the minimum requirement investigated for the test but proved unnecessarily restrictive for the selected RHEL installation and filesystem layout.

The existing virtual disk was therefore expanded:

```text
10 GiB
   ↓
+10 GiB
   ↓
20 GiB
```

RHEL subsequently detected the expanded 20 GiB disk successfully.

This additionally validated Proxmox virtual disk expansion.

---

## Guest Storage Validation

Following installation, RHEL detected the virtual disk as approximately:

```text
/dev/sda    20G
```

The resulting guest storage layout included:

```text
sda
├─sda1
├─sda2        /boot
└─sda3
   ├─rhel-root    /
   └─rhel-swap    [SWAP]
```

This demonstrated the separation between host and guest storage layers:

```text
Physical NVMe
      ↓
vg_northwind
      ↓
tp_workload
      ↓
vm-100-disk-0
════════════════ VM boundary
      ↓
/dev/sda
      ↓
Guest partitions
      ↓
Guest LVM
```

The RHEL guest did not need to know how its virtual disk was physically implemented by HYP01.

---

## Guest Compute and Memory Validation

RHEL successfully detected:

```text
2 virtual CPUs
1 socket
2 cores
```

Because CPU type `host` was configured, the guest also identified the underlying processor as:

```text
12th Gen Intel Core i9-12900KF
```

The guest successfully detected the allocated memory and configured approximately 2 GiB of swap.

Compute and memory allocation were therefore validated.

---

## Guest Networking

TEST01 was connected to:

```text
VirtIO NIC
    ↓
vmbr0
    ↓
nic0
    ↓
Bootstrap LAN
```

The guest received the following configuration using DHCP:

```text
IPv4:    192.168.178.97/24
Gateway: 192.168.178.1
DNS:     192.168.178.1
```

The following were successfully validated:

- Guest network interface operational.
- DHCP address allocation.
- Local gateway connectivity.
- External IP connectivity.
- DNS resolution.
- External hostname connectivity.

The bootstrap network was used because the final Northwind network segmentation had not yet been implemented.

No final Northwind static addressing was introduced during this test.

---

## VM Lifecycle Validation

The following lifecycle operations were successfully tested:

```text
Start       → PASS
Shutdown    → PASS
Start again → PASS
Reboot      → PASS
```

A guest-initiated shutdown correctly transitioned TEST01 to the stopped state in Proxmox.

The VM subsequently started and booted normally.

A guest-initiated reboot also completed successfully.

---

## Thin Provisioning Validation

Before TEST01 contained data, its thin volume consumed effectively no physical pool capacity.

Following RHEL installation, the VM disk reported approximately:

```text
Logical VM disk:       20 GiB
Thin volume Data%:     29.14%
Thin pool Data%:       0.33%
```

This represented approximately 5.8 GiB of physical thin-pool allocation for a 20 GiB logical virtual disk.

This demonstrated the key thin-provisioning principle:

> Logical capacity assigned to a VM does not immediately equal physical storage consumed.

Physical chunks were allocated as RHEL wrote data to the virtual disk.

---

## HYP01 Health Validation

While TEST01 was operational, the Proxmox storage backends remained active:

```text
local
local-lvm
northwind-workload
```

HYP01 reported:

```text
0 failed systemd units
```

The core Proxmox services remained active:

```text
pveproxy
pvedaemon
pvestatd
```

No critical host issue was identified while operating the test workload.

---

## Cleanup

After completing validation, TEST01 was cleanly shut down and removed from Proxmox.

The VM's referenced virtual disk:

```text
vm-100-disk-0
```

was removed with the VM.

The underlying storage architecture remained intact:

```text
vg_northwind
    ↓
tp_workload
    ↓
northwind-workload
```

Following deletion:

```text
tp_workload Data% → 0.00%
```

and `northwind-workload` remained active.

This confirmed that the thin-pool capacity allocated to the temporary workload was successfully reclaimed.

---

## Validation

- [x] Temporary VM created.
- [x] 2 vCPU allocated and detected.
- [x] Guest memory allocated and detected.
- [x] VM disk created on `northwind-workload`.
- [x] Thin-volume creation validated.
- [x] RHEL 10.2 successfully installed.
- [x] Virtual disk expansion validated.
- [x] Guest storage detected correctly.
- [x] Guest network attached to `vmbr0`.
- [x] DHCP configuration received.
- [x] Gateway connectivity validated.
- [x] External connectivity validated.
- [x] DNS resolution validated.
- [x] VM start validated.
- [x] VM shutdown validated.
- [x] VM reboot validated.
- [x] Proxmox resource reporting validated.
- [x] Thin-pool utilisation observed.
- [x] HYP01 storage remained healthy.
- [x] No failed systemd units detected.
- [x] Core Proxmox services remained active.
- [x] Temporary VM removed.
- [x] Temporary virtual disk removed.
- [x] Thin-pool capacity reclaimed.

---

## Findings / Lessons Learned

### Minimum requirements are not necessarily operationally sensible requirements

The initial 10 GiB virtual disk was technically based on the minimum investigated requirement but proved restrictive for the selected RHEL installation.

Future Northwind VM sizing should consider practical operating requirements rather than simply using vendor minimums.

---

### Controlled troubleshooting helps isolate variables

The installer issue was investigated by changing one significant variable at a time.

Increasing RAM did not resolve the issue, while changing the CPU model to `host` did.

This provided useful evidence without claiming an unproven root cause.

---

### CPU model selection is an architecture decision

CPU type affects more than performance.

It can influence guest compatibility and VM portability and should therefore be deliberately standardised when the Northwind VM specification is developed.

---

### Virtualisation creates abstraction layers

The guest saw `/dev/sda` even though its storage was ultimately provided by:

```text
NVMe → LVM PV → VG → Thin Pool → Thin Volume
```

The guest was then free to create its own partitions and LVM configuration inside that virtual disk.

---

### Thin provisioning requires monitoring

TEST01 demonstrated that a 20 GiB logical disk consumed only the physical blocks actually written.

This improves storage efficiency but means thin-pool utilisation must be monitored to prevent physical pool exhaustion.

---

### Cleanup is part of testing

Validation was not complete simply because TEST01 worked.

Removing the temporary workload and verifying that its storage was reclaimed demonstrated the complete workload lifecycle and ensured that test resources were not left behind.

---

## Result

**PASS**

HYP01 successfully demonstrated the ability to provision, execute, network, monitor, resize and remove a RHEL 10.2 virtual workload.

Compute, memory, storage, networking and VM lifecycle operations were successfully validated.

HYP01 remained healthy throughout testing, and all temporary VM resources were removed after completion.

The HYP01 virtualisation platform is therefore considered ready to host the initial Northwind v1 virtual server estate.

---

## Sprint Status

**NIM-14 was the final implementation task in Sprint 1.**

With completion of this task, the planned Sprint 1 implementation activities were complete.

The next project activity is the **Sprint 1 Review**, followed by the **Sprint Retrospective** and preparation for Sprint 2.

---

## Related Documentation

- NIM-10 — Configure HYP01 Local Storage Architecture
- NIM-12 — Configure HYP01 Platform Baseline
- Northwind Low-Level Design
- Sprint 1 implementation records
