# NIM-22 — Rollback and Recovery Guide

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Sprint:** Sprint 2 — Initial Network Infrastructure  
**Jira Issue:** NIM-22  
**Component:** FW01 — Firewall and Network Gateway  
**Operating System:** RHEL 10.2  
**Hypervisor:** HYP01 — Proxmox VE  
**VM ID:** 101  
**Document Type:** Rollback and Recovery Procedure  
**Status:** Prepared — recovery scenarios not yet actively tested

---

## 1. Purpose

This document defines the procedures for recovering FW01 from configuration failures introduced during NIM-22.

The implementation established:

- Static IPv4 connectivity on the external interface.
- An internal VLAN trunk.
- Four VLAN gateway interfaces.
- Persistent NetworkManager connection profiles.
- Restrictive firewalld zone assignments.
- IPv4-only networking.
- A security baseline with IPv4 forwarding disabled.

The objective of this document is to restore the last known working configuration while protecting HYP01 management connectivity and avoiding unnecessary changes to unrelated infrastructure.

## 2. Scope

### In Scope

Recovery from:

- Incorrect static IPv4 addressing.
- Loss of external connectivity.
- Incorrect VLAN IDs or gateway addresses.
- VLAN trunk configuration errors.
- NetworkManager connection failures.
- Incorrect autoconnect settings.
- Unexpected firewall zone assignments.
- Missing interfaces after reboot.
- Accidental IPv4 forwarding changes.
- Severe configuration failures requiring snapshot rollback.

### Out of Scope

- Physical home router failure.
- HYP01 operating system recovery.
- RHEL installation or storage recovery.
- Production NAT and routing policy rollback.
- Production firewall rule rollback.
- Recovery of other Northwind virtual machines.

These are covered by separate operational procedures or future NIMs.

---

## 3. Approved Recovery Baseline

The following configuration represents the expected NIM-22 end state.

| Interface | Connection Profile | IPv4 Address | VLAN | Zone |
|---|---|---|---|---|
| ens18 | ens18 | 192.168.178.11/24 | Untagged | drop |
| ens19 | ens19 | None | Parent | drop |
| ens19.100 | vlan100-management | 10.20.10.1/24 | 100 | drop |
| ens19.200 | vlan200-servers | 10.20.20.1/24 | 200 | drop |
| ens19.300 | vlan300-users | 10.20.30.1/24 | 300 | drop |
| ens19.400 | vlan400-peripherals | 10.20.40.1/24 | 400 | drop |

Additional requirements:

- Default IPv4 gateway: `192.168.178.1`
- DNS server: `192.168.178.1`
- IPv6 disabled on all six NetworkManager profiles
- Autoconnect enabled on all six profiles
- IPv4 forwarding disabled
- FW01 `net0` connected to `vmbr0`
- FW01 `net1` connected to `vmbr1`
- Permitted internal VLANs: 100, 200, 300 and 400
- No IPv4 address assigned to the untagged `ens19` parent

**Important:** This is the NIM-22 configuration baseline, not a declaration that inter-VLAN routing or production firewall policy has been implemented.

---

## 4. Recovery Principles

Recovery must follow these principles:

1. **Preserve management access.** Keep the Proxmox Web UI and FW01 console available before changing network configuration.
2. **Diagnose before modifying.** Determine whether the fault exists in the guest, virtual network adapter, Proxmox bridge or upstream network.
3. **Prefer targeted recovery.** Correct a single connection profile before considering a full VM rollback.
4. **Avoid unnecessary hypervisor changes.** Do not restart HYP01 networking to resolve an isolated FW01 issue.
5. **Preserve evidence.** Capture relevant command output and record the original failure.
6. **Validate recovery.** A command completing successfully does not prove the service has been restored.
7. **Respect subsequent changes.** A NIM-22 snapshot must not be restored after later work without assessing the impact on NIM-23 and subsequent tickets.

### Recovery order

Use the following escalation sequence:

**Inspect → Correct profile → Reactivate or reapply → Validate → Restore saved configuration → Snapshot rollback (last resort).**

---

## 5. Recovery Access

### 5.1 Primary recovery method — Proxmox console

Access the Proxmox Web UI using the HYP01 management address:

`https://192.168.178.10:8006`

Select:

**HYP01 → VM 101 (FW01) → Console**

Log in using the approved local administrative account.

The console is the preferred recovery method because it does not depend on FW01's guest network connectivity.

### 5.2 Alternative access — HYP01 SSH

Where appropriate, connect to HYP01 using the established SSH alias:

```bash
ssh northwind-admin
```

**Command explanation**

- `ssh` — Opens an encrypted remote shell session.
- `northwind-admin` — The locally configured SSH alias for the HYP01 administrative connection.

This accesses HYP01, not FW01.

### 5.3 Verify VM state

Run on **HYP01**:

```bash
sudo qm status 101
```

**Command explanation**

- `sudo` — Executes the command with elevated privileges.
- `qm` — Proxmox QEMU/KVM virtual machine management utility.
- `status` — Reports the VM's runtime state.
- `101` — FW01's virtual machine ID.

Expected:

```text
status: running
```

If FW01 is stopped unexpectedly, investigate the cause before starting it.

---

## 6. Scenario A — External Connectivity Failure

### Symptoms

- FW01 cannot reach `192.168.178.1`.
- External IP connectivity fails.
- DNS resolution fails.
- `ens18` has no address or an incorrect address.
- The default route is missing.

### Step 1 — Inspect interface status

Run on **FW01**:

```bash
nmcli device status
ip -br addr show ens18
ip route
```

**Command explanation**

- `nmcli device status` — Shows the NetworkManager state of each device.
- `ip` — Linux network configuration and inspection utility.
- `-br` — Produces brief output.
- `addr show ens18` — Displays addresses assigned to `ens18`.
- `ip route` — Displays the IPv4 routing table.

Expected:

- `ens18` connected.
- Address `192.168.178.11/24`.
- Default route via `192.168.178.1`.

### Step 2 — Inspect the saved connection

```bash
nmcli -f connection.id,connection.autoconnect,ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns,ipv6.method,connection.zone connection show ens18
```

**Command explanation**

- `-f` — Limits output to the selected fields.
- `connection show ens18` — Displays the saved profile named `ens18`.

Verify that the profile contains the approved addressing, gateway, DNS and firewall zone.

### Step 3 — Correct incorrect configuration

If the saved settings are incorrect:

```bash
sudo nmcli connection modify ens18 \
  ipv4.method manual \
  ipv4.addresses 192.168.178.11/24 \
  ipv4.gateway 192.168.178.1 \
  ipv4.dns 192.168.178.1 \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect yes
```

**Command explanation**

- `connection modify` — Updates an existing NetworkManager profile.
- `ipv4.method manual` — Enables static IPv4 addressing.
- `ipv4.addresses` — Sets the static IPv4 address and prefix.
- `ipv4.gateway` — Sets the default gateway.
- `ipv4.dns` — Sets the DNS server.
- `ipv6.method disabled` — Disables IPv6 for the connection.
- `connection.zone drop` — Associates the connection with the restrictive firewalld zone.
- `connection.autoconnect yes` — Enables automatic activation.

The backslash continues the command onto the following line.

### Step 4 — Apply the configuration

If the interface is already connected:

```bash
sudo nmcli device reapply ens18
```

**Command explanation**

- `device reapply` — Attempts to apply modified connection settings to an active device without fully disconnecting it.

If the interface is disconnected or reapply cannot apply the required changes:

```bash
sudo nmcli connection up ens18
```

**Command explanation**

- `connection up` — Activates the named NetworkManager profile.

Perform potentially disruptive activation changes from the Proxmox console.

### Step 5 — Validate

```bash
ip -br addr show ens18
ip route
ping -c 4 192.168.178.1
ping -c 4 1.1.1.1
getent ahostsv4 redhat.com
```

**Command explanation**

- `ping` — Tests reachability using ICMP echo requests.
- `-c 4` — Sends four requests.
- `getent` — Queries system name-service databases.
- `ahostsv4` — Requests IPv4 address information.

Successful recovery requires the expected address and route, successful router and external connectivity, and working DNS resolution.

**Caution:** If `ens18` has the correct address but the router is unreachable, investigate Layer 2 connectivity, Proxmox adapter attachment, upstream connectivity and neighbor discovery before changing the IP configuration again.

---

## 7. Scenario B — VLAN Interface Missing or Disconnected

### Symptoms

- One or more `ens19.<VLAN>` interfaces are missing.
- A VLAN interface is disconnected.
- An expected connected route is absent.
- A VLAN interface fails to activate after reboot.

### Step 1 — Inspect devices and profiles

Run on **FW01**:

```bash
nmcli device status
nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT connection show
ip -br addr
```

Identify the affected VLAN profile.

### Step 2 — Inspect VLAN properties

Example for VLAN 200:

```bash
nmcli -f connection.id,connection.interface-name,connection.autoconnect,vlan.parent,vlan.id,ipv4.method,ipv4.addresses,ipv4.never-default,ipv6.method,connection.zone connection show vlan200-servers
```

Confirm:

- Parent device: `ens19`
- VLAN ID: `200`
- Interface: `ens19.200`
- IPv4 address: `10.20.20.1/24`
- Autoconnect: `yes`
- Zone: `drop`

### Step 3 — Correct the profile

For an incorrect VLAN ID:

```bash
sudo nmcli connection modify vlan200-servers vlan.id 200
```

For an incorrect IPv4 address:

```bash
sudo nmcli connection modify vlan200-servers ipv4.addresses 10.20.20.1/24
```

For disabled autoconnect:

```bash
sudo nmcli connection modify vlan200-servers connection.autoconnect yes
```

**Command explanation**

- `vlan.id` — Specifies the VLAN tag.
- `ipv4.addresses` — Specifies the IPv4 address and prefix.
- `connection.autoconnect` — Controls automatic activation.

These commands modify only the named connection profile.

### Step 4 — Activate and validate

```bash
sudo nmcli connection up vlan200-servers
nmcli device status
ip -d link show ens19.200
ip -br addr show ens19.200
ip route
```

**Command explanation**

- `ip -d` — Displays detailed interface properties, including VLAN tagging.
- `link show` — Displays network link configuration.

If an active VLAN's ID or parent was changed, a full connection reactivation may be necessary. Use the console and an approved maintenance window where connectivity could be affected.

### Step 5 — Repeat for other VLANs if necessary

| VLAN | Profile | Expected Address |
|---|---|---|
| 100 | vlan100-management | 10.20.10.1/24 |
| 200 | vlan200-servers | 10.20.20.1/24 |
| 300 | vlan300-users | 10.20.30.1/24 |
| 400 | vlan400-peripherals | 10.20.40.1/24 |

Do not recreate all four VLANs if only one profile is affected.

---

## 8. Scenario C — Incorrect Proxmox VLAN Trunk

### Symptoms

- Guest VLAN interfaces are configured correctly.
- VLAN interfaces report `UP`.
- Tagged traffic fails to reach other virtual machines.
- Expected VLANs are absent from the host-side bridge port.
- A Proxmox configuration change has removed permitted VLAN IDs.

**Important:** An interface showing `UP` does not prove that tagged traffic is passing end to end.

### Step 1 — Inspect VM networking

Run on **HYP01**:

```bash
sudo qm config 101 --current
```

**Command explanation**

- `qm config 101` — Displays FW01's VM configuration.
- `--current` — Requests the current configuration view.

Expected `net1`:

```text
net1: virtio=BC:24:11:D9:A0:51,bridge=vmbr1,firewall=1,trunks=100;200;300;400
```

### Step 2 — Inspect bridge VLAN membership

```bash
sudo bridge vlan show dev fwpr101p1
```

Expected VLAN membership includes:

- 100
- 200
- 300
- 400

The host-side interface name must be verified before using this command.

### Step 3 — Restore the approved trunk

If `net1` has incorrect settings:

```bash
sudo qm set 101 \
  --net1 'virtio=BC:24:11:D9:A0:51,bridge=vmbr1,firewall=1,trunks=100;200;300;400'
```

**Command explanation**

- `qm set` — Changes VM configuration.
- `--net1` — Specifies the second virtual NIC.
- `virtio` — Defines the virtual adapter and its MAC address.
- `bridge=vmbr1` — Connects the adapter to the internal bridge.
- `firewall=1` — Retains the Proxmox firewall option.
- `trunks` — Specifies permitted VLAN IDs.

**Caution:** This replaces the complete `net1` configuration. Confirm the approved MAC address and all existing adapter options before applying it.

The command may update the saved configuration without immediately changing every aspect of the running VM's network path. Inspect both the VM's pending/current configuration and live bridge membership afterward. A controlled VM restart may be required for some changes.

### Step 4 — Validate

```bash
sudo qm config 101 --current
sudo bridge vlan show dev fwpr101p1
sudo qm status 101
```

If another VM is available on the same VLAN, perform an end-to-end connectivity test. That test is not part of the completed NIM-22 evidence and must be recorded separately.

---

## 9. Scenario D — Internal Parent Interface Incorrectly Addressed

### Symptoms

- `ens19` has an unexpected IPv4 address.
- The internal parent has acquired a default route.
- VLAN interfaces coexist with an unintended untagged Layer 3 network.

### Step 1 — Inspect the parent

Run on **FW01**:

```bash
ip -br addr show ens19
ip route
nmcli connection show ens19
```

### Step 2 — Restore the intended configuration

```bash
sudo nmcli connection modify ens19 \
  ipv4.method disabled \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect yes
```

### Step 3 — Reactivate if required

```bash
sudo nmcli connection up ens19
```

If the active connection retains unwanted addressing, a controlled disconnect and reconnection may be required:

```bash
sudo nmcli device disconnect ens19
sudo nmcli connection up ens19
```

**Command explanation**

- `device disconnect` — Disconnects the network device and deactivates its active connection.
- `connection up` — Reactivates the selected saved profile.

**Warning:** Disconnecting `ens19` can interrupt every VLAN subinterface using it. Perform this only from the Proxmox console, with the impact understood.

### Step 4 — Validate

```bash
ip -br addr show ens19
nmcli device status
ip route
```

Confirm that the parent has no IPv4 address, all four VLANs are connected, and no unintended default route exists.

---

## 10. Scenario E — Incorrect Firewall Zone Assignment

### Symptoms

- An interface is unexpectedly associated with `public` or another zone.
- Firewall behavior differs from the NIM-22 baseline.
- An interface changes zone after reboot.

### Step 1 — Inspect runtime zones

Run on **FW01**:

```bash
sudo firewall-cmd --get-active-zones
```

Expected:

All six FW01 interfaces assigned to `drop`.

### Step 2 — Inspect saved NetworkManager settings

Example:

```bash
nmcli -f connection.id,connection.zone connection show vlan300-users
```

### Step 3 — Restore the approved zone

```bash
sudo nmcli connection modify vlan300-users connection.zone drop
```

### Step 4 — Apply and verify

For an active connection, first attempt:

```bash
sudo nmcli device reapply ens19.300
```

Then inspect:

```bash
sudo firewall-cmd --get-active-zones
```

If the zone assignment has not updated, use a controlled connection reactivation from the Proxmox console and verify again.

**Security note:** Do not move interfaces into a permissive firewall zone simply to make a connectivity test pass. Determine whether the traffic is expected and whether an approved rule exists.

---

## 11. Scenario F — Unexpected IPv4 Forwarding

### Symptoms

- `net.ipv4.ip_forward` reports `1`.
- FW01 forwards IPv4 packets despite NIM-22's intended baseline.
- A subsequent configuration change has altered the kernel setting.

### Step 1 — Inspect forwarding

Run on **FW01**:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 0
```

### Step 2 — Disable forwarding immediately

If forwarding is unexpectedly enabled:

```bash
sudo sysctl -w net.ipv4.ip_forward=0
```

**Command explanation**

- `-w` — Writes a new value to a kernel parameter.
- `net.ipv4.ip_forward=0` — Disables IPv4 forwarding.

This changes the runtime value. It does not necessarily correct the configuration that caused forwarding to be enabled.

### Step 3 — Inspect persistent settings

```bash
sudo grep -R -n 'net.ipv4.ip_forward' /etc/sysctl.conf /etc/sysctl.d/
```

**Command explanation**

- `grep` — Searches text for a pattern.
- `-R` — Searches directories recursively.
- `-n` — Includes line numbers.
- `/etc/sysctl.conf` — Traditional system-wide kernel parameter configuration.
- `/etc/sysctl.d/` — Directory containing additional persistent kernel parameter configuration files.

Review any matching files and determine which setting is responsible.

Do not remove or overwrite configuration belonging to another approved change without reviewing its ownership and impact.

### Step 4 — Validate

```bash
sysctl net.ipv4.ip_forward
```

Expected value: `0`.

**Important:** This procedure is appropriate for the NIM-22 baseline. Once NIM-23 intentionally enables routing, the approved baseline changes and this procedure must be revised.

---

## 12. Scenario G — Network Configuration Does Not Survive Reboot

### Symptoms

- VLAN interfaces disappear after restart.
- `ens18` reverts to DHCP or loses its address.
- One or more connection profiles remain inactive.
- The firewall zone assignment changes.

### Step 1 — Inspect profiles

Run on **FW01**:

```bash
nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT connection show
```

All six expected profiles should have autoconnect enabled.

### Step 2 — Check NetworkManager

```bash
systemctl status NetworkManager
```

**Command explanation**

- `systemctl status` — Displays a systemd service's state and recent status information.
- `NetworkManager` — The service managing network connections.

### Step 3 — Inspect recent logs

```bash
sudo journalctl -u NetworkManager -b --no-pager
```

**Command explanation**

- `journalctl` — Reads the systemd journal.
- `-u NetworkManager` — Filters messages to the NetworkManager service.
- `-b` — Restricts results to the current boot.
- `--no-pager` — Prints output directly rather than opening an interactive pager.

Look for profile activation failures, missing devices, invalid settings and other network-related errors.

### Step 4 — Restore autoconnect

Example:

```bash
sudo nmcli connection modify vlan400-peripherals connection.autoconnect yes
```

Repeat only for profiles that are incorrectly configured.

### Step 5 — Activate and verify

```bash
sudo nmcli connection up vlan400-peripherals
nmcli device status
ip -br addr
ip route
```

If the failure is recurring, investigate the underlying NetworkManager or device configuration rather than repeatedly activating profiles manually.

---

## 13. Scenario H — Full Snapshot Rollback

### When to use

Snapshot rollback is reserved for severe failures where targeted recovery is unsuccessful or impractical.

Examples:

- Multiple corrupted connection profiles.
- Extensive configuration changes with uncertain scope.
- FW01 cannot be restored to the approved baseline through the console.
- A controlled recovery to the pre-change state is required.

### Preconditions

Before restoring a snapshot:

1. Confirm the exact snapshot name and creation time.
2. Confirm that the snapshot represents the intended pre-NIM-22 state.
3. Identify all changes made since the snapshot.
4. Confirm that restoring the snapshot will not remove required work from later NIMs.
5. Record the reason for rollback.
6. Obtain approval appropriate to the change.
7. Preserve logs and evidence where feasible.

**Critical:** The pre-NIM-22 snapshot will not contain the NIM-22 network configuration. Restoring it reverses the work completed under this ticket.

A snapshot is not an independent backup and should not be treated as protection against hypervisor or storage failure.

### Step 1 — Inspect available snapshots

Run on **HYP01**:

```bash
sudo qm listsnapshot 101
```

**Command explanation**

- `qm listsnapshot` — Lists snapshots associated with a Proxmox VM.
- `101` — FW01's VM ID.

Confirm the exact snapshot name.

### Step 2 — Stop FW01 if required

A controlled shutdown can be initiated from the FW01 console:

```bash
sudo systemctl poweroff
```

**Command explanation**

- `systemctl poweroff` — Initiates an orderly operating system shutdown.

Alternatively, use the Proxmox Web UI's shutdown action.

Wait for the VM to stop.

Do not use a forced stop unless graceful shutdown has failed and the consequences have been assessed.

### Step 3 — Restore the snapshot

Preferred method:

**Proxmox Web UI → VM 101 → Snapshots → Select approved snapshot → Rollback**

Review the confirmation carefully.

Alternatively, the Proxmox CLI provides:

```bash
sudo qm rollback 101 <snapshot-name>
```

**Command explanation**

- `qm rollback` — Restores a VM snapshot.
- `101` — Identifies FW01.
- `<snapshot-name>` — Placeholder for the exact existing snapshot name.

Replace the placeholder before execution.

**Warning:** Snapshot rollback is destructive to changes made after the snapshot. Depending on snapshot type and configuration, VM disk and configuration state can be reverted.

### Step 4 — Start the VM

If FW01 is stopped:

```bash
sudo qm start 101
```

**Command explanation**

- `qm start` — Starts the specified virtual machine.
- `101` — FW01's VM ID.

### Step 5 — Validate the restored baseline

Use the Proxmox console to inspect:

```bash
nmcli device status
ip -br addr
ip route
sudo firewall-cmd --get-active-zones
sysctl net.ipv4.ip_forward
```

Compare the results against the **documented snapshot baseline**, not the completed NIM-22 configuration.

If the snapshot was created before VLAN deployment, the four VLAN interfaces should not be expected to exist.

Record the restored state and determine whether NIM-22 must be reimplemented.

---

## 14. Recovery Validation Checklist

After any recovery action, verify the relevant elements of the approved configuration.

| Check | Expected Result |
|---|---|
| FW01 VM | Running |
| External interface | `ens18` connected |
| External IPv4 | `192.168.178.11/24` |
| Default route | Via `192.168.178.1` |
| Internal parent | `ens19` connected, no IPv4 |
| VLAN 100 | `10.20.10.1/24` |
| VLAN 200 | `10.20.20.1/24` |
| VLAN 300 | `10.20.30.1/24` |
| VLAN 400 | `10.20.40.1/24` |
| NetworkManager autoconnect | Enabled for all six profiles |
| firewalld zones | All six interfaces in `drop` |
| IPv4 forwarding | Disabled |
| IPv6 on ens18 | No IPv6 addresses |
| Router connectivity | Successful |
| External IPv4 connectivity | Successful |
| DNS resolution | Successful |

For guest VLAN connectivity failures, also verify Proxmox trunk membership and perform an end-to-end test when a suitable client is available.

A full reboot validation should be performed after persistent configuration changes where operationally appropriate.

---

## 15. Recovery Evidence and Incident Recording

Any recovery action must be documented in the relevant Jira issue or incident record.

Capture:

- Date and time of the failure.
- Affected VM and interfaces.
- Symptoms and impact.
- Commands used to investigate.
- Relevant log excerpts.
- Root cause, if established.
- Recovery actions taken.
- Before-and-after configuration evidence.
- Validation results.
- Outstanding risks.
- Follow-up actions.

Example incident summary:

**Incident:** FW01 VLAN 200 unavailable after configuration change.

**Impact:** Server VLAN gateway interface unavailable.

**Investigation:** NetworkManager profile inspection identified an incorrect VLAN ID.

**Resolution:** Corrected the VLAN ID, reactivated the connection and validated the interface and connected route.

**Preventive action:** Add a VLAN ID verification check to the deployment procedure.

This is an illustrative incident template, not a claim that the incident occurred.

---

## 16. Recovery Testing Status

| Scenario | Procedure Documented | Actively Tested |
|---|---|---|
| External IPv4 configuration failure | Yes | No |
| Missing VLAN interface | Yes | No |
| Incorrect Proxmox VLAN trunk | Yes | No |
| Incorrect parent addressing | Yes | No |
| Incorrect firewalld zone | Yes | No |
| Unexpected IPv4 forwarding | Yes | No |
| NetworkManager reboot failure | Yes | No |
| Snapshot rollback | Yes | No |

During NIM-22, successful post-reboot persistence validation was completed.

However, deliberate fault injection and recovery testing were not performed. These procedures should therefore be treated as documented recovery plans rather than proven recovery outcomes.

---

## 17. Risks and Limitations

**Snapshot recovery**

The pre-change snapshot is useful for recovering the VM's earlier state, but restoring it would remove NIM-22 changes and potentially subsequent work.

**Guest and hypervisor configuration**

A guest VLAN profile cannot compensate for an incorrectly configured Proxmox bridge or trunk. Both layers must be inspected.

**Firewall security**

The `drop` zone provides a restrictive initial configuration, but it does not constitute the complete future firewall policy.

**Routing behavior**

IPv4 forwarding is intentionally disabled under NIM-22. The expected state will change during NIM-23 and this runbook must be updated accordingly.

**Unverified recovery procedures**

The scenarios are based on the documented configuration and available administration mechanisms. They have not been demonstrated through deliberate failure testing.

**Future infrastructure dependencies**

As additional servers are deployed, changes to FW01 may affect multiple business services. Recovery procedures must account for those dependencies.

---

## 18. Lessons and Operational Improvements

NIM-22 reinforces several enterprise administration principles:

- Network configuration should be implemented incrementally and validated after each change.
- Saved NetworkManager profiles and active interface state must both be checked.
- VLAN configuration spans multiple layers: hypervisor bridge, virtual NIC trunk and guest VLAN interface.
- Layer 2 connectivity and Layer 3 addressing are separate responsibilities.
- Console access is essential when modifying network settings.
- A successful command is not equivalent to a successful service restoration.
- Reboot validation is necessary to confirm persistent configuration.
- Recovery procedures should be prepared before the infrastructure becomes business-critical.

---

## 19. Final Recovery Position

At completion of NIM-22:

- FW01's network configuration was operational.
- Four internal VLAN interfaces were present.
- All connection profiles were configured for automatic activation.
- The external connection was operational.
- Firewall interfaces were assigned to the `drop` zone.
- IPv4 forwarding remained disabled.
- IPv6 was disabled on the external interface.
- The configuration survived a controlled reboot.
- A pre-change Proxmox snapshot was available.

**Recovery readiness:** Documented, with active recovery testing outstanding.

This document establishes the initial recovery procedures for FW01 networking and should be reviewed as routing, NAT, firewall policy and additional Northwind services are introduced.
