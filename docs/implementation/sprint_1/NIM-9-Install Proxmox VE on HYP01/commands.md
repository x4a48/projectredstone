# NIM-9 — Command Reference

Commands encountered while installing Proxmox VE on HYP01 and validating initial host connectivity.

---

## `lsblk`

```bash
lsblk
```

**Purpose:**  
Lists block devices detected by Linux, including disks and partitions.

**Why we used it:**  
To inspect HYP01's storage devices during installation and help confirm that the correct NVMe device was selected as the Proxmox installation target.

---

## `lsblk -d -o NAME,SIZE,MODEL,SERIAL`

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL
```

**Purpose:**  
Displays physical block devices with selected hardware identification information.

- `-d` — show devices without their dependent partitions.
- `-o` — specify the output columns.
- `NAME` — Linux device name.
- `SIZE` — device capacity.
- `MODEL` — hardware model.
- `SERIAL` — hardware serial number.

**Why we used it:**  
To correlate Linux NVMe device names with the persistent hardware serial information collected during NIM-7 and protect the Windows disk from accidental overwrite.

---

## `ip addr`

```bash
ip addr
```

**Purpose:**  
Displays network interfaces and their configured IP addresses.

**Why we used it:**  
To inspect HYP01's network interfaces when troubleshooting why the Proxmox management interface was initially unreachable.

The interface state helped identify that the physical Ethernet connection did not have an active link.

---

## `ip route`

```bash
ip route
```

**Purpose:**  
Displays the Linux routing table, including the default gateway.

**Why we used it:**  
To inspect HYP01's network routing and confirm the configured path toward the bootstrap network gateway.

---

## `nomodeset`

```text
nomodeset
```

**Purpose:**  
A Linux kernel boot parameter that prevents normal kernel graphics mode-setting during early boot.

**Why we used it:**  
The parameter had been identified during NIM-8 as necessary for successfully reaching the Proxmox VE 9.1 graphical installer on HYP01.

---

## Installer Virtual Terminals

```text
Ctrl + Alt + F3
```

**Purpose:**  
Switches from the graphical Proxmox installer to a Linux console.

**Why we used it:**  
To inspect the available storage devices and validate the intended installation target before performing the destructive installation.

```text
Ctrl + Alt + F1
```

**Purpose:**  
Returns to the graphical Proxmox installer.

**Why we used it:**  
To resume the installation after completing CLI storage validation.
