### LLD-01 Design Reconciliation — NIM-15

**Design baseline:** LLD-01 — Platform & Infrastructure Foundation, v1.0 (Approved / Baselined)

**Implementation scope:** Stage 0 — Bootstrap Network Foundation and Stage 1 — HYP01

**Assessment:** Substantially conforms, with one historical validation exception and outstanding verification activities.

The Sprint 1 implementation of HYP01 aligns with the approved physical platform, dual-boot operating model, hypervisor selection, physical storage roles and recovery-based resilience approach.

The temporary management network and direct administrative access are consistent with the staged implementation approach. These arrangements must be reviewed and restricted or replaced as the approved administrative and network architecture is introduced.

**Validation exception:** TEST01's temporary operating-system disk was placed on the dedicated workload NVMe. Appendix A.1 specifies that permanent VM operating-system disks should normally reside on the infrastructure/system NVMe, with workload/business-data disks allocated separately. TEST01 has been removed; no permanent workload requires remediation.

**Implementation decisions:** The Proxmox storage backend, LVM Thin configuration and capacity allocations were determined during implementation, consistent with the detailed decisions deferred by LLD-01.

**Remaining verification:**

- Complete the administrative access configuration review.
- Confirm that future VM build standards preserve the approved system-disk/data-disk separation.
- Update the System Record to clarify that final management addressing is determined by the downstream network design.

**Disposition:** No confirmed material deviation currently requires redesign of the HYP01 foundation. Final NIM-15 acceptance remains subject to the outstanding verification and documentation updates.

### NIM-15-F01 — Bootstrap SSH Authentication ###
HYP01 currently permits direct root SSH authentication using passwords or public keys. SSH listens on all IPv4 and IPv6 interfaces.
This configuration is retained temporarily during the infrastructure bootstrap phase. The effective network exposure remains subject to firewall verification.
Disposition: Review and harden SSH authentication when implementing the approved administrative access architecture. Validate alternative administrative and recovery access before disabling bootstrap authentication.

## NIM-15-F02 — Management Firewall Not Enforcing nftables Rules ##
HYP01 currently has no active nftables rules despite the proxmox-firewall service being active and enabled.
SSH and the Proxmox management interface listen on all interfaces. Access may still be constrained by upstream network controls, but no nftables restrictions are currently applied locally.
Disposition: Retain as a documented bootstrap security finding. Review and implement appropriate firewall restrictions as part of the approved administrative access and network security design.
