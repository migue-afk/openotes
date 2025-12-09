---
title: Server migration
parent: Infraestructure Resilience
layout: home
---

# Migration of Docker Environment Between Servers with LVM Volume Management

## Summary

This repository documents the experience of migrating a Linux server that used LVM, Restic + MinIO for backups, and multiple Docker containers (Nextcloud, Gitea, Grafana, Immich, etc.).

It includes real-world issues encountered and their solutions. The migration was initially complicated by the lack of free space in the **Volume Group (VG)** of **LVM** on the source server, which prevented the creation of _snapshots_.

This problem was overcome by **expanding LVM** with the addition of a new physical disk partitioned for that purpose, allowing the creation of a _snapshot_ and the subsequent _merge_ to return to the initial state in case of serious errors.  
Later, the Docker images were migrated and the volume data was restored.

---

## Contents

- [Phase 1: LVM Expansion and Snapshot Creation](#phase-1-lvm-expansion-and-snapshot-creation)
    
- [Phase 2: Backup of Docker Containers and Volumes](#phase-2-backup-of-docker-containers-and-volumes)
    
- [Phase 3: Reconstruction and Fine-Tuning on the Destination Server](#phase-3-reconstruction-and-fine-tuning-on-the-destination-server)
    

## Phase 1: LVM Expansion and Snapshot Creation

The original server used **LVM** to manage its main disk (`/dev/sda5`), which offered flexibility but presented an immediate problem: the **Volume Group (`debianhp-vg`)** did not have enough free space for a **Logical Volume (LV) Snapshot**.

### 1.1 Initial LVM Diagnostics

The command `sudo vgs` shows that the free space is not sufficient, which complicates the creation of a $1\text{ GB}$ LV Snapshot (256 extents required, only 36 available).

```bash
sudo vgs
VG           #PV #LV #SN Attr   VSize     VFree
debianhp-vg    1   2   0 wz--n- <231.93g   36.00m
```

Attempting to create a snapshot:

```bash
sudo lvcreate -s -L 1G -n linux01-snap /dev/debianhp-vg/root
Volume group "debianhp-vg" has insufficient free space
```

### 1.2 Solution: Adding a New Physical Volume (PV)

The most convenient strategy was to add a new partitioned disk (`/dev/sdb3` with $100\text{ GB}$) and add it to the existing **Volume Group**.

1. **Initialize the new disk as a Physical Volume (PV):**
    
    ```bash
    sudo pvcreate /dev/sdb3
    ```
    
2. **Extend the Volume Group (VG):**
    
    ```bash
    sudo vgextend debianhp-vg /dev/sdb3
    ```
    
3. **Verify free space:**
    

```bash
sudo vgs
VG           #PV #LV #SN Attr   VSize     VFree
debianhp-vg    2   2   0 wz--n- <331.93g  100.03g
```

The VG now has **100 GB** of free space.

### 1.3 Snapshot Creation and Merge

With available space, a _snapshot_ of the root LV (`/dev/mapper/debianhp--vg-root`) was created and later merged.

1. **Create the root LV snapshot:**
    
    ```bash
    sudo lvcreate -s -L 10G -n linux01-snap /dev/mapper/debianhp--vg-root
    #  Logical volume "linux01-snap" created.
    ```
    
    This captured the filesystem state, including Docker data and volumes, at a specific point in time. Any changes after this moment can be discarded via a merge, allowing you to restore an optimal working environment quickly.
    
2. **Snapshot Merge:**
    
    Since the original LV (`/`) is the root mount point, the merge is not immediate — it is scheduled for the next reboot. Otherwise, it could cause irreparable system damage.
    
    ```bash
    sudo lvconvert --merge /dev/debianhp-vg/linux01-snap
    ```
    
    You can verify it with:
    
    ```bash
    sudo lvdisplay /dev/debianhp-vg/linux01-snap
    ```
    
    This ensures that the filesystem state captured in the _snapshot_ becomes the new state of the original LV after reboot, maintaining data integrity at the desired point.
    
    Note: after performing the merge, the _snapshot_ is deleted, so it is advisable to create a new one or store it safely.
    

---

## Phase 2: Backup of Docker Containers and Volumes

The migration strategy was based on backing up images and restoring persistent data (volumes).

### 2.1 Backup of Docker Images

To ensure continuity, exact images of the running containers were backed up (before manipulating volumes or images, all services were stopped).

**Identified improvement:** The images were saved without tags, complicating identification during restoration. It is recommended to tag each `.tar` with the image name and its _tag_ or _ID_.

List containers for more information:

```bash
sudo docker ps -a --format "table {{.Names}}\t{{.Size}}"
```

You can also obtain the internal version (useful when no tags exist):

```bash
docker exec <container> grafana-server -v
```

**Individual backup command:**

```bash
sudo docker save -o /path/to/grafana.tar <IMAGE_ID_OR_NAME:TAG>
# Example used: sudo docker save -o .migration/images/grafana.tar 2e2cb40c55b8
```

### 2.2 Restoration of Volume Data

Persistent data was restored from a **MinIO/S3** repository using **Restic**.  
This volume backup runs every 4 hours, so if more recent volumes are needed, run `restic -r /repo backup`.

1. **Export MinIO credentials:**
    
    ```bash
    export AWS_ACCESS_KEY_ID=<KEY>
    export AWS_SECRET_ACCESS_KEY=<SECRET>
    ```
    
2. **Repository restoration:**
    
    ```bash
    restic -r s3:http://minioserver/debianserver restore 72f4f4c5:/dockervolumes --target .migration/volumes
    ```
    

---

## Phase 3: Reconstruction and Fine Tuning on the Destination Server

On the destination server, the main task was loading the backed-up images, recreating the **Docker Compose** structure, and adjusting restored volume permissions.

### 3.1 Loading Images and Retagging

Since images were backed up without clear tags, they had to be loaded and manually tagged with the image _ID_ to match the `docker-compose.yml`.

1. **Load the image (initially untagged):**
    
    ```bash
        sudo docker load -i /path/to/image.tar
    ```
    
    The output would be:
    
    ```bash
	    REPOSITORY      TAG        IMAGE ID  
        <none>          <none>     1a072cac928d  
    ```

2. **Retag the image:**
    
    ```bash
        sudo docker tag <UNTAGGED_IMAGE_ID> <REPOSITORY:TAG>
        # Example: sudo docker tag 1a072cac928d nextcloud:latest
    ```
    

### 3.2 Volume Migration and Permission Adjustment

This was the most critical step for service functionality. Volumes must have the correct **owner (UID)** and **group (GID)** so the process inside the container can access them.  
To identify which service a volume belongs to on the original server:

```bash
 docker ps -a --filter volume=4d4abcebbb...
 ```

Below are some examples:

#### **Gitea (Bind Mounts):**

Migration was trivial using **Bind Mounts**. The directories were copied and the service started with `docker compose up -d`.

#### **Grafana (Managed Volume):**

- **Volume to migrate:** `grafana_grafana_storage`
    
- **Required owner (inside container):** `472`
    
- **Action:** The restored volume was copied into Docker's directory (`/var/lib/docker/volumes/`) and permissions on the `_data` directory were corrected.
    
```bash
# Copy the volume to the Docker directory
cp -r grafana_grafana_storage /var/lib/docker/volumes/
    
# Adjust owner (472 is the default UID of 'grafana' in the image)
sudo chown -R 472:root /var/lib/docker/volumes/grafana_grafana_storage/_data
```
    

Then `docker compose up -d` was executed. At this point you must have the recreated or original `docker-compose.yml`. Usually they don't differ much and depend on whether Docker-managed volumes or bind mounts were used.

#### **Nextcloud (Volumes and Bind Mounts):**

Nextcloud required migration of several volumes (`nextcloudv3_db`, Redis volumes, etc.) and a **Bind Mount** for (`./html`).

- **Required owner (inside container):** `www-data` (UID: 33 in most Debian/Alpine cases)
    
- **Action:** Permissions were corrected both for managed volumes and the bind mount.
    
1. **Bind mount permission fix (`./html`):**
        
    ```bash
    sudo chown -R www-data:www-data html
    ```
2. **Volume permission fix (`nextcloudv3_db`, etc.):**
        
    ```bash
    sudo chown -R www-data:www-data /var/lib/docker/volumes/nextcloudv3_db/_data
    ```
        
3. **Edit `config.php`:**  


    The file `html/config/config.php` must be updated to remove references to the previous environment ![config.php_https](files/config.php_https) (e.g., HTTPS with Caddy if the new server uses HTTP/Nginx).  
    Otherwise the container fails to start: 
        
    > `Error: Configuration was not read or initialized correctly, not overwriting /var/www/html/config/config.php`
        

---

## Conclusion and Lessons Learned

The migration was completed successfully. For a smooth migration, consider:

- **LVM:** A **Volume Group** can be expanded live, and snapshots can guarantee data integrity during critical operations, even when free space is initially insufficient.
    
- **Docker:** Proper **tagging** (`-o <image_name>.tar`) is crucial during `docker save` to simplify restoration.
    
- **Persistent Volumes:** The fundamental rule is to **know the `UID`/`GID`** used inside the container and ensure restored volumes have the correct ownership (`chown`).

