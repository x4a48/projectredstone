# NIM-20 — Validate and Secure HYP01 Management Access

**Project:** Northwind Outdoor Ltd. — Enterprise Linux Infrastructure  
**Sprint:** Sprint 2 — Secure Infrastructure Foundation  
**Jira:** NIM-20  
**Target System:** HYP01  
**Platform:** Proxmox VE 9.2.1 (Debian-based)  
**Status:** Technically complete — documentation and formal closure pending

---

## 1. Objective

Establish secure, controlled and recoverable administrative access to Northwind's Proxmox hypervisor, HYP01.

The implementation must:

- Provide individual administrative identities with appropriate permissions.
- Secure remote shell access using SSH public-key authentication.
- Prevent direct root SSH login and password-based remote authentication.
- Protect Proxmox Web UI access using dedicated accounts and multifactor authentication.
- Restrict remote management to an approved administrative workstation.
- Minimise unnecessary network services.
- Preserve an emergency recovery method.
- Validate security controls through positive and negative testing.

## 2. Business Context

HYP01 provides the virtualisation platform for Northwind's planned Linux infrastructure.

Unauthorised access to the hypervisor could compromise multiple hosted systems, their configurations and stored data.

Northwind therefore requires management access to be restricted to authorised personnel and devices, while maintaining a reliable recovery path in the event of administrative lockout.

This implementation establishes the initial security baseline before deploying the first infrastructure virtual machines.

## 3. Scope

**In scope**

- Linux administrative accounts and privilege separation
- SSH authentication and hardening
- Proxmox administrative identities and MFA
- Proxmox host firewall configuration
- IPv4 and IPv6 management-access restrictions
- Network-service review and minimisation
- Administrative access validation
- Recovery considerations

**Out of scope**

- FW01 deployment and inter-VLAN firewall policies
- Centralised identity management
- Remote administration from outside the home LAN
- Multi-node Proxmox clustering
- Centralised logging and monitoring

These capabilities will be addressed through subsequent implementation tasks.

## 4. Implementation Summary

### 4.1 Administrative Identity and Privilege Separation

Established separate Linux and Proxmox administrative identities.

| Identity | Purpose | Access |
|---|---|---|
| `hyp-admin` | Linux system administration | SSH key authentication and sudo |
| `hyp-operator` | Restricted operational access | SSH key authentication, no sudo |
| `pve-admin@pve` | Proxmox platform administration | Web UI Administrator role and TOTP MFA |
| `root` | Emergency recovery | Local console; Proxmox account protected with MFA |

Separate Ed25519 SSH keys were generated for the Linux administrative identities.

SSH client aliases were configured on the Ubuntu administrative workstation to simplify access without sharing private keys.

### 4.2 SSH Hardening

Configured SSH daemon security settings using a dedicated configuration file:

`/etc/ssh/sshd_config.d/00-northwind-hardening.conf`

The following controls were implemented:

| Setting | Value | Purpose |
|---|---|---|
| `PermitRootLogin` | `no` | Prevent direct root SSH authentication |
| `PasswordAuthentication` | `no` | Require stronger authentication methods |
| `PubkeyAuthentication` | `yes` | Permit public-key authentication |
| `KbdInteractiveAuthentication` | `no` | Prevent keyboard-interactive authentication |

The configuration was syntax-checked and the effective settings inspected before reloading SSH.

Positive authentication tests confirmed that authorised accounts could connect using their assigned keys.

Negative tests confirmed that password-only authentication was rejected and the operator account could not obtain sudo privileges.

### 4.3 Proxmox Administrative Access

Created a dedicated Proxmox administrative identity with the Administrator role.

Enabled TOTP-based multifactor authentication for the Proxmox administrative account and root Web UI account.

The Linux administrative identity and Proxmox Web UI identity are maintained separately to distinguish host operating-system administration from platform management.

### 4.4 Host Firewall Configuration

Enabled the Proxmox firewall at Datacenter and HYP01 node levels.

Retained the legacy iptables-based Proxmox firewall backend.

Configured the following management access rules:

| Direction | Source | Protocol | Destination port | Action |
|---|---|---|---|---|
| IN | `192.168.178.64/32` | TCP | 22 | ACCEPT |
| IN | `192.168.178.64/32` | TCP | 8006 | ACCEPT |

Datacenter firewall policies:

- INPUT: DROP
- OUTPUT: ACCEPT
- FORWARD: ACCEPT

Inspection of the effective iptables rules identified Proxmox-generated management exceptions permitting traffic from the automatically detected `192.168.178.0/24` network.

To restrict this broader access, configured a user-defined `local_network` alias for `192.168.178.64/32`.

Subsequent inspection confirmed that `PVEFW-0-management-v4` contained only the approved administrative workstation address.

### 4.5 IPv6 Security Review

Reviewed IPv6 addressing, routing, listening services, firewall rules and management IP sets.

HYP01 had a link-local IPv6 address but no IPv6 default route or global IPv6 address in the inspected configuration.

The generated `PVEFW-0-management-v6` IP set contained zero entries.

The effective IPv6 firewall rules retained necessary ICMPv6 functionality while dropping unmatched incoming traffic.

SSH and Proxmox Web UI connection attempts over HYP01's link-local IPv6 address timed out.

IPv6 was retained rather than disabled system-wide.

### 4.6 Service Minimisation

Reviewed RPCBind after identifying TCP/UDP port 111 as a listening service.

The investigation found:

- No mounted NFS/NFSv4 filesystems.
- No identified Northwind dependency.
- No registered RPC programs beyond portmapper itself.

Disabled `rpcbind.socket` and `rpcbind.service`.

Verified that both units were inactive and disabled, and that port 111 was no longer listening.

Retained the Proxmox SPICE proxy for potential VM console functionality, subject to the existing host firewall management restrictions.

## 5. Validation and Test Evidence

| Test | Expected result | Observed result |
|---|---|---|
| SSH public-key login — administrator | Successful | Pass |
| SSH public-key login — operator | Successful | Pass |
| Password-only SSH authentication | Denied | Pass |
| Direct root password SSH authentication | Denied | Pass |
| Operator sudo access | Denied | Pass |
| Proxmox firewall activation | Rules installed | Pass |
| Approved laptop SSH access | Successful | Pass |
| Approved laptop Web UI access | Successful | Pass |
| Unapproved LAN device to Web UI TCP 8006 | Connection blocked | Pass |
| IPv4 management IP set | Approved workstation only | Pass |
| IPv6 management IP set | No authorised entries | Pass |
| IPv6 SSH connection attempt | Connection blocked | Pass |
| IPv6 Web UI connection attempt | Connection blocked | Pass |
| RPCBind socket/service | Inactive and disabled | Pass |
| RPCBind TCP/UDP 111 | No listener | Pass |
| SSH after RPCBind change | Successful | Pass |
| Physical-console recovery access | Available | Confirmed |

**Validation limitations:** Negative IPv4 testing covered the Web UI from a second device, but not every management port. IPv6 testing covered the specified SSH and Web UI connection paths. These tests do not establish that all possible network paths have been exhaustively assessed.

## 6. Security and Design Decisions

The implementation adopted the following decisions:

1. Use dedicated administrative identities rather than routine root access.
2. Require SSH public-key authentication and disable remote password authentication.
3. Use Proxmox's integrated firewall instead of managing persistent host rules manually.
4. Retain the legacy firewall backend during the initial security baseline.
5. Restrict management to the approved Ubuntu workstation.
6. Override Proxmox's automatically detected `local_network` to prevent broad LAN management exceptions.
7. Preserve IPv6 with restrictive firewall rules rather than disable it unnecessarily.
8. Disable RPCBind where no operational dependency was identified.
9. Retain SPICE console functionality for future VM operations.
10. Preserve physical-console access for emergency recovery.

The rationale, alternatives, trade-offs and review triggers are documented in `ADR-NIM20-001-hyp01-management-security.md`.

## 7. Risks, Limitations and Future Considerations

| Risk or consideration | Mitigation / future action |
|---|---|
| Approved laptop unavailable or changes IP | Use physical console and approved firewall change procedure |
| Proxmox cluster expansion | Revisit `local_network` override and cluster networking requirements |
| IPv6 management required in future | Define an explicit IPv6 allowlist and validate access |
| SPICE console requirements change | Review necessary ports and management policy |
| Future NFS/RPC dependency | Reassess RPCBind before enabling related services |
| Single hypervisor | Maintain recovery procedures and backup strategy |
| Limited negative port testing | Extend validation during subsequent security reviews |

## 8. Recovery and Rollback

Physical-console access to HYP01 was confirmed.

If remote management access is lost:

1. Access HYP01 through the physical console.
2. Authenticate using the approved emergency account.
3. Inspect Proxmox firewall configuration and effective rules.
4. Restore the previous approved management alias or firewall configuration if required.
5. Verify SSH and Web UI connectivity from the administrative workstation.
6. Record the incident and corrective action.

For RPCBind, the previous enabled state can be restored by re-enabling its socket and service if a legitimate dependency emerges.

Avoid manually flushing Proxmox-generated iptables chains as a routine rollback method.

## 9. Findings and Lessons Learned

**Firewall activation is not the same as enforcing an intended policy.** Proxmox automatically generated management exceptions that initially permitted the entire home LAN. Inspecting effective rules and IP sets exposed this behaviour.

**Default firewall policies do not tell the whole story.** The built-in INPUT chain used an ACCEPT policy, but Proxmox's user-defined chains enforced explicit DROP rules.

**Rule matching and chain traversal are fundamental.** Understanding `-N`, `-A`, `-j`, `RETURN`, `ACCEPT` and `DROP` made it possible to explain why some traffic was permitted and other traffic blocked.

**IPv4 and IPv6 must both be considered.** Restricting IPv4 management access alone does not establish that IPv6 management paths are secure.

**Service minimisation should be evidence-led.** RPCBind was disabled only after inspecting active services, NFS mounts and registered RPC programs.

**Administrative recovery is part of security.** Strong access restrictions must be paired with a practical way to recover from mistakes or lockouts.

## 10. Result

HYP01 now has a substantially improved administrative security baseline.

Administrative access uses separated identities, SSH key authentication, Proxmox MFA and host firewall restrictions. The approved workstation can connect using IPv4, while the tested unauthorised IPv4 and IPv6 management paths were blocked.

An unnecessary RPCBind service was disabled and validated without disrupting SSH administration.

**Technical outcome:** Implemented and validated for the documented test cases.

**Formal status:** Pending final documentation review, evidence attachment and Jira closure.

## 11. Next Task

**NIM-21 — Deploy FW01**

Begin deployment of Northwind's dedicated RHEL 10 firewall virtual machine, following the approved VM specifications, addressing plan and implementation dependencies.

---

## Supporting Documentation

- `commands.md` — Commands and explanations encountered during implementation.
- `ADR-NIM20-001-hyp01-management-security.md` — Architecture decision record.
- Jira NIM-20 — Acceptance criteria, implementation evidence and closure.
- Northwind LLD documentation — Security baseline, administration and recovery requirements.
