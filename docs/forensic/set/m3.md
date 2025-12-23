---
title: M1. Data Acquisition and Duplication
layout: home
parent: Set
nav_order: 1
tags: [data, adquisition, dd, windows, linux, bootable, virtualmachine]
---
# Data Acquisition and Duplication
---

## Creating an dd image of a file system

---
A `dd` image is a disk image file that is a bit-by-bit copy of a hard drive or disk partition, including all files/folders, deleted files, files remaining in Slack space and unallocated space, file system information, etc.

- Create an image using the dd tool in the Windows operating system.

### Tools
```bash
dd.exe
```

#### Notes
> The physical acquisition of the suspect disk generally requires external data storage, such as an external USB drive. To avoid bottlenecks, SSD or NVMe storage is required to speed up the process and limit bottlenecks.

### Process

1. In Windows we can see information about available hard drives, allowing us to see what the bit-by-bit copy will be of.

```bash
wmic diskdrive list brief /format:list

->
Caption=WDC WD10EZEX-00ZF5A0 
DeviceID=\\.\PHYSICALDRIVE0 
Model=WDC WD10EZEX-00ZF5A0 
Partitions=2 
Size=1000204886016 

Caption=SAMSUNG HD502HJ 
DeviceID=\\.\PHYSICALDRIVE1 
Model=SAMSUNG HD502HJ 
Partitions=1 
Size=500107862016
```

This allows us to identify the hard drives in the system; although `wmic` is outdated, it serves the purpose. Additionally, to complement disk management, we can use the following tools: `Get-Disk` and `Get-PhysicalDisk`.

To perform a bit-by-bit copy, use the following command:
```bash
.\dd.exe if=\\.\PHYSICALDRIVE0 of=F:\Windows_Evidence_002.dd bs=512k --size --progress
```

Where F: Is the destination disk, F must be greater than `PHYSICALDRIVE0` to avoid problems:

```bash
-> if: This is the input file
-> of: This is the output file
-> bs=512k Means block size, that is, the size that reads from the source and writes to the destination. dd needs an amount of RAM equal to the specified block size to store data while it moves to the destination.

-> size = This tells dd to try to determine the size of the input device to ensure it does not read beyond the end of the device. This is useful for devices like USB, where reading beyond the end can cause problems.
```
For disks with damaged sectors, we can use the following configuration along with a smaller bs like `512b` or `4k`, this would allow us to recover as much data as possible.

```bash
.\dd.exe if=\\.\PHYSICALDRIVE0 of=F:\Windows_Evidence_002.dd bs=512k conv=noerror,sync --size --progress
```

The command `conv=noerror,sync` are used to handle read errors during data copying, particularly when dealing with damaged or faulty storage devices.

At the end of the forensic copy, the result is as follows:

```bash
204800+0 records in
204800+0 records out
```

The image obtenience convert the principal soucre pf analysisi for the insvestigtors, dependence of work forensic and the requiemnets, the exan images analysis with forensic tools, or convert to format tools and boot loader.

Image acquisition becomes the primary source of analysis for investigators. Depending on the forensic work and requirements, the images will be analyzed with forensic tools or converted into forensic tool formats and boot images.

2. In Linux the process is as follows

```bash
sudo dd if=/dev/sdX of=/route/to/forensic/imaging.dd bs=4M conv=noerror,sync status=progress
```

#### Note
> You can use other tools, such as `ddrescue` for damaged disks or `dcfldd`, which are specialized command-line tools for forensic imaging.

### Warning:
> Improper use of dd can result in total data loss; review the command several times before execution.

## Convert the acquired image file into a bootable virtual machine

---
#### Lab Scenario:

In some cases, the investigator may need to create a live environment of the machine and run the forensically acquired image file in the virtual machine to extract additional artifacts that may not have been discovered in the static analysis. These additional artifacts could serve as evidence and help solve the case.
### Tools
```bash
qemu
virt-manager
```
### Process

To create an executable from the acquired `.dd` image, we can use the `qemu` tool. The converted image must be adapted to the virtualization platform. If we assume the virtualization medium is **Hyper-V**, then the `.dd` image must be converted to **vhdx** as follows:

```bash
qemu-img convert -f raw Windows_Evidence_002.dd -O vhdx /home/user/Desktop/Windows.vhdx
```

## Acquiring RAM from Windows workstations
---

In this process, it is essential to capture the machine's RAM dump, as RAM contains volatile data that is lost when the machine loses power. In this scenario, the RAM acquisition is performed on a Windows workstation.
### Tools
```bash
Belkasoft RAM Capturer
```
### Process

Use the **belkasoft** tool to extract the RAM, allocate a larger capacity disk to store the RAM dump, the resulting file will be a `.mem`

## Viewing the contents of a forensic image file
---

The **AccessData FTK Imager** tool saves researchers time by eliminating the tedious process of recovering each deleted file from the system.

### Tools
```bash
AccessData_FTK_Imager_4.3.1.exe
```

#### Note

> In Linux you can use **Guymager**
