# NIM-19 — Finalise Network Addressing and Interface Schedule

**Project:** Project Redstone — Northwind Outdoor Ltd.  
**Sprint:** 02 — Initial Infrastructure Foundation  
**Jira Reference:** NIM-19 / NIM-S2-03  
**Priority:** High  
**Parent Epic:** Platform & Virtualisation Foundation  
**Status:** Design approved — documentation and acceptance review pending  
**Target Platform:** HYP01 — Proxmox VE 9.2  
**Guest Operating System:** Red Hat Enterprise Linux 10  
**Predecessors:** NIM-17, NIM-18

---

## 1. Objective

Produce an implementation-ready network addressing and interface schedule for Northwind's initial infrastructure.

The design must establish how FW01 and JUMP01 will connect to the existing HYP01 hypervisor, communicate across the approved VLAN architecture and access temporary network services during initial deployment.

The resulting documentation will provide a controlled baseline for subsequent implementation tasks, reducing configuration ambiguity, addressing conflicts and unintended network exposure.

### 1.1 Scope

**In scope:**

- FW01 external IP addressing.
- Internal VLAN subnet and gateway assignments.
- JUMP01 management addressing.
- Standard IP allocation conventions.
- Proxmox bridge and virtual NIC mapping.
- VLAN trunk and access-port behaviour.
- Temporary routing, DNS and NTP arrangements.
- Address conflict prevention.
- Network isolation requirements.
- Implementation dependencies and validation criteria.

**Out of scope:**

- Creating or modifying Proxmox bridges.
- Deploying FW01 or JUMP01.
- Configuring RHEL network interfaces.
- Enabling routing or NAT.
- Implementing firewall policies.
- Deploying CORE01 or permanent infrastructure services.
- Performing live network connectivity tests.

These activities will be completed under subsequent Jira tasks.

---

## 2. Existing Infrastructure and Design Context

### 2.1 HYP01 network baseline

| Parameter | Existing configuration |
|---|---|
| Hypervisor | HYP01 |
| Platform | Proxmox VE 9.2 |
| Management IP | `192.168.178.10/24` |
| Default gateway | `192.168.178.1` |
| External bridge | `vmbr0` |
| Physical uplink | `nic0` |
| Home network | `192.168.178.0/24` |
| Home router | `192.168.178.1` |
| Home DHCP range | `192.168.178.20–192.168.178.200` |
| Internal bridge | `vmbr1` — planned, not deployed |

The existing HYP01 management configuration must remain operational and independent of FW01.

### 2.2 Approved virtual machines

| VM ID | Hostname | Role | Network requirement |
|---|---|---|---|
| 101 | FW01 | Firewall / Router | External NIC and internal VLAN trunk |
| 201 | JUMP01 | Administrative Jump Server | Single Management VLAN NIC |

These specifications were approved under NIM-18.

### 2.3 Network design principles

1. Separate infrastructure into distinct trust zones.
2. Use FW01 as the routing and firewall enforcement boundary.
3. Maintain independent HYP01 management access.
4. Use predictable addressing conventions.
5. Minimise unnecessary network connectivity.
6. Document addressing and interface assignments before implementation.
7. Validate permitted and prohibited traffic paths.

---

## 3. Design Decision Register

| ID | Decision | Status |
|---|---|---|
| DD01 | FW01 external static addressing | Approved |
| DD02 | Internal VLAN gateway addressing | Approved |
| DD03 | JUMP01 management addressing and IP allocation convention | Approved |
| DD04 | Proxmox bridge and virtual NIC mapping | Approved |
| DD05 | Temporary routing, DNS and NTP bootstrap | Approved |
| DD06 | Address conflict prevention and network isolation | Approved |

---

## 4. DD01 — FW01 External Addressing

**Status:** Approved

### 4.1 Decision

Assign FW01 a static external address on the existing home network.

| Parameter | Approved value |
|---|---|
| Hostname | FW01 |
| VM ID | 101 |
| External IP | `192.168.178.11` |
| Prefix length | `/24` |
| Subnet mask | `255.255.255.0` |
| Network | `192.168.178.0/24` |
| Default gateway | `192.168.178.1` |
| Proxmox NIC | `net0` |
| Proxmox bridge | `vmbr0` |
| Address assignment | Static |

### 4.2 Rationale

The selected address:

- Is adjacent to HYP01's management address (`192.168.178.10`).
- Falls outside the previously identified home DHCP allocation range.
- Provides predictable addressing for a critical network component.
- Supports independent management of HYP01 and FW01.

### 4.3 Constraints

The address must be verified as unused before deployment.

HYP01 must retain its existing default gateway and management configuration.

FW01 must not replace the home router as the default gateway for HYP01.

### 4.4 Validation requirements

- Verify the external address is not already assigned.
- Confirm upstream router connectivity.
- Confirm the intended default route.
- Verify HYP01 remains accessible when FW01 is stopped.
- Document the final guest interface name and configuration.

---

## 5. DD02 — Internal VLAN Gateway Addressing

**Status:** Approved

### 5.1 Decision

Use `.1` as the default gateway address for each approved internal VLAN.

| VLAN | Network zone | Subnet | Gateway | Gateway host |
|---|---|---|---|---|
| 100 | Management | `10.20.10.0/24` | `10.20.10.1` | FW01 |
| 200 | Server | `10.20.20.0/24` | `10.20.20.1` | FW01 |
| 300 | User | `10.20.30.0/24` | `10.20.30.1` | FW01 |
| 400 | Peripheral | `10.20.40.0/24` | `10.20.40.1` | FW01 |

### 5.2 Rationale

Consistent gateway addressing simplifies:

- Network design.
- Documentation.
- Troubleshooting.
- Configuration of static hosts.
- Future DHCP scope creation.

### 5.3 Technical implementation model

FW01 will use one internal VirtIO NIC connected to an 802.1Q trunk.

RHEL NetworkManager will provide VLAN subinterfaces, each assigned its respective gateway address.

Illustrative configuration:

```text
FW01 internal NIC
│
├── VLAN 100 → 10.20.10.1/24
├── VLAN 200 → 10.20.20.1/24
├── VLAN 300 → 10.20.30.1/24
└── VLAN 400 → 10.20.40.1/24
```

The actual Linux interface name must be verified during deployment.

### 5.4 Routing and security implications

Each VLAN represents a separate Layer 2 broadcast domain.

FW01 will provide Layer 3 routing between these networks.

However, routing capability does not imply that traffic should be permitted.

Inter-VLAN connectivity will be controlled using the approved firewall policy, following a default-deny approach with explicit exceptions.

### 5.5 Availability consideration

All four VLAN gateways will initially depend on a single FW01 instance.

This is consistent with Northwind's recovery-based resilience strategy.

High availability is deferred until business requirements justify additional infrastructure.

---

## 6. DD03 — JUMP01 Management Addressing

**Status:** Approved

### 6.1 Decision

Assign JUMP01 a static IP address in the Management VLAN.

| Parameter | Approved value |
|---|---|
| Hostname | JUMP01 |
| VM ID | 201 |
| IP address | `10.20.10.10` |
| Prefix length | `/24` |
| Network | `10.20.10.0/24` |
| Default gateway | `10.20.10.1` |
| VLAN | 100 |
| Zone | Management |
| Proxmox NIC | `net0` |
| Proxmox bridge | `vmbr1` |
| Proxmox VLAN tag | 100 |
| Address assignment | Static |

### 6.2 Rationale

A static address provides a predictable endpoint for administrative access.

JUMP01 is assigned to the Management VLAN to separate administrative traffic from ordinary user and server workloads.

JUMP01 will have one network interface and must not be directly connected to other internal VLANs.

### 6.3 Approved IP allocation convention

The following convention applies to Northwind's internal `/24` networks.

| Address range | Purpose |
|---|---|
| `.0` | Network address |
| `.1` | Default gateway |
| `.2–.9` | Reserved infrastructure |
| `.10–.49` | Static infrastructure and servers |
| `.50–.199` | DHCP allocation where required |
| `.200–.254` | Reserved for future requirements |
| `.255` | Broadcast address |

This is an addressing convention, not a requirement to enable DHCP in every VLAN.

Management and server networks may remain statically addressed.

### 6.4 Address management requirements

- Record every static address assignment.
- Prevent overlapping static and DHCP allocations.
- Record reservations and approved exceptions.
- Review the address register before provisioning new infrastructure.
- Keep the asset register and network schedule consistent.

---

## 7. DD04 — Proxmox Bridge and NIC Mapping

**Status:** Approved

### 7.1 Decision

Use two Proxmox bridges to separate external and internal connectivity.

| Bridge | Purpose | Physical uplink | VLAN-aware | Host management IP |
|---|---|---|---|---|
| `vmbr0` | Home network / external | `nic0` | Not required for current design | `192.168.178.10/24` |
| `vmbr1` | Northwind internal networks | None | Yes | None |

### 7.2 Virtual NIC schedule

| VM | NIC | Model | Bridge | VLAN behaviour |
|---|---|---|---|---|
| FW01 | `net0` | VirtIO | `vmbr0` | Untagged external |
| FW01 | `net1` | VirtIO | `vmbr1` | Tagged trunk: 100, 200, 300, 400 |
| JUMP01 | `net0` | VirtIO | `vmbr1` | Access VLAN 100 |

### 7.3 FW01 internal trunk

FW01's internal NIC will receive tagged frames for the approved VLANs.

The allowed trunk VLANs are:

- 100 — Management
- 200 — Server
- 300 — User
- 400 — Peripheral

The VM NIC must not be configured with a single access VLAN tag.

Proxmox bridge and VM NIC settings must be reviewed together to ensure the intended trunk restrictions are enforced.

### 7.4 JUMP01 access connection

JUMP01's NIC will be assigned VLAN tag 100 in Proxmox.

The guest will use a normal RHEL network interface without a VLAN subinterface.

Proxmox will handle the VLAN tagging and untagging required for access to VLAN 100.

### 7.5 Isolation requirements

`vmbr1` must:

- Have no physical uplink.
- Have no HYP01 management IP address.
- Support the approved VLANs.
- Avoid unintended connectivity to the home network.
- Be validated before production-like workloads are connected.

HYP01 management must remain on `vmbr0`.

### 7.6 Existing configuration considerations

NIM-17 identified stale interface references in the existing Proxmox network configuration.

These must be reviewed before applying bridge changes.

Network changes must be performed with console access and an appropriate rollback procedure to reduce the risk of administrative lockout.

---

## 8. DD05 — Bootstrap Routing, DNS and NTP

**Status:** Approved

### 8.1 Decision

Use temporary upstream network services while CORE01 is unavailable.

| Service | Bootstrap arrangement | Target arrangement |
|---|---|---|
| Default routing | FW01 → `192.168.178.1` | Retained |
| Outbound NAT | FW01 for explicitly permitted traffic | Retained |
| DNS | Home router `192.168.178.1`, subject to validation | CORE01 |
| NTP | Reachable upstream time source | CORE01 |
| DHCP | Static addresses for initial infrastructure | CORE01 where required |
| Administration | Proxmox console and controlled SSH | JUMP01 as standard administrative entry point |

### 8.2 Rationale

FW01 and JUMP01 must be deployed before CORE01 becomes available.

Temporary services prevent a circular dependency in which the initial infrastructure requires internal services that have not yet been built.

### 8.3 DNS requirements

The home router is the preferred temporary DNS resolver.

Its ability to provide DNS forwarding to the lab must be verified.

Temporary DNS must support the resolution required for installation, package updates and bootstrap administration.

Internal Northwind name resolution will be introduced with CORE01.

### 8.4 NTP requirements

Accurate system time is necessary for:

- Consistent logs.
- Troubleshooting.
- Security-related operations.
- Reliable system administration.

An approved reachable upstream time source will be used during bootstrap.

The exact source and configuration must be documented during implementation.

### 8.5 Outbound connectivity

Outbound access must be limited to approved requirements.

Temporary access may be needed for:

- DNS resolution.
- Time synchronisation.
- Operating system updates.
- Approved administrative operations.

JUMP01 must not receive unrestricted Internet access solely for convenience.

### 8.6 Transition to CORE01

When CORE01 is deployed:

1. Validate internal DNS and NTP services.
2. Update relevant client configurations.
3. Verify successful name resolution and time synchronisation.
4. Remove obsolete temporary firewall exceptions.
5. Update operational documentation and asset records.

The transition must be performed through a controlled change.

---

## 9. DD06 — Address Conflict Prevention and Network Isolation

**Status:** Approved

### 9.1 Decision

Adopt seven baseline network controls.

| ID | Control | Requirement |
|---|---|---|
| C01 | Static IP register | Record all assigned infrastructure IPs |
| C02 | DHCP boundaries | Exclude static allocations from DHCP pools |
| C03 | External IP validation | Verify `192.168.178.11` is unused |
| C04 | VLAN isolation | Route inter-VLAN traffic through FW01 |
| C05 | Firewall policy | Default-deny inter-VLAN traffic |
| C06 | Independent management | Preserve HYP01 access without FW01 |
| C07 | Network validation | Test authorised and prohibited connectivity |

### 9.2 Address conflict prevention

Every infrastructure address must be recorded before assignment.

Future DHCP scopes must respect the approved allocation convention.

Static addresses and reservations must be reviewed whenever new servers or services are introduced.

### 9.3 Network isolation

VLAN separation alone is not sufficient to enforce the intended security policy.

FW01 will control routed traffic between Northwind's internal trust zones.

The firewall policy must explicitly permit necessary communication and reject unauthorised traffic.

### 9.4 Independent management

HYP01 management connectivity must not depend on FW01.

If FW01 is stopped or unavailable, administrators must still be able to access the hypervisor through the existing management path.

### 9.5 Validation requirements

Testing must include both positive and negative scenarios.

**Positive tests:**

- FW01 reaches its upstream gateway.
- JUMP01 reaches its approved gateway.
- Authorised management traffic succeeds.
- Approved outbound traffic succeeds.

**Negative tests:**

- Unauthorised inter-VLAN traffic is blocked.
- JUMP01 cannot bypass FW01 to reach other VLANs.
- Internal networks are not unintentionally bridged to the home network.
- Unapproved outbound traffic is denied.

These tests will be performed during subsequent implementation tasks.

---

## 10. Consolidated Network Addressing Schedule

### 10.1 External network

| Asset | Interface | Address | Gateway | Assignment |
|---|---|---|---|---|
| Home router | Existing LAN interface | `192.168.178.1/24` | Upstream router configuration | Existing |
| HYP01 | `vmbr0` | `192.168.178.10/24` | `192.168.178.1` | Existing static |
| FW01 | External NIC | `192.168.178.11/24` | `192.168.178.1` | Approved static; not deployed |

### 10.2 Internal networks

| VLAN | Zone | Subnet | Gateway | Allocation convention |
|---|---|---|---|---|
| 100 | Management | `10.20.10.0/24` | `10.20.10.1` | Static infrastructure; DHCP only if justified |
| 200 | Server | `10.20.20.0/24` | `10.20.20.1` | Static infrastructure; DHCP only if justified |
| 300 | User | `10.20.30.0/24` | `10.20.30.1` | DHCP range available if required |
| 400 | Peripheral | `10.20.40.0/24` | `10.20.40.1` | Static or DHCP by device requirements |

### 10.3 Approved static assignments

| Asset | Network | Address | Status |
|---|---|---|---|
| FW01 external | Home LAN | `192.168.178.11/24` | Approved; pending deployment |
| FW01 MGMT gateway | VLAN 100 | `10.20.10.1/24` | Approved; pending deployment |
| FW01 SERVER gateway | VLAN 200 | `10.20.20.1/24` | Approved; pending deployment |
| FW01 USER gateway | VLAN 300 | `10.20.30.1/24` | Approved; pending deployment |
| FW01 PERIPHERAL gateway | VLAN 400 | `10.20.40.1/24` | Approved; pending deployment |
| JUMP01 | VLAN 100 | `10.20.10.10/24` | Approved; pending deployment |

No other internal static addresses are assigned under NIM-19.

---

## 11. Consolidated Interface Schedule

### 11.1 Proxmox host interfaces

| Host | Interface | Function | Address |
|---|---|---|---|
| HYP01 | `nic0` | Physical external uplink | No separate host IP |
| HYP01 | `vmbr0` | External bridge / hypervisor management | `192.168.178.10/24` |
| HYP01 | `vmbr1` | Internal VLAN-aware bridge | No host IP |

### 11.2 FW01 guest interfaces

| Proxmox NIC | Guest interface | VLAN | IP address | Function |
|---|---|---|---|---|
| `net0` | To be verified | Untagged | `192.168.178.11/24` | External |
| `net1` | To be verified | Trunk | No IP on parent interface | Internal |
| `net1` | VLAN subinterface | 100 | `10.20.10.1/24` | MGMT gateway |
| `net1` | VLAN subinterface | 200 | `10.20.20.1/24` | SERVER gateway |
| `net1` | VLAN subinterface | 300 | `10.20.30.1/24` | USER gateway |
| `net1` | VLAN subinterface | 400 | `10.20.40.1/24` | PERIPHERAL gateway |

The actual RHEL interface names must be confirmed after deployment.

FW01's default route must point to `192.168.178.1` through the external interface.

Internal VLAN interfaces must not introduce competing default routes.

### 11.3 JUMP01 guest interface

| Proxmox NIC | Bridge | VLAN | Guest address | Gateway |
|---|---|---|---|---|
| `net0` | `vmbr1` | 100 access | `10.20.10.10/24` | `10.20.10.1` |

The guest interface name will be verified during deployment.

---

## 12. Implementation Dependencies

| Dependency | Description | Responsible task |
|---|---|---|
| DEP01 | Verify HYP01 management and network readiness | NIM-17 — completed |
| DEP02 | Approve FW01/JUMP01 VM specifications | NIM-18 — completed |
| DEP03 | Validate upstream IP availability | FW01 deployment preparation |
| DEP04 | Review stale Proxmox interface references | NIM-21 |
| DEP05 | Create isolated VLAN-aware `vmbr1` | NIM-21 |
| DEP06 | Deploy FW01 with approved NIC mapping | NIM-21 |
| DEP07 | Configure VLAN gateways and routing | NIM-22 / NIM-23 |
| DEP08 | Implement firewall and NAT policy | NIM-23 / NIM-24 |
| DEP09 | Validate isolation and connectivity | NIM-26 |
| DEP10 | Deploy JUMP01 with VLAN 100 access | NIM-27 |
| DEP11 | Implement permanent internal DNS/NTP/DHCP | CORE01 deployment |
| DEP12 | Validate temporary DNS/NTP and retire obsolete exceptions | Relevant bootstrap and CORE01 tasks |

---

## 13. Risks and Mitigations

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R01 | Duplicate external IP | Connectivity disruption | Verify `.11` before deployment |
| R02 | Incorrect VLAN tagging | Loss of connectivity or unintended access | Validate trunk and access configurations |
| R03 | Incorrect Proxmox bridge configuration | HYP01 management lockout | Preserve `vmbr0`, use console access and rollback |
| R04 | Unrestricted inter-VLAN routing | Security boundary failure | Apply default-deny firewall policy |
| R05 | Bootstrap DNS/NTP unavailable | Installation or administration issues | Validate upstream services and document alternatives |
| R06 | Temporary firewall rules retained | Unnecessary network exposure | Track and remove bootstrap exceptions |
| R07 | FW01 failure | Internal routing unavailable | Maintain recovery procedures and independent hypervisor management |
| R08 | Future DHCP overlap | Address conflicts | Enforce IP allocation policy and maintain register |

---

## 14. Validation and Acceptance Criteria

### 14.1 NIM-19 design acceptance

- [x] FW01 external static IP and gateway documented.
- [x] Internal VLAN subnets and gateway addresses documented.
- [x] JUMP01 static management IP documented.
- [x] Standard IP allocation convention approved.
- [x] Proxmox bridge assignments documented.
- [x] FW01 external and internal NIC mappings defined.
- [x] JUMP01 VLAN 100 access configuration defined.
- [x] Temporary routing, DNS and NTP arrangements approved.
- [x] Address conflict prevention and isolation controls approved.
- [x] Implementation dependencies and risks identified.
- [ ] Asset register updated with NIM-19 addressing details.
- [ ] Network schedule and README committed to GitHub.
- [ ] Final documentation reviewed and Jira acceptance confirmed.

### 14.2 Deferred implementation validation

- [ ] Verify `192.168.178.11` is unused.
- [ ] Validate `vmbr1` isolation and VLAN awareness.
- [ ] Confirm RHEL guest interface names.
- [ ] Confirm VLAN interfaces and gateway connectivity.
- [ ] Verify FW01 forwarding and NAT.
- [ ] Verify permitted and prohibited inter-VLAN traffic.
- [ ] Confirm temporary DNS and NTP functionality.
- [ ] Verify HYP01 remains accessible without FW01.
- [ ] Confirm configuration persistence after reboot.

These are implementation tests and are not prerequisites for completing the NIM-19 design ticket unless explicitly required by its Jira acceptance criteria.

---

## 15. Deliverables

| Deliverable | Status |
|---|---|
| Design decision register DD01–DD06 | Complete |
| FW01 external addressing schedule | Complete |
| Internal VLAN addressing schedule | Complete |
| JUMP01 management addressing | Complete |
| Proxmox bridge and NIC schedule | Complete |
| Bootstrap network services plan | Complete |
| Addressing and isolation controls | Complete |
| Risk and dependency register | Complete |
| Asset register updates | Pending |
| GitHub documentation and review | Pending |

---

## 16. Findings and Lessons Learned

### 16.1 Findings

- The existing home network can serve as the upstream network for Northwind's initial firewall design, subject to address and connectivity validation.
- Internal VLANs can share one virtual trunk interface on FW01 while remaining separate Layer 2 domains.
- Proxmox VLAN-aware bridges and per-VM NIC settings must be configured consistently.
- A dedicated management VLAN does not eliminate the need for firewall access controls.
- Temporary bootstrap services are necessary because CORE01 has not yet been deployed.
- Independent hypervisor management is essential when the firewall is itself a virtual machine.

### 16.2 RHCSA learning relevance

The approved design prepares for hands-on work with:

- NetworkManager and `nmcli`.
- Static IP addressing.
- VLAN interfaces.
- Linux routing and IP forwarding.
- DNS resolver configuration.
- `chrony` time synchronisation.
- `firewalld` zones and policies.
- Network troubleshooting and validation.

### 16.3 Key lesson

Network design is not complete when IP addresses and VLAN IDs have been selected.

A reliable implementation also requires clear interface ownership, controlled routing, isolation policies, dependency planning and verification of both permitted and prohibited connectivity.

---

## 17. Final Assessment

**Design outcome:** Approved.

NIM-19 has established an implementation-ready addressing and interface baseline for FW01, JUMP01 and Northwind's four initial internal VLANs.

All six design decisions have been approved.

No Proxmox or RHEL network configuration changes were performed as part of this ticket.

**NIM-19 is ready for final documentation review and closure once the asset register and GitHub records have been updated.**

### Next steps

1. Update FW01 and JUMP01 asset register entries.
2. Commit the consolidated README and addressing/interface schedules.
3. Review Jira acceptance criteria.
4. Merge the feature branch into `main`.
5. Close NIM-19.
6. Continue with the next approved Sprint 2 task, preserving dependencies on NIM-20 and NIM-21.