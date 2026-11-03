# NIM-21 — FW01 Rollback and Recovery Guide

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Jira:** NIM-21 (NIM-S2-05)  
**Document:** `rollback.md`  
**Target systems:** HYP01 / FW01  
**Platform:** Proxmox VE 9.2.1 / RHEL 10.2  
**Document status:** Recovery procedures — not yet recovery-tested  
**Related documents:** `README.md`, `deployment.md`, `commands.md`  
**Design references:** LLD-01, LLD-02, LLD-06, LLD-09

---

## 1. Purpose

This document defines the procedures for recovering from failures introduced during NIM-21 or reversing its implementation when required.

NIM-21 introduced two infrastructure components:

1. `vmbr1` — HYP01's isolated, VLAN-aware internal Linux bridge.
2. FW01 — a RHEL 10.2 virtual machine intended to provide Northwind's future network routing and firewall services.

The recovery procedures aim to:

- Restore HYP01 management connectivity.
- Preserve existing hypervisor services and virtual machines.
- Recover FW01 without unnecessary data loss.
- Restore approved virtual hardware and network settings.
- Avoid introducing unintended connectivity between Northwind's internal network and the physical home LAN.
- Provide a controlled method of reversing the deployment.

### Recovery priority

**HYP01 management access takes precedence over FW01 availability.**

At the completion of NIM-21, FW01 does not yet provide production routing or firewall services. It can therefore remain offline while HYP01 networking is recovered.

This assumption must be reviewed once Northwind workloads begin depending on FW01.

## 2. Recovery principles

All recovery operations must follow these principles:

1. **Diagnose before changing configuration.** Identify the failing component and collect available evidence.
2. **Use the least disruptive recovery method.** Restarting a failed service is preferable to rebuilding a virtual machine.
3. **Protect HYP01 management.** Avoid changing `vmbr0`, its physical uplink or management addressing unless necessary.
4. **Preserve existing VM disks.** Do not delete, recreate or reformat disks without verifying the correct recovery target.
5. **Use console access.** Recovery must not depend on FW01 having operational networking.
6. **Verify after every change.** A successful command does not necessarily mean the service has been restored.
7. **Document deviations.** Record any recovery action that differs from the approved configuration.
8. **Escalate when uncertain.** Do not continue with destructive actions if the failure or its consequences are unclear.

## 3. Recovery prerequisites

Before starting recovery, establish:

| Requirement | Purpose |
|---|---|
| HYP01 physical console access | Recovery if management networking fails |
| HYP01 administrative credentials | Authorised hypervisor recovery |
| Proxmox Web UI access, if available | VM and bridge administration |
| FW01 Proxmox console | Guest recovery without networking |
| FW01 local administrator credentials | Operating-system administration |
| Protected root recovery credentials | Recovery when normal administrative access fails |
| Approved VM specification | Recreating virtual hardware accurately |
| Approved network interface schedule | Restoring correct bridge/NIC mappings |
| RHEL installation ISO | Boot or reinstall recovery |
| Latest verified backup, if available | Restoring lost or damaged configuration/data |

Do not assume that a backup exists merely because the VM's Proxmox disk has the **Backup** option enabled.

That option makes the disk eligible for applicable backup jobs; it does not create a backup by itself.

### 3.1 Record the incident

Before making changes, capture:

- Date and time.
- Affected system.
- Symptoms and error messages.
- Last known working configuration.
- Most recent change.
- Available backup or recovery point.
- Proposed recovery action.
- Expected impact.
- Person authorising the action.

Use the project's incident or change-management process where applicable.

## 4. Recovery scenario matrix

| Scenario | Preferred recovery action | Impact |
|---|---|---|
| `vmbr1` configuration incorrect | Correct bridge through Proxmox | Internal VM networking may be interrupted |
| HYP01 management access lost | Recover from physical console | Hypervisor administration affected |
| FW01 NIC attached to wrong bridge | Correct VM virtual NIC configuration | Brief guest network interruption |
| FW01 starts with unintended DHCP connectivity | Disable the affected NetworkManager connection | Guest connectivity interrupted |
| FW01 does not boot | Inspect boot configuration and console errors | FW01 unavailable |
| Filesystem or LVM issue | Diagnose and repair using appropriate recovery environment | Possible extended outage or data risk |
| QEMU Guest Agent unavailable | Diagnose and restart service | Reduced hypervisor/guest integration |
| FW01 configuration unusable | Restore a verified backup or rebuild | FW01 unavailable during recovery |
| Full NIM-21 reversal required | Remove FW01 and then `vmbr1`, after dependency checks | Removes the NIM-21 infrastructure |

## 5. Scenario A — Incorrect `vmbr1` configuration

### Symptoms

- `vmbr1` missing from Proxmox.
- VLAN awareness disabled.
- Incorrect physical interface attached.
- Unexpected IP address assigned to the bridge.
- Internal VM network connectivity not behaving as designed.

### Step 1 — Inspect the bridge

If HYP01 management access is available, open:

**HYP01 → System → Network**

Confirm the expected settings:

| Property | Expected |
|---|---|
| Bridge | `vmbr1` |
| Type | Linux Bridge |
| Autostart | Yes |
| VLAN aware | Yes |
| Bridge ports | None |
| IP address | None |
| Gateway | None |

From HYP01, inspect the configuration:

```bash
ip -br addr
cat /etc/network/interfaces
sudo bridge link show
sudo bridge vlan show
cat /sys/class/net/vmbr1/bridge/vlan_filtering
```

The last command requires `vmbr1` to exist. If the bridge is missing, the file will not exist.

### Step 2 — Correct the configuration

Using the Proxmox Web UI:

1. Select `vmbr1`.
2. Click **Edit**.
3. Correct the settings to match the approved design.
4. Review the pending changes.
5. Confirm that `vmbr0` remains unchanged.
6. Apply the configuration.

Do not remove a bridge while virtual machines are actively dependent on it.

### Step 3 — Validate

Repeat the inspection commands.

Confirm:

- `vmbr1` exists.
- VLAN filtering is enabled.
- No physical interface is attached.
- No HYP01 IP address is assigned.
- HYP01 management access still works.

If a reboot is necessary to validate persistence, schedule it separately and ensure console recovery is available.

## 6. Scenario B — HYP01 management connectivity lost

### Symptoms

- Proxmox Web UI unreachable.
- SSH to HYP01 fails.
- HYP01 no longer responds at its approved management address.
- Networking failure occurred immediately after applying bridge changes.

### Recovery priority

**Do not attempt repeated remote network changes when access is already unstable.**

Use the HYP01 physical console.

### Step 1 — Establish local access

1. Connect to the HYP01 physical console.
2. Log in using authorised recovery credentials.
3. Confirm that the operating system is running.
4. Record any networking errors.

### Step 2 — Inspect interfaces and routes

Run:

```bash
ip -br addr
ip route
cat /etc/network/interfaces
```

Command explanations:

- `ip -br addr` displays interface state and addresses.
- `ip route` displays the IPv4 routing table.
- `cat /etc/network/interfaces` displays the persistent host network configuration.

Check that the management bridge still has:

```text
vmbr0
192.168.178.10/24
Gateway: 192.168.178.1
Physical bridge port: nic0
```

These are the recorded HYP01 bootstrap-management settings. They must be revalidated if the management design changes.

### Step 3 — Compare against the known-good configuration

The expected `vmbr0` section is equivalent to:

```text
auto vmbr0
iface vmbr0 inet static
    address 192.168.178.10/24
    gateway 192.168.178.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
```

Do not overwrite the entire networking configuration merely because one setting differs.

First determine whether the difference is responsible for the failure.

### Step 4 — Restore the known-good configuration

If an approved pre-change copy of `/etc/network/interfaces` exists, compare it with the current configuration.

A useful command is:

```bash
sudo diff -u /etc/network/interfaces.pre-nim21 /etc/network/interfaces
```

This example assumes a backup was previously created at that path. The backup filename is illustrative and was not verified as part of the original NIM-21 implementation.

`diff -u` displays differences between two files using unified output.

If the backup is verified and appropriate for the current host state, restore the required configuration through the approved console recovery procedure.

Do not blindly overwrite the current file if subsequent approved changes have been made.

For a simple bridge misconfiguration, prefer correcting only the affected settings.

### Step 5 — Apply and validate

Use the supported Proxmox networking workflow where available.

If operating from the physical console, confirm that the installed networking management utilities and their supported recovery procedure are understood before applying changes.

Avoid restarting all networking services or rebooting repeatedly without identifying the failure.

Verify:

```bash
ip -br addr
ip route
```

Then test the Proxmox Web UI and SSH from the approved management workstation.

### Escalation

If local management networking cannot be restored:

- Preserve console access.
- Collect network configuration and relevant system logs.
- Compare the active configuration against the approved design.
- Escalate before attempting destructive network reconfiguration or host reinstallation.

## 7. Scenario C — FW01 attached to the wrong bridge

### Symptoms

- External NIC cannot reach the expected upstream network when enabled.
- Internal NIC unexpectedly receives home-LAN DHCP configuration.
- Proxmox Hardware view does not match the approved interface schedule.

### Recovery procedure

1. Open **FW01 → Hardware**.
2. Inspect both network devices.
3. Confirm:

| Device | Correct bridge |
|---|---|
| `net0` | `vmbr0` |
| `net1` | `vmbr1` |

4. Record the existing MAC addresses and configuration.
5. Shut down FW01 gracefully if required to make the correction safely.
6. Edit the incorrect virtual NIC.
7. Select the approved bridge.
8. Apply the change.
9. Start FW01 if it was shut down.
10. Verify the NIC mapping in RHEL.

Inside FW01:

```bash
ip -br link
nmcli device status
```

`ip -br link` displays network interface names, link state and hardware information in brief format.

Confirm that the guest NIC identities correspond to the intended Proxmox virtual adapters.

**Do not enable final network routing or addressing as part of this recovery.**

## 8. Scenario D — Unexpected DHCP or network connectivity

### Symptoms

- `ens18` obtains a DHCP address after reboot.
- An unexpected IPv4 default route appears.
- A connection profile activates automatically.
- FW01 becomes reachable on a network where connectivity was not yet approved.

### Step 1 — Inspect NetworkManager

From the FW01 Proxmox console:

```bash
nmcli device status
nmcli connection show
ip -br addr
ip route
```

Determine which interface and connection profile are active.

### Step 2 — Identify the active profile

For the external interface:

```bash
nmcli -g GENERAL.CONNECTION device show ens18
```

This returns the name of the active NetworkManager connection profile.

If the interface is already disconnected, identify its saved profile from `nmcli connection show` instead.

### Step 3 — Disable autoconnect

If the external profile is active:

```bash
sudo nmcli connection modify "$(nmcli -g GENERAL.CONNECTION device show ens18)" connection.autoconnect no
```

This prevents the selected saved profile from activating automatically.

If the connection is no longer active, substitute the verified saved profile name or UUID instead of using the command substitution.

### Step 4 — Disconnect the interface

```bash
sudo nmcli device disconnect ens18
```

This disconnects the network interface without removing the virtual NIC.

### Step 5 — Validate

```bash
nmcli device status
ip -br addr
ip route
```

Expected for the NIM-21 baseline:

- `ens18` disconnected.
- `ens19` disconnected.
- No assigned IPv4 address.
- No IPv4 default route.

If the profile activates again after reboot, inspect all saved connection profiles and their autoconnect settings.

### Reversal of the temporary baseline

When a later approved NIM requires external connectivity, the saved profile may be modified and reactivated using the approved network configuration procedure.

Do not simply re-enable DHCP if the subsequent design requires static addressing.

## 9. Scenario E — FW01 fails to boot

### Symptoms

- VM fails to start.
- UEFI firmware cannot find a bootable operating system.
- Bootloader errors.
- Kernel panic.
- Emergency mode.
- Filesystem mount failures.

### Step 1 — Inspect Proxmox

Open **FW01 → Summary** and **Hardware**.

Confirm:

- VM ID `101` exists.
- The operating-system disk is attached.
- OVMF firmware is configured.
- The EFI disk exists.
- The virtual disk is on the expected storage.
- The VM boot order includes the installed operating-system disk.

Do not delete or recreate the EFI disk as an initial troubleshooting step.

### Step 2 — Inspect the console

Open the Proxmox console and record the exact error.

Determine whether the failure occurs during:

1. Virtual machine startup.
2. UEFI firmware.
3. Bootloader execution.
4. Kernel loading.
5. Filesystem mounting.
6. Systemd startup.

This distinction determines the appropriate recovery method.

### Step 3 — Attempt a controlled recovery

If RHEL reaches emergency mode or a recovery shell, inspect the displayed error and available logs.

If the installed system cannot boot, attach the approved RHEL installation ISO and use its supported rescue environment.

**Do not reinstall immediately.**

The existing operating-system disk may still contain recoverable configuration and data.

### Step 4 — Verify storage

Where available in a suitable recovery environment:

```bash
lsblk -f
sudo pvs
sudo vgs
sudo lvs
```

Check whether the expected disk, partitions, volume group and logical volumes are visible.

Expected:

- EFI partition on `sda1`.
- `/boot` partition on `sda2`.
- LVM physical volume on `sda3`.
- Volume group `vg_os`.
- Logical volumes `lv_root` and `lv_swap`.

Device names may differ in a rescue environment. Identify devices by filesystem, UUID and LVM metadata rather than relying exclusively on `/dev/sda`.

### Step 5 — Select the recovery method

Choose according to the diagnosed failure:

- Incorrect boot order: correct Proxmox boot configuration.
- Missing or incorrect virtual hardware: restore approved hardware settings.
- Bootloader failure: follow the supported RHEL UEFI bootloader recovery procedure.
- Missing LVM activation: investigate the volume group and logical volumes.
- Filesystem failure: follow the supported filesystem-specific recovery procedure.
- Unrecoverable guest installation: restore a verified backup or rebuild.

Do not use filesystem repair commands until the filesystem type and failure have been confirmed.

In particular, XFS and FAT/VFAT require different recovery tools and procedures.

## 10. Scenario F — LVM or filesystem problems

### Symptoms

- Root filesystem fails to mount.
- Expected logical volume is missing.
- Swap does not activate.
- Volume group is unavailable.
- Filesystem reports corruption or I/O errors.

### Initial diagnostic commands

```bash
lsblk -f
sudo pvs
sudo vgs
sudo lvs
findmnt /
findmnt /boot
findmnt /boot/efi
swapon --show
```

These commands inspect the storage configuration without deliberately modifying it.

### Recovery considerations

Before making changes:

1. Confirm the affected device.
2. Determine whether the issue is LVM metadata, filesystem damage or virtual disk availability.
3. Confirm the availability of a verified backup.
4. Preserve existing data where possible.
5. Use a suitable rescue environment if the filesystem cannot be safely inspected while mounted.
6. Consult the applicable RHEL storage recovery procedure.

**Do not run destructive formatting commands such as `mkfs` during diagnosis.**

Do not use `fsck` indiscriminately against XFS. XFS requires its own diagnostic and repair approach, and repairs must follow the appropriate offline recovery procedure.

If data integrity cannot be established, stop and escalate before attempting repair.

## 11. Scenario G — QEMU Guest Agent unavailable

### Symptoms

- Guest-agent status is inactive or failed.
- Proxmox reports that guest-agent communication is unavailable.
- Guest information is not available through Proxmox.

### Step 1 — Inspect the service

Inside FW01:

```bash
systemctl status qemu-guest-agent
```

Confirm whether the service is loaded, enabled, active or failed.

### Step 2 — Inspect recent logs

```bash
sudo journalctl -u qemu-guest-agent -b --no-pager
```

Explanation:

- `journalctl` reads the systemd journal.
- `-u qemu-guest-agent` filters messages for the specified service.
- `-b` limits output to the current boot.
- `--no-pager` prints directly to the terminal instead of opening a pager.

### Step 3 — Restart if appropriate

If the service is installed and the diagnostics indicate a restart is safe:

```bash
sudo systemctl restart qemu-guest-agent
```

This requests that systemd restart the guest-agent service.

### Step 4 — Validate

```bash
systemctl is-active qemu-guest-agent
```

Expected output:

```text
active
```

If the service repeatedly fails, collect the logs and investigate the guest-agent communication channel and Proxmox configuration.

Do not reinstall RHEL to resolve a guest-agent problem.

## 12. Scenario H — Restore or rebuild FW01

This scenario applies when the installed operating system cannot be recovered economically or when an approved rebuild is required.

### 12.1 Preferred recovery order

1. Correct a specific configuration error.
2. Recover the installed operating system.
3. Restore a verified VM backup, if available.
4. Rebuild the VM using `deployment.md`.

### 12.2 Backup restoration

Before restoring:

- Confirm the backup exists.
- Confirm the backup belongs to FW01.
- Verify its timestamp and recovery suitability.
- Determine whether restoration would overwrite existing VM data.
- Confirm the correct target VM ID and storage.
- Obtain approval for any destructive replacement.

Use the supported Proxmox backup restoration workflow.

After restoration, validate the same acceptance criteria used during NIM-21.

### 12.3 Rebuild

If no suitable backup exists:

1. Preserve the existing VM and virtual disk where possible.
2. Confirm whether the existing VM ID can safely be reused.
3. Review the approved virtual hardware specification.
4. Confirm the installation ISO is available.
5. Follow `deployment.md`.
6. Recreate the approved RHEL storage and hostname configuration.
7. Restore approved credentials and configuration through the restricted operational process.
8. Run the complete acceptance checklist.
9. Record any deviations from the original build.

A rebuild is not considered complete until the guest has successfully rebooted and passed the required validation tests.

## 13. Scenario I — Full NIM-21 rollback

A full rollback removes both FW01 and the internal bridge introduced by this ticket.

This is a **destructive reversal**, not a routine troubleshooting action.

It requires explicit approval.

### 13.1 Dependency assessment

Before removing anything, confirm:

- FW01 is not providing active routing or firewall services.
- No Northwind workloads depend on FW01.
- No other VMs use `vmbr1`.
- No required data remains solely on FW01.
- Backups or required evidence have been preserved.
- Subsequent NIMs have not introduced dependencies on the bridge.

The original NIM-21 deployment had no production workloads depending on FW01. This must not be assumed in future.

### 13.2 Remove FW01

Using the Proxmox Web UI:

1. Identify VM ID `101` and confirm its name is FW01.
2. Shut down the VM gracefully.
3. Verify it is stopped.
4. Preserve required configuration, disks and evidence.
5. Obtain approval for VM removal.
6. Use the supported Proxmox VM removal workflow.
7. Review any option that would remove associated disks or other resources.
8. Confirm that only the approved FW01 resources will be removed.
9. Complete the removal.

Do not remove a virtual disk if its contents may still be required.

### 13.3 Remove `vmbr1`

Only after all attached workloads have been removed or migrated:

1. Navigate to **HYP01 → System → Network**.
2. Select `vmbr1`.
3. Confirm it is no longer required.
4. Click **Remove**.
5. Review pending changes.
6. Confirm `vmbr0` is unaffected.
7. Apply the configuration.

### 13.4 Validate the rollback

On HYP01:

```bash
ip -br addr
cat /etc/network/interfaces
sudo qm list
```

Confirm:

- HYP01 management connectivity remains operational.
- `vmbr0` retains the approved configuration.
- FW01 is no longer present if removal was approved and completed.
- `vmbr1` is absent if its removal was approved and completed.
- Other virtual machines and storage remain unaffected.

Record the completed rollback and any residual resources.

## 14. Post-recovery validation

Following any recovery action, validate the relevant components.

### HYP01

- [ ] Proxmox Web UI accessible.
- [ ] Approved SSH management access operational.
- [ ] `vmbr0` management address correct.
- [ ] Default route correct.
- [ ] Physical uplink correct.
- [ ] `vmbr1` configuration matches the approved target state.
- [ ] No unintended physical uplink on `vmbr1`.
- [ ] No unintended host IP on `vmbr1`.

### FW01

- [ ] VM starts successfully.
- [ ] Console login works.
- [ ] Hostname correct.
- [ ] Operating-system release correct.
- [ ] EFI and boot filesystems mounted.
- [ ] Root logical volume mounted.
- [ ] Swap active.
- [ ] LVM free capacity understood.
- [ ] External NIC attached to `vmbr0`.
- [ ] Internal NIC attached to `vmbr1`.
- [ ] NetworkManager configuration matches the approved phase.
- [ ] SELinux enforcing.
- [ ] QEMU Guest Agent active.
- [ ] Controlled reboot successful.

The network-state checks must reflect the **currently approved design**, not necessarily the disconnected baseline from NIM-21.

## 15. Recovery documentation and evidence

After recovery, update the incident or change record with:

- Incident reference.
- Start and end times.
- Affected components.
- Root cause, if established.
- Recovery actions performed.
- Commands executed.
- Configuration changes.
- Validation results.
- Data loss or downtime, if any.
- Remaining risks.
- Follow-up actions.
- Approval and closure information.

Update the GitHub documentation if the recovery process revealed missing steps, incorrect assumptions or changed architecture.

Never publish credentials, recovery secrets or sensitive backup contents.

## 16. Recovery testing status

| Recovery capability | Current evidence |
|---|---|
| HYP01 controlled reboot | Successfully tested during NIM-21 |
| `vmbr1` persistence | Successfully tested during NIM-21 |
| HYP01 management after reboot | Successfully tested |
| FW01 controlled reboot | Successfully tested |
| FW01 console login | Successfully tested |
| FW01 storage persistence | Successfully tested |
| FW01 network-state persistence | Successfully tested |
| Guest-agent persistence | Successfully tested |
| HYP01 management-loss recovery | Not tested |
| Bridge configuration rollback | Not tested |
| FW01 backup restoration | Not tested |
| RHEL rescue/bootloader recovery | Not tested |
| Full NIM-21 reversal | Not tested |

**Important:** A documented recovery procedure is not equivalent to a tested recovery capability.

The untested scenarios should be reviewed and exercised through future approved recovery drills.

## 17. Glossary

| Term | Definition |
|---|---|
| Acceptance criteria | Conditions that must be satisfied before work can be formally accepted. |
| Backup | A recoverable copy of data or system state. |
| Bootloader | Software responsible for loading the operating-system kernel during startup. |
| Bridge | A software-based Layer 2 switching component. |
| Console access | Direct access to a system's interactive display and keyboard, independent of guest network services. |
| DHCP | Dynamic Host Configuration Protocol; automatically assigns network configuration. |
| EFI System Partition | A partition containing files required for UEFI boot. |
| Emergency mode | A limited Linux operating state used when normal startup cannot complete. |
| Filesystem | The structure used by an operating system to organise and access stored data. |
| Guest agent | Software inside a VM that communicates with its hypervisor. |
| Hypervisor | A platform that hosts and manages virtual machines. |
| LVM | Logical Volume Manager; provides flexible Linux storage management. |
| NetworkManager | The service used by RHEL to manage network connections and profiles. |
| Out-of-band management | Administrative access that does not depend on the system's normal production network path. |
| Recovery point | A saved state from which data or a system can be restored. |
| Rollback | Reversal of a change to restore an earlier approved state. |
| RPO | Recovery Point Objective; the maximum acceptable amount of data loss measured in time. |
| RTO | Recovery Time Objective; the target time for restoring a service. |
| SELinux | Security-Enhanced Linux; a mandatory access-control security framework. |
| systemd | Linux system and service manager. |
| UEFI | Modern firmware interface used during system startup. |
| VLAN | Virtual Local Area Network; a logical Layer 2 network segment. |
| Volume group | An LVM storage pool containing capacity available for logical volumes. |
| XFS | A journaling filesystem commonly used on RHEL. |

---

## 18. Final recovery statement

NIM-21 introduced a standalone RHEL virtual machine and an isolated virtual network bridge.

At the completion of the ticket, FW01 does not yet support production network services. Consequently, failures can generally be investigated without disrupting Northwind application traffic.

The preferred recovery approach is to restore the affected component while preserving HYP01 management access, existing VM disks and approved configurations.

As Northwind's infrastructure expands, this recovery guide must be reviewed to account for new service dependencies, backups, recovery objectives and operational requirements.
