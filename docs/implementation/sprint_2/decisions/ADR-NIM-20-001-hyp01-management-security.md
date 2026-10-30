# ADR-NIM20-001 — HYP01 Management Access Security

| Field | Value |
|---|---|
| **Project** | Northwind Outdoor Ltd. — Enterprise Linux Infrastructure |
| **Decision ID** | ADR-NIM20-001 |
| **Title** | Secure Hypervisor Management Access |
| **Status** | Accepted — Implemented |
| **Related Jira Ticket** | NIM-20 |
| **Affected System** | HYP01 — Proxmox VE 9.2.1 |
| **Decision Owner** | Northwind Infrastructure Consultancy |
| **Decision Date** | 30 October 2026 |
| **Review Trigger** | Management architecture change, cluster expansion or security review |

---

## 1. Context

HYP01 provides the virtualisation platform for Northwind Outdoor Ltd.'s planned Linux infrastructure.

As the hypervisor will host multiple infrastructure and business-service virtual machines, compromise of its administrative interfaces could affect the confidentiality, integrity and availability of the wider environment.

The initial configuration presented several security considerations:

- Remote administrative access required formal identity and privilege separation.
- SSH authentication and root access required hardening.
- Proxmox management interfaces were reachable from the wider home LAN.
- The Proxmox firewall automatically trusted the detected local network (`192.168.178.0/24`) for certain management services.
- IPv6 management exposure had not been validated.
- RPCBind was enabled despite no identified operational requirement.
- Restrictive firewall changes introduced a potential administrative lockout risk.

Northwind requires a secure and recoverable management baseline before additional virtual machines are deployed.

The current environment consists of one standalone Proxmox hypervisor and one approved Ubuntu administrative workstation.

## 2. Decision

Northwind will adopt a **restricted, identity-based administrative access model** for HYP01.

The implementation comprises the following controls.

### 2.1 Administrative identities

Separate administrative identities will be maintained for Linux host administration and Proxmox platform management.

- `hyp-admin` — Linux administrative account with controlled sudo access.
- `hyp-operator` — Linux operational account without sudo privileges.
- `pve-admin@pve` — Dedicated Proxmox administrative identity.
- `root` — Reserved for emergency local recovery and authorised platform administration.

Routine direct root SSH access is prohibited.

### 2.2 SSH authentication

SSH access will use Ed25519 public-key authentication.

The following SSH daemon settings are enforced:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
```

Separate private keys will be maintained for distinct administrative identities.

### 2.3 Proxmox authentication

Proxmox administrative accounts will use multifactor authentication.

TOTP MFA is enabled for the dedicated Proxmox administrator and root Web UI account.

### 2.4 Host firewall

The integrated Proxmox firewall will be used to protect HYP01.

The existing legacy iptables backend will be retained for the initial implementation.

Firewall policy:

| Direction | Policy |
|---|---|
| INPUT | DROP |
| OUTPUT | ACCEPT |
| FORWARD | ACCEPT |

Incoming SSH (TCP 22) and Proxmox Web UI (TCP 8006) access will be explicitly permitted from the approved administrative workstation:

`192.168.178.64/32`

Proxmox's automatically detected `local_network` will be overridden using a user-defined alias to prevent the entire `192.168.178.0/24` subnet from qualifying for built-in management exceptions.

The effective generated IPv4 management IP set must contain only the approved workstation address.

### 2.5 IPv6 management access

IPv6 will remain enabled.

Incoming IPv6 management access will be restricted through the Proxmox host firewall.

No IPv6 addresses are currently authorised in the generated management IP set.

Required IPv6 network functions, including Neighbor Discovery, will be preserved.

### 2.6 Service minimisation

RPCBind will be disabled while no operational dependency exists.

Both `rpcbind.socket` and `rpcbind.service` will remain disabled unless a future approved requirement justifies their use.

The Proxmox SPICE proxy will be retained for potential virtual-machine console functionality, with access governed by the host firewall.

### 2.7 Administrative recovery

Physical-console access will be retained as the emergency recovery method.

Firewall changes must be performed with a verified recovery path and followed by positive and negative access tests.

## 3. Alternatives Considered

| Alternative | Decision | Rationale |
|---|---|---|
| Permit management from entire home LAN | Rejected | Broader exposure than operationally required |
| Use root for routine SSH administration | Rejected | Weak privilege separation and accountability |
| Retain password-based SSH authentication | Rejected | Public-key authentication better supports the approved security baseline |
| Maintain firewall rules directly using iptables | Rejected | Proxmox should remain the authoritative firewall configuration manager |
| Migrate immediately to nftables backend | Deferred | Introduces additional change risk during initial hardening |
| Disable IPv6 entirely | Rejected for current scope | Restrictive firewall rules provide management protection without removing IPv6 functionality |
| Disable SPICE proxy | Deferred | Console functionality may be needed during VM deployment |
| Retain RPCBind | Rejected for current scope | No active dependency was identified |

## 4. Rationale

The selected approach balances security, operational simplicity and recoverability.

Using Proxmox's integrated firewall maintains consistency with the virtualisation platform and avoids conflicts with automatically generated rules.

Restricting management to one approved workstation reduces unnecessary exposure on the shared home network.

Separate administrative identities support least privilege and clearer accountability.

SSH key authentication and Proxmox MFA provide complementary controls for different administrative interfaces.

Retaining the existing firewall backend avoids introducing an unrelated platform migration during the security-baseline implementation.

Preserving physical-console access provides a recovery mechanism if remote management becomes unavailable.

## 5. Consequences

### Positive consequences

- Reduced exposure of privileged management interfaces.
- Stronger remote authentication.
- Separation between administrative and operational privileges.
- Consistent platform-managed firewall configuration.
- IPv4 and IPv6 management paths explicitly reviewed.
- Reduced attack surface through removal of an unnecessary network service.
- Documented recovery and validation procedures.

### Negative consequences and trade-offs

- Remote administration depends on the approved workstation and its reserved IPv4 address.
- Workstation replacement or address changes require a controlled firewall update.
- Restrictive firewall changes can cause administrative lockout.
- The `local_network` override must be reviewed before introducing additional Proxmox nodes.
- Future IPv6 management access requires an explicit policy change.
- The legacy firewall backend will require future lifecycle review.
- Additional administrators or management devices require approved access-policy changes.

## 6. Implementation and Validation

The decision was implemented during NIM-20.

Evidence collected includes:

- Successful SSH public-key authentication for authorised Linux accounts.
- Rejection of password-only SSH authentication.
- Verification of restricted sudo permissions.
- Inspection of effective SSH daemon configuration.
- Confirmation of active Proxmox firewall chains.
- Verification that the generated IPv4 management IP set contains only `192.168.178.64`.
- Successful SSH and Web UI access from the approved workstation.
- Failed Web UI connection from another device on the home LAN.
- Verification that the IPv6 management IP set contains zero entries.
- Failed IPv6 SSH and Web UI connection attempts.
- Confirmation that RPCBind units are inactive and disabled.
- Confirmation that TCP/UDP port 111 is no longer listening.
- Successful SSH authentication following RPCBind removal.
- Confirmation of physical-console recovery access.

**Validation limitation:** The recorded negative tests cover specified management paths and do not constitute exhaustive testing of every service, protocol or possible source network.

## 7. Rollback and Recovery

If a firewall change causes loss of remote administrative access:

1. Access HYP01 through its physical console.
2. Authenticate using the authorised emergency account.
3. Inspect the current Proxmox firewall configuration.
4. Restore the previously approved management-access configuration.
5. Verify that the Proxmox firewall is operating correctly.
6. Establish fresh SSH and Web UI connections.
7. Document the incident and corrective actions.

Proxmox-generated iptables chains must not be manually flushed as a routine rollback procedure.

If an approved service subsequently requires RPCBind, its socket and service may be re-enabled following dependency and security review.

## 8. Review Triggers

This decision must be reviewed when:

- JUMP01 becomes Northwind's primary administrative entry point.
- Additional administrators or approved workstations are introduced.
- A second Proxmox hypervisor is deployed.
- Proxmox clustering is implemented.
- Remote administration from outside the local network is introduced.
- IPv6 management access becomes a business requirement.
- A migration to the nftables firewall backend is proposed.
- NFS or another RPC-dependent service is introduced.
- A security audit identifies weaknesses in the current management model.

## 9. Related Documentation

- NIM-20 — Validate and Secure HYP01 Management Access
- NIM-20 `README.md`
- NIM-20 `commands.md`
- Northwind Security Baseline LLD
- Northwind Remote Administration and Recovery LLDs
- NIM-21 — Deploy FW01

---

## 10. Decision Outcome

**Accepted and implemented.**

HYP01 now uses separated administrative identities, hardened SSH authentication, Proxmox MFA and restricted host-management access.

The selected configuration meets the initial single-hypervisor security requirements within the documented validation scope.

The decision remains subject to review as Northwind's infrastructure and management architecture evolve.
