---
title: Image analysis
layout: home
parent: Forensic
nav_order: 2
---

# Image analysis

tags: #analysis #photorec #recovery #foremost #strings

-----

In the obtained image there may be files that were deleted but are still present. We can try to recover potential files using the following tools:

---

### `photorec` (recovers files regardless of the filesystem)

Allows scanning the image and recovering files (.jpg, .pdf, .docx, etc.), even if there is no partition table.

```bash
sudo photorec imgforensics.img
```

> Recovery may take some time.
> It is recommended to choose a destination disk with capacity larger than the image size.

Once files are recovered, they can be filtered by type for a more detailed inspection. For example:

```bash
# To filter .txt files
find . -iname "*.txt" | xargs -I {} cp {} alltxt/

# To filter .png files
find . -iname "*.png" | xargs -I {} cp {} allpng/
```

---

### `foremost` (file-type-based recovery)

In case you want to recover files of a specific type, you can use the following command:

```bash
foremost -t jpeg -o output -i summer.img
```

> Make sure that the `output` directory exists before running the command.

---

### `strings` (extraction of text strings)

There are text strings that may contain useful information, such as passwords, commands, error messages, usernames, among others. We can extract this information using the `strings` command, which can be used with images, memory dumps, executable files, among others.

```bash
strings imgforensics.img
```

We can also use it together with `grep` to improve the search, especially if we have clues about what we are looking for.

```bash
strings imgforensics.img | grep -i "pass"
```
---

### `bulk_extractor` (pattern extraction)

```bash
bulk_extractor -o outputdir -e all -x gzip -x zip imgforensics.img
```
