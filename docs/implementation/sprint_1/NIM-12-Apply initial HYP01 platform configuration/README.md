# NIM-12 — Configure HYP01 Platform Baseline

## Overview

This task established and validated the initial operating baseline for **HYP01** following the installation of Proxmox VE and configuration of local storage.

The objective was to ensure that HYP01 had a consistent system identity, correct time configuration, working network and name resolution, appropriate software repositories, an up-to-date Proxmox installation, and no unresolved platform-level service failures before virtual workloads were introduced.

---

## Target Component

**HYP01 — Northwind v1 Virtualisation Host**

---

## Objectives

- Validate HYP01 system identity.
- Validate hostname and local name resolution.
- Configure and validate system time and timezone.
- Validate NTP synchronisation.
- Validate bootstrap network configuration.
- Review and configure software repositories.
- Disable repositories requiring an unavailable subscription.
- Enable the appropriate Proxmox repository for the lab environment.
- Review available package updates before installation.
- Update HYP01 to the current stable platform release.
- Reboot and perform post-update validation.
- Confirm that core Proxmox services and storage remained healthy.

---

## System Identity

The following identity was confirmed:

```text
Hostname: hyp01
FQDN:     hyp01.northwind.internal
OS:       Debian GNU/Linux 13 (trixie)
```

Local hostname resolution was configured through `/etc/hosts`:

```text
192.168.178.10 hyp01.northwind.internal hyp01
```

This allowed the host's short name and FQDN to resolve locally during the bootstrap phase.

---

## Time Configuration

HYP01 was configured with:

```text
Timezone: Europe/Berlin
RTC:      UTC
```

Chrony was confirmed as the active network time synchronisation service.

The host successfully communicated with multiple external NTP sources and selected a valid synchronisation source.

Accurate system time is important for:

- Log correlation
- Authentication
- Scheduled operations
- Monitoring
- Troubleshooting
- Future distributed Northwind services

External NTP remains a bootstrap dependency until the final Northwind time-service design is implemented.

---

## Bootstrap Network Validation

The existing bootstrap network configuration was validated:

```text
HYP01:   192.168.178.10/24
Gateway: 192.168.178.1
DNS:     192.168.178.1
```

The management address remained configured on the Proxmox Linux bridge `vmbr0`, with the physical Ethernet interface acting as the bridge port.

Connectivity to the local gateway and external network was confirmed.

DNS resolution was also validated.

The bootstrap network remains temporary and will eventually be replaced by the final Northwind management-zone configuration.

---

## Software Repository Configuration

The configured Debian and Proxmox package repositories were reviewed.

The following repository sources were enabled:

```text
Debian Trixie
Debian Trixie Updates
Debian Security
Proxmox VE No-Subscription
```

The following were disabled:

```text
Proxmox VE Enterprise
Ceph Enterprise
```

The enterprise repositories require an appropriate Proxmox subscription and were therefore not suitable for the current Northwind lab environment.

The `pve-no-subscription` repository was enabled instead.

No `pve-test` repository was enabled.

> The no-subscription repository is appropriate for this lab environment but should not automatically be treated as the preferred repository strategy for a production enterprise deployment.

---

## Package Update Review

The package metadata was refreshed and available upgrades were reviewed before making changes.

A simulated full upgrade was performed first.

The simulation reported approximately:

```text
247 packages upgrading
8 packages installing
1 package removing
```

The package removal was investigated before proceeding.

It represented a package transition from:

```text
libzfs6linux
```

to:

```text
libzfs7linux
```

rather than an unexpected removal of required ZFS functionality.

This demonstrated the importance of reviewing package-manager actions before approving significant platform updates.

---

## Proxmox Update

After reviewing the proposed package changes and relevant Proxmox release considerations, the full system upgrade was performed.

HYP01 was subsequently rebooted.

The update moved the host from the initially installed Proxmox VE 9.1 platform to Proxmox VE 9.2.

---

## Post-Update Baseline

Following the reboot, the platform reported:

```text
proxmox-ve:  9.2.0
pve-manager: 9.2.21
Kernel:      7.0.14-20-pve
```

This established the updated HYP01 software baseline for subsequent Sprint 1 validation.

---

## Platform Health Validation

Following the update and reboot, HYP01 was checked for failed systemd units.

Result:

```text
0 loaded units listed
```

The core Proxmox services were also confirmed active:

```text
pveproxy
pvedaemon
pvestatd
```

Storage remained available and the host retained network connectivity following the reboot.

Chrony also returned to a healthy synchronised state.

---

## Proxmox Update Observation

Historical service information showed an earlier non-zero status associated with a `pveupdate` operation executed through `pveproxy`.

The core `pveproxy` service itself remained healthy and its current journal activity appeared normal.

`pveupdate` was subsequently executed manually and completed successfully.

The observation was therefore documented but was not considered an ongoing platform fault.

---

## Boot Configuration Observation

`proxmox-boot-tool status` reported that the host did not contain:

```text
/etc/kernel/proxmox-boot-uuids
```

This indicated that HYP01 was not using that Proxmox-managed EFI System Partition configuration.

No change was required as part of this task.

Windows remained the default UEFI boot target in accordance with the agreed dual-boot operating model.

---

## Bootstrap Dependencies

The following temporary dependencies remain:

- Management connectivity through `192.168.178.0/24`
- Home router `192.168.178.1` providing gateway services
- Home router providing DNS forwarding
- External NTP sources
- Proxmox No-Subscription repository for the lab environment
- Manual selection of Proxmox when starting a Northwind lab session

These are documented bootstrap conditions rather than final Northwind design decisions.

---

## Validation

The following checks were successfully completed:

- [x] HYP01 hostname validated.
- [x] HYP01 FQDN validated.
- [x] Local hostname resolution validated.
- [x] Operating system version recorded.
- [x] Timezone configured and validated.
- [x] RTC configuration validated.
- [x] Chrony active.
- [x] NTP synchronisation validated.
- [x] Bootstrap IP configuration validated.
- [x] Default route validated.
- [x] DNS configuration validated.
- [x] Local gateway connectivity validated.
- [x] External connectivity validated.
- [x] Debian repositories validated.
- [x] Proxmox Enterprise repository disabled.
- [x] Ceph Enterprise repository disabled.
- [x] Proxmox No-Subscription repository enabled.
- [x] Package metadata refreshed.
- [x] Proposed upgrade reviewed before implementation.
- [x] HYP01 successfully updated.
- [x] HYP01 successfully rebooted.
- [x] Updated Proxmox version recorded.
- [x] Running kernel recorded.
- [x] No failed systemd units detected.
- [x] Core Proxmox services active.
- [x] Storage remained available.
- [x] Network connectivity survived the reboot.

---

## Findings / Lessons Learned

### Establish a baseline before introducing workloads

Recording identity, networking, repositories, time configuration, package state and software versions provides a known starting point for future troubleshooting.

---

### Review updates before applying them

A package removal during an upgrade does not automatically indicate a problem.

Simulating the upgrade first allowed the `libzfs6linux` to `libzfs7linux` transition to be investigated before changes were made.

---

### `apt update` does not install updates

Refreshing repository metadata and installing package upgrades are separate operations.

This allows administrators to inspect available changes before modifying a system.

---

### Time synchronisation is infrastructure

Correct time is required for reliable logs, authentication, monitoring and troubleshooting.

NTP validation should therefore form part of a server baseline rather than being treated as an optional configuration detail.

---

### Reboot validation is part of the change

A successful package installation does not by itself prove that an update was successful.

HYP01 was rebooted and its networking, storage, time synchronisation and Proxmox services were revalidated.

---

### Temporary dependencies should be visible

The current gateway, DNS, NTP and management network allow HYP01 to operate during initial implementation.

Documenting these dependencies prevents temporary bootstrap configuration from silently becoming permanent architecture.

---

## Result

**PASS**

HYP01 successfully completed its initial platform baseline configuration.

System identity, networking, time synchronisation and package repositories were validated. The host was successfully updated from the initial Proxmox VE 9.1 installation to Proxmox VE 9.2 and passed post-update health validation.

HYP01 was ready to proceed to functional virtualisation validation.

---

## Next Task

**NIM-14 — Validate HYP01 Virtualisation Capability**

NIM-14 validates that the completed HYP01 platform can provision, execute, network, monitor and remove a real Linux virtual workload.

---

## Related Documentation

- NIM-9 — Install Proxmox VE on HYP01
- NIM-10 — Configure HYP01 Local Storage Architecture
- Northwind Low-Level Design
- Sprint 1 implementation records
