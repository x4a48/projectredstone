### NIM-20-DD01 — Administrative Identity Separation

**Status:** Approved  
**Scope:** HYP01 — Proxmox VE / Debian Linux

**Decision**

Northwind will implement separate administrative identities for Proxmox management and underlying Linux administration, applying least privilege and separating routine operations from privileged activities.

| Identity | Purpose | Access Model |
|---|---|---|
| `pve-admin@pve` | Proxmox Web UI administration | Proxmox RBAC with MFA |
| `hypervisor admin` | Privileged HYP01 Linux administration | SSH key authentication; full `sudo` |
| `hypervisor operator` | Routine HYP01 Linux administration | SSH key authentication; task-specific `sudo` permissions |
| `root` | Emergency recovery | Physical console; protected credentials |

**Rationale**

- Separate hypervisor management from operating system administration.
- Apply least privilege to routine administrative activities.
- Enable controlled privilege elevation without requiring routine root logins.
- Improve accountability through named administrative accounts.
- Support RHCSA learning through practical user, group, SSH and `sudo` administration.

**Operational Workflow**

1. `hyp-operator` identifies required elevated permissions.
2. Permissions are reviewed and granted by `hyp-admin` through an approved sudo policy.
3. The operator performs and validates the task.
4. Temporary permissions are revoked when no longer required.
5. Permission changes and validation evidence are recorded in the relevant NIM.

**Implementation & Security Controls**

- Use separate SSH keys for `hyp-admin` and `hyp-operator`.
- Establish and validate alternative administrative access before modifying existing authentication controls.
- Disable direct root SSH login and SSH password authentication only after successful validation.
- Retain physical root console access for emergency recovery.
- Review and restrict HYP01 management network exposure separately.

**Implementation Status:** Approved — not yet implemented or validated.
