# NIM-17 — HYP01 Platform Readiness: Command Reference

**Project:** Northwind Outdoor Ltd.  
**Sprint:** 02  
**Ticket:** NIM-17 — Verify HYP01 Platform Readiness  
**Target:** HYP01 — Proxmox VE 9.2  
**Purpose:** Record the commands used to verify platform health, storage, networking, management access and deployment readiness.

> **Scope:** Verification only. No infrastructure configuration changes were made.
>
> Commands marked **New / Extended** introduce a new command or a more advanced use of an existing command. This is a learning classification, not confirmation that the command was never encountered previously.

---

## 1. Platform Health and System Information

### `hostnamectl`

**TL;DR:** Display the computer's identity and operating-system information.

```bash
hostnamectl
```

Displays the hostname, operating system, kernel, architecture and available hardware information.

**Key terms:**
- *Hostname:* The name assigned to a computer.
- *Kernel:* The core component of the operating system that manages hardware and system resources.
- *Architecture:* The CPU instruction-set family, such as x86-64.

**NIM-17:** Confirmed HYP01's identity, Debian 13 and running kernel.

### `pveversion -v`

**TL;DR:** Display the installed Proxmox version and component versions.

```bash
pveversion -v
```

- `-v` — Verbose output, including installed Proxmox packages and component versions.

**NIM-17:** Confirmed Proxmox VE 9.2.0 and associated package versions.

**Platform-specific:** Proxmox VE.

### `uptime`

**TL;DR:** Show how long the system has been running and how busy it is.

```bash
uptime
```

**Key terms:**
- *Uptime:* Time elapsed since the last system boot.
- *Load average:* Average demand for CPU execution and uninterruptible system resources over 1, 5 and 15 minutes.

**NIM-17:** Reported approximately 1 hour and 28 minutes of uptime with load averages of 0.00.

### `free -h`

**TL;DR:** Display available and used system memory.

```bash
free -h
```

- `-h` — Human-readable sizes, such as MiB and GiB.

**Key terms:**
- *RAM:* Physical memory used by running programs and the operating system.
- *Swap:* Disk-backed space that can hold memory pages.
- *Available memory:* An estimate of memory available for starting applications without swapping.

**NIM-17:** Confirmed approximately 62 GiB of total RAM and 60 GiB available.

### `lscpu`

**TL;DR:** Display CPU specifications and virtualisation capabilities.

```bash
lscpu
```

**Key terms:**
- *Logical CPU:* A processor execution context exposed to the operating system.
- *VT-x:* Intel's hardware-assisted virtualisation technology.
- *NUMA:* Non-Uniform Memory Access, an architecture where memory access latency can vary between processor nodes.

**NIM-17:** Confirmed the Intel Core i9-12900KF, 24 logical CPUs and VT-x support.

### `systemctl --failed`

**TL;DR:** Show systemd units that have failed.

```bash
systemctl --failed
```

- `--failed` — Filter the unit list to units in the failed state.

**Key terms:**
- *systemd:* The Linux system and service manager.
- *Unit:* A resource managed by systemd, such as a service, mount or device.

**NIM-17:** Returned zero failed units.

### `systemctl`

**TL;DR:** Display systemd units and their states.

```bash
systemctl
```

Shows the default systemd unit listing. Use `systemctl --failed` when specifically investigating failures.

**NIM-17:** Used to inspect the broader unit inventory.

---

## 2. Storage and Capacity Verification

### `pvesm status` — New / Extended

**TL;DR:** Show Proxmox storage backends and available capacity.

```bash
pvesm status
```

**Key terms:**
- *Storage backend:* A configured storage location available to Proxmox.
- *LVM-thin:* Logical Volume Manager storage that allocates physical space as data is written.
- *Thin provisioning:* Presenting virtual storage capacity without allocating all underlying physical space immediately.

**NIM-17:** Confirmed `local`, `local-lvm` and `northwind-workload` were active.

**Platform-specific:** Proxmox VE.

### `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS` — Extended

**TL;DR:** Display disk and partition information using selected columns.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

- `-o` — Select the output columns.
- `NAME` — Device name.
- `SIZE` — Device capacity.
- `TYPE` — Disk, partition or logical-volume type.
- `FSTYPE` — Filesystem or storage format.
- `MOUNTPOINTS` — Locations where the device's filesystem is mounted.

**NIM-17:** Verified the physical NVMe devices, partitions and LVM layout.

**Important:** NVMe device names can change between boots. Use persistent identifiers when identifying disks for configuration changes.

### `pvs`

**TL;DR:** List LVM physical volumes.

```bash
pvs
```

**Key term:**
- *Physical volume (PV):* A disk or partition initialised for use by LVM.

**NIM-17:** Verified the physical volumes associated with the `pve` and `vg_northwind` volume groups.

### `vgs`

**TL;DR:** List LVM volume groups and their available space.

```bash
vgs
```

**Key term:**
- *Volume group (VG):* A storage pool formed from one or more LVM physical volumes.

**NIM-17:** Verified the volume-group capacities and remaining unallocated space.

### `lvs -a -o ...` — Extended

**TL;DR:** Display detailed logical-volume and thin-pool information.

```bash
lvs -a -o lv_name,vg_name,lv_size,segtype,data_percent,metadata_percent
```

- `-a` — Include internal and hidden logical volumes.
- `-o` — Select the output columns.
- `lv_name` — Logical-volume name.
- `vg_name` — Parent volume group.
- `lv_size` — Logical-volume size.
- `segtype` — Storage segment type, such as `linear` or `thin-pool`.
- `data_percent` — Percentage of thin-pool data space consumed.
- `metadata_percent` — Percentage of thin-pool metadata space consumed.

**Key term:**
- *Thin-pool metadata:* Information used by LVM to track mappings and allocations inside a thin pool.

**NIM-17:** Confirmed that both thin pools were empty, with low metadata utilisation.

**Administration consideration:** Monitor both thin-pool data and metadata capacity.

### `df -h`

**TL;DR:** Show used and available space on mounted filesystems.

```bash
df -h
```

- `-h` — Display sizes in human-readable units.

**Key term:**
- *Filesystem:* The structure used to organise and access files on a storage device.

**NIM-17:** Confirmed the Proxmox root filesystem had approximately 20% utilisation.

**Important:** `df` reports mounted filesystem usage, not the remaining capacity inside an LVM thin pool.

---

## 3. Network Verification

### `ip -br link` — Extended

**TL;DR:** Show network interfaces and their operational states.

```bash
ip -br link
```

- `-br` — Brief output.
- `link` — Display network interface information.

**NIM-17:** Identified `nic0`, `vmbr0` and the wireless interfaces.

### `ip -br addr` — Extended

**TL;DR:** Show network interfaces and their assigned IP addresses.

```bash
ip -br addr
```

- `-br` — Brief output.
- `addr` — Display interface addresses.

**NIM-17:** Confirmed the management address `192.168.178.10/24` on `vmbr0`.

### `ip route`

**TL;DR:** Display the kernel's IP routing table.

```bash
ip route
```

**Key terms:**
- *Routing table:* Rules used to determine where IP packets should be forwarded.
- *Default gateway:* Router used when no more specific route matches a destination.

**NIM-17:** Confirmed the default route through `192.168.178.1` via `vmbr0`.

### `cat /etc/network/interfaces`

**TL;DR:** Display the persistent network interface configuration.

```bash
cat /etc/network/interfaces
```

**Key terms:**
- *Linux bridge:* A software-based Layer 2 network switch.
- *Bridge port:* An interface attached to a Linux bridge.
- *Persistent configuration:* Settings intended to remain effective across restarts.

**NIM-17:** Confirmed the `vmbr0` configuration and identified that `vmbr1` was not configured.

**Platform note:** This is the networking configuration approach used on HYP01. RHEL commonly uses NetworkManager and `nmcli`.

### `bridge vlan show` — New

**TL;DR:** Display VLAN membership and tagging settings on Linux bridge ports.

```bash
bridge vlan show
```

**Key terms:**
- *VLAN:* Virtual Local Area Network, used to logically separate Layer 2 networks.
- *PVID:* Port VLAN Identifier, used to classify incoming untagged traffic.
- *Egress Untagged:* Indicates that outgoing traffic for a VLAN is transmitted without a VLAN tag.
- *802.1Q:* Ethernet VLAN tagging standard.

**NIM-17:** Displayed VLAN 1 as the PVID and untagged egress VLAN for `nic0` and `vmbr0`.

**Finding:** The intended VLAN-aware internal bridge `vmbr1` was absent.

### `ls /sys/class/net/` — New

**TL;DR:** List network interfaces currently known to the Linux kernel.

```bash
ls /sys/class/net/
```

**Key terms:**
- *sysfs:* A virtual filesystem exposing kernel device and system information.
- *Network interface:* A physical or virtual endpoint used to send and receive network traffic.

**NIM-17:** Confirmed that `nic1` and `nic2` were not present in the current interface inventory.

### `lspci | grep -i ethernet` — New / Extended

**TL;DR:** Find Ethernet controllers detected on the PCI bus.

```bash
lspci | grep -i ethernet
```

- `lspci` — List PCI devices.
- `|` — Pipe the output of one command into another.
- `grep` — Search text for a matching pattern.
- `-i` — Match without distinguishing uppercase and lowercase.

**Key terms:**
- *PCI:* Peripheral Component Interconnect, a hardware bus family used to connect devices.
- *Pipe:* A shell operator that passes one command's output to another command's input.

**NIM-17:** Identified the Realtek RTL8125 2.5GbE controller.

**Limitation:** This command does not list every possible network adapter, such as USB Ethernet devices.

---

## 4. Virtual Machine Inventory

### `qm list` — New

**TL;DR:** List virtual machines configured in Proxmox.

```bash
qm list
```

**Key terms:**
- *QEMU:* Software used to emulate and virtualise computer hardware.
- *KVM:* Kernel-based Virtual Machine, the Linux kernel's hardware-assisted virtualisation infrastructure.

**NIM-17:** Confirmed that no QEMU/KVM virtual machines were configured.

**Platform-specific:** Proxmox VE.

### `pct list` — New

**TL;DR:** List LXC containers configured in Proxmox.

```bash
pct list
```

**Key term:**
- *LXC:* Linux Containers, an operating-system-level virtualisation technology that shares the host kernel.

**NIM-17:** Confirmed that no LXC containers were configured.

**Platform-specific:** Proxmox VE.

---

## 5. Administrative Connectivity

### `ssh root@192.168.178.10`

**TL;DR:** Connect securely to HYP01's command-line interface.

```bash
ssh root@192.168.178.10
```

**Key terms:**
- *SSH:* Secure Shell, a protocol for encrypted remote administration.
- *Root:* The Linux superuser account with extensive system privileges.
- *Authentication:* The process of verifying a user's identity.

**NIM-17:** Successful SSH connectivity confirmed from the administration workstation.

**Security note:** Root/password-based SSH access is part of the temporary bootstrap arrangement. Its hardening is tracked separately.

### `exit`

**TL;DR:** End the current shell session.

```bash
exit
```

Used to leave an interactive shell or disconnect from an SSH session.

**Execution note:** Suggested during verification; not separately confirmed.

---

## 6. Additional Command — Live Monitoring

### `watch` — New

**TL;DR:** Repeatedly execute a command and refresh its output.

```bash
watch -n 2 free -h
```

- `watch` — Execute another command repeatedly.
- `-n 2` — Refresh every two seconds.
- `free -h` — Display human-readable memory usage.

**Useful for:** Observing changes in memory usage, service state or other system metrics during troubleshooting.

Press `Ctrl+C` to stop.

**Execution note:** Added to the command reference during NIM-17; execution of this exact example was not separately confirmed.

---

## 7. NIM-17 Verification Summary

| Area | Commands | Result |
|---|---|---|
| Platform health | `hostnamectl`, `pveversion`, `uptime`, `free`, `lscpu`, `systemctl` | No immediate health issues |
| Storage | `pvesm`, `lsblk`, `pvs`, `vgs`, `lvs`, `df` | Storage active with substantial available capacity |
| Networking | `ip`, `cat`, `bridge`, `ls`, `lspci`, `grep` | Bootstrap networking operational; internal bridge pending |
| VM inventory | `qm`, `pct` | No configured guests |
| Remote access | `ssh` | Connectivity confirmed |

### Findings

- **NIM-17-F01:** Internal bridge `vmbr1` not yet implemented.
- **NIM-17-F02:** Unused `nic1` and `nic2` declarations require later review.
- **NIM-17-F03:** Final FW01/JUMP01 resource sizing remains dependent on NIM-18.

### Lessons Learned

1. Readiness assessments should use non-disruptive verification commands.
2. LVM thin-pool capacity and filesystem capacity must be checked separately.
3. A configured network interface is not necessarily a currently detected interface.
4. Missing infrastructure may be a planned dependency rather than a defect.
5. Successful SSH access verifies connectivity and authentication, not the security of the final administrative access model.
6. Proxmox-specific commands complement, rather than replace, transferable Linux administration skills.watch -n 1 free -h

