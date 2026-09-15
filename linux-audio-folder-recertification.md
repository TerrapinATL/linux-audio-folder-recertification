### linux-audio-folder-recertification

**Version: v4** — Current version; supersedes v3.

Change log and version history are maintained separately:
[linux-audio-folder-recertification-changelog.md](linux-audio-folder-recertification-changelog.md)

---

01. Introduction

---

These procedures are based on a FOLDER LEVEL approach. Each bash script is designed to be executed manually from the appropriate target folder and is not intended to operate recursively.

-- What This Guide Is For

This is the QUICK path. It lets you add or re-certify an album or two without running the full library-wide guides. Use these commands after making intentional, small changes such as:

- Replacing a bad track
- Updating album artwork
- Correcting tags
- Adding or removing tracks

-- What This Guide Bypasses

Because this is a quick, folder-scoped path, it deliberately bypasses some of the safety checks built into the full library workflows. In particular, it does NOT perform the standalone stray-file audit or the comprehensive library-wide verification pass. It relies on the fact that a single album is small enough to inspect by eye, and it validates only the folder you point it at.

For a fully audited, library-wide certification of your collection, run the full guides instead. Use this guide when you only need to bring a handful of albums back under certification quickly.

All scripts direct their tracking data to `$HOME/.logs/linux-audio-folder-recertification`. See the Log File Reference in Section 10 for details.

-- Anticipated folder structure for this process:

```
Artist/
└── YYYY Album Name/
    ├── 01 Track One.flac
    ├── 02 Track Two.mp3
    └── 03 Track Three.m4a
```

This guide works with any file naming, but the suite's `NN Title` convention (zero-padded track number, no dash) keeps the checksum manifests and moOde's display consistent with the other guides.

-- Supported Audio Formats

- flac
- mp3
- m4a / mp4
- ogg
- opus
- wav
- aiff / aif
- aac

-- Requirements

To successfully execute the scripts in this guide, your system must have the following command-line tools installed and available in your shell's PATH:

* flac – Required for FLAC integrity testing (Step 1).

* ffmpeg / ffprobe – Required for multi-format integrity testing (Steps 1 and 4A) and MP4 container sanitizing (Step 2B).

* loudgain – Required for calculating and writing ReplayGain metadata (Steps 2A and 2B) across FLAC, MP3, OGG, Opus, WAV, AIFF, M4A, and MP4.

* sha512sum – Required for album and artist checksum creation and verification (Steps 3 and 5).

* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname, xargs, mktemp, tee).

---

02. Album Folder

---

Run these commands from the album folder containing supported audio files.

---

03. Step 1: Verify Audio Files

---

-- Purpose

Run from the album folder. Tests the decodability of every supported audio file without modifying anything, and reports a per-file pass/fail tally.

-- Logging

This step writes `step1-verify-audio.log` to `$HOME/.logs/linux-audio-folder-recertification`.

--- Bash Script Step 1 Start ---
```bash

#!/usr/bin/env bash
# Step 1: Verify Audio Files (Run from Album folder)

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"

# AUTOPURGE: remove all logs from any previous recertification run.
# This runs at the START of the workflow, so the previous run's logs remain
# on disk for review until the next run replaces them.
rm -rf "$LOG_ROOT"
mkdir -p "$LOG_ROOT"
LOG_FILE="$LOG_ROOT/step1-verify-audio.log"
: > "$LOG_FILE"

verify_audio() {
    local passed=0 failed=0 total=0

    echo "Scanning audio files in $(pwd)..." | tee -a "$LOG_FILE"

    while IFS= read -r -d '' file; do
        ((total++))

        case "${file,,}" in
            *.flac)
                if flac -s -t -- "$file" >/dev/null 2>&1; then
                    echo "[OK]     $(basename "$file")" | tee -a "$LOG_FILE"
                    ((passed++))
                else
                    echo "[FAILED] $(basename "$file")" | tee -a "$LOG_FILE"
                    ((failed++))
                fi
                ;;

            *.mp3|*.m4a|*.wav|*.ogg|*.aac|*.opus|*.aiff|*.aif)
                if ffmpeg -nostdin -v error -hide_banner -nostats \
                    -i "$file" -f null - >/dev/null 2>&1; then
                    echo "[OK]     $(basename "$file")" | tee -a "$LOG_FILE"
                    ((passed++))
                else
                    echo "[FAILED] $(basename "$file")" | tee -a "$LOG_FILE"
                    ((failed++))
                fi
                ;;
        esac

    done < <(
        find . -maxdepth 1 -type f \( \
            -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
            -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
            -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif"  \
        \) -print0 | LC_ALL=C sort -z -V
    )

    {
        echo "----------------------------------------"
        echo "SUMMARY: $total file(s) scanned, $passed passed, $failed failed."
    } | tee -a "$LOG_FILE"

    (( failed == 0 ))
}

verify_audio

```

--- Bash Script Step 1 End ---

---

04. Step 2: Apply Gain

---

-- Purpose

Applies ReplayGain volume metadata. Handles FLAC, MP3, OGG, OPUS, WAV, and AIFF (M4A/MP4 is handled separately in Step 2B).

-- Step 2A: Apply Album & Track Gain (FLAC, MP3, OGG, OPUS, etc.)

```bash

#!/usr/bin/env bash
# Step 2A: Apply Album & Track Gain (FLAC, MP3, OGG, OPUS, WAV, AIFF)

set -o pipefail

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"
mkdir -p "$LOG_ROOT"
LOG_FILE="$LOG_ROOT/step2a-album-track-gain.log"

# CLEANUP: reset this step's log from any previous run
: > "$LOG_FILE"

shopt -s nullglob nocaseglob
files=( *.flac *.mp3 *.ogg *.opus *.wav *.aiff *.aif )
shopt -u nullglob nocaseglob

[ ${#files[@]} -eq 0 ] && exit 0

if loudgain -k -s e -a -- "${files[@]}" 2>&1 | tee -a "$LOG_FILE"; then
    echo "SUMMARY: ReplayGain applied to ${#files[@]} file(s) (album + track)." | tee -a "$LOG_FILE"
else
    echo "ERROR: Loudgain failed in $(pwd). Details saved to $LOG_FILE" >&2
    exit 1
fi

```

-- Step 2B: Apply Track Gain Only (M4A / MP4 Workaround)

Note: M4A/MP4 Files: `loudgain` suffers from an upstream C-level memory corruption bug (segmentation fault) when writing album-wide metadata into MP4/M4A atom headers. To prevent crashes while keeping standard EBU R128 tags (-s e), M4A containers are intentionally processed using Track Gain only (by omitting the -a flag).

`loudgain` relies on a legacy MP4 atom parser that frequently encounters memory corruption bugs (segmentation faults) when reading or writing tags on non-standard M4A containers.

To guarantee stability:
   1. Files are pre-processed through `ffmpeg` using stream copy (`-c copy -movflags +faststart`) to rewrite and sanitize the MP4 container structure without re-encoding the audio.
   2. `loudgain` is run in **Track Gain mode** (omitting `-a`) to prevent album-level tag aggregation crashes while preserving standard EBU R128 values (`-s e`).

```bash

#!/usr/bin/env bash
# Step 2B: Apply Track Gain to M4A/MP4 Files (Container Repair & Workaround)

set -o pipefail

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"
mkdir -p "$LOG_ROOT"
LOG_FILE="$LOG_ROOT/step2b-track-gain-m4a.log"

# CLEANUP: reset this step's log from any previous run
: > "$LOG_FILE"

shopt -s nullglob nocaseglob
files=( *.m4a *.mp4 )
shopt -u nullglob nocaseglob

[ ${#files[@]} -eq 0 ] && exit 0

echo "Sanitizing MP4 containers..." | tee -a "$LOG_FILE"
for f in "${files[@]}"; do
    tmp=$(mktemp "$LOG_ROOT/step2b-fixed.XXXXXX.${f##*.}")
    if ffmpeg -nostdin -v error -i "$f" -map 0 -map_metadata 0 -c copy -movflags +faststart "$tmp" 2>>"$LOG_FILE"; then
        mv "$tmp" "$f"
    else
        echo "Warning: FFmpeg container fix failed for $f" | tee -a "$LOG_FILE"
        rm -f "$tmp"
    fi
done

if loudgain -k -s e -L -- "${files[@]}" 2>&1 | tee -a "$LOG_FILE"; then
    echo "SUMMARY: Track gain applied to ${#files[@]} M4A/MP4 file(s)." | tee -a "$LOG_FILE"
else
    echo "ERROR: Loudgain failed in $(pwd). Details saved to $LOG_FILE" >&2
    exit 1
fi

```

---

05. Step 3: Create and Verify Album Checksum

---

-- Purpose

Run from the album folder. Creates a checksum file for the current folder, verifies that the generated hashes match immediately. Non-recursive: operates only on the current album folder.

-- Logging

This step writes `step3-album-checksum.log` to `$HOME/.logs/linux-audio-folder-recertification`.

--- Bash Script Step 3 Start ---
```bash

#!/usr/bin/env bash
# Step 3: Create and Verify Album Checksum

set -o pipefail

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"
mkdir -p "$LOG_ROOT"
LOG_FILE="$LOG_ROOT/step3-album-checksum.log"

# CLEANUP: reset this step's log from any previous run
: > "$LOG_FILE"

CHECKSUM="ALBUM.sha512sums.txt"

find . -maxdepth 1 -type f ! -name "$CHECKSUM" \( \
    -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
    -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
    -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif" \
\) -print0 |
LC_ALL=C sort -z |
xargs -0 -r sha512sum > "$CHECKSUM"

if [ -s "$CHECKSUM" ]; then
    echo "CREATED: $CHECKSUM ($(wc -l < "$CHECKSUM") files)" | tee -a "$LOG_FILE"

    if sha512sum -c "$CHECKSUM" 2>&1 | tee -a "$LOG_FILE" | awk -F': ' '{printf "%-6s %s\n", $2, $1}'; then
        echo "SUMMARY: Album checksum verified OK." | tee -a "$LOG_FILE"
    else
        echo "FAILED: ALBUM CHECKSUM ERROR" | tee -a "$LOG_FILE"
        exit 1
    fi
else
    echo "FAILED: NO SUPPORTED AUDIO FILES FOUND" | tee -a "$LOG_FILE"
    exit 1
fi

```

--- Bash Script Step 3 End ---

---

06. Artist Folder

---

Run these commands from the Artist directory.

---

07. Step 4: Recursive Artist Audio Validation

---

-- Purpose

Recursively inspect only supported audio files beneath that folder. Verify audio integrity before any Artist-level checksum is created. Does not modify files. Produces a clear pass/fail report.

-- Logging

This step writes `step4a-artist-audio-*.log` files (validation, passed, failed, errors) to `$HOME/.logs/linux-audio-folder-recertification`.

--- Bash Script Step 4A Start ---
```bash

#!/usr/bin/env bash
# Step 4A - Recursive Artist Audio Validation (Run from Artist folder)

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"
mkdir -p "$LOG_ROOT"

LOG="$LOG_ROOT/step4a-artist-audio-validation.log"
PASSED="$LOG_ROOT/step4a-artist-audio-passed.log"
FAILED="$LOG_ROOT/step4a-artist-audio-failed.log"
ERRORS="$LOG_ROOT/step4a-artist-audio-errors.log"

: > "$LOG"
: > "$PASSED"
: > "$FAILED"
: > "$ERRORS"

total=0; passed=0; failed=0

echo "Starting recursive validation for: $(pwd)" | tee -a "$LOG"

while IFS= read -r -d '' file; do
    total=$((total + 1))
    clean_file=$(printf '%s' "$file" | tr -d '\r')

    if ffmpeg -nostdin -v error -i "$clean_file" -f null - 2>>"$ERRORS"; then
        passed=$((passed + 1))
        echo "OK: $clean_file" >> "$PASSED"
        echo "OK: $clean_file" >> "$LOG"
    else
        failed=$((failed + 1))
        echo "FAILED: $clean_file" >> "$FAILED"
        echo "FAILED: $clean_file" | tee -a "$LOG"
    fi
done < <(
    find . -type f ! -ipath '*/Ignore/*' \( \
        -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
        -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
        -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif"  \
    \) -print0
)

echo "----------------------------------------" | tee -a "$LOG"
echo "SUMMARY: $total file(s) scanned, $passed passed, $failed failed." | tee -a "$LOG"

if (( failed > 0 )); then
    echo "Validation failed for $failed file(s). Review logs in: $LOG_ROOT" >&2
    exit 1
fi

```

--- Bash Script Step 4A End ---

-- Separate Results

Step 4A generates four distinct log files inside $HOME/.logs/linux-audio-folder-recertification/:

    step4a-artist-audio-validation.log

        Complete validation report summary.

    step4a-artist-audio-passed.log

        List of all audio files that passed integrity checks.

    step4a-artist-audio-failed.log

        List of audio files that failed integrity checks.

    step4a-artist-audio-errors.log

        Raw ffmpeg decoder output recorded during failure inspection.

-- Review Results

```bash

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"

# View raw FFmpeg error output
cat "$LOG_ROOT/step4a-artist-audio-errors.log"

# View overall summary report
cat "$LOG_ROOT/step4a-artist-audio-validation.log"

# View list of passed files
cat "$LOG_ROOT/step4a-artist-audio-passed.log"

# View list of failed files
cat "$LOG_ROOT/step4a-artist-audio-failed.log"

```

---

08. Step 5: Create and Verify Artist Checksum

---

-- Purpose

Run from the artist folder only. Each immediate child folder is treated as an album folder. Requires completed `ALBUM.sha512sums.txt` files. Does not recursively inspect audio files.

-- Logging

This step writes `step5-artist-checksum.log` to `$HOME/.logs/linux-audio-folder-recertification`.

--- Bash Script Step 5 Start ---
```bash

#!/usr/bin/env bash
# Step 5 - Create and Verify Artist Checksum (Run from Artist folder)

set -o pipefail

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"
mkdir -p "$LOG_ROOT"
LOG_FILE="$LOG_ROOT/step5-artist-checksum.log"

# CLEANUP: reset this step's log from any previous run
: > "$LOG_FILE"

CHECKSUM="ARTIST.sha512sums.txt"
TEMP_CHECKSUM="${CHECKSUM}.tmp"
FAILED=0

: > "$TEMP_CHECKSUM"

while IFS= read -r -d '' album; do
    name=$(basename "$album")

    if [ ! -f "$album/ALBUM.sha512sums.txt" ]; then
        echo "FAILED: MISSING ALBUM CHECKSUM: $name" | tee -a "$LOG_FILE"
        FAILED=1
        continue
    fi

    hash=$(
        cd "$album" &&
        find . -maxdepth 1 -type f \( \
            -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
            -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
            -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif" \
        \) -print0 |
        LC_ALL=C sort -z |
        xargs -0 -r sha512sum |
        sha512sum |
        cut -d' ' -f1
    )

    printf "%s  %s\n" "$hash" "$name" >> "$TEMP_CHECKSUM"
done < <(
    find . -mindepth 1 -maxdepth 1 -type d ! -ipath '*/Ignore/*' -print0 |
    LC_ALL=C sort -z
)

if [ "$FAILED" -ne 0 ] || [ ! -s "$TEMP_CHECKSUM" ]; then
    rm -f "$TEMP_CHECKSUM"
    echo "FAILED: ARTIST CHECKSUM CREATION ABORTED" | tee -a "$LOG_FILE"
    exit 1
fi

mv "$TEMP_CHECKSUM" "$CHECKSUM"
echo "CREATED: $CHECKSUM ($(wc -l < "$CHECKSUM") albums)" | tee -a "$LOG_FILE"

# Verification Phase
MISMATCH_COUNT=0
while read -r stored_hash album; do
    actual_hash=$(
        cd "$album" 2>/dev/null &&
        find . -maxdepth 1 -type f \( \
            -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
            -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
            -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif" \
        \) -print0 |
        LC_ALL=C sort -z |
        xargs -0 -r sha512sum |
        sha512sum |
        cut -d' ' -f1
    )

    if [ "$stored_hash" = "$actual_hash" ]; then
        printf "%-10s %s\n" "OK" "$album" | tee -a "$LOG_FILE"
    else
        printf "%-10s %s\n" "MISMATCH" "$album" | tee -a "$LOG_FILE"
        ((MISMATCH_COUNT++))
    fi
done < "$CHECKSUM"

echo "SUMMARY: $MISMATCH_COUNT album mismatch(es) found." | tee -a "$LOG_FILE"

# Logs are retained after this step so Steps 1-4A results can be reviewed.
# They are purged automatically at the start of the next recertification
# run (Step 1), or manually via Step 6.

if [ "$MISMATCH_COUNT" -ne 0 ]; then
    echo "----------------------------------------" >&2
    echo "ARTIST CHECKSUM ERRORS DETECTED. Logs retained in $LOG_ROOT" >&2
    exit 1
fi

```

--- Bash Script Step 5 End ---

---

09. Step 6: Log Cleanup

---

-- Purpose

Finds existing log files, waits for user confirmation, removes them, and verifies cleanup.

--- Bash Script Step 6 Start ---
```bash

#!/usr/bin/env bash
# Step 6: Log Cleanup

LOG_ROOT="$HOME/.logs/linux-audio-folder-recertification"

echo "Log directory:"
echo "$LOG_ROOT"
echo

echo "Step 6A: Finding Log Files"
echo "----------------------------------------"

LOG_COUNT=$(find "$LOG_ROOT" -type f -name "*.log" | wc -l)

if [ "$LOG_COUNT" -eq 0 ]; then
    echo "No log files found."
    exit 0
fi

find "$LOG_ROOT" -type f -name "*.log"

echo
echo "----------------------------------------"
echo "Found $LOG_COUNT log file(s)."

read -rp "Continue and delete these log files? (y/N): " CONFIRM

if [[ ! "$CONFIRM" =~ ^[Yy]$ ]]; then
    echo "Cleanup cancelled. No files were deleted."
    exit 0
fi

echo
echo "Step 6B: Deleting Log Files"
echo "----------------------------------------"

find "$LOG_ROOT" -type f -name "*.log" -delete

echo "Log deletion complete."

echo
echo "Step 6C: Verify Log Deletion"
echo "----------------------------------------"

REMAINING=$(find "$LOG_ROOT" -type f -name "*.log" | wc -l)

if [ "$REMAINING" -eq 0 ]; then
    echo "SUMMARY: All $LOG_COUNT log file(s) removed successfully."
else
    echo "WARNING: $REMAINING log file(s) remain."
    find "$LOG_ROOT" -type f -name "*.log"
fi

echo
echo "Log cleanup finished."

```

--- Bash Script Step 6 End ---

---

10. Troubleshooting & Reference Information

---

-- Common Issues and Fixes

1. Loudgain and M4A/MP4 Files

WAV and AIFF are supported by loudgain and are handled by Step 2A along with FLAC, MP3, OGG, and OPUS. M4A/MP4 must use Step 2B (track gain only), because of the loudgain MP4 atom bug described there.

2. "MISSING ALBUM CHECKSUM" During Artist Checksum

Step 5 requires a completed `ALBUM.sha512sums.txt` in every album folder before the artist checksum can be created. Run Step 3 in any album folder that is missing one, then re-run Step 5.

3. Permission Denied Errors

If tools like sha512sum, ffmpeg, or loudgain fail to write to a directory, it usually indicates a file ownership or permission issue. To grant your current user full access:

--- Bash Script Start ---
```bash

sudo chown -R $USER:$USER /path/to/your/music/library
chmod -R u+rw /path/to/your/music/library

```
--- Bash Script End ---

-- Log File Reference & Understanding Your Log Files

Throughout this workflow, all scripts direct their tracking data to a dedicated log directory created in your home directory ($HOME/.logs/linux-audio-folder-recertification), regardless of which library folder you're working in. This ensures your music directories remain completely free of random text files and gives you a single, centralized place to review the results across every run.

Each step writes its own named log file directly in that one directory:

  1. step1-verify-audio.log
    Per-file audio integrity results with a final pass/fail summary.

  2. step2a-album-track-gain.log
    Loudgain output for album + track gain across lossless/lossy formats.

  3. step2b-track-gain-m4a.log
    Loudgain track-gain output plus any FFmpeg MP4 container repair messages.

  4. step3-album-checksum.log
    Album checksum creation and verification result.

  5. step4a-artist-audio-validation.log
    Recursive artist audio validation summary.

  6. step4a-artist-audio-passed.log
    List of files that passed validation.

  7. step4a-artist-audio-failed.log
    List of files that failed validation.

  8. step4a-artist-audio-errors.log
    Raw ffmpeg decoder output for failed files.

  9. step5-artist-checksum.log
    Artist checksum creation and per-album verification result.

-- General Cleanup

Every recertification run starts fresh: Step 1 automatically purges the entire $HOME/.logs/linux-audio-folder-recertification directory, so the previous run's logs remain reviewable until the next run begins. Between runs, you can also delete the directory manually, or run Step 6 to remove the log files with confirmation. The log directory is completely independent of the audio files.

\---------------------------------------------------------------------------------------

-- Disclaimer

This guide was developed through iterative collaborative effort between ChatGPT, Claude, Gemini, Mistral and the user. I cannot thank OpenCode project enough. I was about to give up on the other four (well, actually I did) when I came across OpenCode. I run a 10+ year old laptop yet OpenCode ran perfectly well, offloading the heaving lifting to an offsite server.

https://opencode.ai/
