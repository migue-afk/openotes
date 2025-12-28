---
title: Forensic analysis workflow
layout: home
parent: Forensic
nav_order: 1
---

# Forensic analysis workflow
---

## 1. Basic Write Blocker Setup

The _write blocker_ can be configured directly on the device where the acquired image analysis is intended to be performed. However, it is highly recommended to use a dedicated device.

> Configuring a software-based _write blocker_ does not replace a dedicated device for such purposes, but it helps avoid accidental writes to the analysis disk.

### Write Blocker Setup on a Dedicated Device

For this setup, we will use a device with Linux. The first step is to disable automount and the **udisks** service:

```bash
sudo systemctl stop udisks2
sudo systemctl disable --now udisks2
```

This will prevent the operating system from automatically mounting the drive and potentially overwriting it.

Additionally, we can configure the device so that when any external storage device is connected, it will be in **read-only** mode using a rule in `/etc/udev/rules.d`:

```bash
touch /etc/udev/rules.d/90-usb-readonly.rules
```

The rule content will use the `blockdev` tool as follows:

```bash
KERNEL=="sd[a-z]", ACTION=="add", RUN+="/sbin/blockdev --setro /dev/%k"
KERNEL=="sd[a-z][0-9]", ACTION=="add", RUN+="/sbin/blockdev --setro /dev/%k"
```

With this, any external device connected will automatically be set to **read-only** mode.

This example will only work with devices identified as `sda, sda1, ...`. For others, like `nvme0n1`, the rule must be adjusted or the command run manually:

```bash
sudo blockdev --setro /dev/sda
```

To verify the drive's state:

```bash
sudo blockdev --getro /dev/sdX
```

Possible results:

```bash
0 -> The drive is in write mode (*write*)
1 -> The drive is in read-only mode (*read-only*)
```

---

## 2. Forensic Image Acquisition

Acquiring the forensic image is a critical step, as the integrity of the original data must be maintained. We will use the `dcfldd` tool to generate the image and calculate its **hash**.

### Cloning and Hash Calculation

If cloning is done on the same drive:

```bash
sudo dcfldd if=/dev/sdX of=imgforensics.img bs=4M hash=sha256 hashlog=hashes.txt errlog=errorlog.txt statusinterval=5
```

If we are using a dedicated device with a _write blocker_ and want to send the copy to a remote server:

```bash
sudo dcfldd if=/dev/sdX bs=4M hash=sha256 hashlog=hashes.txt errlog=errorlog.txt | ssh user@ip "cat > /destiny/imgforensics.img"
```

The completion time will depend on the hardware and transfer speed.

The resulting bit-by-bit image can have formats such as **.dd, .E01, .img**, among others, depending on the needs of the analysis.

### Cloned in two destinations
To carry out a cloning in two destinations, from the acquired image we use the following command line.

```bash
sudo dcfldd if=/dev/sdX bs=4M of=/mnt/disk1/imgforen1.img of=/mnt/disk2/imgforen2.img hash=sha256 hashlog=hashlog.txt errlog=errorlog.txt statusinterval=5
```
> of=/mnt/disk1 is the destiny 1 and of=/mnt/disk2 is the destity 2

### Integrity Check

```bash
sha256sum imgforensics.img
```

Additional Recommendations:

- Make a secure backup of the original evidence.
    
- Always work with a copy for analysis.
    

---

## 3. Mounting the Image for Analysis

### Inspecting the Partition Table

```bash
sudo fdisk -l imgforensics.img
```

Example output:

```bash
Disk imgforensics.img: 14.41 GiB, 15472047104 bytes, 30218842 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xa27785c7

Device          Boot Start      End  Sectors  Size Id Type
imgforensics.img1     8192 30218841 30210650 14.4G  c W95 FAT32 (LBA)
```

In this case, the **Device** column indicates that there is a partition. To mount it, we must calculate the **offset** by multiplying the sector where the partition starts by `512`:

```bash
starting_sector * 512
8192 * 512 = 4194304
```

Now we create a loop device for that partition using `losetup`:

```bash
sudo losetup --find --show --offset $((8192 * 512)) imgforensics.img
```

The command will return something like `/dev/loop0`, which is the corresponding loop device. Finally, we mount the partition in the desired directory:

```bash
sudo mount /dev/loop0 /mnt/img
```
