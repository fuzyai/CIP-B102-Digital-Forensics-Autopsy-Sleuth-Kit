# CIP-B102 Lab 4 – File System Forensics & File Recovery

**Student:** Fuseini Imoru Kantuogaa
**Course:** CIP-B102
**Lab:** Lab 4 – Digital Forensics Analysis

## Overview

This lab demonstrates a structured digital forensics workflow: acquiring
evidence, verifying its integrity, analyzing a FAT12 disk image with both
command-line (The Sleuth Kit) and GUI (Autopsy) tools, recovering deleted
files through multiple methods, keyword searching, and mounting an image
read-only for direct inspection.

## Case Folder Structure

A case directory was created with a clear separation of evidence,
recovered files, reports, screenshots, and notes:

```
~/CIP-B102-Lab4/
├── evidence/
├── recovered/
├── reports/
├── screenshots/
├── autopsy-notes/
└── mount/
```

```bash
mkdir -p ~/CIP-B102-Lab4/{evidence,recovered,reports,screenshots,autopsy-notes}
cd ~/CIP-B102-Lab4
```

## 1. Evidence Acquisition

- Downloaded the evidence disk image (`Ch01InChap01.dd`, a FAT12 image)
  directly into the `evidence/` folder using `wget`
- Confirmed successful download and file size (1.4 MB)

```bash
cd ~/CIP-B102-Lab4/evidence
wget -O Ch01InChap01.dd "<source_url>"
```

## 2. Integrity Verification

- Identified the file type with `file` (confirmed FAT12 boot sector)
- Generated **MD5** and **SHA-256** hashes immediately after acquisition
  and saved them to `reports/` using `tee`, establishing a baseline for
  chain of custody

```bash
file Ch01InChap01.dd
md5sum Ch01InChap01.dd | tee ../reports/image_md5.txt
sha256sum Ch01InChap01.dd | tee ../reports/image_sha256.txt
```

| Hash | Value |
|---|---|
| MD5 | `a117773bcf1fc88ec0ab8e0a349fbbcb` |
| SHA-256 | `3ce8053e4f3d9c8ab98b3aadb2480685efb8e4980d34297b83bd5a09b1a7b122` |

## 3. GUI Analysis with Autopsy

- Launched the **Autopsy Forensic Browser** (v2.24) as a local web
  service (`sudo autopsy`) on `http://localhost:9999/autopsy`
- Created a new case: **CIP-B102-Lab4-Fuseini-Imoru**, with investigator
  name recorded
- Added a host (`host1`) and imported the disk image via **symlink**
  import method (non-destructive — original evidence file untouched)
- Calculated and verified the image's MD5 hash on import, matching the
  command-line hash exactly
- Autopsy auto-detected **Partition 1: FAT12**, mounted as `C:`
- Used the **File Analysis** view to browse **All Deleted Files**,
  identifying 4 deleted entries: `Billing_Letter.doc`, `confirmation.txt`,
  `letter1.txt`, `Regrets.doc` — each with full MAC (Modified/Accessed/
  Created) timestamps and metadata (inode) links
- Ran **Keyword Search** (ASCII + Unicode, case-sensitive) for terms of
  interest:
  - `"letter"` → 1 hit, Sector 241
  - `"password"` → 1 hit, Sector 284 (`password: 342`)

## 4. Command-Line File System Analysis (The Sleuth Kit)

All outputs piped through `tee` to simultaneously display and log results
to `reports/`:

| Command | Output File | Purpose |
|---|---|---|
| `img_stat` | `01_img_stat.txt` | Image type (raw), size, sector size |
| `mmls` | `02_mmls.txt` | Partition table check |
| `fls` | `04_fls_root.txt` | Root directory listing (allocated + deleted) |
| `fls -r` | `05_fls_recursive.txt` | Recursive listing incl. system metadata |
| `fls -r -d` | `06_deleted_files.txt` | Deleted files only |

```bash
cd ~/CIP-B102-Lab4/evidence
img_stat Ch01InChap01.dd | tee ../reports/01_img_stat.txt
mmls Ch01InChap01.dd | tee ../reports/02_mmls.txt
fls Ch01InChap01.dd | tee ../reports/04_fls_root.txt
fls -r Ch01InChap01.dd | tee ../reports/05_fls_recursive.txt
fls -r -d Ch01InChap01.dd | tee ../reports/06_deleted_files.txt
```

### Files found on the image

| Inode | Name | Status |
|---|---|---|
| 5 | `Client Info.mdb` | Allocated |
| 8 | `Billing Letter.doc` | Deleted |
| 11 | `confirmation.txt` | Deleted |
| 13 | `Income.xls` | Allocated |
| 15 | `letter1.txt` | Deleted |
| 17 | `Regrets.doc` | Deleted |

## 5. Targeted File Recovery — `Income.xls`

Located and recovered the allocated file `Income.xls` (inode 13) using
**two independent methods**, then cross-verified the results matched:

**Method A — `icat` (inode-based extraction):**
```bash
fls -r Ch01InChap01.dd | grep -i "INCOME"
icat Ch01InChap01.dd 13 > ../recovered/INCOME_icat.xls
md5sum ../recovered/INCOME_icat.xls | tee ../reports/income_icat_md5.txt
sha256sum ../recovered/INCOME_icat.xls | tee ../reports/income_icat_sha256.txt
file ../recovered/INCOME_icat.xls
```

**Method B — `istat` + `blkcat` (manual sector-level reconstruction):**
```bash
istat Ch01InChap01.dd 13 | tee ../reports/income_istat.txt
# Sectors: 285–311
for blk in 285 286 287 288 289 290 291 292 293 294 295 296 297 298 299 300 \
           301 302 303 304 305 306 307 308 309 310 311; do
  blkcat Ch01InChap01.dd "$blk" >> ../recovered/INCOME_blkcat.xls
done
md5sum ../recovered/INCOME_blkcat.xls | tee ../reports/income_blkcat_md5.txt
sha256sum ../recovered/INCOME_blkcat.xls | tee ../reports/income_blkcat_sha256.txt
```

**Method C — `tsk_recover` (bulk automated recovery):**
```bash
mkdir -p ../recovered/tsk_recover_all
tsk_recover -e Ch01InChap01.dd ../recovered/tsk_recover_all
find ../recovered/tsk_recover_all -iname "*INCOME*" -ls | tee ../reports/income_tsk_recover_location.txt
```

**Verification:** All three recovery methods (`icat`, `blkcat` sector
reconstruction, and `tsk_recover`) produced files with **identical MD5
hashes** (`6a2e65afc5af4fc5f9da2859df134eac`), confirming the file was
recovered completely and accurately regardless of method.

```bash
find ../recovered/tsk_recover_all -iname "*INCOME*" -exec md5sum {} \;
```

## 6. Keyword / String Search Across the Raw Image

Searched the entire raw disk image for a name of interest, independent of
file system structure:

```bash
strings -a Ch01InChap01.dd | grep -i "amelia" | tee ../reports/search_amelia.txt
```

Result: Multiple hits for **"Amelia Phillips"**, matching the document
author metadata found in `Billing_Letter.doc` and `Income.xls`.

## 7. Read-Only Mount & Direct Inspection

Mounted the image directly onto the file system (loopback, read-only) to
inspect it as a live volume without any forensic tool abstraction:

```bash
sudo mkdir -p /mnt/cipb102_lab4
sudo mount -o loop,ro ~/CIP-B102-Lab4/evidence/Ch01InChap01.dd /mnt/cipb102_lab4
ls -la /mnt/cipb102_lab4 | tee ~/CIP-B102-Lab4/reports/mounted_listing.txt
sudo umount /mnt/cipb102_lab4
```

Confirmed only the **allocated** files (`Client Info.mdb`, `Income.xls`)
are visible via a normal mount — deleted files are invisible at this
level and only recoverable through forensic tools, illustrating the
difference between file-system-level and forensic-tool-level visibility.

## Key Tools Used

| Tool | Purpose |
|---|---|
| `wget` | Evidence acquisition |
| `file` | File type identification |
| `md5sum` / `sha256sum` | Hash-based integrity verification |
| **Autopsy** | GUI-based case management, file browsing, keyword search |
| `img_stat`, `mmls` (Sleuth Kit) | Image/partition metadata |
| `fls` | Directory listing (allocated + deleted) |
| `icat` | Inode-based file content extraction |
| `istat` | Inode metadata (timestamps, sector map) |
| `blkcat` | Raw sector-level block extraction |
| `tsk_recover` | Automated bulk file recovery |
| `strings` / `grep` | Raw keyword search across the image |
| `mount` (loop, ro) | Read-only direct file system inspection |

## Report Files Generated

All command output was logged to `~/CIP-B102-Lab4/reports/` for the case
file, including hash values, `fls`/`mmls`/`img_stat` output, deleted file
listings, keyword search results, and recovery verification hashes.

## Forensic Notes

- Evidence was hashed immediately upon acquisition, before any analysis,
  to preserve chain of custody.
- The disk image was imported into Autopsy via **symlink** (not copy or
  move) to avoid duplicating or altering the original evidence file.
- Three independent recovery methods for `Income.xls` produced matching
  hashes, cross-validating the integrity of the recovered data.
- A standard read-only mount only exposes allocated files, reinforcing
  why forensic tools (not the OS file system driver) are required to
  recover and analyze deleted content.
