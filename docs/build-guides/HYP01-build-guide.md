# HYP01 Build / Rebuild Guide

**Northwind Outdoor Ltd. | Sprint 1 platform foundation**  
**Version:** 0.2 — Draft for review | **Prepared:** 8 October 2026  
**Related work:** NIM-15 — Document HYP01 build and configuration

## 1. Confirm scope and build inputs

Recreate the Sprint 1 HYP01 virtualisation foundation while preserving the existing Windows NVMe installation. This guide describes the execution sequence; the concise System Record should hold the authoritative values and resulting configuration.

**Evidence boundary:** Available project conversation confirms that Proxmox / Windows dual boot, the HYP01 core platform, NVMe2 workload storage and Proxmox storage configuration were implemented and validated in Sprint 1. Guest lifecycle validation was also reported complete. The implementation conversation returned reference markers rather than its underlying text, and the concise System Record was unavailable in this workspace. Detailed values below therefore remain **[VERIFY]**. Suggested safety and validation steps are procedural recommendations, not claims about steps performed in Sprint 1.

1. Obtain the approved concise HYP01 System Record and Sprint 1 implementation evidence. Record their paths and revisions: **[VERIFY: references]**.
2. Select the activity: **[initial build / host rebuild / host and workload-storage rebuild]**. State exactly which disks and data may be overwritten.
3. Resolve the inputs below against the records and current hardware. Do not infer device identity from an NVMe number alone.

| Required input | Established value / verification needed |
| --- | --- |
| Host | HYP01; exact configured hostname/FQDN **[VERIFY]** |
| Platform | Proxmox VE; release, installer filename and checksum **[VERIFY]** |
| Hardware and firmware | Model, CPU, memory, virtualisation settings and boot mode **[VERIFY]** |
| Protected Windows disk | NVMe; model, capacity, serial and current device path **[VERIFY]** |
| Proxmox installation disk | Physical identity, device path and approved partition/filesystem layout **[VERIFY]** |
| Workload disk | NVMe2 project designation; physical identity and device path **[VERIFY]** |
| Storage configuration | IDs, backing devices, storage types, paths and permitted content **[VERIFY]** |
| Bootstrap management | NIC/bridge, address/prefix, gateway, DNS and access URL **[VERIFY]** |
| Platform configuration | Repositories, update baseline, time settings and administrator access **[VERIFY]** |
| Validation guest | TEST01 was used for baseline learning; confirm Sprint 1 test guest, ID, image and specifications **[VERIFY]** |

**Checkpoint:** Required inputs are reconciled and the intended erase scope is explicit. This draft is not ready for destructive execution until the relevant placeholders are resolved.

## 2. Protect Windows and existing workloads

1. For a rebuild, record the current host, network, storage and guest configuration before changing it. Identify affected workloads and agree the outage window.
2. Confirm recoverable copies of Windows data and any host/guest data within the erase scope. Record backup location, date and recovery verification: **[VERIFY]**. A copy on a disk being erased does not satisfy this gate.
3. Record the Windows boot entry and current firmware settings. If Windows disk encryption is enabled, confirm recovery-key access before firmware or disk changes; keep secrets outside this guide.
4. Shut down affected guests and the host cleanly. Use the established Windows-disk protection method: **[VERIFY: Sprint 1 method]**. Prefer physical disconnection where supported; if it remains connected, require independent confirmation of every target disk before any write.
5. Label protected and permitted-write disks using their physical identities. Preserve Windows partitions, its EFI boot files and its boot entry.

**Safety gate — STOP:** Do not install, initialise, partition or format anything if the Windows disk or an erase target cannot be positively identified. Disk numbering may change after disconnection or reboot.

## 3. Prepare firmware and installation media

1. Power on HYP01 and press **[VERIFY: firmware setup key]** to open firmware setup.
2. Open **[VERIFY: firmware menu path]**. Select **[VERIFY: virtualisation setting]** and set it to the recorded value **[VERIFY]**. Set **[VERIFY: boot-mode setting]** to **[VERIFY: recorded value]**. Save using **[VERIFY: save/exit control]**.
3. Download the recorded Proxmox VE installer **[VERIFY: filename and release]**. Compare its checksum with **[VERIFY: approved checksum and source]** using **[VERIFY: established verification method]**. Stop if the values differ.
4. Connect the installation USB device and open **[VERIFY: media-writing application]**.
5. In **[VERIFY: device selector]**, select the USB device **[VERIFY: identity and capacity]**. In **[VERIFY: image selector]**, choose the verified installer file. Set **[VERIFY: recorded writing options]**.
6. Check that the destination is the USB device, then click **[VERIFY: write/start button]**. Wait for **[VERIFY: successful completion indication]**. This action overwrites the selected USB device.
7. Connect the completed USB to HYP01. Restart and press **[VERIFY: boot-menu key]**. Select **[VERIFY: USB boot entry matching the recorded boot mode]**.

**Checkpoint:** Installer starts in the intended boot mode; permitted-write disks are identifiable and Windows protection remains effective.

## 4. Install Proxmox VE

**Screen-label reference:** The graphical installer sequence and the **Options** control are described in the [official Proxmox installation documentation](https://pve.proxmox.com/pve-docs/chapter-pve-installation.html). Confirm labels against the recorded HYP01 installer release before execution. These generic controls do not establish the values selected in Sprint 1.

1. At the installer boot menu, select **Install Proxmox VE (Graphical)**. Wait for the graphical installer to open.
2. Read the licence agreement and use the displayed acceptance control to continue.
3. On the installation-disk screen, select **[VERIFY: approved Proxmox disk]** from the target-disk selector. Match its displayed identity with the model, capacity and serial recorded in Section 1. If the screen does not expose enough information, stop and identify the disk before continuing.
4. Click **Options**. Select the recorded filesystem **[VERIFY]** and enter the recorded disk-layout values **[VERIFY: each option name and value]**. Do not accept defaults merely because they are preselected. Confirm the options using **[VERIFY: confirmation button]**.
5. Continue to the location/time screen. Select **[VERIFY: country]**, **[VERIFY: time zone]** and **[VERIFY: keyboard layout]** in the corresponding controls.
6. Continue to the administrator-credentials screen. Enter and confirm the administrator password using the approved credential source. Enter **[VERIFY: administrator email]** in the email field. Do not save the password in this document.
7. Continue to the network screen. Select **[VERIFY: management NIC]**. Enter **[VERIFY: FQDN]**, **[VERIFY: IP address/prefix]**, **[VERIFY: gateway]** and **[VERIFY: DNS server]** in their corresponding fields. Compare every value with Section 1 before continuing.
8. On the final summary screen, check the selected disk, filesystem, hostname and network values. Confirm that the Windows disk and NVMe2 workload disk are excluded from the installation targets. **Do not activate the installation control until this check passes.**
9. Click **[VERIFY: final install button label]**. Wait for successful completion. Remove the installation USB when prompted or before the subsequent boot, then use **[VERIFY: reboot control]** to restart HYP01.
10. At the local console, check that the displayed hostname and management address match the record. On the administration workstation, open a browser and enter **[VERIFY: management URL]**. Sign in using **[VERIFY: recorded account and authentication realm]**. Check that HYP01 appears in the management interface.

**Safety gate:** If the installer selects Windows, workload storage or an unidentified device, cancel before committing changes. If installation fails, preserve the protection controls and investigate before retrying.

## 5. Restore the initial platform configuration

1. In the management interface, select HYP01. Open **[VERIFY: repository page/menu path]**. Select each recorded repository entry and use **[VERIFY: enable/disable/edit control]** to reproduce **[VERIFY: Sprint 1 repository configuration]**.
2. Open **[VERIFY: update page/menu path]**. Click **[VERIFY: refresh control]** and review the result. Apply the approved update baseline through **[VERIFY: established update control or recorded command]**. Wait for the task to complete successfully.
3. If a reboot is required, shut down affected guests, then click **[VERIFY: host reboot control]** and confirm the action. After HYP01 restarts, reopen its management URL, sign in and record the displayed platform version **[VERIFY: version location]**.
4. Open **[VERIFY: time-settings page]** and set **[VERIFY: each setting and value]**. Open **[VERIFY: name-resolution page]** and enter **[VERIFY: recorded settings]**. Save each change with **[VERIFY: save/apply control]**.
5. Open **[VERIFY: administrator-access page]** and reproduce **[VERIFY: established access settings]**. Test the intended administrator login while retaining local console access.
6. Open **[VERIFY: bootstrap-network page]**. Compare the NIC/bridge, address, gateway and DNS settings with Section 1. Record the outcome; do not apply a proposed future network design.

**Checkpoint:** HYP01 remains accessible after reboot; recorded platform settings, time and bootstrap connectivity match the intended baseline.

**Scope boundary:** NIM-11 management networking and NIM-13 virtual networking foundation remain dependent on LLD-06. Network/security zones and inter-zone policy were not implemented in Sprint 1. JUMP01, CORE01, OPS01, BACKUP01 and FILE01 are subsequent work. Do not mark them complete through this guide.

## 6. Configure or reconnect workload storage

1. Select HYP01 and open **[VERIFY: disk inventory page/menu path]**. Select the disk designated NVMe2 and compare its serial and capacity with Section 1. Inspect the displayed partitions/storage layout. Do not click an initialise, wipe or format control.
2. **For a host-only rebuild:** open **[VERIFY: existing-storage recovery/import page or procedure]**. Select **[VERIFY: existing storage identifier]**, enter **[VERIFY: required recovery/import fields]**, and activate **[VERIFY: reconnect/import control]**. Confirm that existing data is retained. The storage backend and recovery procedure must be verified before this step can be completed.
3. **For an initial build or approved workload-storage rebuild:** confirm the erase gate below first. Open **[VERIFY: storage creation page]**, select the verified NVMe2 disk and enter **[VERIFY: layout fields and values]**. Click **[VERIFY: create/initialise control]** only after checking its target. Wait for successful completion.
4. Open **[VERIFY: Proxmox storage configuration page]**. Select or add **[VERIFY: recorded storage type]** through **[VERIFY: control]**. Enter **[VERIFY: storage ID, backing device/path and permitted content]**, then click **[VERIFY: save/add control]**.
5. Select the configured storage and open **[VERIFY: capacity/status view]**. Check that it is available, shows the expected capacity and uses the recorded backing device/path. For a host-only rebuild, confirm the expected existing workload content is present.

**Safety gate:** Formatting or initialising NVMe2 requires explicit inclusion in the erase scope and verified recovery arrangements. Never apply a storage command using an assumed device path.

**Checkpoint:** Workload storage matches the record and is available. Windows storage is not configured as a workload or installation target.

## 7. Validate virtualisation and dual boot

1. Select HYP01 and click **[VERIFY: create-guest control]**. Follow the guest wizard, entering **[VERIFY: recorded guest ID/name, installation image, CPU, memory, disk storage/size and network settings]** in the corresponding fields. Review the summary and click **[VERIFY: finish control]**. Use a disposable validation guest where appropriate.
2. Select the validation guest. Click **[VERIFY: start control]**, then open **[VERIFY: console control]**. Confirm that the guest reaches its expected boot screen or login prompt.
3. Open **[VERIFY: guest hardware/storage view]**. Check that its virtual disk uses the intended workload storage. Record the observed storage ID.
4. Return to the guest console and shut down the guest using **[VERIFY: established guest shutdown method]**. Check that its status becomes stopped. Start it again through **[VERIFY: start control]** and confirm another successful boot. Record any host or storage errors.
5. Shut down all guests cleanly, then select HYP01 and click **[VERIFY: host shutdown control]**. Confirm it has powered off before reconnecting Windows NVMe, if previously disconnected. Restore **[VERIFY: recorded boot arrangement]**.
6. Power on HYP01, press **[VERIFY: boot-menu key]** and select **[VERIFY: Windows boot entry]**. Sign in to Windows and open **[VERIFY: agreed files/applications]** to check the expected data. Record any recovery prompt or boot error.
7. Shut down Windows cleanly. Power on HYP01 and use the boot menu to select **[VERIFY: Proxmox boot entry]**. Open the management URL and sign in. Select workload storage and check availability, then start the validation guest and check its console again.
8. If the test guest is disposable, select it and confirm its ID/name before using **[VERIFY: remove control and confirmation sequence]**. Retain TEST01 if it remains an active project asset.

**Checkpoint:** Both operating systems boot independently as intended, and HYP01 hosts a working guest on the intended storage. A failed Windows boot is a failed preservation check; investigate before declaring success.

## 8. Record results and hand over for review

1. Update the concise System Record with actual versions, disk identities, storage definitions, bootstrap networking and validation evidence. Document deviations from the approved baseline.
2. Replace verified placeholders in this guide and add exact commands only where supported by the implementation record. No executable command has been included in this draft because none could be verified from the available HYP01 evidence.
3. Record execution date, administrator, change reference and evidence location: **[VERIFY]**. Record each checkpoint as **Pass / Fail / Not tested**.
4. Return HYP01 to the agreed operating state: **[VERIFY]**. If a checkpoint fails, halt dependent work and use the verified recovery procedure; do not attempt speculative bootloader or disk repairs.
5. Submit the guide and updated System Record for review under NIM-15. Keep technical validation separate from formal HYP01 acceptance; confirm the applicable NIM-10 acceptance criteria before closure.

**Review disposition:** **[Pending / Approved / Changes required]**  
**Reviewer and date:** **[VERIFY]**

Maintain this guide alongside the System Record when future sprints change HYP01. Keep the procedure concise and link to detailed evidence rather than duplicating it.
