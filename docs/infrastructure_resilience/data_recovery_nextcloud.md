---
title: Nextcloud data recovery
parent: Infraestructure Resilience
layout: home
---

# Nextcloud data recovery
---

## Case Overview
A host synchronized data with Nextcloud and was subsequently infected with a Trojan. The synchronization interval was one hour, during which the infected host synchronized malicious or corrupted files with the Nextcloud server. Fortunately a backup was available, allowing for recovery as described below.

---

## 1. Disable the Nextcloud User

To prevent additional writes to the account, first disable the user both physically and logically. To disable the user in Nextcloud, use one of the following commands:

```bash
# Docker installation
sudo docker exec -u www-data -it nextcloudID php occ user:disable USER

# Standalone installation
sudo -u www-data php occ user:disable USER
```

Now the user is blocked, and no new writes to their directory are possible.

As an additional step, verify the ownership and group of the directory `/var/html/data/USER/files`. Running `ls -la` should display `www-data:www-data`.

---

## 2. Delete Compromised Files

To continue with data recovery, it is necessary to delete the compromised files located in `/var/html/data/USER/files`. These files will be replaced with a backup from two hours before the incident.  
Inside the container, run:

```bash
rm -r /var/html/data/USER/files/*
```

This deletes all files belonging to **USER**. Now proceed to restore the data from the backup.

In this case, the backup was stored on MinIO S3 and on the local disk with restic; the restore operation used a mounted local **restic** repository to allow navigation to this user's specific directory.

---

## 3. Mount the Restic Repository

```bash
# Mount Restic repo to a local directory
sudo restic -r resticrepo mount /mount/point/restic
cd /mount/point/restic
```

Navigate through the directories and locate the data corresponding to **USER**. Use `rsync` or `cp` to copy the files to:

```
/var/html/data/USER/files/
```

After completing the restore, unmount the Restic repository:

```bash
sudo umount /mount/point/restic
```

---

## 4. Fix Ownership and Rescan Files

At this point, the restored files for **USER** are in place, but Nextcloud will **not recognize them** because they were copied directly into the filesystem rather than uploaded through the Nextcloud interface. This creates inconsistencies.

To solve this, run a file scan for the user:

```bash
# Docker installation
sudo docker exec -u www-data -it nextcloudID php occ files:scan USER

# Standalone installation
sudo -u www-data php occ files:scan USER
```

After the scan completes, re-enable the user:

```bash
# Docker installation
sudo docker exec -u www-data -it nextcloudID php occ user:enable USER

# Standalone installation
sudo -u www-data php occ user:enable USER
```

At this point, the **USER** account should contain files as they existed two hours prior to the incident.

---

## Note

This procedure guarantees data recovery on the **Nextcloud server**, but **does not guarantee** recovery on the infected host.

In this case, the hard drive of the affected host was isolated for further analysis, and the operating system and Nextcloud client were reinstalled to quickly and securely recover the user's data.

