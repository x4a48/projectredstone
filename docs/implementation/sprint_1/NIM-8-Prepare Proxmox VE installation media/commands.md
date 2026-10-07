# NIM-8 — Command Reference

Commands and command-line options encountered while preparing and validating the Proxmox VE installation media for HYP01.

---

## `sha256sum`

```bash
sha256sum <ISO-file>
```

**Purpose:**  
Calculates the SHA-256 checksum of a file.

**Why we used it:**  
To compare the downloaded Proxmox VE ISO against the vendor-published checksum and confirm that the installation media had downloaded correctly and had not been corrupted.

---

## `lsblk`

```bash
lsblk
```

**Purpose:**  
Lists block devices detected by Linux, including disks and partitions.

**Why we used it:**  
To inspect the storage devices detected by the Proxmox installer and help identify the three NVMe SSDs installed in HYP01.

---

## `lsblk -d -o NAME,SIZE,MODEL,SERIAL`

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL
```

**Purpose:**  
Displays physical block devices with selected identification information.

- `-d` — show devices without their dependent partitions.
- `-o` — specify which output columns to display.
- `NAME` — Linux device name.
- `SIZE` — device capacity.
- `MODEL` — hardware model.
- `SERIAL` — hardware serial number.

**Why we used it:**  
To correlate the NVMe devices detected by Linux with the persistent hardware information recorded during NIM-7 and avoid selecting the Windows disk for installation.

---

## `nomodeset`

```text
nomodeset
```

**Purpose:**  
A Linux kernel boot parameter that prevents normal kernel graphics mode-setting during early boot.

**Why we used it:**  
As a troubleshooting measure when the Proxmox installer failed to reach the graphical installation environment. It allowed the installer to progress further and was ultimately used successfully with the Proxmox VE 9.1 installer.

> `nomodeset` changed the installer behaviour, but this does not establish graphics initialisation as the definitive root cause of the Proxmox VE 9.2 issue.

---

## Installer Virtual Terminals

```text
Ctrl + Alt + F3
```

**Purpose:**  
Switches from the graphical installer to a Linux console.

**Why we used it:**  
To access the installer shell and inspect the detected storage devices before selecting an installation target.

```text
Ctrl + Alt + F1
```

**Purpose:**  
Returns to the graphical installer.

**Why we used it:**  
To continue the Proxmox installation workflow after performing CLI validation.
