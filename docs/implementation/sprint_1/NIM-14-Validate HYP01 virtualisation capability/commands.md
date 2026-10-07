# NIM-14 — Command Reference

Commands encountered while validating HYP01's ability to provision and operate a RHEL 10.2 virtual workload.

---

## `sha256sum`

```bash
sha256sum <ISO-file>
```

**Purpose:**  
Calculates the SHA-256 checksum of a file.

**Why we used it:**  
To verify the downloaded RHEL installation media against the checksum published by Red Hat before using it on HYP01.

---

## `lvs`

```bash
lvs
```

**Purpose:**  
Displays LVM Logical Volumes and thin-pool utilisation.

**Why we used it:**  
To observe the creation of `vm-100-disk-0`, monitor its physical thin-storage consumption after RHEL installation, and verify its removal during cleanup.

---

## `lscpu`

```bash
lscpu
```

**Purpose:**  
Displays CPU architecture and topology information visible to Linux.

**Why we used it:**  
To confirm that TEST01 detected the expected 2 vCPUs and to observe the CPU presented to RHEL when Proxmox CPU type `host` was used.

---

## `free -h`

```bash
free -h
```

**Purpose:**  
Displays system memory and swap utilisation in human-readable units.

**Why we used it:**  
To verify the memory available to the RHEL guest and confirm that swap had been configured.

---

## `lsblk`

```bash
lsblk
```

**Purpose:**  
Displays block devices, partitions and their relationships.

**Why we used it:**  
To inspect the RHEL guest's 20 GiB virtual disk and verify the resulting partition and guest LVM layout.

---

## `ip addr`

```bash
ip addr
```

**Purpose:**  
Displays network interfaces and their configured IP addresses.

**Why we used it:**  
To confirm that RHEL detected the VirtIO network interface and received an IP address on the bootstrap network.

---

## `ip route`

```bash
ip route
```

**Purpose:**  
Displays the Linux routing table.

**Why we used it:**  
To confirm TEST01's directly connected network and default route through the bootstrap gateway.

---

## `ping`

```bash
ping -c 4 <destination>
```

**Purpose:**  
Sends ICMP echo requests to test basic IP connectivity.

- `-c 4` — send four requests and then stop.

**Why we used it:**  
To test connectivity progressively to the local gateway, an external IP address and external hostnames.

Examples included:

```bash
ping -c 4 192.168.178.1
ping -c 4 1.1.1.1
ping -c 4 google.com
```

---

## `systemctl poweroff`

```bash
sudo systemctl poweroff
```

**Purpose:**  
Performs an orderly shutdown of the Linux operating system.

**Why we used it:**  
To validate guest-initiated shutdown and confirm that Proxmox correctly reported TEST01 as stopped.

---

## `systemctl reboot`

```bash
sudo systemctl reboot
```

**Purpose:**  
Performs an orderly restart of the Linux operating system.

**Why we used it:**  
To validate TEST01's guest-initiated reboot lifecycle operation.

---

## `pvesm status`

```bash
pvesm status
```

**Purpose:**  
Displays the status and utilisation of storage configured in Proxmox.

**Why we used it:**  
To verify that `northwind-workload` remained active and observe its utilisation while TEST01 existed and after the VM was removed.

---

## `systemctl --failed`

```bash
systemctl --failed
```

**Purpose:**  
Lists systemd units currently in a failed state.

**Why we used it:**  
To confirm that operating TEST01 had not introduced any failed services on HYP01.

---

## `systemctl status`

```bash
systemctl status pveproxy pvedaemon pvestatd
```

**Purpose:**  
Displays the current status of specified systemd services.

**Why we used it:**  
To confirm that HYP01's core Proxmox management services remained active while hosting the validation workload.
