# FLAC Library SHA-512 Checksum & Verification Guide

01. Introduction

This document is a guide and automated script procedure for generating, verifying, and auditing cryptographic hashes across large audio libraries on Linux systems.

It is intended for users who require strict data preservation, routine auditing for silent data corruption (bit rot), and absolute verification.

Typically, these SHA-512 checksums are verified after data is copied to ensure the copy is a perfect match to the original.

---

02. Requirements

To successfully execute the scripts in this guide, your system must have the following command-line tools installed and available in your shell's PATH:

* sha512sum – Required for calculating and verifying the SHA-512 cryptographic hashes.
* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname) commonly available in Linux environments like Linux Mint.
* Zenity / Nemo Actions (Optional) – For users integrating visual verification into the Nemo file manager via custom action scripts.

-- Note on File Naming Conventions

To prioritize strict data preservation and structural clarity, this workflow utilizes a specific tiered naming convention for its manifests: ARTIST.sha512sums.txt for top-level artist directories and ALBUM.sha512sums.txt for individual album folders. Maintaining this exact convention is critical for the automated auditing scripts and Nemo action configurations to function correctly.

-- Library Structure
```

Parent/
├── Artist1/
│   ├── artist-level files
│   ├── Album1/
│   │   ├── album files
│   │   └── ...
│   └── Album2/
└── Artist2/

```
---

03. Design Philosophy

This guide is built on four core data-management principles:
    
* Decentralized Verification: Hash files are stored locally within the directory of the files they protect, ensuring that data and its verification record travel together during transfers.
* Relative Pathing: Scripts operate within subshells to generate strictly relative paths in the hash files, preventing verification failures if the library is moved to a different drive or mount point.
* Traceable Execution: All operations generate clean, timestamped logs and separate success/failure lists, ensuring total transparency and auditability across massive data sets.
* Built-In Permission Guards: Every script includes an automated permission check and interactive chown prompt to catch and resolve root-locked files or mounts (common when formatting drives on desktop systems for Raspberry Pi environments) before processing begins.

---

04. Workflow Overview

The hashing process follows a comprehensive six-step operational pipeline designed to audit, establish, confirm, and maintain data integrity:

* Step 1: Stray File Audit and Identification — Scan and isolate unrecognized files prior to hashing
* Step 2: Create SHA-512 Checksums for Album Folders
* Step 3: Verification of Album Folders
* Step 4: Create SHA-512 Checksums for Artist Folders
* Step 5: Verification of Artist Folders
* Step 6: Auditing the Library — Scan the file system to identify rogue folders or incorrect checksum naming

---

05. Step 1 – Stray File Audit and Identification

---

-- Purpose

This step scans the Parent folder, all Artist folders, and all Album folders to identify and report any stray, unrecognized, or misplaced files anywhere within the library hierarchy.

Before generating checksum manifests, it is critical to ensure that no unexpected or extraneous files are included in the cryptographic hash baseline. This step flags any files that are not authorized audio files, valid album artwork, or existing checksum manifests.

--- Bash Script Start ---
```bash

#!/usr/bin/env bash

LOGDIR="$HOME/sha512_library_logs"
mkdir -p "$LOGDIR"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R \$USER:\$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by \$USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

{
    echo "Stray file audit started..."

    find "$PWD" -type f \
        ! -iname "*.flac" \
        ! -iname "*.mp3" \
        ! -iname "*.m4a" \
        ! -iname "*.wav" \
        ! -iname "*.ogg" \
        ! -iname "*.opus" \
        ! -iname "*.alac" \
        ! -iname "*.jpg" \
        ! -iname "*.jpeg" \
        ! -iname "*.png" \
        ! -name "ARTIST.sha512sums.txt" \
        ! -name "ALBUM.sha512sums.txt" \
        -print | tee "$LOGDIR/step1_stray_files_found.log"

    count=$(wc -l < "$LOGDIR/step1_stray_files_found.log")

    if [ "$count" -gt 0 ]; then
        echo "WARNING: Found $count stray/unrecognized file(s). Review '$LOGDIR/step1_stray_files_found.log' before generating checksums."
    else
        echo "OK: No unauthorized stray files detected. Library is clean for checksum generation."
    fi

    echo "Stray file audit completed."
} | tee "$LOGDIR/step1_run.log"


```
--- Bash Script End ---

-- Review Results

View the generated reports by running:

--- Bash Script Start ---
```bash

cat "$LOGDIR/step1_run.log"

```
--- Bash Script End ---

---

06. Step 2 – Create SHA-512 Checksums for Album Folders

---

-- Purpose

This step establishes the cryptographic fingerprint for individual files located inside nested Album folders. It creates the standard ALBUM.sha512sums.txt file.

--- Bash Script Start ---
```bash

#!/usr/bin/env bash
LOGDIR="$HOME/sha512_library_logs"
mkdir -p "$LOGDIR"
# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R \$USER:\$USER \"$PWD\""
    echo ""
    exit 1
fi
if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by \$USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------
# Detect folder level by probing actual directory structure below $PWD
# (relative to $PWD, not absolute path depth -- works regardless of mount point)
if find "$PWD" -mindepth 2 -maxdepth 2 -type d -print -quit 2>/dev/null | grep -q .; then
    min=2; max=2   # Parent folder: Parent/Artist/Album
elif find "$PWD" -mindepth 1 -maxdepth 1 -type d -print -quit 2>/dev/null | grep -q .; then
    min=1; max=1   # Artist folder: Artist/Album
else
    min=0; max=0   # Album folder itself
fi
mapfile -d '' dirs < <(find "$PWD" -mindepth $min -maxdepth $max -type d -print0 | LC_ALL=C sort -z)
total=${#dirs[@]}
i=0
for d in "${dirs[@]}"; do
    i=$((i+1))
    
    if [ $min -eq 0 ]; then
        album=$(basename "$d"); artist=$(basename "$(dirname "$d")"); label="$artist-$album"
    elif [ $min -eq 1 ]; then
        album=$(basename "$d"); artist=$(basename "$PWD"); label="$artist-$album"
    else
        album=$(basename "$d"); artist=$(basename "$(dirname "$d")"); label="$artist-$album"
    fi
    (
        cd "$d" || exit 1
        shopt -s nullglob
        files=(*)
        shopt -u nullglob
        target_files=()
        for f in "${files[@]}"; do
            if [[ -f "$f" && "$f" != "ARTIST.sha512sums.txt" && "$f" != "ALBUM.sha512sums.txt" ]]; then
                target_files+=("$f")
            fi
        done
        if [ ${#target_files[@]} -gt 0 ]; then
            sha512sum "${target_files[@]}" > "ALBUM.sha512sums.txt" 2>"$LOGDIR/temp_err.log"
            exit $?
        else
            exit 0
        fi
    )
    rc=$?
    if [ $rc -ne 0 ]; then
        echo "FAIL [$i/$total] $label"
        sed "s/^/[$i\/$total] ERROR: $label :: /" "$LOGDIR/temp_err.log" >> "$LOGDIR/step2_errors.log"
    else
        echo "OK [$i/$total] $label"
    fi
done | tee "$LOGDIR/step2_run.log"
rm -f "$LOGDIR/temp_err.log"

```
--- Bash Script End ---

-- Review Results

View the generated reports by running:

--- Bash Script Start ---
```bash

cat "$LOGDIR/step2_run.log"

```
--- Bash Script End ---

---

07. Step 3 – Verification of Album Folders

---

-- Purpose

This step validates the integrity of the individual album folders, confirming that no audio files have suffered bit rot or silent corruption.

--- Bash Script Start ---
```bash

#!/usr/bin/env bash

LOGDIR="$HOME/sha512_library_logs"
mkdir -p "$LOGDIR"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R \$USER:\$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by \$USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

mapfile -d '' manifests < <(find "$PWD" -type f -name "ALBUM.sha512sums.txt" -print0 | LC_ALL=C sort -z)

total=${#manifests[@]}
i=0

if [ "$total" -eq 0 ]; then
    echo "ALERT: No ALBUM.sha512sums.txt files found under $PWD."
    echo "Nothing to verify — check that you are in the correct directory."
    exit 1
fi

for m in "${manifests[@]}"; do
    i=$((i+1))
    dir_path=$(dirname "$m")
    album_name=$(basename "$dir_path")
    artist_name=$(basename "$(dirname "$dir_path")")
    label="$artist_name-$album_name"

    (
        cd "$dir_path" || exit 1
        sha512sum -c --quiet --strict "ALBUM.sha512sums.txt" 2>&1
    ) > "$LOGDIR/temp_err.log"

    rc=$?

    if [ $rc -ne 0 ]; then
        echo "FAIL [$i/$total] $label"
        sed "s/^/[$i\/$total] ERROR: $label :: /" "$LOGDIR/temp_err.log" >> "$LOGDIR/step3_errors.log"
    else
        echo "OK [$i/$total] $label"
    fi
done | tee "$LOGDIR/step3_run.log"
rm -f "$LOGDIR/temp_err.log"

```
--- Bash Script End ---

-- Review Results

View the generated reports by running:

--- Bash Script Start ---
```bash

cat "$LOGDIR/step3_run.log"

```
--- Bash Script End ---

---

08. Step 4 – Create SHA-512 Checksums for Artist Folders

---

-- Purpose

This step establishes the baseline cryptographic fingerprint for files located directly within the parent Artist directories. It creates the standard ARTIST.sha512sums.txt manifest.

--- Bash Script Start ---
```bash

#!/usr/bin/env bash

LOGDIR="$HOME/sha512_library_logs"
mkdir -p "$LOGDIR"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R \$USER:\$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by \$USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

RUNLOG="$LOGDIR/step4_run.log"
ERRLOG="$LOGDIR/step4_errors.log"

: > "$RUNLOG"
: > "$ERRLOG"

mapfile -d '' artists < <(
    find "$PWD" -mindepth 1 -maxdepth 1 -type d -print0 |
    LC_ALL=C sort -z
)

total=${#artists[@]}
i=0

for artist in "${artists[@]}"; do
    i=$((i+1))
    name=$(basename "$artist")

    errfile=$(mktemp)

    (
        cd "$artist" || exit 1

        : > ARTIST.sha512sums.txt

        mapfile -d '' albums < <(
            find . -mindepth 1 -maxdepth 1 -type d -print0 |
            LC_ALL=C sort -z
        )

        for album in "${albums[@]}"; do
            album_name=$(basename "$album")

            hash=$(
                cd "$album" || exit 1

                find . -type f \
                    ! -name "ALBUM.sha512sums.txt" \
                    -print0 |
                LC_ALL=C sort -z |
                xargs -0 sha512sum |
                sha512sum |
                cut -d" " -f1
            )

            printf "%s  %s\n" "$hash" "$album_name" >> ARTIST.sha512sums.txt
        done

    ) 2>"$errfile"

    rc=$?

    if [ $rc -ne 0 ]; then
        echo "FAIL [$i/$total] $name"
        sed "s/^/[$i\/$total] ERROR: $name :: /" "$errfile" >> "$ERRLOG"
    else
        echo "OK [$i/$total] $name"
    fi

    cat "$errfile" >> "$RUNLOG"
    rm -f "$errfile"

done | tee -a "$RUNLOG"

```

--- Bash Script End ---

-- Review Results

View the generated reports by running:

--- Bash Script Start ---

```bash

cat "$LOGDIR/step4_run.log"

```
--- Bash Script End ---

---

09. Step 5 – Verification of Artist Folders

---

-- Purpose

This step validates the integrity of the top-level artist hashes against the current state of the files to detect missing or corrupted data.

--- Bash Script Start ---
```bash

#!/usr/bin/env bash

LOGDIR="$HOME/sha512_library_logs"
mkdir -p "$LOGDIR"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R \$USER:\$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by \$USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

mapfile -d '' artist_dirs < <(find "$PWD" -mindepth 1 -maxdepth 1 -type d -print0 | LC_ALL=C sort -z)
mapfile -d '' manifests < <(find "$PWD" -type f -name "ARTIST.sha512sums.txt" -print0 | LC_ALL=C sort -z)

total_artists=${#artist_dirs[@]}
total=${#manifests[@]}
i=0
missing=0
missing_names=()

: > "$LOGDIR/step5_missing.log"

for a in "${artist_dirs[@]}"; do
    artist_name=$(basename "$a")
    if [ ! -f "$a/ARTIST.sha512sums.txt" ]; then
        missing=$((missing+1))
        missing_names+=("$artist_name")
        echo "MISSING $artist_name :: no ARTIST.sha512sums.txt in $a" >> "$LOGDIR/step5_missing.log"
    fi
done

if [ "$total" -eq 0 ]; then
    echo "ALERT: no ARTIST.sha512sums.txt files found anywhere under $PWD ($total_artists artist folders scanned, 0 have manifests)."
    echo "Nothing to verify — check you're in the right directory or that checksums were generated here."
    exit 1
fi

if [ "$missing" -gt 0 ]; then
    echo "ALERT: $missing of $total_artists artist folder(s) have no ARTIST.sha512sums.txt:"
    for name in "${missing_names[@]}"; do
        echo "  - $name"
    done
    echo "(Full detail in $LOGDIR/step5_missing.log)"
    echo ""
fi

for m in "${manifests[@]}"; do
    i=$((i+1))
    dir_path=$(dirname "$m")
    artist=$(basename "$dir_path")
    (
        cd "$dir_path" || exit 1
        sha512sum -c --quiet --strict "ARTIST.sha512sums.txt" 2>&1
    ) > "$LOGDIR/temp_err.log"
    rc=$?
    if [ $rc -ne 0 ]; then
        echo "FAIL [$i/$total] $artist"
        sed "s/^/[$i\/$total] ERROR: $artist :: /" "$LOGDIR/temp_err.log" >> "$LOGDIR/step5_errors.log"
    else
        echo "OK [$i/$total] $artist"
    fi
done | tee "$LOGDIR/step5_run.log"
rm -f "$LOGDIR/temp_err.log"

if [ "$missing" -gt 0 ]; then
    echo ""
    echo "Reminder: $missing artist folder(s) were skipped (no manifest):"
    for name in "${missing_names[@]}"; do
        echo "  - $name"
    done
fi

```
--- Bash Script End ---

-- Review Results

View the generated reports by running:

--- Bash Script Start ---
```bash

cat "$LOGDIR/step5_run.log"

```
--- Bash Script End ---

---

10. Step 6 – Auditing the Library

---

-- Purpose

This step audits the structural integrity of the library itself. It scans for any directories that are missing their required manifest files, or files that violate the strict naming convention, ensuring nothing escapes the verification safety net.

--- Bash Script Start ---
```bash

#!/usr/bin/env bash

LOGDIR="$HOME/sha512_library_logs"
mkdir -p "$LOGDIR"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R \$USER:\$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by \$USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

echo "Auditing library for rogue files and missing manifests..." | tee "$LOGDIR/step6_run.log"

find "$PWD" -type f -iname "*.sha512*" \
    ! -name "ARTIST.sha512sums.txt" \
    ! -name "ALBUM.sha512sums.txt" \
    -print | tee -a "$LOGDIR/step6_rogue_names.log"

mapfile -d '' dirs < <(find "$PWD" -type d -print0 | LC_ALL=C sort -z)

for d in "${dirs[@]}"; do
    if [[ ! -f "$d/ARTIST.sha512sums.txt" && ! -f "$d/ALBUM.sha512sums.txt" ]]; then
        shopt -s nullglob
        files=("$d"/*)
        shopt -u nullglob

        has_files=0
        for f in "${files[@]}"; do
            if [[ -f "$f" ]]; then
                has_files=1
                break
            fi
        done

        if [ $has_files -eq 1 ]; then
            echo "WARNING: Directory contains files but no manifest: $d" | tee -a "$LOGDIR/step6_missing_manifests.log"
        fi
    fi
done

echo "Audit complete. Check step6 logs for detailed anomalies." | tee -a "$LOGDIR/step6_run.log"

```
--- Bash Script End ---

-- Review Results

View the generated reports by running:

--- Bash Script Start ---
```bash

cat "$LOGDIR/step6_run.log"

```
--- Bash Script End ---

--

11. Troubleshooting & Reference Information

1. Checksum Mismatch ("computed checksum did NOT match")
If an audit reveals a mismatch, the file has been altered since the hash was generated. Do not regenerate the hash to "fix" the error, as you will be validating corrupted data. Delete the corrupted file and replace it from a known-good backup.

2. Missing File Errors ("FAILED open or read")
This occurs if a file tracked in the manifest has been renamed or deleted. You must delete the existing manifest in that directory and re-run the Generation scripts to establish a new baseline.
