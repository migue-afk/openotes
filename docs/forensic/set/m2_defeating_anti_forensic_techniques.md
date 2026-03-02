---
title: M2. Defeating anti-forensic techniques
layout: home
parent: Set
nav_order: 2
tags: [carving, diskpartition, password, steganography]
---
#  Defeating anti-forensic techniques
---
## Carving files on an SSD of a Linux file system
---
Carving is a technique for recovering files and file fragments from unallocated hard drive space in the absence of file metadata. Carving files on a Linux system will be done using Autopsy.

### Tools
```bash
- PhotoRec 
- Scalpel 
- Foremost
- Autopsy 
```

Carving refers to extracting pieces of data directly from the raw content of the disk, based on signatures known as headers and/or pieces of files.

### How carving works

> - Scan without file system, FAT file system, NTFs, etc. are ignored. useful when damaged.
> - Signature Search: Sectors are scanned for byte patterns that mark the start and/or end of files, for example JPEG files start with FFD8 and end with FFD9.
> - Reconstruction: Once the signatures have been identified, the content is extracted between them and the files are reconstructed.

### Manual identification
> - Open the suspicious file or sector with a Hexadecimal viewer such as HxD or 010 Editor
> - Find the file type header as example FF D8 for JPEG Extract the data until you find the FF D9 footer 
> - Save that fragment in a new file with the corresponding extension, for example for a serious image .jpeg

---
## Data recovery from a lost or deleted disk partition
---

When a partition is deleted from a disk, the files inside the disk are lost and the computer deletes the entries related to the deleted partition from the MBR partition table. However, as long as the corresponding section of the disk is not overwritten, there is a possibility to recover the deleted partition and the files it contains.

### Tools

```bash

EaseUS Data Recovery Wizard # Windows

# Alternative for Linux
- TestDisk + Photorec # TestDisk recovers partitions and FAT/NTFS/EXT systems without GUI, while Photorec recovers files based on file signatures
- Foremost # Craving (analysis without taking into consideration partitions)
- Extundelete # Only for EXT3/EXT4 files
- R-Linux # Only for EXT2...4, GUI
- ddrescue # Damaged disk cloner sector by sector.
```

---
## App password cracking
---
Passwords are usually a string of characters that are used to verify the identity of a user, the objective is to try to decipher the passwords of files and applications protected with passwords.


### Tools
```bash
Passware Kit Forensic
```

With the tool we can select the protected file and try to decrypt the password, for Linux we can use `John The Riper`

`Passware` would provide us with a GUI however it is limited by the DEMO version

---
## Detecting Steganography
---

Occasionally, attackers try to trick users and system security by hiding a malicious program with a seemingly useful image or file. By doing this, they can bypass security checks and lure victims into downloading and running malware, as well as prevent forensic identification.


### Tools
```bash
StegSpy2.1.exe
OpenStego
```

By selecting the suspicious file and clicking RUN, the tool gives us the following information.

`Steganography found at maker position 103431 Hiderman program detected!`

Which means that the tool scanned the file and showed the type of steganography technique used to hide another file in it.

`OpenStego` allows us to both hide data and Extract data
