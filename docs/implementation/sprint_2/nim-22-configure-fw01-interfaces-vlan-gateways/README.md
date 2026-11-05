# NIM-22 — Configure FW01 Interfaces and VLAN Gateways

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Sprint:** Sprint 2 — Initial Network Infrastructure  
**Jira Issue:** NIM-22  
**Component:** FW01 — Network Firewall and Gateway  
**Platform:** Red Hat Enterprise Linux 10.2 / Proxmox VE  
**Status:** Technical implementation complete — documentation and acceptance review pending  
**Predecessors:** NIM-19 — Network Addressing and Interface Schedule; NIM-21 — Deploy FW01

---

## 1. Overview

NIM-22 establishes the network interface and VLAN gateway configuration for FW01, Northwind Outdoor Ltd.'s virtual firewall and routing server.

The implementation builds upon the RHEL virtual machine deployed in NIM-21 and the addressing and interface design established in NIM-19.

FW01 requires connectivity to the existing home network for external communication and separate logical interfaces for Northwind's internal Management, Server, User and Peripheral networks.

The solution uses a VLAN-aware Proxmox Linux bridge, an 802.1Q trunk and NetworkManager-managed VLAN subinterfaces within RHEL.

All interfaces are configured with persistent connection profiles and a restrictive firewall baseline. Routing between networks, NAT and production firewall access policies remain outside this implementation.

## 2. Business Requirements

Northwind Outdoor Ltd. requires a network foundation capable of supporting multiple business functions while maintaining appropriate separation between them.

The implementation must support:

- Separate network segments for management systems, servers, users and peripheral devices.
- Predictable, documented IP addressing for each network.
- A central gateway interface for each internal network.
- An isolated internal virtual network that does not connect directly to the physical home LAN.
- Reliable external connectivity for FW01.
- Persistent configuration across system restarts.
- A secure initial configuration that does not permit unintended traffic between networks.

This provides the foundation for later deployment of internal servers, administration services and business applications.

## 3. Objectives

The objectives of NIM-22 were to:

1. Validate the existing Proxmox virtual networking configuration.
2. Configure the FW01 external interface with a static IPv4 address.
3. Establish the internal interface as an 802.1Q VLAN trunk.
4. Create VLAN gateway interfaces for VLANs 100, 200, 300 and 400.
5. Configure static IPv4 gateway addresses on each VLAN interface.
6. Disable IPv6 in accordance with the approved IPv4-only design.
7. Apply a restrictive firewalld zone to all FW01 network interfaces.
8. Preserve IPv4 forwarding in its disabled state.
9. Ensure all connection profiles activate automatically after reboot.
10. Validate configuration, connectivity and persistence.

## 4. Scope

### In Scope

- Review of HYP01 network bridge configuration.
- Verification of FW01 virtual network adapter assignments.
- Configuration of Proxmox VLAN trunk allowances.
- NetworkManager configuration of external and internal interfaces.
- Creation of RHEL VLAN subinterfaces.
- Static IPv4 addressing and connected routes.
- NetworkManager autoconnect configuration.
- Initial firewalld zone assignments.
- IPv6 disabling on FW01 connection profiles.
- Controlled reboot and persistence validation.
- Documentation of implementation and findings.

### Out of Scope

- Enabling IPv4 packet forwarding.
- Inter-VLAN routing and access.
- Network Address Translation (NAT).
- Final firewall access-control policies.
- DHCP services for internal VLANs.
- DNS and NTP service deployment.
- Deployment of internal test clients and production servers.
- End-to-end VLAN connectivity and isolation testing.

These capabilities are addressed through subsequent implementation tickets.

## 5. Architecture

### 5.1 Platform

| Component | Configuration |
|---|---|
| Hypervisor | HYP01 — Proxmox VE |
| Virtual machine | FW01 |
| VM ID | 101 |
| Operating system | RHEL 10.2 |
| Network management | NetworkManager (`nmcli`) |
| Firewall management | firewalld |
| VLAN encapsulation | IEEE 802.1Q |
| External virtual bridge | `vmbr0` |
| Internal virtual bridge | `vmbr1` |

### 5.2 Network Architecture

```text
                 Home Network
                192.168.178.0/24
                        |
                 Home Router
                192.168.178.1
                        |
                 Proxmox vmbr0
                        |
                 FW01 — ens18
                192.168.178.11/24
                        |
                      FW01
                        |
                 FW01 — ens19
                802.1Q VLAN Trunk
                        |
                 Proxmox vmbr1
                  (No uplink)
                        |
           +------------+------------+
           |            |            |
       VLAN 100     VLAN 200      VLAN 300      VLAN 400
       Management   Servers       Users         Peripherals
       10.20.10.0   10.20.20.0    10.20.30.0    10.20.40.0
          /24          /24           /24           /24
```

**Architecture note:** The diagram illustrates logical connectivity and the intended internal network segmentation. Internal VLANs are carried over `ens19` and the VLAN-aware `vmbr1` bridge. No additional internal client VM was deployed as part of NIM-22.

The presence of VLAN gateway addresses does not imply that forwarding between these networks is enabled.

### 5.3 Proxmox Virtual Networking

**`vmbr0` — External Bridge**

- Connected to the physical home network through `nic0`.
- Provides connectivity for HYP01 and FW01 to the existing home LAN.
- HYP01 management address: `192.168.178.10/24`.
- FW01 external address: `192.168.178.11/24`.

**`vmbr1` — Internal Bridge**

- Configured as a VLAN-aware Linux bridge.
- No physical uplink.
- No configured IPv4 management address.
- VLAN filtering enabled.
- Supports the Northwind internal VLANs.

The FW01 virtual network adapters are configured as follows:

| VM Adapter | Proxmox Bridge | RHEL Interface | Purpose |
|---|---|---|---|
| `net0` | `vmbr0` | `ens18` | External connectivity |
| `net1` | `vmbr1` | `ens19` | Internal VLAN trunk |

The Proxmox configuration for `net1` was updated to permit VLAN IDs:

`100;200;300;400`

The resulting bridge VLAN membership was verified using `bridge vlan show`.

## 6. As-Built Interface Configuration

### 6.1 Interface Schedule

| Interface | Network Role | IPv4 Address | VLAN ID |
|---|---|---|---|
| `ens18` | External / Home LAN | `192.168.178.11/24` | Untagged |
| `ens19` | Internal trunk parent | None | Parent |
| `ens19.100` | Management | `10.20.10.1/24` | 100 |
| `ens19.200` | Servers | `10.20.20.1/24` | 200 |
| `ens19.300` | Users | `10.20.30.1/24` | 300 |
| `ens19.400` | Peripherals | `10.20.40.1/24` | 400 |

### 6.2 External Interface

**Interface:** `ens18`  
**NetworkManager profile:** `ens18`

| Property | Value |
|---|---|
| IPv4 method | Manual |
| IPv4 address | `192.168.178.11/24` |
| Default gateway | `192.168.178.1` |
| DNS server | `192.168.178.1` |
| IPv6 method | Disabled |
| firewalld zone | `drop` |
| Autoconnect | Enabled |

The static address was selected outside the home router's DHCP allocation range, following verification that the address was not assigned to another known device.

The external interface was activated and tested successfully for local gateway connectivity, internet connectivity and DNS resolution.

### 6.3 Internal Parent Interface

**Interface:** `ens19`  
**NetworkManager profile:** `ens19`

The parent interface is used to carry tagged Ethernet traffic for the internal VLANs.

It is configured without IPv4 or IPv6 addressing.

| Property | Value |
|---|---|
| IPv4 method | Disabled |
| IPv6 method | Disabled |
| Autoconnect | Enabled |
| firewalld zone | `drop` |
| Proxmox bridge | `vmbr1` |

The parent interface itself does not require an IP address because Layer 3 gateway addresses are configured on its VLAN subinterfaces.

### 6.4 Internal VLAN Interfaces

| NetworkManager Profile | Interface | VLAN | Gateway Address |
|---|---|---|---|
| `vlan100-management` | `ens19.100` | 100 | `10.20.10.1/24` |
| `vlan200-servers` | `ens19.200` | 200 | `10.20.20.1/24` |
| `vlan300-users` | `ens19.300` | 300 | `10.20.30.1/24` |
| `vlan400-peripherals` | `ens19.400` | 400 | `10.20.40.1/24` |

Each VLAN profile uses:

- A static IPv4 address.
- The appropriate 802.1Q VLAN ID.
- `ens19` as its parent interface.
- No default gateway.
- IPv6 disabled.
- The `drop` firewalld zone.
- Autoconnect enabled.

The VLAN profiles were initially created with autoconnect disabled to permit inspection before activation.

Each profile was then activated, validated and configured for automatic activation.

## 7. Implementation Summary

### 7.1 Baseline Inspection

The existing configuration was reviewed before modification.

Checks included:

- Proxmox bridge configuration and VLAN filtering.
- FW01 virtual network adapter assignments.
- RHEL interface and NetworkManager state.
- Existing IPv4 addresses and routes.
- NetworkManager connection profile properties.
- firewalld runtime and permanent configuration.
- IPv4 forwarding state.

The initial configuration confirmed that FW01's internal interface did not have an IPv4 address or default route.

### 7.2 Proxmox Trunk Configuration

The FW01 `net1` adapter was configured to allow tagged traffic for VLANs 100, 200, 300 and 400 over `vmbr1`.

The updated VM configuration was verified using `qm config`, and bridge membership was inspected using the Linux `bridge` utility on HYP01.

The VM remained running during the configuration change.

### 7.3 External IPv4 Configuration

The `ens18` NetworkManager profile was updated to use static IPv4 addressing.

The interface was then activated and verified for:

- Correct IP address.
- Correct default route.
- Connectivity to the home router.
- Internet reachability.
- DNS resolution.

IPv6 was subsequently disabled on the saved connection profile and successfully reapplied without a full interface disconnection.

### 7.4 Internal VLAN Deployment

VLAN interfaces were created individually in the following order:

1. VLAN 100 — Management.
2. VLAN 200 — Servers.
3. VLAN 300 — Users.
4. VLAN 400 — Peripherals.

Each interface was validated before proceeding to the next.

Verification included NetworkManager activation, 802.1Q VLAN ID, IPv4 addressing, connected routes, firewalld zone assignment and autoconnect configuration.

### 7.5 Controlled Reboot

After configuration, FW01 underwent a controlled reboot.

The post-reboot checks confirmed that the interface addresses, connected routes, default gateway and firewall zone assignments were retained.

External network connectivity and DNS resolution also remained operational.

## 8. Security Baseline

### 8.1 firewalld

All six FW01 network interfaces were assigned to the `drop` zone.

The `public` zone remained the default zone but had no active interfaces assigned.

The `drop` zone provides a restrictive initial policy for unsolicited inbound traffic while the final firewall design remains unimplemented.

The assignment was verified both before and after reboot.

**Important:** This is an interim security baseline. Final ingress, forwarding and administrative access policies are to be designed and implemented in NIM-24.

### 8.2 IPv4 Forwarding

The following kernel parameter was verified:

```text
net.ipv4.ip_forward = 0
```

This confirms that IPv4 forwarding was disabled during implementation and remained disabled after reboot.

The configured connected routes identify directly attached networks; they do not, by themselves, permit FW01 to route packets between those networks.

### 8.3 IPv6

The implementation follows the approved IPv4-only design.

IPv6 was disabled in the relevant NetworkManager connection profiles.

Following the controlled reboot, the external interface `ens18` had no IPv6 addresses.

### 8.4 Administrative Access

FW01 configuration was performed through its Proxmox console using the existing `nw-admin` administrative account.

SSH access and its final permitted administrative network paths are addressed separately in the project's security and management-access work.

## 9. Validation Results

### 9.1 Interface and VLAN Validation

| Test | Expected Result | Actual Result | Status |
|---|---|---|---|
| External interface | Static IPv4 assigned | `192.168.178.11/24` | PASS |
| Internal parent | No IPv4 address | No IPv4 address | PASS |
| VLAN 100 | `10.20.10.1/24` | Correct | PASS |
| VLAN 200 | `10.20.20.1/24` | Correct | PASS |
| VLAN 300 | `10.20.30.1/24` | Correct | PASS |
| VLAN 400 | `10.20.40.1/24` | Correct | PASS |
| VLAN encapsulation | 802.1Q, correct IDs | Verified | PASS |
| NetworkManager | All six profiles autoconnect | Confirmed | PASS |
| VLAN trunk allowance | IDs 100, 200, 300, 400 | Verified on HYP01 | PASS |

### 9.2 Routing and Security Validation

| Test | Expected Result | Actual Result | Status |
|---|---|---|---|
| Default route | Via `192.168.178.1` on `ens18` | Correct | PASS |
| Management route | `10.20.10.0/24` | Present | PASS |
| Server route | `10.20.20.0/24` | Present | PASS |
| User route | `10.20.30.0/24` | Present | PASS |
| Peripheral route | `10.20.40.0/24` | Present | PASS |
| firewalld assignment | All interfaces in `drop` | Confirmed | PASS |
| IPv4 forwarding | Disabled | `0` | PASS |
| IPv6 on external interface | No IPv6 addresses | None | PASS |

### 9.3 Connectivity Validation

| Test | Result | Status |
|---|---|---|
| Ping home router `192.168.178.1` | 4 transmitted, 4 received | PASS |
| Ping external IP `1.1.1.1` | 4 transmitted, 4 received | PASS |
| Resolve `redhat.com` | IPv4 addresses returned | PASS |
| Reboot and recheck IPv4 addresses | All five addresses retained | PASS |
| Reboot and recheck connected routes | All four VLAN routes retained | PASS |
| Reboot and recheck firewall assignments | All six interfaces in `drop` | PASS |
| Reboot and recheck forwarding | Still disabled | PASS |

**Validation limitation:** Successful activation of VLAN subinterfaces and installation of connected routes confirm local configuration. They do not independently prove end-to-end tagged connectivity with other virtual machines.

## 10. Findings and Lessons Learned

### 10.1 VLAN Trunking Requires Configuration at Multiple Layers

The implementation reinforced the distinction between the Proxmox virtual networking layer and the guest operating system.

Proxmox permits the required VLAN tags to traverse the virtual bridge and VM adapter.

RHEL then uses NetworkManager to create the corresponding VLAN interfaces and assign their gateway addresses.

Both layers must be configured consistently.

### 10.2 A VLAN Parent Interface Does Not Require an IP Address

The internal interface `ens19` carries tagged Layer 2 traffic but has no Layer 3 address of its own.

The IP addresses belong to the VLAN interfaces created on top of it.

This separation allows one virtual network adapter to support multiple logical networks.

### 10.3 Interface Configuration Is Not the Same as Routing

The creation of four gateway interfaces automatically installs connected routes in the Linux routing table.

However, IPv4 forwarding remained disabled.

This demonstrates the distinction between a system having knowledge of directly connected networks and being configured to forward traffic between them.

### 10.4 NetworkManager Profiles and Active Device State Are Different

NetworkManager stores connection profiles separately from the runtime state of network interfaces.

Configuration changes may require activation or reapplication before they affect the live system.

During this implementation, `nmcli device reapply` successfully applied the IPv6 change to `ens18` without interrupting IPv4 connectivity.

### 10.5 Persistence Must Be Tested

The successful controlled reboot demonstrated that the NetworkManager configuration and relevant firewall assignments were retained.

It also confirmed that the intended routing and forwarding baseline remained unchanged.

This is stronger evidence than relying solely on successful initial configuration.

### 10.6 Troubleshooting Requires the Correct Execution Context

Proxmox utilities such as `qm` and host-side bridge inspection commands must be executed on HYP01.

Guest operating system tools such as `nmcli`, `ip`, `sysctl` and `firewall-cmd` are executed within FW01.

An initial command attempt against a Proxmox virtual firewall interface failed because it was run inside FW01 rather than on HYP01. Running the command in the correct environment resolved the issue.

### 10.7 Route Metrics May Change After Reboot

After reboot, the connected routes remained present, but their metrics differed slightly from the pre-reboot values.

This was consistent with the VLAN profiles activating in a different order.

No unexpected default route or missing network was observed.

## 11. Outstanding Work and Limitations

| Item | Description | Planned Follow-up |
|---|---|---|
| End-to-end VLAN testing | Tagged connectivity with separate test VMs not yet demonstrated | NIM-26 |
| Inter-VLAN routing | IPv4 forwarding and routed traffic not enabled | NIM-23 |
| NAT | Internal network internet access not configured | NIM-23 |
| Final firewall policy | Production access-control rules not implemented | NIM-24 |
| Internal network services | DHCP, DNS and other services not yet deployed | Subsequent core-services work |
| Internal client validation | No additional client VM deployed for VLAN traffic testing | NIM-26 or appropriate deployment ticket |

These items are deliberate scope boundaries rather than failures of the local interface configuration.

## 12. Evidence Register

The following evidence was collected during implementation:

| Evidence | Description |
|---|---|
| EV-01 | HYP01 bridge and VLAN-aware configuration |
| EV-02 | FW01 virtual network adapter configuration |
| EV-03 | Proxmox VLAN trunk membership |
| EV-04 | External interface static IPv4 profile |
| EV-05 | External connectivity and DNS tests |
| EV-06 | VLAN 100 profile and runtime verification |
| EV-07 | VLAN 200 profile and runtime verification |
| EV-08 | VLAN 300 profile and runtime verification |
| EV-09 | VLAN 400 profile and runtime verification |
| EV-10 | NetworkManager autoconnect verification |
| EV-11 | firewalld zone and IPv4 forwarding verification |
| EV-12 | Post-reboot interface, routing, connectivity and security verification |

Evidence screenshots and command outputs should be retained in the project's established documentation repository and OneNote evidence records.

## 13. Related Documentation

- **NIM-19:** Network Addressing and Interface Schedule.
- **NIM-21:** Deploy FW01 RHEL VM.
- **`deployment.md`:** Detailed implementation steps, commands and verification procedures.
- **`rollback.md`:** Recovery and rollback procedures.
- **`commands.md`:** New Linux and Proxmox commands encountered during implementation.

## 14. Next Steps

Following documentation and final acceptance review, NIM-22 can be submitted for closure.

The next implementation activities are:

**NIM-23 — FW01 Routing and NAT**

Introduce controlled routing functionality and outbound network address translation.

**NIM-24 — FW01 Firewall Policy**

Implement the approved security policy governing permitted traffic between network segments and external destinations.

**NIM-26 — FW01 Connectivity and Isolation Validation**

Demonstrate end-to-end connectivity and confirm that the network segmentation behaves according to the approved design.

## 15. Implementation Outcome

**Technical outcome: Successful**

FW01 now has a persistent external IPv4 interface, an internal VLAN trunk and four configured VLAN gateway interfaces.

The configuration survived a controlled reboot, with all expected IPv4 addresses and routes retained. External connectivity and DNS resolution remained operational, and the restrictive firewall and forwarding baseline was preserved.

The work establishes the required interface foundation for Northwind's future internal infrastructure while keeping routing and traffic permissions under explicit change control.

**Final ticket status:** Pending documentation completion and acceptance review.
