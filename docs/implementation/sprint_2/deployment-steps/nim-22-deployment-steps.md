# NIM-22 — Deployment Guide

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Sprint:** Sprint 2 — Initial Network Infrastructure  
**Jira Issue:** NIM-22  
**Component:** FW01 — Firewall and Network Gateway  
**Operating System:** RHEL 10.2  
**Hypervisor:** HYP01 — Proxmox VE  
**Implementation Status:** Completed and reboot-validated

---

## 1. Purpose

This guide documents the implementation of FW01's external interface, internal VLAN trunk and four VLAN gateway interfaces.

It records the configuration procedure, the Linux and Proxmox commands used, the purpose of their options, and the checks performed to validate the implementation.

The configuration establishes the network interface foundation required for future routing, firewall policy and internal service deployments.

**Important:** NIM-22 does not enable inter-VLAN routing, NAT or production firewall access policies.

## 2. Prerequisites

Before starting, confirm:

- HYP01 is operational and accessible through the Proxmox Web UI.
- FW01 (VM 101) has been deployed and is running RHEL 10.2.
- FW01 has two VirtIO network adapters.
- `net0` connects to `vmbr0`.
- `net1` connects to `vmbr1`.
- `vmbr1` is VLAN-aware and has no physical uplink.
- The NIM-19 addressing schedule is approved.
- Administrative console access to FW01 is available.
- A pre-change Proxmox snapshot has been taken and its successful creation verified.

The implementation was performed using the HYP01 SSH terminal and the FW01 Proxmox console.

The pre-change snapshot is a recovery aid, not a substitute for an independent backup.

## 3. Approved Network Configuration

| Interface | Purpose | IPv4 Address | VLAN |
|---|---|---|---|
| `ens18` | External connectivity | `192.168.178.11/24` | Untagged |
| `ens19` | Internal trunk parent | None | Parent |
| `ens19.100` | Management | `10.20.10.1/24` | 100 |
| `ens19.200` | Servers | `10.20.20.1/24` | 200 |
| `ens19.300` | Users | `10.20.30.1/24` | 300 |
| `ens19.400` | Peripherals | `10.20.40.1/24` | 400 |

External default gateway and DNS server: `192.168.178.1`.

All interfaces use the `drop` firewalld zone as their initial security baseline.

IPv6 is disabled in the NetworkManager profiles.

---

## 4. Inspect Existing Proxmox Networking

**Execution host: HYP01**

Before modifying the VM networking, inspect the hypervisor's current network configuration.

### 4.1 Inspect interface addresses

```bash
ip -br addr
```

**Command explanation**

- `ip` — Linux utility for inspecting and managing network interfaces, addresses and routes.
- `-br` — Brief output format, reducing unnecessary detail.
- `addr` — Displays IP address information.

**Purpose:** Confirm the current HYP01 interfaces and ensure the management IP is assigned to the expected bridge.

Observed configuration:

- `vmbr0` carries HYP01's management address `192.168.178.10/24`.
- `vmbr1` exists as the internal bridge without a configured IPv4 address.

### 4.2 Inspect persistent bridge configuration

```bash
cat /etc/network/interfaces
```

**Command explanation**

- `cat` — Displays the contents of a file.
- `/etc/network/interfaces` — The primary network configuration file used by this Proxmox installation.

**Purpose:** Verify that `vmbr1` is configured as an isolated VLAN-aware bridge.

Relevant configuration:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

**Configuration explanation**

- `auto vmbr1` — Brings the bridge up automatically.
- `iface vmbr1 inet manual` — Does not configure an IPv4 address automatically.
- `bridge-ports none` — No physical network interface is attached.
- `bridge-stp off` — Disables Spanning Tree Protocol on this bridge.
- `bridge-fd 0` — Sets the bridge forwarding delay to zero.
- `bridge-vlan-aware yes` — Enables VLAN-aware bridge operation.
- `bridge-vids 2-4094` — Defines the permitted VLAN range at the bridge configuration level.

**Important:** `bridge-ports none` is the critical setting preventing the internal bridge from having a direct physical uplink. It does not, by itself, replace end-to-end isolation testing.

### 4.3 Verify VLAN filtering

```bash
cat /sys/class/net/vmbr1/bridge/vlan_filtering
```

**Command explanation**

- `cat` — Displays file contents.
- `/sys/class/net/...` — A kernel-provided interface for inspecting network device state.
- `vlan_filtering` — Indicates whether VLAN filtering is enabled on the Linux bridge.

Expected result:

```text
1
```

`1` means enabled; `0` means disabled.

### 4.4 Inspect bridge VLAN membership

```bash
sudo bridge vlan show
```

**Command explanation**

- `sudo` — Runs the command with elevated privileges.
- `bridge` — Utility for inspecting and managing Linux bridge configuration.
- `vlan` — Selects VLAN-related operations.
- `show` — Displays current VLAN membership.

**Purpose:** Inspect the VLANs assigned to bridge ports and identify the FW01 virtual network adapter's host-side interface.

A VLAN-aware bridge can show a long list of VLAN IDs. This is expected when a broad VLAN range is configured.

---

## 5. Verify FW01 Virtual Network Adapters

**Execution host: HYP01**

```bash
sudo qm config 101
```

**Command explanation**

- `qm` — Proxmox command-line utility for managing QEMU/KVM virtual machines.
- `config` — Displays a VM's saved configuration.
- `101` — The Proxmox VM ID assigned to FW01.

**Purpose:** Confirm that the virtual machine has the correct network adapter assignments.

Relevant configuration:

```text
net0: virtio=BC:24:11:C6:70:7D,bridge=vmbr0,firewall=1
net1: virtio=BC:24:11:D9:A0:51,bridge=vmbr1,firewall=1
```

`net0` connects FW01 to the external bridge.

`net1` connects FW01 to the internal VLAN-aware bridge.

The `firewall=1` setting enables Proxmox firewall processing for that virtual NIC. It does not replace the guest operating system's firewalld configuration.

---

## 6. Configure Proxmox VLAN Trunking

**Execution host: HYP01**

FW01 must receive tagged traffic for VLANs 100, 200, 300 and 400.

### 6.1 Update the virtual network adapter

```bash
sudo qm set 101 \
  --net1 'virtio=BC:24:11:D9:A0:51,bridge=vmbr1,firewall=1,trunks=100;200;300;400'
```

**Command explanation**

- `qm set` — Changes the saved configuration of a Proxmox VM.
- `101` — Identifies FW01.
- `--net1` — Configures the VM's second virtual network adapter.
- `virtio=...` — Defines the VirtIO network adapter and its MAC address.
- `bridge=vmbr1` — Attaches the adapter to the internal bridge.
- `firewall=1` — Retains the Proxmox firewall option.
- `trunks=100;200;300;400` — Permits the four specified VLAN IDs on this adapter.

**Important:** The command supplies the complete `net1` configuration. Existing adapter settings must be preserved when using `qm set`.

The single quotes are important because semicolons have special meaning to the shell.

### 6.2 Verify the saved configuration

```bash
sudo qm config 101 --current | grep '^net'
```

**Command explanation**

- `qm config 101` — Displays FW01's configuration.
- `--current` — Requests the current configuration view.
- `|` — Sends the output of the first command into the next command.
- `grep` — Filters text matching a pattern.
- `'^net'` — Matches lines beginning with `net`.

Expected result includes:

```text
net1: virtio=BC:24:11:D9:A0:51,bridge=vmbr1,firewall=1,trunks=100;200;300;400
```

### 6.3 Verify the VM is running

```bash
sudo qm status 101
```

**Command explanation**

- `qm status` — Displays a virtual machine's runtime status.
- `101` — Identifies FW01.

Observed result:

```text
status: running
```

### 6.4 Verify bridge-port VLAN membership

```bash
sudo bridge vlan show dev fwpr101p1
```

**Command explanation**

- `bridge vlan show` — Displays VLAN membership.
- `dev` — Restricts the output to a particular network device.
- `fwpr101p1` — The relevant host-side Proxmox firewall bridge interface observed during implementation.

The output included VLAN IDs 100, 200, 300 and 400.

**Important:** `fwpr101p1` is a Proxmox host-side interface, not a RHEL guest interface. This command must run on HYP01. Generated interface names may differ between deployments.

---

## 7. Inspect FW01 Network Baseline

**Execution host: FW01**

Before assigning addresses, inspect the existing guest networking.

### 7.1 Inspect interfaces

```bash
ip -br addr
```

Confirm that the guest contains:

- `ens18` — External interface.
- `ens19` — Internal interface.

### 7.2 Inspect NetworkManager devices

```bash
nmcli device status
```

**Command explanation**

- `nmcli` — Command-line interface for NetworkManager.
- `device` — Selects operations concerning network devices.
- `status` — Displays each device's current state and active connection.

**Purpose:** Determine whether the interfaces are connected, disconnected or attempting automatic configuration.

### 7.3 Inspect saved connection profiles

```bash
nmcli connection show
```

**Command explanation**

- `connection` — Selects NetworkManager connection-profile operations.
- `show` — Lists saved connection profiles.

A connection profile is a stored set of network settings. It is not the same thing as a physical or virtual network device.

### 7.4 Inspect routes

```bash
ip route
```

**Command explanation**

- `ip route` — Displays the kernel's IPv4 routing table.

**Purpose:** Establish whether a default route or any unexpected routes already exist.

### 7.5 Verify forwarding

```bash
sysctl net.ipv4.ip_forward
```

**Command explanation**

- `sysctl` — Reads or changes Linux kernel parameters at runtime.
- `net.ipv4.ip_forward` — Controls whether the kernel forwards IPv4 packets between interfaces.

Expected:

```text
net.ipv4.ip_forward = 0
```

A value of `0` disables forwarding. A value of `1` enables it.

**Purpose:** Ensure that FW01 is not forwarding traffic between networks before the routing and firewall policies are implemented.

---

## 8. Inspect the Existing Firewall Baseline

**Execution host: FW01**

### 8.1 Verify firewalld is running

```bash
sudo systemctl is-active firewalld
```

**Command explanation**

- `systemctl` — Controls and inspects systemd-managed services.
- `is-active` — Returns the current service activity state.
- `firewalld` — The Linux firewall management service.

Expected:

```text
active
```

### 8.2 Inspect active firewall zones

```bash
sudo firewall-cmd --get-active-zones
```

**Command explanation**

- `firewall-cmd` — Command-line management tool for firewalld.
- `--get-active-zones` — Displays firewall zones currently associated with interfaces or source addresses.

**Purpose:** Identify which firewall policies are currently applied to FW01 interfaces.

### 8.3 Inspect the runtime firewall configuration

```bash
sudo firewall-cmd --list-all
```

**Command explanation**

- `--list-all` — Displays the complete configuration of the selected or default zone in the runtime firewall configuration.

### 8.4 Inspect permanent configuration

```bash
sudo firewall-cmd --permanent --list-all
```

**Command explanation**

- `--permanent` — Queries the persistent firewalld configuration rather than the current runtime configuration.
- `--list-all` — Displays the zone's configuration.

**Why this matters:** Runtime firewall changes and permanent firewall changes can differ. Persistent configuration must be considered when validating behavior after reboot.

---

## 9. Configure the External Interface

**Execution host: FW01**

The external interface connects FW01 to the home LAN through Proxmox `vmbr0`.

### 9.1 Configure static IPv4 addressing

```bash
sudo nmcli connection modify ens18 \
  ipv4.method manual \
  ipv4.addresses 192.168.178.11/24 \
  ipv4.gateway 192.168.178.1 \
  ipv4.dns 192.168.178.1 \
  connection.autoconnect yes
```

**Command explanation**

- `connection modify` — Changes a saved NetworkManager connection profile.
- `ens18` — The connection profile being modified.
- `ipv4.method manual` — Uses manually assigned IPv4 addressing instead of DHCP.
- `ipv4.addresses` — Sets the interface's IPv4 address and subnet prefix.
- `ipv4.gateway` — Sets the default IPv4 gateway.
- `ipv4.dns` — Defines the DNS server.
- `connection.autoconnect yes` — Enables automatic activation when the interface becomes available.

The backslash (`\`) continues the command onto the next line for readability.

### 9.2 Inspect the saved configuration

```bash
nmcli -f connection.id,connection.autoconnect,ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns connection show ens18
```

**Command explanation**

- `-f` — Specifies which fields NetworkManager should display.
- The comma-separated names — Select the precise connection properties to inspect.

**Purpose:** Verify the saved values before activating the interface.

### 9.3 Assign the restrictive firewall zone

The `ens18` connection was assigned to the firewalld `drop` zone.

```bash
sudo nmcli connection modify ens18 connection.zone drop
```

**Command explanation**

- `connection.zone` — Associates the NetworkManager profile with a firewalld zone.
- `drop` — Selects a restrictive zone that drops unsolicited inbound traffic.

This initial assignment prevents FW01 from accepting unsolicited inbound connections while the final firewall policy is pending.

The `drop` zone does not, by itself, block all traffic initiated by FW01.

### 9.4 Activate the connection

```bash
sudo nmcli connection up ens18
```

**Command explanation**

- `connection up` — Activates the specified NetworkManager profile.

**Purpose:** Apply the saved network configuration to the interface.

### 9.5 Validate IPv4 configuration

```bash
ip -br addr show ens18
ip route
nmcli device status
```

Expected:

- `ens18` has `192.168.178.11/24`.
- The default route uses `192.168.178.1`.
- NetworkManager reports `ens18` connected.

### 9.6 Validate connectivity

```bash
ping -c 4 192.168.178.1
ping -c 4 1.1.1.1
getent ahostsv4 redhat.com
```

**Command explanation**

- `ping` — Sends ICMP echo requests to test network reachability.
- `-c 4` — Sends four requests before stopping.
- `192.168.178.1` — Tests reachability of the local router.
- `1.1.1.1` — Tests external IP connectivity without depending on DNS.
- `getent` — Queries system databases using the configured name-service mechanisms.
- `ahostsv4` — Retrieves IPv4 address information for a hostname.
- `redhat.com` — The DNS test hostname.

**Observed results:**

- Router: 4/4 successful.
- External IP: 4/4 successful.
- DNS: IPv4 addresses returned.

---

## 10. Configure the Internal Parent Interface

**Execution host: FW01**

The internal interface `ens19` connects to `vmbr1` and carries tagged traffic for the four Northwind VLANs.

It does not require an IPv4 address.

### 10.1 Disable IP addressing on the parent

```bash
sudo nmcli connection modify ens19 \
  ipv4.method disabled \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect yes
```

**Command explanation**

- `ipv4.method disabled` — Prevents NetworkManager from configuring IPv4 on the parent.
- `ipv6.method disabled` — Disables IPv6 configuration on the parent.
- `connection.zone drop` — Assigns the restrictive firewall zone.
- `connection.autoconnect yes` — Enables activation after reboot.

**Why this configuration?**

The parent interface provides Layer 2 connectivity. The VLAN subinterfaces will hold the Layer 3 gateway addresses.

Disabling IP addressing on the parent does not create VLAN isolation by itself. Isolation is provided by VLAN tagging and filtering, together with the later routing and firewall policies.

### 10.2 Activate the parent interface

```bash
sudo nmcli connection up ens19
```

### 10.3 Verify

```bash
nmcli device status
ip -br addr show ens19
ip route
```

Expected:

- `ens19` connected.
- No IPv4 address on `ens19`.
- No additional default route.

---

## 11. Create VLAN 100 — Management

**Execution host: FW01**

### 11.1 Create the connection profile

```bash
sudo nmcli connection add type vlan \
  con-name vlan100-management \
  ifname ens19.100 \
  dev ens19 \
  id 100 \
  ipv4.method manual \
  ipv4.addresses 10.20.10.1/24 \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect no
```

**Command explanation**

- `connection add` — Creates a new NetworkManager profile.
- `type vlan` — Creates a VLAN connection.
- `con-name` — Assigns a descriptive profile name.
- `ifname ens19.100` — Specifies the VLAN interface name.
- `dev ens19` — Selects the parent network device.
- `id 100` — Sets the IEEE 802.1Q VLAN ID.
- `ipv4.method manual` — Uses static IPv4 addressing.
- `ipv4.addresses` — Assigns the gateway address.
- `ipv4.never-default yes` — Prevents the VLAN profile from becoming a default route.
- `ipv6.method disabled` — Disables IPv6 on the VLAN interface.
- `connection.zone drop` — Applies the restrictive firewall zone.
- `connection.autoconnect no` — Prevents automatic activation until validation is complete.

**Why create the profile with autoconnect disabled?**

This provides a controlled implementation sequence: create, inspect, activate, validate and then enable persistence.

### 11.2 Verify the saved profile

```bash
nmcli -f connection.id,connection.interface-name,connection.autoconnect,vlan.parent,vlan.id,ipv4.method,ipv4.addresses,ipv4.never-default,ipv6.method,connection.zone connection show vlan100-management
```

This uses `-f` to display only the relevant configuration properties.

### 11.3 Activate VLAN 100

```bash
sudo nmcli connection up vlan100-management
```

### 11.4 Verify VLAN tagging and addressing

```bash
nmcli device status
ip -d link show ens19.100
ip -br addr show ens19.100
ip route
```

**New command option: `ip -d link show`**

- `-d` — Requests detailed information.
- `link show` — Displays network interface properties.

For a VLAN interface, the detailed output includes the 802.1Q protocol and VLAN ID.

Expected:

- VLAN ID 100.
- IPv4 `10.20.10.1/24`.
- Connected route `10.20.10.0/24`.
- No new default route.

### 11.5 Verify firewall and forwarding

```bash
sudo firewall-cmd --get-active-zones
sysctl net.ipv4.ip_forward
```

Expected:

- `ens19.100` assigned to `drop`.
- IPv4 forwarding remains `0`.

### 11.6 Enable autoconnect

```bash
sudo nmcli connection modify vlan100-management connection.autoconnect yes
```

Verify:

```bash
nmcli -f connection.id,connection.autoconnect connection show vlan100-management
```

---

## 12. Create VLAN 200 — Servers

**Execution host: FW01**

```bash
sudo nmcli connection add type vlan \
  con-name vlan200-servers \
  ifname ens19.200 \
  dev ens19 \
  id 200 \
  ipv4.method manual \
  ipv4.addresses 10.20.20.1/24 \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect no
```

This follows the same procedure as VLAN 100, with the VLAN ID, interface name, profile name and IPv4 address changed.

### Activate

```bash
sudo nmcli connection up vlan200-servers
```

### Validate

```bash
nmcli device status
ip -d link show ens19.200
ip -br addr show ens19.200
ip route
sudo firewall-cmd --get-active-zones
sysctl net.ipv4.ip_forward
```

Expected:

- VLAN ID 200.
- IPv4 `10.20.20.1/24`.
- Connected route `10.20.20.0/24`.
- firewalld `drop`.
- IPv4 forwarding disabled.

### Enable persistence

```bash
sudo nmcli connection modify vlan200-servers connection.autoconnect yes
```

Verify:

```bash
nmcli -f connection.id,connection.autoconnect connection show vlan200-servers
```

---

## 13. Create VLAN 300 — Users

**Execution host: FW01**

```bash
sudo nmcli connection add type vlan \
  con-name vlan300-users \
  ifname ens19.300 \
  dev ens19 \
  id 300 \
  ipv4.method manual \
  ipv4.addresses 10.20.30.1/24 \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect no
```

### Activate

```bash
sudo nmcli connection up vlan300-users
```

### Validate

```bash
nmcli device status
ip -d link show ens19.300
ip -br addr show ens19.300
ip route
sudo firewall-cmd --get-active-zones
sysctl net.ipv4.ip_forward
```

Expected:

- VLAN ID 300.
- IPv4 `10.20.30.1/24`.
- Connected route `10.20.30.0/24`.
- firewalld `drop`.
- IPv4 forwarding disabled.

### Enable persistence

```bash
sudo nmcli connection modify vlan300-users connection.autoconnect yes
```

Verify:

```bash
nmcli -f connection.id,connection.autoconnect connection show vlan300-users
```

---

## 14. Create VLAN 400 — Peripherals

**Execution host: FW01**

```bash
sudo nmcli connection add type vlan \
  con-name vlan400-peripherals \
  ifname ens19.400 \
  dev ens19 \
  id 400 \
  ipv4.method manual \
  ipv4.addresses 10.20.40.1/24 \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect no
```

### Activate

```bash
sudo nmcli connection up vlan400-peripherals
```

### Validate

```bash
nmcli device status
ip -d link show ens19.400
ip -br addr show ens19.400
ip route
sudo firewall-cmd --get-active-zones
sysctl net.ipv4.ip_forward
```

Expected:

- VLAN ID 400.
- IPv4 `10.20.40.1/24`.
- Connected route `10.20.40.0/24`.
- firewalld `drop`.
- IPv4 forwarding disabled.

### Enable persistence

```bash
sudo nmcli connection modify vlan400-peripherals connection.autoconnect yes
```

Verify:

```bash
nmcli -f connection.id,connection.autoconnect connection show vlan400-peripherals
```

---

## 15. Disable IPv6 on the External Interface

**Execution host: FW01**

The initial external connection had automatically configured IPv6 addresses.

The approved design requires IPv4-only networking.

### 15.1 Modify the saved profile

```bash
sudo nmcli connection modify ens18 ipv6.method disabled
```

**Command explanation**

- `ipv6.method disabled` — Configures NetworkManager to disable IPv6 on this connection.

### 15.2 Verify the profile

```bash
nmcli -f connection.id,ipv4.method,ipv4.addresses,ipv4.gateway,ipv6.method,connection.zone connection show ens18
```

Expected:

- Static IPv4 unchanged.
- IPv6 disabled.
- firewalld zone still `drop`.

### 15.3 Apply the change without disconnecting

```bash
sudo nmcli device reapply ens18
```

**Command explanation**

- `device reapply` — Attempts to apply changes from the saved profile to the active device without fully deactivating its connection.

**Why use reapply?**

A full network disconnect can interrupt remote administration. Reapplying supported changes is less disruptive.

The command succeeded during implementation.

### 15.4 Verify the runtime state

```bash
ip -br addr show ens18
ip -6 addr show dev ens18
ip route
```

**Command explanation**

- `ip -6` — Restricts the query to IPv6 information.
- `addr show dev ens18` — Displays addresses assigned to the specified interface.

Observed:

- Static IPv4 retained.
- No IPv6 addresses on `ens18`.
- Default IPv4 route unchanged.

External connectivity was retested successfully.

---

## 16. Pre-Reboot Validation

**Execution host: FW01**

### 16.1 Verify all saved profiles

```bash
nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT connection show
```

**Command explanation**

- `NAME` — Connection profile name.
- `TYPE` — Connection type, such as Ethernet or VLAN.
- `DEVICE` — Associated interface.
- `AUTOCONNECT` — Whether the profile activates automatically.

Observed:

All six FW01 network profiles had autoconnect enabled.

### 16.2 Verify external profile activation

```bash
nmcli -f GENERAL.NAME,GENERAL.DEVICES,GENERAL.STATE,connection.autoconnect connection show ens18
```

The output confirmed that the `ens18` connection was activated and autoconnect enabled.

### 16.3 Verify firewall zones

```bash
sudo firewall-cmd --get-active-zones
```

Observed:

```text
drop
  interfaces: ens18 ens19 ens19.100 ens19.200 ens19.300 ens19.400
public (default)
```

The interface ordering may differ.

### 16.4 Verify forwarding

```bash
sysctl net.ipv4.ip_forward
```

Observed:

```text
net.ipv4.ip_forward = 0
```

---

## 17. Controlled Reboot

**Execution host: FW01**

After completing pre-reboot validation, the virtual machine was restarted.

```bash
sudo systemctl reboot
```

**Command explanation**

- `systemctl` — Manages systemd services and system state.
- `reboot` — Initiates a controlled operating system restart.

**Purpose:** Confirm that the configuration survives a restart.

The Proxmox console remained available for recovery if networking failed.

HYP01 was not rebooted.

---

## 18. Post-Reboot Validation

**Execution host: FW01**

### 18.1 Verify interfaces and addresses

```bash
nmcli device status
ip -br addr
```

Observed:

```text
ens18       192.168.178.11/24
ens19       No IPv4 address
ens19.100   10.20.10.1/24
ens19.200   10.20.20.1/24
ens19.300   10.20.30.1/24
ens19.400   10.20.40.1/24
```

All expected addresses were retained.

### 18.2 Verify routes

```bash
ip route
```

Observed:

```text
default via 192.168.178.1 dev ens18
10.20.10.0/24 dev ens19.100
10.20.20.0/24 dev ens19.200
10.20.30.0/24 dev ens19.300
10.20.40.0/24 dev ens19.400
192.168.178.0/24 dev ens18
```

The output above omits kernel-generated route attributes and metrics for readability.

The actual route metrics changed slightly after reboot. No unexpected default route or missing connected network was observed.

### 18.3 Verify firewall zones

```bash
sudo firewall-cmd --get-active-zones
```

All six interfaces remained assigned to `drop`.

### 18.4 Verify forwarding

```bash
sysctl net.ipv4.ip_forward
```

Observed:

```text
net.ipv4.ip_forward = 0
```

### 18.5 Verify local gateway connectivity

```bash
ping -c 4 192.168.178.1
```

Result: 4/4 successful.

### 18.6 Verify external connectivity

```bash
ping -c 4 1.1.1.1
```

Result: 4/4 successful.

### 18.7 Verify DNS

```bash
getent ahostsv4 redhat.com
```

Result: IPv4 addresses returned.

### 18.8 Verify IPv6 remains disabled

```bash
ip -6 addr show dev ens18
```

Result: No IPv6 addresses reported.

---

## 19. Validation Summary

| Area | Result |
|---|---|
| Proxmox VLAN-aware bridge | PASS |
| Proxmox VLAN trunk allowance | PASS |
| External static IPv4 | PASS |
| Internal trunk parent | PASS |
| VLAN 100 | PASS |
| VLAN 200 | PASS |
| VLAN 300 | PASS |
| VLAN 400 | PASS |
| Default route | PASS |
| firewalld zone assignments | PASS |
| IPv4 forwarding disabled | PASS |
| IPv6 disabled on external interface | PASS |
| NetworkManager autoconnect | PASS |
| Router connectivity | PASS |
| Internet connectivity | PASS |
| DNS resolution | PASS |
| Reboot persistence | PASS |

**Not tested:** End-to-end connectivity between FW01 and separate tagged VLAN clients, inter-VLAN forwarding, NAT and production firewall access policies.

These remain outside the NIM-22 implementation scope.

## 20. Operational Considerations

### Network Configuration Changes

NetworkManager changes should be verified before activation, particularly when modifying an interface used for remote administration.

The Proxmox console should remain available as an alternative access method.

### Firewall Configuration

The `drop` zone is intentionally restrictive. Future changes must follow the approved access-control design rather than broadly permitting traffic to resolve connectivity problems.

### VLAN Configuration

The Proxmox bridge and guest VLAN configurations must remain consistent.

When adding a new VLAN, administrators must review both the VM's permitted VLAN trunk list and the guest NetworkManager profile.

### Routing

IPv4 forwarding remains disabled.

Enabling forwarding alone is not sufficient to establish a secure routing policy. Routing, NAT and firewall behavior must be configured and validated together.

### Change Recovery

If an interface configuration causes loss of connectivity, use the Proxmox console to inspect or restore the relevant NetworkManager profile.

Detailed recovery procedures are maintained separately in `rollback.md`.

## 21. Implementation Result

**Result: Successful**

FW01 was configured with a persistent external IPv4 interface and four internal VLAN gateway interfaces carried over a VLAN-aware Proxmox virtual network.

All expected interfaces, IPv4 addresses, routes and security settings survived a controlled reboot.

The configuration establishes the network interface foundation required for NIM-23 routing and NAT implementation.

End-to-end VLAN connectivity and isolation remain subject to later integration testing.
