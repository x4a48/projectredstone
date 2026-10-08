# NIM-15 — HYP01 Command Reference

**Scope:** Commands used during HYP01 documentation and administrative access verification.  
**Platform:** Proxmox VE 9.2 (Debian-based host).  
**Purpose:** Quick operational reference and RHCSA learning notes; not a hardening procedure.

> **RHCSA relevance:** “High” means directly useful for RHEL administration and aligned with common RHCSA skills, not a guarantee that a particular command appears on an exam. “Background” means useful underlying knowledge. “Indirect” means the concept transfers, but the command is Proxmox-specific.

## 1. Services and listening ports

| Command | Platform | RHCSA relevance | Purpose / interpretation |
|---|---|---|---|
| `systemctl status ssh` | General Linux (service name varies) | High | Show whether OpenSSH is loaded, active and enabled. On RHEL the service is normally `sshd`. |
| `systemctl is-active proxmox-firewall` | General `systemctl`; Proxmox service | High (tool) / Indirect (service) | Report whether the nftables-based Proxmox firewall service is currently active. |
| `systemctl is-enabled proxmox-firewall` | General `systemctl`; Proxmox service | High (tool) / Indirect (service) | Report whether that service is configured to start at boot. |
| `ss -tlnp` | General Linux | High | List **T**CP, **L**istening sockets, **N**umeric addresses/ports and **P**rocesses. Run with sufficient privilege to see process names. |

**HYP01 findings:** SSH was active/enabled; TCP/22 and TCP/8006 listened on all interfaces. A listening socket does **not** prove that a remote client can reach it: firewalls and upstream network controls may restrict access.

## 2. SSH authentication configuration

```bash
sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|authenticationmethods|allowusers|allowgroups|listenaddress) '
```

- **`sshd`** — OpenSSH *server daemon*, available on RHEL as well as Debian/Proxmox.
- **`-T`** — Print effective server configuration (including defaults), rather than just showing uncommented lines in a configuration file.
- **`grep -E`** — Use extended regular expressions; `-i` ignores case. The `^` anchors the match at the start of each line and `|` separates alternatives.
- **Caveat:** `sshd -T` without `-C` shows a general effective configuration; `Match` rules can alter settings for particular users or source addresses. For a specific connection, use `sshd -T -C user=<user>,host=<host>,addr=<client-ip>`.

**HYP01 findings:** `permitrootlogin yes`, `passwordauthentication yes`, `pubkeyauthentication yes`, `kbdinteractiveauthentication no`, and `authenticationmethods any`. These settings permit direct root password authentication *at the SSH configuration level*; they do not establish external reachability.

**RHCSA connection:** Configure and troubleshoot OpenSSH server access, understand SSH key authentication, and inspect configuration safely before making changes.

## 3. Proxmox users, realms, permissions and MFA

| Command | Platform | RHCSA relevance | Purpose / interpretation |
|---|---|---|---|
| `pveum user list` | Proxmox only | Indirect | List users registered in Proxmox's access-control system; not all local Linux users. |
| `pveum realm list` | Proxmox only | Indirect | List authentication realms, such as PAM and PVE. |
| `pveum acl list` | Proxmox only | Indirect | Show explicit Proxmox ACL assignments. An empty list does **not** remove `root@pam`'s built-in privileges. |
| `pveum user tfa list root@pam` | Proxmox only | Indirect | List registered two-factor authentication entries for the specified Proxmox account. Blank output indicates no entries were listed. |

**HYP01 findings:** `root@pam` was the sole listed Proxmox user; PAM and PVE realms were present; no explicit ACL entries or registered TFA credentials for `root@pam` were listed.

**RHCSA connection:** The equivalent general concepts are local Linux users/groups, PAM, permissions, `sudo` and authentication controls—not `pveum` itself.

## 4. Firewall services and effective rules

| Command | Platform | RHCSA relevance | Purpose / interpretation |
|---|---|---|---|
| `pve-firewall status` | Proxmox only | Indirect | Check **legacy** Proxmox firewall status. `disabled/running` means its enforcement is disabled even though its service is running. |
| `systemctl is-active proxmox-firewall` | Proxmox service, general Linux tool | High (tool) / Indirect (service) | Check the nftables-based Proxmox firewall daemon's running state. |
| `systemctl is-enabled proxmox-firewall` | Proxmox service, general Linux tool | High (tool) / Indirect (service) | Check whether the daemon starts at boot. |
| `nft list ruleset` | General Linux | Background | Display currently loaded nftables tables, chains and rules. Requires appropriate privileges. |

**HYP01 findings:** Legacy firewall `disabled/running`; `proxmox-firewall` active/enabled; `nft list ruleset` returned no rules. **Running service ≠ filtering policy enforced.** This check does not by itself rule out every possible network or host access control.

**RHCSA connection:** RHEL typically administers host firewall policy with `firewalld` and `firewall-cmd`. `nft` is useful for understanding/troubleshooting the underlying packet-filtering framework; use the approved firewall-management tool for configuration.

## 5. Proxmox storage

| Command | Platform | RHCSA relevance | Purpose / interpretation |
|---|---|---|---|
| `pvesm status` | Proxmox only | Indirect | List configured Proxmox storage IDs, types, status and capacity. |
| `pvesm list <storage-id>` | Proxmox only | Indirect | List volumes available through a named Proxmox storage definition. |

**Important distinction:** `pvesm` manages the Proxmox *storage abstraction* (for example, `local-lvm` and `northwind-workload`). On RHEL, the underlying administration skills involve commands such as `lsblk`, `pvs`, `vgs`, `lvs`, `findmnt` and filesystem tools. Those commands are not substitutes for `pvesm`, but help explain the storage beneath it.

## 6. Safe verification checklist

1. Confirm service state with `systemctl` and socket state with `ss`.
2. Inspect effective SSH authentication settings with `sshd -T`.
3. Review Proxmox identities, realms, ACLs and MFA with `pveum`.
4. Distinguish the legacy and nftables-based Proxmox firewall services.
5. Inspect active nftables rules with `nft list ruleset`.
6. Record **observed results**, **limitations**, and **follow-up actions** in the implementation record; keep only the current-state summary in `servers/HYP01/README.md`.

**Security:** Redact personal email addresses, passwords, secrets, tokens and identifying inventory data before publishing terminal output or screenshots.

**Scope boundary:** These are read-only inspection examples. Hardening and firewall changes belong to a separately planned implementation task, with validated alternative administrative access and rollback procedures.
