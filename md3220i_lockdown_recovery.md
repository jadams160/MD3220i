# Dell PowerVault MD3200i / MD3220i Lockdown Recovery Guide

This document summarizes a recovery procedure that was successfully used on a Dell PowerVault MD3220i running controller firmware **08.20.24.60** after the array entered controller lockdown because of a corrupt primary database.

It is intended as a field-recovery reference for technically experienced administrators working on equipment they own or are authorized to service.

> **Warning**
>
> Some commands in this guide are low-level controller service commands. A mistake can make recovery harder or destroy array configuration. Do not run destructive commands such as `sysWipeAllConfigData` unless you explicitly intend to erase the array configuration and understand the consequences.

---

## Symptoms

The affected controller booted far enough to initialize most hardware, but serial output showed:

```text
CmgrLockdownException: CORRUPT DBM DATABASE DETECTED - LOCKDOWN Type=6

This controller is locked down due to detected
corruption in the primary database.

In order to prevent any configuration loss, the
backed up database must be restored.
```

Management software could report the array in **Recovery Mode** and the second controller could remain **Failed / Offline** even after the database lockdown was cleared.

The successful recovery required solving **two separate problems**:

1. Clear the controller's database lockdown.
2. Clear the surviving controller's persistent belief that the alternate controller had failed and should remain held in reset.

---

## Tested hardware / software

- Dell PowerVault MD3200i / MD3220i family
- Dual RAID controllers
- Firmware tested: **08.20.24.60**
- NVSRAM tested: **N26X0-820890-008**
- Dell MD Storage Manager / `SMcli`
- Serial console at **115200 8-N-1**

The exact commands and behavior may differ on other firmware generations.

---

## Important note about the rear password-reset switch

On later MD3200-series firmware, the rear password-reset switch functionality was disabled.

For firmware in the **08.20.09.60 and later** family, do not assume the physical reset switch will provide access to recovery functions. A serial console may be required.

---

## Serial console interfaces

During boot, send an actual serial **BREAK**.

Two different interfaces may be available.

### Normal Service Interface

At the boot prompt, press:

```text
S
```

This opens the Service Interface.

Use the appropriate Dell service credentials for your environment.

The Service Interface can be useful for:

- displaying the management IP configuration
- displaying controller LED/status codes
- some service-level configuration functions

### Hidden VxWorks / SYMbol shell

During the same boot window, entering the controller shell instead of the normal Service Interface provides a VxWorks-style prompt:

```text
->
```

On the tested firmware, the shell exposed low-level recovery functions such as:

```text
lemClearLockdown
clearHardwareLockdown
cmgrSetAltToFailed
cmgrSetAltToOptimal
```

Before running any low-level command, verify that the symbol exists using `lkup`.

Example:

```text
lkup "lemClearLockdown"
```

Expected style of output:

```text
lemClearLockdown    0x........ text (RAID)
value = 0 = 0x0
```

---

# Recovery Procedure

## 1. Diagnose the array before changing anything

Use `SMcli` to check health:

```bash
SMcli <management-ip> -c 'show storageArray healthStatus;'
```

Also collect the array summary:

```bash
SMcli <management-ip> -c 'show storageArray summary;'
```

If the controller is in database lockdown, confirm the serial console shows a message similar to:

```text
CORRUPT DBM DATABASE DETECTED - LOCKDOWN Type=6
```

This is distinct from a dead controller, failed iSCSI hardware, or a simple network problem.

---

## 2. Recover Controller A in isolation

Power down as required and physically isolate one controller so it can boot without the alternate controller interfering.

With only Controller A active, normal serial output may include:

```text
No attempt made to open Inter-Controller Communication Channels
LockMgr Role is Master
```

Allow the normal power-on diagnostics to complete.

Typical successful diagnostics include:

```text
Processor DRAM                 Passed
NVSRAM                         Passed
Flash Test                     Passed
iSCSI hardware tests           Passed
RT Clock Tick                  Passed
```

An RTC warning may still appear even if the RTC tick diagnostic passes.

---

## 3. Clear the database lockdown on Controller A

At the controller shell:

```text
lkup "lemClearLockdown"
```

If the symbol exists, run:

```text
lemClearLockdown
```

On the tested controller, this immediately rebooted the controller.

Allow it to complete a full reboot.

---

## 4. Clear Storage Array Recovery Mode

After Controller A becomes reachable through normal management again:

```bash
SMcli <management-ip> -c 'clear storageArray recoveryMode;'
```

If the controller has only just come online, the first connection attempt may fail while networking or management services are still initializing. Retry after the controller is fully up.

Then check:

```bash
SMcli <management-ip> -c 'show storageArray healthStatus;'
```

The goal at this stage is for the previous controller-lockdown / recovery-mode fault to disappear.

---

## 5. Recover Controller B in isolation

If Controller B still appears failed or offline, isolate it and boot it by itself.

Verify the hardware-lockdown symbol:

```text
lkup "clearHardwareLockdown"
```

If present, run:

```text
clearHardwareLockdown
```

Then verify and clear the normal lockdown state:

```text
lkup "lemClearLockdown"
lemClearLockdown
```

The `lemClearLockdown` command may reboot the controller.

After reboot, verify that Controller B can complete a normal startup.

Successful output should eventually include:

```text
SOD: Initialization Phase Complete
sodMain Normal sequence finished
sodMain complete
```

There should be no new:

```text
CORRUPT DBM DATABASE DETECTED - LOCKDOWN Type=6
```

message.

When Controller B is booted alone, messages such as these are expected:

```text
Alt unavailable
IconSendInfeasibleException
Peering Disabled
Unable to initialize mirror device
```

They occur because the alternate controller is physically absent.

---

# Fixing a Second Controller That Remains Failed / Offline

Clearing Controller B's own lockdown may not be sufficient.

In the successful recovery, Controller A still had a persistent record that the alternate controller had failed.

Controller A's serial console showed:

```text
Failing The Alternate Controller
Alt Ctl Reboot:
Reboot reason: 0x6

holding alt ctl in reset
HealthCheck: Alt Ctl: 1 Reset_Failure
```

This explains a common symptom where the second controller appears to begin booting and then seems to hang when both controllers are installed.

---

## 6. Clear Controller A's stale alternate-controller failure state

Remove Controller B again and boot Controller A normally.

Enter Controller A's shell.

Verify these functions exist:

```text
lkup "cmgrSetAltToFailed"
lkup "cmgrSetAltToOptimal"
```

Then enable the debug command set:

```text
loadDebug
```

Run:

```text
cmgrSetAltToFailed
```

followed by:

```text
cmgrSetAltToOptimal
```

The important successful output is:

```text
releasing alt ctl from reset
```

This indicates that Controller A has cleared the state that was holding the alternate controller in reset.

---

## 7. Hot-insert Controller B

Leave Controller A running.

Hot-insert Controller B and allow the controllers to communicate and synchronize.

Do not immediately interrupt the serial boot unless the original lockdown error returns.

Monitor array health:

```bash
SMcli <management-ip> -c 'show storageArray healthStatus;'
```

And periodically check:

```bash
SMcli <management-ip> -c 'show storageArray summary;'
```

A successful recovery should eventually show:

```text
RAID Controller Modules: 2
Consistency mode: Duplex (dual RAID controller modules)
```

Both controller slots should report matching controller firmware and NVSRAM versions.

The following faults should disappear:

```text
Controller lockdown
Recovery Mode
Offline RAID Controller Module
SAS Port Failed
Expansion Enclosure - Loss of Path Consistency
```

---

# Minimal Proven Command Sequence

## Controller A - isolated

```text
lkup "lemClearLockdown"
lemClearLockdown
```

Allow reboot.

Then from the management station:

```bash
SMcli <management-ip> -c 'clear storageArray recoveryMode;'
SMcli <management-ip> -c 'show storageArray healthStatus;'
```

---

## Controller B - isolated

```text
lkup "clearHardwareLockdown"
clearHardwareLockdown

lkup "lemClearLockdown"
lemClearLockdown
```

Allow reboot and verify normal SOD completion.

---

## Controller A - with B removed

```text
lkup "cmgrSetAltToFailed"
lkup "cmgrSetAltToOptimal"

loadDebug
cmgrSetAltToFailed
cmgrSetAltToOptimal
```

Expected:

```text
releasing alt ctl from reset
```

Then hot-insert Controller B.

---

# Commands Investigated but Not Required

The following commands were identified during troubleshooting but were **not required for the successful recovery described here**.

## `bdbmClearPrimaryDatabase`

This is associated with clearing/rebuilding the primary database as part of a database-recovery procedure.

It may be appropriate when preserving or restoring the existing array configuration is required.

Do not use it casually.

---

## `sysWipeAllConfigData`

This is an extremely destructive clean-start operation.

Firmware text associated with the command warns that it should not be used when configuration restoration or user-data recovery is desired.

Only consider it if:

- the old array configuration is not needed
- all user data can be discarded
- normal database recovery has failed
- you explicitly want a factory-like clean configuration

It was **not necessary** in this recovery.

---

# Post-Recovery Checks

## Array health

```bash
SMcli <management-ip> -c 'show storageArray healthStatus;'
```

## Full array summary

```bash
SMcli <management-ip> -c 'show storageArray summary;'
```

## Physical disks

```bash
SMcli <management-ip> -c 'show allPhysicalDisks;'
```

## Synchronize controller clocks

```bash
SMcli <management-ip> -c 'set storageArray time;'
```

Then verify:

```bash
SMcli <management-ip> -c 'show storageArray time;'
```

---

# Collecting SAS Drive Health / SMART-like Data

The MD3200-series reports high-level drive state through:

```bash
SMcli <management-ip> -c 'show allPhysicalDisks;'
```

However, a drive can still report `Optimal` while accumulating significant read-recovery or other SCSI error counters.

To collect the per-disk SCSI LOG SENSE diagnostic information:

```bash
SMcli <management-ip> \
  -c 'save allPhysicalDisks logFile="/path/to/drive-logs.zip";'
```

Depending on the MD Storage Manager version, the output may be a ZIP archive regardless of the filename extension supplied.

The archive contains:

```text
physicalDiskDiagnosticData.bin
```

The binary contains per-drive diagnostic records, including SCSI LOG SENSE information.

Useful health indicators include:

- corrected read errors
- delayed corrections
- rereads / rewrites
- correction-algorithm invocation counts
- uncorrected errors
- verify errors
- non-medium errors
- temperature
- self-test results
- informational-exception / predictive-failure data

A drive that remains `Optimal` but has dramatically higher corrected-read or retry counts than its peers should be treated with suspicion before building a new RAID group.

---

# Troubleshooting Notes

## Controller appears to hang at `loading flash file: iSCSI`

Do not assume the iSCSI hardware has failed.

If the controller can boot normally in isolation but stops when the alternate controller is present, check the surviving controller's serial console for:

```text
holding alt ctl in reset
Reset_Failure
```

The issue may be the controller-manager state rather than the iSCSI subsystem itself.

---

## `IconSendInfeasibleException`

When only one controller is installed, repeated messages such as:

```text
IconSendInfeasibleException
Alt unavailable
Peering Disabled
```

can be normal because inter-controller communication is impossible.

Interpret them in context.

---

## FPGA firmware warning

The tested unit reported:

```text
FPGA FW is out of date
"Rhone03 rev17" currently in use
"Rhone03 rev20" available for update
```

Firmware/FPGA upgrades should be deferred until:

- controller lockdown is resolved
- both controllers are stable
- the array is operating normally in duplex mode

Do not combine a firmware update with a controller-lockdown recovery unless required.

---

# Recovery Logic Summary

The successful repair can be understood as two independent state problems:

```text
Primary DB corruption
        |
        v
Controller database lockdown
        |
        |  lemClearLockdown
        v
Controller can boot normally
        |
        v
Storage Array Recovery Mode
        |
        |  clear storageArray recoveryMode
        v
Controller A operational


Controller B still marked failed
        |
        v
A holds alternate controller in reset
        |
        |  clearHardwareLockdown on B
        |  lemClearLockdown on B
        |
        |  cmgrSetAltToFailed on A
        |  cmgrSetAltToOptimal on A
        v
"releasing alt ctl from reset"
        |
        v
Hot-insert Controller B
        |
        v
Controllers synchronize
        |
        v
Normal duplex array
```

---

# Final Warning

These service-shell commands bypass normal management safeguards.

Before using them:

- record current firmware and NVSRAM versions
- save any configuration that is still recoverable
- isolate controllers when instructed
- verify each symbol with `lkup`
- avoid destructive database commands unless data/configuration loss is acceptable
- make only one state-changing intervention at a time
- capture the full serial console output

This procedure documents one successful MD3220i recovery and should be treated as a troubleshooting reference, not an official Dell service procedure.
