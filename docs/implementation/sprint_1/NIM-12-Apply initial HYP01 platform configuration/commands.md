# NIM-12 — Command Reference

Commands encountered while configuring and validating the HYP01 platform baseline.

---

## `hostnamectl`

```bash
hostnamectl
```

**Purpose:**  
Displays the system hostname and related operating system information.

**Why we used it:**  
To validate HYP01's system identity and confirm the configured hostname.

---

## `cat /etc/hostname`

```bash
cat /etc/hostname
```

**Purpose:**  
Displays the hostname stored in `/etc/hostname`.

**Why we used it:**  
To confirm the persistent hostname configuration for HYP01.

---

## `cat /etc/hosts`

```bash
cat /etc/hosts
```

**Purpose:**  
Displays the host's local static hostname-to-IP mappings.

**Why we used it:**  
To confirm that `hyp01.northwind.internal` and `hyp01` mapped to the bootstrap management address.

---

## `getent hosts`

```bash
getent hosts hyp01.northwind.internal
```

**Purpose:**  
Queries the system's configured name-resolution databases.

**Why we used it:**  
To confirm that the HYP01 FQDN resolved correctly using the host's normal name-resolution mechanism.

---

## `timedatectl`

```bash
timedatectl
```

**Purpose:**  
Displays system time, timezone, RTC and time-synchronisation information.

**Why we used it:**  
To validate the `Europe/Berlin` timezone and UTC hardware clock configuration.

---

## `systemctl status chrony`

```bash
systemctl status chrony
```

**Purpose:**  
Displays the current state of the Chrony systemd service.

**Why we used it:**  
To confirm that the service responsible for HYP01 network time synchronisation was enabled and running.

---

## `chronyc sources -v`

```bash
chronyc sources -v
```

**Purpose:**  
Displays the NTP sources known to Chrony and their synchronisation status.

**Why we used it:**  
To verify that HYP01 could reach external time sources and had selected a valid source for synchronisation.

---

## `ip addr`

```bash
ip addr
```

**Purpose:**  
Displays network interfaces and configured IP addresses.

**Why we used it:**  
To validate the HYP01 bootstrap management address and confirm that it was assigned to `vmbr0`.

---

## `ip route`

```bash
ip route
```

**Purpose:**  
Displays the system routing table.

**Why we used it:**  
To confirm that HYP01 had the expected default route through `192.168.178.1`.

---

## `cat /etc/resolv.conf`

```bash
cat /etc/resolv.conf
```

**Purpose:**  
Displays the DNS resolver configuration currently used by the host.

**Why we used it:**  
To confirm the bootstrap DNS server and Northwind search domain configuration.

---

## `apt update`

```bash
apt update
```

**Purpose:**  
Downloads the latest package metadata from configured APT repositories.

**Why we used it:**  
To verify repository accessibility and refresh HYP01's knowledge of available package versions before reviewing updates.

---

## `apt list --upgradeable`

```bash
apt list --upgradeable
```

**Purpose:**  
Lists installed packages for which newer versions are available.

**Why we used it:**  
To review the packages that could be updated before changing the platform.

---

## `apt full-upgrade --simulate`

```bash
apt full-upgrade --simulate
```

**Purpose:**  
Simulates a full package upgrade without actually changing the system.

**Why we used it:**  
To review the packages that would be upgraded, installed or removed before approving the HYP01 platform update.

---

## `apt full-upgrade`

```bash
apt full-upgrade
```

**Purpose:**  
Upgrades installed packages and permits dependency changes, including installing or removing packages where required.

**Why we used it:**  
To apply the reviewed HYP01 platform updates and move the host to the current Proxmox VE 9.2 software baseline.

---

## `uname -r`

```bash
uname -r
```

**Purpose:**  
Displays the currently running Linux kernel release.

**Why we used it:**  
To record and verify the kernel running after the HYP01 platform update and reboot.

---

## `systemctl --failed`

```bash
systemctl --failed
```

**Purpose:**  
Lists systemd units currently in a failed state.

**Why we used it:**  
To perform a quick post-update health check and confirm that HYP01 had no failed systemd units.

---

## `systemctl status`

```bash
systemctl status pveproxy pvedaemon pvestatd
```

**Purpose:**  
Displays the current state of specified systemd services.

**Why we used it:**  
To confirm that the core Proxmox management services were active following the update and reboot.

---

## `journalctl -u`

```bash
journalctl -u pveproxy
```

**Purpose:**  
Displays systemd journal entries associated with a specific service.

**Why we used it:**  
To investigate the historical non-zero `pveupdate` status associated with `pveproxy` and distinguish it from an ongoing service failure.

---

## `pveupdate`

```bash
pveupdate
```

**Purpose:**  
Runs the Proxmox update-information process manually.

**Why we used it:**  
To reproduce and validate the operation associated with the earlier `pveproxy` observation. The manual execution completed successfully.

---

## `proxmox-boot-tool status`

```bash
proxmox-boot-tool status
```

**Purpose:**  
Displays the status of Proxmox-managed bootloader/EFI System Partition configuration.

**Why we used it:**  
To inspect HYP01's boot configuration while reviewing the platform following the update.

---

## `pveversion`

```bash
pveversion
```

**Purpose:**  
Displays the installed Proxmox VE version.

**Why we used it:**  
To confirm and record the Proxmox version after the platform update.

---

## `pvesm status`

```bash
pvesm status
```

**Purpose:**  
Displays the status and utilisation of storage configured in Proxmox.

**Why we used it:**  
To confirm that HYP01's storage backends remained active following the platform update and reboot.

---

## `reboot`

```bash
reboot
```

**Purpose:**  
Restarts the system.

**Why we used it:**  
To load the updated platform/kernel and verify that HYP01 could return to a healthy operational state after the upgrade.
