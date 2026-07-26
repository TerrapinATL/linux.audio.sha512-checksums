# linux-audio-sha512-checksums
Create checksum files to verify the integrity of audio files and folder contents. 

# FLAC Library SHA-512 Verification

A collection of Bash scripts for generating, verifying, and auditing cryptographic hashes across FLAC music libraries on Linux.

## Overview

This repository contains automated procedures designed for strict data preservation and bit-rot auditing. It establishes a baseline of SHA-512 checksums to ensure library integrity, using a decentralized approach where manifests travel with the data they protect.

## Included Tools & Workflows

* **Stray File Audit:** Identifies unrecognized files before hashing to ensure a clean baseline.
* **Checksum Generation:** Creates `ARTIST.sha512sums.txt` and `ALBUM.sha512sums.txt` using relative pathing.
* **Integrity Verification:** Validates file integrity against existing manifests in strict mode.
* **Library Audit:** Scans for missing manifests or non-compliant checksum filenames.

## Prerequisites

Ensure the following command-line utilities are installed and available in your shell's PATH:

* `sha512sum`
* GNU Core Utilities (`find`, `sort`, `awk`, `grep`, `wc`, `basename`, `dirname`)

### Recommended Workflow

A three part series to clean, verify, and lockdown securely the integrity of an audio file library. 

1. linux.audio.flac-clean-up: https://github.com/TerrapinATL/linux.audio.flac-clean-up

2. linux.audio.sha512-checksums: https://github.com/TerrapinATL/linux.audio.sha512-checksums

3. linux.os.nemo.sha512-shortcut: https://github.com/TerrapinATL/linux.os.nemo.sha512-shortcut

