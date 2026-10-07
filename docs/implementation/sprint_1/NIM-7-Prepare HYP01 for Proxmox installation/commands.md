# NIM-7 — Command Reference

Commands encountered while validating HYP01 hardware and storage before the Proxmox VE installation.

---

## `Get-PhysicalDisk`

```powershell
Get-PhysicalDisk | Select DeviceId, FriendlyName, SerialNumber, AdapterSerialNumber
```

**Purpose:**  
Displays physical storage devices detected by Windows and selected hardware identification information.

**Why we used it:**  
To inventory HYP01's three NVMe SSDs and record persistent serial information before performing any destructive installation work.

---

## `Get-Disk`

```powershell
Get-Disk | Select Number, FriendlyName, SerialNumber, BusType, Size
```

**Purpose:**  
Displays disks recognised by Windows together with their Windows disk number and selected hardware information.

**Why we used it:**  
To correlate the Windows disk layout with the physical NVMe devices and identify which disk contained the existing Windows installation.

---

## `Select`

```powershell
... | Select DeviceId, FriendlyName, SerialNumber
```

**Purpose:**  
Selects specific properties from objects passed through the PowerShell pipeline.

**Why we used it:**  
To reduce the output of the storage commands to the identification information relevant to the HYP01 hardware inventory.

---

## PowerShell Pipeline `|`

```powershell
command | command
```

**Purpose:**  
Passes the objects produced by one PowerShell command to another command for further processing.

**Why we used it:**  
To pass the disk information returned by `Get-PhysicalDisk` and `Get-Disk` to `Select`, allowing us to display only the required properties.
