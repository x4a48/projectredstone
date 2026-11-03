# NIM-20 — HYP01 Secure Management Access Deployment Runbook

**Project:** Northwind Outdoor Ltd.  
**System:** HYP01 — Proxmox VE 9.2.1  
**Related Jira:** NIM-20  
**Document Type:** Deployment and Validation Procedure  
**Status:** Draft — requires comparison against original command history  
**Execution:** Authorised Linux administrator with physical-console recovery available

---

## 1. Purpose

This runbook describes the repeatable procedure for securing administrative access to the Northwind Proxmox hypervisor.

It covers:

1. Pre-deployment checks and recovery preparation.
2. Linux administrator and operator account configuration.
3. SSH public-key authentication and daemon hardening.
4. Proxmox administrative account and MFA configuration.
5. Host firewall configuration.
6. IPv4 management access restrictions.
7. IPv6 security validation.
8. Unnecessary service removal.
9. Final acceptance testing.

This procedure applies to the standalone HYP01 deployment. It must be reviewed before reuse on a cluster or another Proxmox version.

## 2. Target Configuration

| Parameter | Value |
|---|---|
| Hypervisor | HYP01 |
| Management IPv4 | `192.168.178.10/24` |
| Default gateway | `192.168.178.1` |
| Approved workstation | `192.168.178.64/32` |
| SSH port | TCP 22 |
| Proxmox Web UI | TCP 8006 |
| Firewall backend | Legacy Proxmox firewall (iptables) |
| Incoming firewall policy | DROP |
| Outgoing firewall policy | ACCEPT |
| Forwarding policy | ACCEPT |
| IPv6 management allowlist | Empty |
| Emergency access | Physical console |

The addresses above represent the recorded NIM-20 deployment. Adapt them only through an approved configuration change.

## 3. Prerequisites and Safety Checks

Before implementation, confirm:

- Physical access to HYP01 is available.
- The root console credentials are available through approved secure storage.
- Proxmox Web UI access is operational.
- The approved administrative workstation has a stable address or DHCP reservation.
- Current Proxmox configuration has been backed up or a known-good recovery method is available.
- A maintenance window has been agreed if hosted workloads could be affected.
- A second SSH session remains open during SSH and firewall changes.

**Critical:** Never disable password authentication until public-key authentication has been successfully tested in a separate session. Never activate a restrictive firewall without verified console recovery.

### Step 3.1 — Verify host identity

```bash
hostnamectl
```

Displays the hostname, operating system and system architecture.

```bash
ip -br addr
```

Displays a concise summary of network interfaces and their addresses.

- `ip` — Linux network configuration utility.
- `-br` — Brief output.
- `addr` — Display interface addresses.

```bash
ip route
```

Displays the IPv4 routing table.

**Expected:** HYP01 has the approved management address and a route through the home network gateway.

### Step 3.2 — Confirm Proxmox version

```bash
pveversion
```

Displays the installed Proxmox VE version.

Record the version before making changes.

### Step 3.3 — Record initial firewall state

```bash
sudo iptables -S
sudo ip6tables -S
sudo nft list ruleset
```

- `iptables` — Inspects IPv4 firewall rules.
- `ip6tables` — Inspects IPv6 firewall rules.
- `-S` — Displays rules in command-style syntax.
- `nft list ruleset` — Displays rules managed through nftables.

These commands are read-only.

Save the output as implementation evidence.

## 4. Linux Administrative Accounts

### Step 4.1 — Create the administrative account

The original account-creation commands should be verified against `commands.md`.

A reproducible Debian-based procedure is:

```bash
sudo adduser hyp-admin
```

Creates a Linux user, including a home directory and initial account configuration.

Complete the interactive prompts using the approved credential procedure.

### Step 4.2 — Create the operator account

```bash
sudo adduser hyp-operator
```

Creates the restricted operational account.

The operator must not receive unrestricted administrative privileges.

### Step 4.3 — Configure sudo for the administrator

First, verify sudo is installed:

```bash
command -v sudo
```

Displays the executable path if sudo is available.

If sudo is missing, install it through the approved Debian package-management procedure from the root console.

Then configure the administrative account:

```bash
sudo usermod -aG sudo hyp-admin
```

- `usermod` — Modifies an existing Linux account.
- `-a` — Appends supplementary group membership.
- `-G sudo` — Specifies the `sudo` group.
- `hyp-admin` — Account to modify.

The `-a` option is important: omitting it can replace existing supplementary group memberships.

### Step 4.4 — Validate permissions

```bash
id hyp-admin
id hyp-operator
```

Displays each account's user ID and group memberships.

From a new `hyp-admin` session:

```bash
sudo -l
```

Lists permitted sudo operations.

From a `hyp-operator` session, run the same command and confirm administrative privileges are denied.

**Checkpoint:** Administrator has the intended privileges; operator does not.

## 5. SSH Key Authentication

These steps are performed on the Ubuntu administrative workstation unless otherwise stated.

### Step 5.1 — Generate the administrator key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/northwind_hyp_admin
```

- `ssh-keygen` — Generates SSH authentication keys.
- `-t ed25519` — Selects the Ed25519 key algorithm.
- `-f` — Specifies the output file.

Protect the private key with a strong passphrase.

### Step 5.2 — Generate the operator key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/northwind_hyp_operator
```

Creates a separate key pair for the operator identity.

Private keys must remain on the administrative workstation and must never be committed to GitHub.

### Step 5.3 — Install the administrator public key

```bash
ssh-copy-id -i ~/.ssh/northwind_hyp_admin.pub hyp-admin@192.168.178.10
```

- `ssh-copy-id` — Installs a public key into the remote user's authorised-key file.
- `-i` — Specifies which public key to install.
- `.pub` — Public-key file.
- `hyp-admin@192.168.178.10` — Remote account and host.

This requires an existing permitted authentication method. If password authentication is already disabled, use an approved console-based installation procedure instead.

### Step 5.4 — Install the operator public key

```bash
ssh-copy-id -i ~/.ssh/northwind_hyp_operator.pub hyp-operator@192.168.178.10
```

### Step 5.5 — Test administrator key authentication

```bash
ssh -i ~/.ssh/northwind_hyp_admin \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  -o PasswordAuthentication=no \
  hyp-admin@192.168.178.10
```

- `-i` — Specifies the private key.
- `-o` — Sets an SSH client option.
- `IdentitiesOnly=yes` — Limits authentication to explicitly configured identities.
- `PreferredAuthentications=publickey` — Prefers public-key authentication.
- `PasswordAuthentication=no` — Prevents the client from falling back to password authentication.

**Expected:** Successful login without remote account-password authentication.

Repeat with the operator key and identity.

**Checkpoint:** Both identities authenticate successfully before proceeding.

## 6. SSH Daemon Hardening

Perform these steps on HYP01.

### Step 6.1 — Create the hardening configuration

Use an approved editor to create:

`/etc/ssh/sshd_config.d/00-northwind-hardening.conf`

Contents:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
```

### Step 6.2 — Validate syntax

```bash
sudo sshd -t
```

- `sshd` — OpenSSH server daemon.
- `-t` — Tests configuration syntax and key validity.

**Expected:** No errors.

### Step 6.3 — Inspect effective settings

```bash
sudo sshd -T
```

- `-T` — Displays the effective SSH server configuration.

Confirm the intended authentication settings are active. Account- or source-specific `Match` rules may require testing with connection criteria.

### Step 6.4 — Reload SSH

```bash
sudo systemctl reload ssh
```

- `systemctl` — Controls systemd units.
- `reload` — Requests configuration reload without a full service stop.
- `ssh` — Debian OpenSSH server unit.

Keep the existing SSH session open.

### Step 6.5 — Validate in a new terminal

Open a fresh administrator SSH session.

Verify:

- Administrator public-key login succeeds.
- Operator public-key login succeeds.
- Password-only authentication is denied.
- Direct root SSH authentication is denied according to the effective policy.
- Operator sudo access is denied.

**Checkpoint:** Do not proceed until new SSH sessions work.

## 7. Proxmox Administrative Identity and MFA

Use the Proxmox Web UI.

### Step 7.1 — Create the administrator identity

Navigate to:

**Datacenter → Permissions → Users**

Create:

`pve-admin@pve`

Use the Proxmox internal authentication realm.

### Step 7.2 — Assign permissions

Navigate to:

**Datacenter → Permissions**

Assign the Administrator role to the new identity at the root resource path `/`, as approved for the current single-node lab.

Verify that the account can access the required administrative functions.

### Step 7.3 — Configure MFA

Configure TOTP multifactor authentication for:

- `pve-admin@pve`
- `root@pam`

Store recovery information in approved secure storage.

Do not include recovery codes, TOTP seeds or QR codes in screenshots or GitHub documentation.

### Step 7.4 — Validate

Log out and authenticate in a new browser session using the dedicated Proxmox administrator account and MFA.

Confirm the expected administrative permissions.

**Checkpoint:** Dedicated Proxmox administration works before firewall changes.

## 8. Proxmox Firewall Deployment

### Step 8.1 — Confirm backend and initial state

```bash
sudo pve-firewall status
```

Displays the Proxmox legacy firewall service status.

```bash
sudo pve-firewall localnet
```

Displays the local network detected or configured for Proxmox firewall management exceptions.

Before the override, HYP01 detected:

`192.168.178.0/24`

This was broader than Northwind's approved management source.

### Step 8.2 — Enable the Datacenter firewall

Navigate to:

**Datacenter → Firewall → Options**

Configure:

| Option | Value |
|---|---|
| Firewall | Yes |
| Input Policy | DROP |
| Output Policy | ACCEPT |
| Forward Policy | ACCEPT |

Keep the Web UI session open.

### Step 8.3 — Enable the node firewall

Navigate to:

**HYP01 → Firewall → Options**

Set:

- Firewall: Yes
- nftables (tech preview): No

Do not switch firewall backends as part of this NIM.

### Step 8.4 — Create explicit management rules

Navigate to:

**HYP01 → Firewall**

Add these incoming rules:

| Type | Action | Source | Protocol | Destination port |
|---|---|---|---|---|
| IN | ACCEPT | `192.168.178.64/32` | TCP | 22 |
| IN | ACCEPT | `192.168.178.64/32` | TCP | 8006 |

Ensure both rules are enabled.

**Important:** These explicit rules alone do not remove Proxmox's automatically generated local-network management exceptions.

### Step 8.5 — Restrict the automatic management network

Configure the reserved `local_network` alias through the Proxmox Datacenter firewall alias interface:

**Datacenter → Firewall → Alias**

Create:

| Name | IP/CIDR |
|---|---|
| `local_network` | `192.168.178.64/32` |

This is a special Proxmox alias used to override the automatically detected management network.

Confirm the resulting configuration using:

```bash
sudo pve-firewall localnet
```

**Expected:**

```text
using user defined local_network: 192.168.178.64
```

The output may display the address without the `/32` suffix.

### Step 8.6 — Verify the generated management set

```bash
sudo ipset list PVEFW-0-management-v4
```

- `ipset` — Manages collections of addresses used by firewall rules.
- `list` — Displays a set and its members.
- `PVEFW-0-management-v4` — Proxmox-generated IPv4 management set.

**Expected:** One entry, `192.168.178.64`.

### Step 8.7 — Inspect effective firewall rules

```bash
sudo iptables -S PVEFW-HOST-IN
```

Displays the rules in the Proxmox host input chain.

Verify that the explicit TCP 22 and 8006 source rules exist and that unmatched incoming traffic reaches DROP rules.

### Step 8.8 — Validate IPv4 access

From the approved laptop:

- Open a new SSH connection.
- Open a fresh Proxmox Web UI session.

From another device on the same LAN:

- Attempt a connection to TCP 8006.
- Confirm that the connection is blocked.

Record both successful and unsuccessful tests.

**Checkpoint:** The approved workstation retains access; the tested unapproved device does not.

## 9. IPv6 Security Validation

### Step 9.1 — Inspect IPv6 addressing

```bash
ip -6 addr show
```

- `-6` — Selects IPv6.
- `addr show` — Displays interface addresses.

### Step 9.2 — Inspect IPv6 routes

```bash
ip -6 route show
```

Displays the IPv6 routing table.

The recorded HYP01 configuration had only a link-local route and no IPv6 default route.

### Step 9.3 — Inspect IPv6 firewall rules

```bash
sudo ip6tables -S
```

Verify that management exceptions reference `PVEFW-0-management-v6` and unmatched incoming traffic reaches DROP rules.

### Step 9.4 — Inspect the management IP set

```bash
sudo ipset list PVEFW-0-management-v6
```

**Expected:** Zero entries.

### Step 9.5 — Inspect listening TCP services

```bash
sudo ss -lntp -6
```

- `ss` — Displays socket information.
- `-l` — Listening sockets.
- `-n` — Numeric addresses and ports.
- `-t` — TCP sockets.
- `-p` — Associated processes.
- `-6` — IPv6 only.

### Step 9.6 — Test IPv6 management connectivity

From the Ubuntu laptop, identify the local interface:

```bash
ip -br link
```

Then attempt SSH:

```bash
ssh -6 -o ConnectTimeout=5 \
  hyp-admin@fe80::1e69:7aff:febc:f1f9%YOUR_INTERFACE
```

Replace `YOUR_INTERFACE` with the laptop's actual LAN interface.

- `-6` — Forces IPv6.
- `ConnectTimeout=5` — Limits connection setup time to five seconds.
- `%YOUR_INTERFACE` — Identifies the interface for the link-local destination.

Test the Web UI:

```bash
curl -g -6 -k -I --connect-timeout 5 \
  'https://[fe80::1e69:7aff:febc:f1f9%YOUR_INTERFACE]:8006/'
```

- `-g` — Disables curl URL globbing.
- `-6` — Uses IPv6.
- `-k` — Skips certificate verification for this diagnostic.
- `-I` — Requests HTTP response headers.
- `--connect-timeout 5` — Limits connection setup time.

**Expected:** Both tested management connections fail.

Do not treat timeouts alone as proof of the firewall's cause; interpret them alongside routing and effective firewall rules.

## 10. RPCBind Service Review and Removal

### Step 10.1 — Inspect reverse dependencies

```bash
systemctl list-dependencies --reverse rpcbind.socket
```

- `list-dependencies` — Displays unit dependencies.
- `--reverse` — Shows units depending on the specified unit.

### Step 10.2 — Check NFS mounts

```bash
findmnt -t nfs,nfs4
```

- `findmnt` — Displays mounted filesystems.
- `-t` — Filters filesystem types.
- `nfs,nfs4` — NFS and NFSv4 filesystems.

**Recorded result:** No NFS mounts.

### Step 10.3 — Inspect unit startup state

```bash
systemctl list-unit-files 'rpcbind*'
```

Displays RPCBind-related unit files and enablement states.

### Step 10.4 — Inspect RPC registrations

```bash
sudo rpcinfo -p localhost
```

- `rpcinfo` — Queries registered RPC services.
- `-p` — Lists registered programs and ports.
- `localhost` — Queries the local system.

**Recorded result:** Only portmapper registrations.

### Step 10.5 — Disable RPCBind

```bash
sudo systemctl disable --now rpcbind.socket rpcbind.service
```

- `disable` — Prevents automatic startup through normal enablement links.
- `--now` — Also stops the units immediately.
- `rpcbind.socket` — Socket-activation unit.
- `rpcbind.service` — RPCBind daemon.

### Step 10.6 — Validate

```bash
systemctl is-active rpcbind.socket rpcbind.service
systemctl is-enabled rpcbind.socket rpcbind.service
sudo ss -lntup | grep ':111 '
```

The last command checks TCP and UDP listeners for port 111.

**Expected:**

- Both units inactive.
- Both units disabled.
- No TCP/UDP port 111 listener.

Verify a fresh SSH login and Proxmox Web UI access.

## 11. Final Acceptance Checklist

- [ ] Administrator SSH key authentication works.
- [ ] Operator SSH key authentication works.
- [ ] Operator has no unintended sudo privileges.
- [ ] Direct root SSH access is disabled.
- [ ] Password-only SSH authentication is rejected.
- [ ] Dedicated Proxmox administrator and MFA work.
- [ ] Proxmox host firewall is enabled.
- [ ] IPv4 management IP set contains only the approved source.
- [ ] Approved laptop SSH and Web UI access work.
- [ ] Unapproved LAN Web UI connection is blocked.
- [ ] IPv6 management IP set is empty.
- [ ] Tested IPv6 management connections fail.
- [ ] RPCBind is inactive and disabled.
- [ ] Port 111 is not listening.
- [ ] Physical-console recovery is available.
- [ ] Evidence is recorded and secrets are excluded.
- [ ] `README.md`, `commands.md`, ADR and rollback runbook are updated.

## 12. Glossary

| Term | Definition |
|---|---|
| ADR | Architecture Decision Record; explains a technical decision and its rationale |
| Authentication | Verifying the identity of a user or system |
| Authorisation | Determining what an authenticated identity may do |
| CIDR | Classless Inter-Domain Routing; notation used to describe IP networks and address prefixes |
| Conntrack | Linux connection tracking; records connection states for firewall processing |
| DROP | Firewall action that silently discards matching traffic |
| Ed25519 | Modern public-key signature algorithm commonly used for SSH authentication |
| Firewall chain | Ordered sequence of packet-filtering rules |
| IP set | Collection of IP addresses or networks referenced by firewall rules |
| iptables | Linux firewall rule-management interface used by the legacy Proxmox firewall |
| IPv6 link-local address | IPv6 address used for communication on the local network link |
| Least privilege | Granting only the permissions needed to perform a task |
| MFA | Multifactor authentication; requires more than one authentication factor |
| NFS | Network File System; protocol for accessing files over a network |
| Proxmox VE | Virtualisation platform used to host Northwind's virtual machines |
| RPC | Remote Procedure Call; mechanism allowing software to invoke functions across processes or systems |
| RPCBind | Service that maps registered RPC programs to network ports |
| SSH | Secure Shell; encrypted protocol for remote system administration |
| sudo | Mechanism for running permitted commands with another user's privileges |
| systemd | Linux service and system manager |
| TOTP | Time-based One-Time Password; temporary code used for authentication |
| VM | Virtual machine; software-defined computer hosted by a hypervisor |
| Web UI | Browser-based management interface |
