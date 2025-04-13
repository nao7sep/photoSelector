<!-- 2025-04-13T12:03:31Z -->

# **photoSelector – Specifications Document v0.1**

## **1. Introduction**
**photoSelector** is a C# **console** application to help organize a massive collection of photos and videos spanning many years. It is designed to:
- **Extract timestamps** (from EXIF/QuickTime metadata or file system)
- **Index** files in a **SQLite** database using their **relative paths** as IDs
- Support **safe partial organization** in **batches** or by **day ranges**
- **Verify** backup copies to ensure no corruption
- Handle **duplicates** only at the **filename** level (no hashing)
- Allow **brutal termination** at any point without database corruption

The end goal is to whittle down a large folder structure in safe, incremental steps and avoid confusion across multiple backup drives.

---

## **2. Project Goals & Philosophy**
1. **Timestamp-Focused**: The main metadata used is the **timestamp**. All other metadata fields are unnecessary.
2. **Minimal Duplication**: The app does not attempt robust deduplication — only basic checks for **file length + metadata timestamps**.
3. **Relative Path as Unique ID**: The user’s backup software maintains identical directory trees across drives, so storing absolute paths is unnecessary.
4. **Safe, Atomic Operations**:
   - The database must remain uncorrupted if the user forcibly terminates the app.
   - Moved files are immediately updated in the database so that partial moves do not cause conflicts.
5. **Incremental Organization**: The user can extract a manageable number of files at a time (e.g., a day’s worth, or the first 250 files), move them into a designated directory, and compare them with backups on the spot.

---

## **3. Media & Metadata Requirements**
- **Supported Image Formats**: `.jpg`, `.jpeg`, `.tiff`
  - `.tiff` is presumed to have a corresponding `.jpeg` due to “JPEG + RAW” shooting.
- **Supported Video Formats**: Any file with **QuickTime**-type metadata that `MetadataExtractor` can parse. Otherwise, fallback to the file’s **last write time**.
- **Timestamp Extraction**:
  - Use [`MetadataExtractor`](https://github.com/drewnoakes/metadata-extractor) for both images and videos (QuickTime).
  - Fallback to `FileInfo.LastWriteTimeUtc` if metadata does not exist.

---

## **4. Directory Structure**
1. **Main Directory**: Contains all disorganized media.
2. **Backup Directories**: Mirrors of the main directory on other drives; same relative structure.
3. **Extraction Subdirectory**: `__EXTRACTED__` is used on both the main and backup drives as the location for files once they are “processed.”
   - This name is chosen for visibility and to reduce accidental deletion.
   - The subdirectory **must not** be rescanned or reindexed by the app.

### **4.1 Duplicate Filenames**
If multiple files have the **exact same filename** during extraction:
- The first file goes into `__EXTRACTED__\`
- The second file goes into `__EXTRACTED__\dup1\`, the third into `dup2\`, and so on

---

## **5. Database Design**
- **SQLite** database with journaling/WAL for resilience.
- **Schema** (primary fields):
  - `RelativePath` (Primary Key)
  - `TimestampUtc`
  - `TimestampSource` (`Exif`, `QuickTime`, `FileSystem`)
  - `FileSize`
  - `MediaType` (`Photo` or `Video`)

**No absolute paths** are stored.
When a file is moved to `__EXTRACTED__`, its `RelativePath` is updated accordingly (e.g., from `IMG_1234.JPG` to `__EXTRACTED__\IMG_1234.JPG`).

---

## **6. Configuration (in `appsettings.json`)**
```json
{
  "MainDirectory": "D:\\Media",
  "BackupDirectories": [
    "E:\\Backup",
    "F:\\Archive"
  ]
}
```

- These paths are used for scanning and verifying across drives.

---

## **7. CLI System**
- Uses `System.CommandLine` for:
  - Full autocomplete
  - Interactive prompt
  - Aliases and sugar commands

---

## **8. Core Commands**

### **8.1 `scan`**
1. Reads configuration from `appsettings.json`.
2. Recursively scans the **main directory**, ignoring any `__EXTRACTED__` subfolders.
3. **Adds** new files not in the database (by `RelativePath`), extracting timestamps.
4. **Removes** entries from the database for files that no longer exist.
5. Generates an **HTML stats report** showing:
   - Yearly/monthly totals
   - Photo vs. video counts
   - Total file sizes
   - Possibly largest/smallest days

---

### **8.2 `extract`**
Selects and moves files into `__EXTRACTED__`. Multiple modes:

- **Time-based**
  - `--first-day`, `--last-day`
  - `--first-n-days`, `--last-n-days <n>`
  - `--since 2023-01-01 --before 2023-04-01`
- **Quantity-based**
  - `--first-n-files <n>`
  - `--last-n-files <n>`

The logic ensures that if you request N files but the sum of days is slightly over N (e.g., day 1 + day 2 < N, day 3 pushes total over N), it includes that entire last day.

**Flow**:
1. **Show** a preview of what will be moved (counts, days, etc.).
2. **Confirm** with the user (Y/N).
3. For each file:
   1. Compare **byte-by-byte** (or size + timestamp) with the backup copies that share the same `RelativePath`.
   2. If they match, move the file in the **main directory** to `__EXTRACTED__` (or `__EXTRACTED__\dup1\` if duplicate name), move the same file in backups to `__EXTRACTED__` as well, **update** the database record.
   3. If mismatch → color-coded warning (red/yellow) and beep. The user can choose how to handle that file.
4. This is **incremental** per file, so partial results remain valid on interruption.

---

### **8.3 Sugar Commands**

1. `sfN`
   - **Scan** + extract the **first** `N` days.
2. `slN`
   - **Scan** + extract the **last** `N` days.
3. `sbN`
   - **Scan** + extract both the **first** and **last** `N` days.

**Example**:
```
photoSelector> sf3
[Runs scan]
Found 100 new files, removed 2. HTML stats generated.
Extracting first 3 days...
Proceed? y
...
```

---

## **9. Rescanning & Sync Behavior**
If new files are added or old ones removed from the main directory:
- Running `scan` again will **add** newly found ones to the DB and **remove** DB entries that disappeared on disk.
- Backup directories are similarly excluded from scanning under `__EXTRACTED__`.
- The app ensures the DB remains a faithful mirror of the on-disk structure, ignoring files in `__EXTRACTED__`.

---

## **10. Byte-Wise Verification**
During `extract`, each file is checked against its backup copy (same relative path) by:
- **Byte comparison** (preferred for corruption checks)
  - If mismatch: beep + red/yellow console text warning
  - The user decides whether to skip or proceed.

This ensures data integrity prior to moving.

---

## **11. Moving & DB Updates**
- Once a file is verified:
  1. The file is moved on the **main drive** to `__EXTRACTED__`.
  2. Its backup copies are also moved to `__EXTRACTED__`.
  3. The DB `RelativePath` field is updated immediately for each copy.
- If the app terminates in the middle, only fully processed files are marked as moved.

---

## **12. Performance & Scalability**
- **No hashing** or heavy checks except for the file-by-file verification step when extracting.
- DB operations are minimal and incremental.
- The scanning and indexing, even for ~500k files, should remain performant because only file size + timestamp are retrieved, plus metadata extraction for new files.

---

## **13. Resiliency**
- **SQLite** must use WAL or journaling mode to prevent corruption on abrupt exit.
- The tool is designed so that half-finished operations do not break anything.
- Any subsequent run of `scan` or `extract` will pick up where it left off.

---

## **14. Potential Future Enhancements**
1. **Facial or scene recognition** for auto-tagging.
2. **Grouping** burst shots automatically.
3. **Cloud Integration** (e.g., Google Photos).
4. **Detailed Logs** for each mismatch or skip.
5. **Shell Scripts** for quicker backup drive cleanups.
6. **Web or GUI** for a more visual approach.

---

## **15. Example Workflows**

### **Initial Scan**
```
photoSelector> scan
Indexed 412,323 files. Removed 0.
Report: scan_report_2025-04-13.html
```

### **Extract 250 Files from Oldest**
```
photoSelector> extract --first-n-files 250
Days needed: Jan 1 (100 files), Jan 2 (120 files), Jan 3 (50 files) = 270 total
Proceed? y
# Byte-check each file, move to __EXTRACTED__, update DB
```

### **Sugar Command – `sf3`**
```
photoSelector> sf3
[Under the hood: scan + extract first 3 days]
```

### **Rescan After Adding New Media**
```
photoSelector> scan
Found 93 new files, removed 0.
Report: scan_report_2025-05-01.html
```
