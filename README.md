# Dell PowerVault MD3200i / MD3220i Homelab Setup Guide

This guide explains the storage concepts and the normal workflow for turning physical SAS disks in a Dell PowerVault MD3200i/MD3220i into storage that Linux, VMware, or another server can use over iSCSI.

It is written for someone new to SAN terminology and pairs with `powervault_wizard.py`.

> **Safety:** The wizard is conservative by design. Read-only checks run automatically. Any array-changing command is printed first and asks for confirmation. Formatting a Linux block device requires typing `FORMAT`.

---

## 1. The mental model

The easiest way to understand a PowerVault is as a stack:

```text
Physical SAS disks
        |
        v
RAID disk group
(example: 5 disks in RAID 6)
        |
        v
Virtual disk
(the usable SAN volume carved from the RAID group)
        |
        v
LUN mapping
(the number by which that virtual disk is presented to a host)
        |
        v
Host / host group
(whose iSCSI IQN is allowed to see it)
        |
        v
iSCSI target + network portal
        |
        v
Linux / VMware sees a block device
        |
        +--> Linux: ext4/XFS -> mount point
        |
        +--> VMware: VMFS datastore
```

### Physical disk

A real SAS drive in a numbered enclosure/slot, for example `[0,7]`.

### RAID disk group

A set of physical disks combined with RAID 0/1/5/6. The disk group provides redundancy and raw RAID capacity, but by itself it is **not yet a LUN that a server can mount**.

Example:

```text
5 x 1.2 TB drives -> RAID 6 disk group -> about 3 drives' worth of usable capacity
```

### Virtual disk

Dell calls the logical SAN volume created inside a RAID disk group a **virtual disk**. Other storage vendors may call this a volume.

A disk group can contain one large virtual disk or multiple smaller virtual disks.

### LUN

**LUN = Logical Unit Number.**

A LUN is mainly the address/number used when presenting a virtual disk to a host. It is not another RAID layer.

Example:

```text
Virtual disk LAB_DATA -> mapped to Linux server as LUN 0
Virtual disk HA_VM    -> mapped to VMware host group as LUN 0
```

Those can both be LUN 0 because they are mapped in different host contexts.

### Host

A PowerVault host object represents a server allowed to access storage. It has an operating-system type such as:

- Linux (DM-MP)
- VMware
- Windows

The exact host-type index should always be checked with:

```bash
SMcli <management-ip> -c 'show storageArray hostTypeTable;'
```

On the tested MD3220i configuration the table reported:

```text
Linux (DM-MP)  1
VMWare          2
Windows         0
```

### iSCSI initiator

The server identifies itself with an IQN such as:

```text
iqn.2004-10.com.ubuntu:01:...
```

On Ubuntu/Linux:

```bash
cat /etc/iscsi/initiatorname.iscsi
```

### iSCSI target and portal

The PowerVault is the **target**. Each iSCSI Ethernet port is a network **portal** through which the same target can be reached.

Factory-style MD3200i addressing commonly looks like:

```text
Controller 0 port 0  192.168.130.101/24
Controller 0 port 1  192.168.131.101/24
Controller 0 port 2  192.168.132.101/24
Controller 0 port 3  192.168.133.101/24

Controller 1 port 0  192.168.130.102/24
Controller 1 port 1  192.168.131.102/24
Controller 1 port 2  192.168.132.102/24
Controller 1 port 3  192.168.133.102/24
```

Do not put a normal Internet/default gateway on a dedicated iSCSI NIC.

---

# 2. Using the wizard

Copy the script to the Linux management/server host and run:

```bash
chmod +x powervault_wizard.py
sudo ./powervault_wizard.py
```

On first use choose **Configure saved defaults** and enter items such as:

```text
SMcli path:        /opt/dell/mdstoragemanager/client/SMcli
Array management:  <management IP>
iSCSI NIC:         eno3
Local iSCSI CIDR:  192.168.130.10/24
iSCSI portal:      192.168.130.101
Mount root:        /mnt
```

The configuration is stored in:

```text
~/.config/powervault-wizard/config.json
```

The wizard menu handles:

1. iSCSI NIC/network checks
2. array/RAID/drive status
3. new drives -> RAID disk group -> virtual disk
4. another virtual disk in an existing disk group
5. adding a server and mapping a volume
6. Linux iSCSI login and mounting
7. physical-disk health/log collection
8. troubleshooting report generation
9. SNMP traps/basic status
10. safe homelab power-down procedure

---

# 3. Scenario A: New drives -> new RAID volume

## Step A1: Inspect drives

```bash
SMcli <management-ip> -c 'show allPhysicalDisks;'
```

Look for drives with:

```text
Status: Optimal
Mode:   Unassigned
```

Do not blindly include a questionable disk just because it says `Optimal`; historical SCSI error logs can provide additional information.

## Step A2: Create the RAID disk group

Example: five disks in RAID 6:

```bash
SMcli <management-ip> \
  -c 'create diskGroup physicalDisks=(0,3 0,4 0,5 0,6 0,7) raidLevel=6 userLabel="LAB_DATA_RG";'
```

This creates the RAID set only.

## Step A3: Create a virtual disk inside the group

Use all free capacity:

```bash
SMcli <management-ip> \
  -c 'create virtualDisk diskGroup="LAB_DATA_RG" userLabel="LAB_DATA" owner=0 usageHint=fileSystem;'
```

Now you have an actual SAN volume.

Check it:

```bash
SMcli <management-ip> -c 'show allVirtualDisks;'
```

## Step A4: Assign a hot spare

Example:

```bash
SMcli <management-ip> \
  -c 'set physicalDisk [0,8] hotSpare=TRUE;'
```

Then verify coverage:

```bash
SMcli <management-ip> \
  -c 'show storageArray hotSpareCoverage;'
```

A hot spare should be at least as large as the drives it may replace.

---

# 4. Scenario B: Add a Linux server to an existing volume

## Step B1: Install the initiator tools

Ubuntu/Debian:

```bash
sudo apt update
sudo apt install open-iscsi ethtool
sudo systemctl enable --now iscsid
```

## Step B2: Configure the dedicated iSCSI NIC

Temporary example:

```bash
sudo ip link set eno3 up
sudo ip addr replace 192.168.130.10/24 dev eno3
```

Check:

```bash
ip -br addr show eno3
ethtool eno3
ping -c 2 192.168.130.101
```

The wizard can generate an Ubuntu netplan snippet for persistence.

## Step B3: Get the server's IQN

```bash
cat /etc/iscsi/initiatorname.iscsi
```

## Step B4: Check the PowerVault host type table

```bash
SMcli <management-ip> \
  -c 'show storageArray hostTypeTable;'
```

Use the index returned by your array.

## Step B5: Create the PowerVault host object

Example for Linux if Linux is index 1:

```bash
SMcli <management-ip> \
  -c 'create host userLabel="linux01" hostType=1;'
```

## Step B6: Register the server's iSCSI initiator

```bash
SMcli <management-ip> \
  -c 'create iscsiInitiator iscsiName="<SERVER-IQN>" userLabel="linux01-iscsi" host="linux01";'
```

## Step B7: Map the virtual disk to the host

```bash
SMcli <management-ip> \
  -c 'set virtualDisk ["LAB_DATA"] logicalUnitNumber=0 host="linux01";'
```

This is the moment at which `LAB_DATA` becomes **LUN 0 for linux01**.

---

# 5. Scenario C: Discover, login, and mount on Linux

## Discovery

```bash
iscsiadm -m discovery -t sendtargets -p 192.168.130.101
```

A PowerVault may report all controller portals even though only one of your Linux NICs can currently reach one subnet. That is normal.

## Login

```bash
iscsiadm -m node \
  -T <TARGET-IQN> \
  -p 192.168.130.101:3260 \
  --login
```

Confirm:

```bash
iscsiadm -m session
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL,FSTYPE,MOUNTPOINTS
```

Use stable paths rather than guessing `/dev/sdX`:

```bash
ls -l /dev/disk/by-path/ | grep iscsi
```

Example:

```text
...ip-192.168.130.101:3260-iscsi-<target>-lun-0 -> ../../sdc
```

## Creating a Linux filesystem

Only do this if the LUN is meant to be a normal Linux filesystem and is empty.

First inspect it:

```bash
wipefs -n /dev/disk/by-path/<iscsi-path>
lsblk -f /dev/disk/by-path/<iscsi-path>
```

Example ext4 creation:

```bash
mkfs.ext4 -L LAB_DATA /dev/disk/by-path/<iscsi-path>
```

Mount:

```bash
mkdir -p /mnt/lab-data
mount /dev/disk/by-path/<iscsi-path> /mnt/lab-data
```

For automatic login:

```bash
iscsiadm -m node -T <TARGET-IQN> -p 192.168.130.101:3260 \
  --op update -n node.startup -v automatic
```

For `/etc/fstab`, prefer the filesystem UUID and `_netdev`:

```text
UUID=<uuid> /mnt/lab-data ext4 defaults,_netdev,nofail,x-systemd.automount,x-systemd.device-timeout=30 0 2
```

---

# 6. Scenario D: VMware shared HA datastore

Do **not** create ext4/XFS on the shared VMware LUN.

For an ESXi cluster:

```text
PowerVault virtual disk HA_VM
             |
             v
          LUN 0
             |
      VMware host group
         /        \
      ESXi01     ESXi02
```

Create each ESXi host with the VMware host type, register each IQN, add the hosts to one PowerVault host group, and map the shared virtual disk to the **host group**.

Create a host group:

```bash
SMcli <management-ip> \
  -c 'create hostGroup userLabel="VMWARE_CLUSTER";'
```

Add a host to it:

```bash
SMcli <management-ip> \
  -c 'set host ["esxi01"] hostGroup="VMWARE_CLUSTER";'
```

Map the LUN to the group:

```bash
SMcli <management-ip> \
  -c 'set virtualDisk ["HA_VM"] logicalUnitNumber=0 hostGroup="VMWARE_CLUSTER";'
```

Then create VMFS from VMware, not Linux.

> Never mount one ordinary ext4/XFS filesystem read-write on multiple independent hosts. Shared block storage needs a cluster-aware filesystem or a hypervisor that coordinates access.

---

# 7. Multipath and dual controllers

One iSCSI cable is enough to learn and get storage online, but it is not highly available.

A proper redundant layout eventually looks like:

```text
server NIC/path A -> Controller 0 iSCSI port
server NIC/path B -> Controller 1 iSCSI port
                      |
                      v
             same virtual disk/LUN
                      |
                      v
               Linux DM-Multipath
```

Linux hosts should use the `Linux (DM-MP)` host type when your array's host-type table identifies it.

Do the simple single-path setup first. Add `multipath-tools` after basic iSCSI login and LUN mapping are understood.

---

# 8. Troubleshooting checklist

The wizard's **Generate a troubleshooting report** option collects the common information into one text file.

Manual checklist:

```bash
ip -br addr
ethtool <iscsi-nic>
ping <portal-ip>
iscsiadm -m session
iscsiadm -m node
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL,FSTYPE,UUID,MOUNTPOINTS
```

PowerVault:

```bash
SMcli <management-ip> -c 'show storageArray healthStatus;'
SMcli <management-ip> -c 'show storageArray summary;'
SMcli <management-ip> -c 'show storageArray longRunningOperations;'
SMcli <management-ip> -c 'show storageArray hotSpareCoverage;'
SMcli <management-ip> -c 'show storageArray hostTopology;'
SMcli <management-ip> -c 'show allVirtualDisks;'
SMcli <management-ip> -c 'show allPhysicalDisks;'
```

Common failure patterns:

### `NO-CARRIER`

This is a physical Ethernet problem, not an iSCSI/IP problem. Check cable, NIC, port, link LEDs, and `ethtool`.

### Ping works, TCP 3260 fails

The physical/IP path works but the iSCSI service/port is unavailable or incorrectly configured.

### Discovery works but no disk appears

Usually one of these:

- no virtual disk exists yet
- the virtual disk is not mapped to the host/host group
- the server IQN is not registered correctly
- login has not occurred

### iSCSI login works but the wrong/no LUN appears

Check:

```bash
SMcli <management-ip> -c 'show storageArray hostTopology;'
SMcli <management-ip> -c 'show allVirtualDisks;'
```

and inspect `/dev/disk/by-path/`.

---

# 9. Physical-disk status and SMART-like information

High-level drive status:

```bash
SMcli <management-ip> -c 'show allPhysicalDisks;'
```

This shows items such as:

- Optimal/failed state
- model and firmware
- capacity
- SAS path consistency
- current link rate
- slot and serial number

For deeper SAS drive health, collect the drive diagnostic archive:

```bash
SMcli <management-ip> \
  -c 'save allPhysicalDisks logFile="/tmp/powervault-drive-logs.zip";'
```

The output is normally a ZIP archive containing:

```text
physicalDiskDiagnosticData.bin
```

It contains per-drive SCSI LOG SENSE / diagnostic data. This can reveal historical corrected errors, retry activity, verify errors, non-medium errors, temperature and related counters even when the array still says a disk is `Optimal`.

---

# 10. Can drive status be monitored over SNMP?

**Use SNMP primarily for alerts/traps, not as the detailed disk-health interface.**

There is some firmware-generation nuance:

- Dell historically documented MD3xxx systems as supporting SNMP traps rather than useful detailed polling.
- Later 8.x firmware added an embedded SNMP agent with communities and basic MIB-II system variables.
- The MD32xx CLI exposes SNMP community/trap configuration.
- Detailed physical-disk, RAID-group, virtual-disk and SCSI diagnostic information is still much better obtained with `SMcli`.

A practical homelab monitoring design is therefore:

```text
PowerVault SNMP traps -> monitoring server for immediate fault alerts
           +
periodic SMcli healthStatus / allPhysicalDisks -> detailed polling/status
```

Show configured communities:

```bash
SMcli <management-ip> -c 'show allSnmpCommunities summary;'
```

Create a community:

```bash
SMcli <management-ip> \
  -c 'create snmpCommunity communityName="homelab";'
```

Add a trap destination:

```bash
SMcli <management-ip> \
  -c 'create snmpTrapDestination trapReceiverIP=<monitor-ip> communityName="homelab" sendAuthenticationFailureTraps=FALSE;'
```

Test it:

```bash
SMcli <management-ip> \
  -c 'start snmpTrapDestination trapReceiverIP=<monitor-ip> communityName="homelab";'
```

For detailed health polling, a simple cron/systemd job can periodically run:

```bash
SMcli <management-ip> -c 'show storageArray healthStatus;'
```

and alert if the result is not healthy.

---

# 11. Safe homelab power-down

If the lab is idle, the PowerVault does not need to remain powered continuously.

Recommended sequence:

1. Stop VMs and applications using the LUNs.
2. Unmount Linux filesystems / detach hypervisor datastores cleanly.
3. Run `sync` on Linux.
4. Log out iSCSI sessions, or shut down the hosts.
5. Confirm the PowerVault is not rebuilding, initializing or copying data.
6. With host I/O stopped, switch off both PowerVault PSUs.

Check long-running work with:

```bash
SMcli <management-ip> \
  -c 'show storageArray longRunningOperations;'
```

Power-up order:

```text
Expansion shelves (if any)
        -> PowerVault RAID enclosure
        -> wait for healthy controllers/disks
        -> hosts / iSCSI login
```

---

# 12. Useful official Dell references

- Dell PowerVault MD32xx/MD36xx CLI Guide — create disk group:
  https://www.dell.com/support/manuals/en-us/powervault-md3200/32xx_36xx_cli_pub/create-disk-group
- Create virtual disk in existing disk group:
  https://www.dell.com/support/manuals/en-us/powervault-md3200i/32xx_36xx_cli_pub/create-raid-virtual-disk-free-extent-base-select
- Create iSCSI initiator:
  https://www.dell.com/support/manuals/en-us/powervault-md3200i/32xx_36xx_cli_pub/create-iscsi-initiator
- Set virtual disk/LUN mapping:
  https://www.dell.com/support/manuals/en-ae/powervault-md3200/hogs_cli_pub/set-virtual-disk-mapping
- Show storage-array health/profile/hot-spare coverage:
  https://www.dell.com/support/manuals/en-ca/powervault-md3200/32xx_36xx_cli_pub/show-storage-array

---

## Short version

When you forget the terminology, remember:

```text
Drives
  -> RAID disk group
  -> virtual disk
  -> map it as LUN N to a host/host group
  -> host logs into PowerVault over iSCSI
  -> OS sees block device
  -> Linux filesystem + mount, OR VMware VMFS
```

That is the whole workflow.
