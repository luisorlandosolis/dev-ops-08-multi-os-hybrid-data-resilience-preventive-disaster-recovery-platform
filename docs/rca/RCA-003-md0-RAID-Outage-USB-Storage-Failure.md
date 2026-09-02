# RCA-003 - md0 RAID Outage, USB Storage Failure, and Replication Recovery

## Overview

**Date:** August 2026

**Host:** pve (Proxmox)

**Affected Services:**

- RAID1-STORAGE (/dev/md0)
- Samba Shares
- Backup Replication Tier
- Archive Replication Workflow

**Status:** Resolved

---

## Executive Summary

An investigation into a storage outage revealed a combination of RAID degradation, USB-attached storage instability, and a separate backup replication failure.

The RAID1 array remained operational but lost redundancy after one mirror member disappeared from the USB bus. The filesystem remained accessible but operated in a degraded state.

Additional investigation discovered that backup replication from the ExternalBackup repository to RAID1-STORAGE had silently failed because the repository was not mounted when scheduled synchronization jobs executed.

No backup data was lost.

Repository data was successfully recovered and backup operations were restored.

---

## Backup Architecture

```text
A. Primary Backup Tier
--------------------------------
ExternalBackup
/mnt/externalbackup

ProxmoxBackups
WindowsImageBackup

        |
        | Monday 02:00
        v

B/C. Replication Tier
--------------------------------
RAID1-STORAGE
/dev/md0
/srv/storage

ProxmoxBackups
WindowsImageBackup
Media
ProxmoxConfigBackup

        |
        | Monday 04:00
        v

D. Archive Tier
--------------------------------
   Archive Tier

F:\Archive

        |
        | Tuesday
        v

E. Cloud Tier
--------------------------------
iDrive Cloud
```

This incident affected the replication tier while simultaneously exposing a separate backup synchronization failure.

---

## Hardware Context

### Storage Inventory

#### Primary Repository Tier

```text
ExternalBackup
Mount Point: /mnt/externalbackup
Filesystem: NTFS

Purpose:
- ProxmoxBackups
- WindowsImageBackup
```

#### Replication Tier

```text
RAID1-STORAGE
Device: /dev/md0
Filesystem: ext4
Mount Point: /srv/storage
```

#### Archive Tier

```text
Archive Tier

Archive Storage
F:\Archive
```

### ArmorATD Storage Design

All external storage devices involved in the backup platform are G-Technology ArmorATD USB drives.

```text
Vendor ID : 4791
Product ID: 205a
```

Characteristics:

- Bus-powered
- No dedicated power brick
- USB-attached storage
- Shared USB resources
- Susceptible to USB autosuspend behavior
- Susceptible to power instability under sustained load

---

## Root Cause Analysis

Two distinct failure modes were identified.

### Failure Mode #1 - USB Autosuspend

USB-attached ArmorATD drives may enter a low-power state during extended idle periods.

```text
Idle Device
     ↓
USB Autosuspend
     ↓
Wake-Up Failure
     ↓
Device Disappears
     ↓
RAID Degraded
```

Observed symptoms:

- Device removed from USB subsystem
- Device disappears from lsblk
- RAID member removed from md0
- Loss of RAID redundancy

### Failure Mode #2 - Power Starvation Under Load

All ArmorATD devices are bus-powered.

Under sustained workload:

```text
RAID Resync
rsync Replication
Samba Activity
Backup Operations
```

current demand can temporarily exceed what the USB subsystem reliably supplies.

```text
Heavy I/O
     ↓
Power Demand Spike
     ↓
USB Reset / Brownout
     ↓
Drive Disconnect
     ↓
RAID Degradation
```

This was determined to be the most likely explanation for USB device drops that occurred during active disk activity.

Both failure modes produced the same result:

```text
Drive disappears
       ↓
md0 degrades
       ↓
Loss of redundancy
```

---

## Incident Timeline

### August 15, 2026

Kernel logs recorded:

```text
md/raid1:md0: Disk failure on sdc, disabling device.
```

RAID1 degraded from:

```text
[UU]
```

to:

```text
[U_]
```

The array remained online and continued servicing reads and writes using the surviving member.

### August 15 Through Investigation

The array operated in a degraded state without alerting.

Characteristics:

```text
Filesystem Available  : Yes
Data Accessible       : Yes
Redundancy Available  : No
```

No automated monitoring was in place to alert on degraded RAID health.

---

## Additional Discovery - Replication Failure

Comparison of backup locations revealed a major discrepancy.

```text
ExternalBackup Repository
≈118 GB

RAID1-STORAGE Copy
≈22 GB
```

Investigation of:

```text
/var/log/a-to-raid-sync.log
```

revealed repeated failures:

```text
rsync:
change_dir
"/mnt/externalbackup/ProxmoxBackups"
failed:
No such file or directory

rsync:
change_dir
"/mnt/externalbackup/WindowsImageBackup"
failed:
No such file or directory
```

### Replication Root Cause

The synchronization script:

```bash
/usr/local/bin/a-to-raid-sync.sh
```

was executing successfully from cron but the repository drive was not mounted at runtime.

Script:

```bash
#!/bin/bash

rsync -avh --delete \
/mnt/externalbackup/ProxmoxBackups/ \
/srv/storage/ProxmoxBackups/

rsync -avh --delete \
/mnt/externalbackup/WindowsImageBackup/ \
/srv/storage/WindowsImageBackup/
```

Result:

```text
Cron Running       ✅
Script Running     ✅
Source Mounted     ❌
Replication        ❌
```

This explains why backup data continued accumulating in the repository while replication remained stale.

---

## Repair Actions Performed

### Repository Recovery

Verified active mount:

```bash
mount | grep externalbackup
```

Result:

```text
/dev/sda2 on /mnt/externalbackup type ntfs3
```

Repository successfully recovered.

### Backup Validation

Executed:

```bash
vzdump 104 --storage externalbackup --mode snapshot
```

Result:

```text
Backup completed successfully.
```

New backup files appeared in:

```text
/mnt/externalbackup/ProxmoxBackups/dump
```

### Backup Storage Recovery

Created and validated direct Proxmox storage:

```text
ID: externalbackup
Path: /mnt/externalbackup/ProxmoxBackups
Type: Directory
```

This removed dependency on the failed SMB backup target.

---

## RAID Status After Recovery

```text
md0 : active raid1 sdb[0]
      [2/1] [U_]
```

Meaning:

```text
Data Available      ✅
Filesystem Mounted  ✅
Backups Functional  ✅
Redundancy Missing  ❌
```

The RAID mirror remains degraded pending restoration of the missing mirror member.

---

## Remediation Deployed

### USB Autosuspend Disabled

File:

```text
/etc/udev/rules.d/99-armoratd-no-autosuspend.rules
```

Contents:

```bash
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="4791", ATTR{idProduct}=="205a", ATTR{power/control}="on"
```

Purpose:

```text
Prevent USB autosuspend
Reduce wake-up failures
Keep ArmorATD devices active
```
### Ansible Drive Keepalive Mitigation

File:

automation/ansible/playbooks/keep-drives-active.yml

A sanitized portfolio version of the remediation playbook has been included in this repository to demonstrate the automation approach while omitting environment-specific identifiers.

```

Purpose:

- Periodically read from RAID members
- Detect missing devices
- Reduce prolonged idle periods
- Generate audit logging

```yaml
- name: Deploy USB drive keepalive to prevent idle/autosuspend drops
  hosts: all
  become: true

  vars:
    keepalive_script_path: /usr/local/bin/drive-keepalive.sh
    keepalive_log_path: /var/log/drive-keepalive.log
    drives_to_ping:
      - /dev/sdc
      - /dev/sdd

  tasks:
    - name: Deploy keepalive script
      ansible.builtin.copy:
        dest: "{{ keepalive_script_path }}"
        mode: '0755'

    - name: Ensure log file exists with sane permissions
      ansible.builtin.file:
        path: "{{ keepalive_log_path }}"
        state: touch

    - name: Schedule keepalive every 2 hours via cron
      ansible.builtin.cron:
        name: "USB drive keepalive"
        minute: "0"
        hour: "*/2"
        user: root
```

### Important Caveat

The keepalive playbook is a mitigation, not a root-cause fix.

It may reduce failures associated with idle devices but does not eliminate power-starvation events under sustained load.

---

## Known Recurrence During Recovery

During RAID resynchronization, Repository A independently disappeared from the USB bus and required physical reconnection.

Observations:

```text
Device disappeared from lsusb
Device disappeared from lsblk
Re-authorize attempt failed
Physical reconnect restored operation
```

This occurred after autosuspend mitigation had already been implemented.

This behavior further supports the conclusion that power instability remains an unresolved architectural weakness.

---

## Future Jenkins Architecture Lessons

This incident reinforces the future Jenkins storage design philosophy.

```text
Jenkins Deployment
       ↓
Jenkins Pod
       ↓
Persistent Volume Claim
       ↓
Persistent Volume
       ↓
Primary Storage
       ↓
Replication Storage
       ↓
Archive Storage
       ↓
Cloud Backup
```

If the pod dies:

```text
Pod dies
     ↓
New Pod Created
     ↓
PVC Mounts Existing Storage
     ↓
Jobs Preserved
Plugins Preserved
Build History Preserved
```

Core principle:

```text
Application
      ↓
Persistent Storage
      ↓
Replication
      ↓
Archive
      ↓
Offsite Copy
```

---

## Lessons Learned

1. RAID degradation must trigger alerting.
2. Mounted storage dependencies must be validated before synchronization jobs execute.
3. USB bus-powered storage introduces operational risk.
4. Replication success should be verified, not assumed.
5. Backup repositories should be directly accessible wherever possible.
6. RAID redundancy loss can remain hidden while services continue operating normally.
7. Multi-tier backups significantly reduced recovery risk during this incident.

---

## Open Items

```text
[ ] Verify A→RAID replication catches up successfully
[ ] Validate August backup copies exist in /srv/storage
[ ] Restore md0 from [U_] to [UU]
[ ] Deploy keep-drives-active.yml
[ ] Convert device references to /dev/disk/by-id paths
[ ] Implement RAID degradation alerting
[ ] Verify ExternalBackup auto-mount persistence
[ ] Evaluate powered USB hub or powered enclosures
```

---

## Outcome

Backup operations were restored successfully.

The backup repository remained intact throughout the incident.

Replication root cause was identified.

The remaining work focuses on restoring RAID redundancy and strengthening long-term storage resiliency controls.
