# NIM-21 — FW01 Deployment Guide

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Jira:** NIM-21 (NIM-S2-05)  
**Document:** `deployment.md`  
**Target:** FW01 on HYP01  
**Operating system:** Red Hat Enterprise Linux 10.2  
**Implementation date:** 2026-11-03  
**Document status:** As-built deployment procedure  
**Design references:** LLD-01-004, LLD-02-001, LLD-02-002, LLD-06, NIM-19 Interface Schedule

---

## 1. Purpose

This document describes the procedure used to provision Northwind Outdoor Ltd.'s FW01 virtual machine and its supporting internal virtual network.

It covers:

- Creating and validating the internal Proxmox bridge.
- Provisioning FW01's virtual hardware.
- Installing RHEL 10.2 Minimal.
- Configuring storage, hostname and local administration.
- Establishing a network-independent initial operating-system baseline.
- Validating system configuration and reboot persistence.

The guide is intended for repeatable deployment, technical review, disaster recovery preparation and RHCSA learning.

**Scope boundary:** This procedure does not configure VLAN gateways, inter-VLAN routing, NAT, IP forwarding or production firewall policies. These are delivered through subsequent NIMs.

## 2. Deployment prerequisites

Before implementation, confirm:

- HYP01 is operational and accessible through the Proxmox Web UI.
- Existing HYP01 management access through `vmbr0` works.
- A tested administrative account is available.
- Physical console or equivalent out-of-band recovery access is available.
- The RHEL 10.2 x86_64 installation ISO is present in Proxmox ISO storage.
- VM ID `101` is available.
- Sufficient CPU, RAM and disk capacity are available.
- The approved network interface schedule and VM specification have been reviewed.
- No dependent production workloads are using the proposed internal bridge.
- A maintenance window has been approved if HYP01 networking or reboot operations may affect other workloads.

**Safety requirement:** Do not remove or reconfigure `vmbr0` or its physical uplink during this deployment. Loss of HYP01 management access could require physical-console recovery.

### 2.1 Approved deployment values

| Parameter | Value |
|---|---|
| Hypervisor | HYP01 |
| VM name | FW01 |
| VM ID | 101 |
| RHEL hostname | `fw01.internal.northwindoutdoor.xyz` |
| External bridge | `vmbr0` |
| Internal bridge | `vmbr1` |
| External guest NIC | `ens18` |
| Internal guest NIC | `ens19` |
| Installation ISO | `rhel-10.2-x86_64-dvd.iso` |
| Local administrator | `nw-admin` |

The interface names are the observed names from this deployment. In a rebuild, verify the mapping using the Proxmox NIC configuration and guest MAC addresses rather than assuming the names will always be identical.

## 3. Pre-change HYP01 validation

Log in to HYP01 using the approved administrative account.

Run:

```bash
ip -br addr
cat /etc/network/interfaces
sudo pvesm status
free -h
sudo qm list
```

### Command explanations

| Command | Purpose |
|---|---|
| `ip -br addr` | Displays network interfaces and addresses in brief format. `-br` means brief. |
| `cat /etc/network/interfaces` | Displays the persistent Proxmox host networking configuration. |
| `sudo pvesm status` | Shows configured Proxmox storage and available capacity. `pvesm` is the Proxmox storage manager. |
| `free -h` | Displays system memory and swap usage. `-h` uses human-readable units. |
| `sudo qm list` | Lists QEMU/KVM virtual machines on the Proxmox node. `qm` is Proxmox's VM management command. |

Record the results as pre-change evidence.

Confirm that `vmbr0` has the expected management configuration and that VM ID `101` is not already assigned to another VM.

## 4. Create the internal virtual bridge

### 4.1 Proxmox Web UI procedure

1. Open the Proxmox Web UI.
2. Select node **hyp01**.
3. Navigate to **System → Network**.
4. Click **Create → Linux Bridge**.
5. Configure the bridge as follows:

| Setting | Value |
|---|---|
| Name | `vmbr1` |
| Autostart | Enabled |
| VLAN aware | Enabled |
| Bridge ports | Blank / none |
| IPv4/CIDR | Blank |
| IPv4 gateway | Blank |
| IPv6/CIDR | Blank |
| IPv6 gateway | Blank |
| Comment | Northwind internal VLAN-aware bridge — no physical uplink |

6. Save the bridge.
7. Review the pending configuration changes.
8. Confirm that the existing `vmbr0` management configuration has not been altered.
9. Click **Apply Configuration**.

**Do not assign a physical NIC to `vmbr1`.** The bridge is intended to provide isolated virtual Layer 2 connectivity between Northwind VMs.

### 4.2 Verify the persistent configuration

Run on HYP01:

```bash
cat /etc/network/interfaces
```

The relevant configuration should be equivalent to:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

Explanation:

- `auto vmbr1` brings the bridge up automatically during system startup.
- `iface vmbr1 inet manual` defines the interface without assigning a static IPv4 address or using DHCP.
- `bridge-ports none` specifies that no physical network interface is attached.
- `bridge-stp off` disables Spanning Tree Protocol for this bridge.
- `bridge-fd 0` sets the bridge forwarding delay to zero.
- `bridge-vlan-aware yes` enables VLAN-aware bridging.
- `bridge-vids 2-4094` permits the configured VLAN ID range.

The bridge should not contain an `address` or `gateway` directive.

### 4.3 Verify the active bridge

Run:

```bash
ip -br addr
sudo bridge link show
sudo bridge vlan show
cat /sys/class/net/vmbr1/bridge/vlan_filtering
```

Expected observations:

- `vmbr1` exists.
- No IPv4 management address is assigned to `vmbr1`.
- No physical NIC is attached to `vmbr1`.
- The VLAN-filtering value is `1`.
- Existing HYP01 connectivity through `vmbr0` remains operational.

**Command explanations:**

`bridge link show` displays interfaces attached to Linux bridges and their operational state.

`bridge vlan show` displays VLAN membership information associated with bridge ports.

The `/sys/class/net/.../vlan_filtering` file exposes the running kernel's VLAN-filtering setting for the bridge. A value of `1` means enabled; `0` means disabled.

The absence of attached VM ports at this stage is expected because FW01 has not yet been created.

### 4.4 Validate bridge persistence

Perform a controlled HYP01 reboot during the approved maintenance window, after confirming recovery access is available.

Use the Proxmox node **Reboot** action or an approved orderly host reboot procedure.

After HYP01 returns:

1. Reconnect to the Proxmox Web UI.
2. Confirm `vmbr0` management connectivity remains operational.
3. Navigate to **System → Network**.
4. Confirm `vmbr1` is present, active, autostart-enabled and VLAN-aware.
5. Repeat the command-line bridge checks from Section 4.3.

**Observed implementation result:** HYP01 rebooted successfully, `vmbr1` persisted, VLAN filtering remained enabled and management connectivity was preserved.

**Isolation evidence limitation:** Configuration inspection confirmed no physical uplink. A separate active Layer 2 isolation test was not recorded as completed.

## 5. Create the FW01 virtual machine

### 5.1 General tab

In Proxmox:

1. Select HYP01.
2. Click **Create VM**.
3. Enter:

| Setting | Value |
|---|---|
| VM ID | `101` |
| Name | `FW01` |
| Start at boot | Leave at the approved default; do not enable solely for this installation |

Proceed to the next tab.

### 5.2 OS tab

Configure:

| Setting | Value |
|---|---|
| Installation media | Use CD/DVD disc image file (ISO) |
| Storage | `local` |
| ISO image | `rhel-10.2-x86_64-dvd.iso` |
| Guest OS type | Linux |

### 5.3 System tab

Configure:

| Setting | Value |
|---|---|
| Graphic card | Default |
| Machine | `q35` |
| BIOS | OVMF (UEFI) |
| Add EFI Disk | Enabled |
| EFI Storage | `local-lvm` |
| Pre-Enroll keys | Disabled |
| SCSI Controller | VirtIO SCSI single |
| QEMU Agent | Enabled |
| TPM | Not added |

**Design note:** Secure Boot is deferred. Do not enable pre-enrolled Secure Boot keys as part of this baseline without a separate approved change and validation procedure.

### 5.4 Disks tab

Configure the operating-system disk:

| Setting | Value |
|---|---|
| Bus/Device | SCSI 0 |
| Storage | `local-lvm` |
| Disk size | 40 GiB |
| Cache | Default (No cache) |
| Discard | Enabled |
| IO thread | Enabled |
| SSD emulation | Enabled |
| Backup | Enabled |
| Async IO | Default (`io_uring`) |

The operating-system disk resides on HYP01's VM OS storage tier, not the separate Northwind workload-data tier.

### 5.5 CPU tab

Configure:

| Setting | Value |
|---|---|
| Sockets | 1 |
| Cores | 2 |
| Total vCPUs | 2 |
| CPU type | host |
| NUMA | Disabled |
| CPU limit | Unlimited |

Leave individual advanced CPU flags at their default settings unless the approved VM specification requires otherwise.

### 5.6 Memory tab

Configure:

| Setting | Value |
|---|---|
| Memory | 4096 MiB |
| Ballooning device | Disabled |
| Allow KSM | Default enabled |

Disabling memory ballooning gives FW01 a fixed memory allocation, simplifying initial capacity planning and troubleshooting.

### 5.7 Network tab

Configure the first virtual NIC:

| Setting | Value |
|---|---|
| Bridge | `vmbr0` |
| Model | VirtIO (paravirtualized) |
| VLAN Tag | No VLAN |
| Firewall | Enabled |
| Disconnect | Disabled |
| MTU | Same as bridge |

This creates `net0`, FW01's external/upstream network adapter.

### 5.8 Confirm and create

Review the complete VM configuration.

Confirm:

- VM ID is `101`.
- Name is `FW01`.
- Firmware is OVMF.
- Machine type is q35.
- CPU and memory match the specification.
- The 40 GiB disk is on `local-lvm`.
- The installation ISO is attached.
- `net0` is attached to `vmbr0`.

Click **Finish**.

Do not start the VM until the internal NIC has been added and verified.

## 6. Add the internal network adapter

1. Select **FW01 → Hardware**.
2. Click **Add → Network Device**.
3. Configure:

| Setting | Value |
|---|---|
| Bridge | `vmbr1` |
| Model | VirtIO (paravirtualized) |
| VLAN Tag | No VLAN |
| Firewall | Enabled |
| Disconnect | Disabled |
| MTU | Same as bridge |

4. Confirm the change.
5. Review the Hardware tab.

The expected network hardware is:

```text
net0 → VirtIO → vmbr0
net1 → VirtIO → vmbr1
```

Do not apply a single access VLAN tag to the internal NIC at this stage. The intended future design requires FW01 to support multiple internal VLANs, which will be configured in later NIMs.

Record the virtual NIC MAC addresses for interface identification and inventory purposes.

## 7. Install RHEL 10.2 Minimal

### 7.1 Start the installer

1. Select FW01.
2. Click **Start**.
3. Open **Console**.
4. Boot from the attached RHEL installation ISO.
5. Launch the RHEL 10.2 installer (Anaconda).

Use the approved language, locale and keyboard settings.

### 7.2 Software selection

Select:

**Base Environment: Minimal Install**

Do not select additional software groups for this NIM.

Additional packages should be installed only when required by subsequent approved configuration tasks.

### 7.3 Installation destination

Select FW01's 40 GiB virtual operating-system disk.

Choose **Custom / Manual Partitioning**.

Use the following layout:

| Mount point | Device type | Filesystem | As-built capacity |
|---|---|---|---|
| `/boot/efi` | Standard partition | EFI System Partition | Approximately 572 MiB |
| `/boot` | Standard partition | XFS | Approximately 953 MiB |
| `/` | LVM | XFS | Approximately 23.28 GiB |
| Swap | LVM | Linux swap | Approximately 1.86 GiB |

Create the volume group:

```text
Volume group: vg_os
Size policy: As large as possible
```

Create logical volumes:

```text
vg_os/lv_root
vg_os/lv_swap
```

Leave the remaining volume-group capacity unallocated.

**As-built note:** The displayed capacities reflect the installed configuration. They are slightly smaller than the initially proposed binary-unit sizes because the original capacity entries were interpreted as decimal units.

For an exact-size rebuild, use explicit `MiB` and `GiB` suffixes where required by the approved specification, and document any deviation.

### 7.4 EFI partition verification

Ensure `/boot/efi` uses **EFI System Partition**, not XFS.

During the original installation, the EFI partition was initially created with an incorrect XFS selection.

The issue was resolved by removing and recreating the `/boot/efi` mount point, after which Anaconda recognised it as an EFI System Partition.

Before accepting the storage changes:

1. Click **Done**.
2. Review the **Summary of Changes**.
3. Verify the EFI partition is being created using the correct EFI-compatible format.
4. Verify `/boot` and `/` use XFS.
5. Verify the logical volumes and swap.
6. Verify changes affect only the intended FW01 virtual disk.
7. Accept the changes only after the summary is correct.

Do not continue if the installer proposes formatting `/boot/efi` as XFS.

### 7.5 Network and hostname

Open **Network & Host Name**.

The original installation detected:

```text
ens18 → External NIC → vmbr0
ens19 → Internal NIC → vmbr1
```

Set the hostname:

```text
fw01.internal.northwindoutdoor.xyz
```

Click **Apply**.

For the console-first baseline, do not configure final static addressing or VLAN gateways during installation.

If the installer activates the external DHCP connection, this must be reviewed and disabled before NIM-21 acceptance.

**Important:** Anaconda displays guest interface names (`ens18`, `ens19`), not the Proxmox bridge names (`vmbr0`, `vmbr1`).

### 7.6 User configuration

Create the named local administrator:

```text
nw-admin
```

Grant the approved administrative privileges required for `sudo`.

Set a strong, unique password according to Northwind's credential policy.

Configure root recovery credentials according to the restricted recovery procedure. Do not store passwords, password hashes, recovery secrets or credential screenshots in the public repository.

### 7.7 Begin installation

Review the installation summary.

Confirm:

- Correct installation disk.
- Minimal Install selected.
- Hostname configured.
- Administrator account created.
- Correct UEFI and storage configuration.
- No dependency on operational Northwind routing.

Begin installation and wait for completion.

## 8. First boot and initial operating-system validation

After installation:

1. Reboot into the installed operating system.
2. Confirm the VM boots from its virtual disk.
3. Log in through the Proxmox console as `nw-admin`.

Run:

```bash
cat /etc/redhat-release
hostnamectl
lsblk -f
sudo pvs
sudo vgs
sudo lvs
ip -br addr
getenforce
```

### Command reference

| Command | What it does |
|---|---|
| `cat /etc/redhat-release` | Displays the installed RHEL release. |
| `hostnamectl` | Displays system hostname, OS, kernel and platform information. |
| `lsblk -f` | Lists block devices and filesystem details; `-f` means filesystem information. |
| `pvs` | Displays LVM physical volumes. |
| `vgs` | Displays LVM volume groups and available capacity. |
| `lvs` | Displays LVM logical volumes. |
| `ip -br addr` | Shows interfaces and assigned addresses in a compact format. |
| `getenforce` | Displays the current SELinux mode. |

Expected observations:

- RHEL 10.2 installed.
- Correct fully qualified hostname.
- EFI partition formatted as VFAT.
- `/boot` and `/` formatted as XFS.
- Root and swap logical volumes present in `vg_os`.
- Approximately 13.36 GiB free in `vg_os`.
- SELinux reports `Enforcing`.

## 9. Configure the network-independent baseline

### 9.1 Inspect NetworkManager

Run:

```bash
nmcli device status
nmcli connection show
ip route
```

These commands display:

- The connection status of network devices.
- Saved NetworkManager connection profiles.
- The current IPv4 routing table.

In the original deployment, `ens18` obtained the temporary DHCP address `192.168.178.80/24`, with a default route through `192.168.178.1`.

This was corrected before acceptance.

### 9.2 Identify the active external connection profile

Run:

```bash
nmcli -g GENERAL.CONNECTION device show ens18
```

Explanation:

- `nmcli` is the NetworkManager command-line interface.
- `-g` selects specific output fields.
- `GENERAL.CONNECTION` identifies the active connection profile associated with the device.
- `device show ens18` inspects the external network interface.

Record the returned profile name or UUID.

Do not assume the connection profile is named `ens18`; the original installation generated a UUID-like profile name.

### 9.3 Disable automatic activation

While the external connection is active, run:

```bash
sudo nmcli connection modify "$(nmcli -g GENERAL.CONNECTION device show ens18)" connection.autoconnect no
```

This updates the active external connection profile so NetworkManager does not automatically activate it during future boots.

Explanation:

- `connection modify` edits a saved connection profile.
- `$(...)` substitutes the output of the inner command.
- `connection.autoconnect no` disables automatic activation.
- `sudo` provides the privileges required to modify system network configuration.

Run this command from the Proxmox console, not over an SSH connection that depends on `ens18`.

### 9.4 Disconnect the external interface

Run:

```bash
sudo nmcli device disconnect ens18
```

This deactivates the current network connection without removing the virtual network adapter or deleting its saved profile.

The internal `ens19` connection should also remain disconnected.

### 9.5 Validate the result

Run:

```bash
nmcli device status
ip -br addr
ip route
```

Expected:

```text
ens18  ethernet  disconnected
ens19  ethernet  disconnected
```

Neither NIC should have an assigned IPv4 address.

The IPv4 routing table should contain no default route.

The NICs may still display `UP` at the kernel interface level. This does not mean a NetworkManager connection is active.

**Observed result:** Both interfaces were disconnected, neither had an assigned IPv4 address, and the IPv4 routing table was empty.

## 10. Validate storage, swap and security

Run:

```bash
findmnt /
findmnt /boot
findmnt /boot/efi
swapon --show
sudo pvs
sudo vgs
sudo lvs
getenforce
systemctl status qemu-guest-agent
```

### Command reference

| Command | Purpose |
|---|---|
| `findmnt /` | Identifies the filesystem mounted at `/`. |
| `findmnt /boot` | Verifies the boot filesystem mount. |
| `findmnt /boot/efi` | Verifies the EFI filesystem mount. |
| `swapon --show` | Lists active swap devices and usage. |
| `sudo pvs` | Verifies the physical volume. |
| `sudo vgs` | Verifies the volume group and free capacity. |
| `sudo lvs` | Verifies the logical volumes. |
| `getenforce` | Verifies SELinux enforcement. |
| `systemctl status qemu-guest-agent` | Displays the guest-agent service state and recent logs. |

Expected:

- Root filesystem: XFS on `vg_os/lv_root`.
- Boot filesystem: XFS on `/dev/sda2`.
- EFI filesystem: VFAT on `/dev/sda1`.
- Swap: active on `vg_os/lv_swap`.
- LVM volume group: `vg_os`.
- Free capacity: approximately 13.36 GiB.
- SELinux: `Enforcing`.
- QEMU Guest Agent: enabled and active.

**Observed result:** All checks passed. The QEMU Guest Agent was already installed and running.

## 11. Controlled FW01 reboot

### 11.1 Reboot procedure

Confirm that installation has completed and that the Proxmox console is available.

Run inside FW01:

```bash
sudo systemctl reboot
```

Explanation:

- `sudo` executes with administrative privileges.
- `systemctl` communicates with the systemd service manager.
- `reboot` requests an orderly operating-system restart.

Do not forcibly power off the VM for this validation unless a separate recovery procedure requires it.

### 11.2 Post-reboot validation

After FW01 restarts, log in through the Proxmox console as `nw-admin`.

Run:

```bash
hostnamectl
findmnt /
findmnt /boot
findmnt /boot/efi
swapon --show
sudo vgs
nmcli device status
ip -br addr
ip route
getenforce
systemctl is-active qemu-guest-agent
```

The `systemctl is-active` command returns the concise current service state. For the guest agent, the expected output is:

```text
active
```

Confirm:

1. The correct hostname persists.
2. The root, boot and EFI filesystems are mounted.
3. Swap is active.
4. The volume group remains available.
5. Both network connections remain disconnected.
6. No IPv4 default route exists.
7. SELinux remains enforcing.
8. The QEMU Guest Agent is active.

**Observed result:** FW01 successfully rebooted and passed the recorded persistence checks.

## 12. Final acceptance checklist

- [x] `vmbr1` created.
- [x] VLAN awareness enabled.
- [x] No physical uplink on `vmbr1`.
- [x] No HYP01 management address assigned to `vmbr1`.
- [x] Bridge persisted after HYP01 reboot.
- [x] HYP01 management access preserved.
- [x] FW01 VM 101 created.
- [x] Approved CPU, memory and virtual disk provisioned.
- [x] External NIC attached to `vmbr0`.
- [x] Internal NIC attached to `vmbr1`.
- [x] RHEL 10.2 Minimal installed.
- [x] EFI, XFS and LVM layout validated.
- [x] Hostname configured.
- [x] Local administrator login demonstrated.
- [x] Both guest network connections disconnected.
- [x] No IPv4 default route present.
- [x] SELinux enforcing.
- [x] QEMU Guest Agent active.
- [x] FW01 reboot and configuration persistence verified.
- [ ] Root recovery credential handling formally confirmed.
- [ ] Active Layer 2 isolation test completed or deferral approved.
- [ ] Recovery documentation reviewed and approved.

## 13. Recovery considerations

This guide records the deployment procedure. Detailed recovery actions should be maintained in `rollback.md`.

Recovery scenarios include:

- Loss of HYP01 management connectivity after bridge changes.
- Incorrect `vmbr1` configuration.
- Failed FW01 virtual hardware provisioning.
- Failed RHEL installation.
- Incorrect EFI or LVM partitioning.
- NetworkManager activating an unintended connection.
- Failed FW01 reboot or bootloader recovery.

**Recovery priority:** Restore HYP01 management access before attempting further changes to the internal virtual networking environment.

Do not delete an existing VM or reformat storage during a rebuild without verifying the target VM ID, disk and backup status.

## 14. Glossary

| Term | Definition |
|---|---|
| Anaconda | The installation program used by RHEL. |
| Bridge | A software-based Layer 2 switching component that connects network interfaces. |
| DHCP | Dynamic Host Configuration Protocol; automatically supplies network configuration such as IP addresses. |
| EFI System Partition (ESP) | A partition containing boot files read by UEFI firmware. |
| FQDN | Fully Qualified Domain Name; the complete DNS hostname of a system. |
| GPT | GUID Partition Table; a modern disk partition-table format. |
| Hypervisor | Software that hosts and manages virtual machines. |
| IO thread | A dedicated execution thread used to handle certain virtual disk I/O operations. |
| KVM | Kernel-based Virtual Machine; Linux virtualisation technology. |
| LVM | Logical Volume Manager; manages flexible storage volumes using physical volumes, volume groups and logical volumes. |
| Logical volume (LV) | A storage volume allocated from an LVM volume group. |
| Mount point | A directory where a filesystem is attached to the Linux directory tree. |
| NAT | Network Address Translation; translates network addresses between networks. |
| NetworkManager | The Linux service responsible for managing network connections and profiles. |
| NIC | Network Interface Card; a physical or virtual network adapter. |
| OVMF | UEFI firmware implementation used by virtual machines. |
| Physical volume (PV) | A disk or partition prepared for use by LVM. |
| Proxmox VE | Virtualisation management platform hosting FW01. |
| QEMU Guest Agent | Software inside a virtual machine that allows supported communication with its hypervisor. |
| q35 | A modern virtual machine hardware model based on a PCI Express chipset. |
| RHCSA | Red Hat Certified System Administrator certification. |
| SELinux | Security-Enhanced Linux; a mandatory access-control system. |
| systemd | Linux system and service manager responsible for starting and managing services and system operations. |
| UEFI | Unified Extensible Firmware Interface; modern firmware used to boot operating systems. |
| VFAT | Linux filesystem driver/type commonly used for FAT-formatted EFI partitions. |
| VirtIO | Paravirtualised device interface designed for efficient virtual machine I/O. |
| VLAN | Virtual Local Area Network; separates Layer 2 network traffic into logical segments. |
| VLAN-aware bridge | A bridge that understands and handles VLAN-tagged Ethernet frames. |
| Volume group (VG) | An LVM storage pool from which logical volumes are allocated. |
| XFS | A high-performance journaling filesystem commonly used by RHEL. |

---

## 15. Deployment outcome

FW01 was successfully provisioned on HYP01 with RHEL 10.2 Minimal, the approved virtual hardware, UEFI boot, XFS/LVM storage and dual virtual network interfaces.

The internal VLAN-aware bridge was created and verified across a HYP01 reboot. FW01 was validated through console login, storage inspection, SELinux verification, guest-agent checks and a controlled guest reboot.

**The base deployment is complete.** Formal NIM-21 closure remains subject to the outstanding evidence and documentation items identified above.

The next implementation task is **NIM-22 — FW01 Interfaces and VLAN Gateways**.
