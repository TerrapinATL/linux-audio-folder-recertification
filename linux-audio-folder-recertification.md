# Linux: Audio Folder-Level 

---

01. Introduction

These procedures are based on a FOLDER LEVEL approach. Each bash script is designed to be executed manually from the appropriate target folder and is not intended to operate recursively.

Use these commands after making intentional library changes such as:

- Replacing a bad track
- Updating album artwork
- Correcting tags
- Adding or removing tracks


      -----------------------------------------------------------------

-- Anticipated folder structure for this process:

```
Artist/
└── Album/
    ├── Track 01.flac
    ├── Track 02.mp3
    ├── Track 03.m4a    
```

      -----------------------------------------------------------------

-- Supported Audio Formats

- flac
- mp3
- m4a
- ogg
- opus
- wav
- aiff

---

02. Album Folder

---
Run these commands from the album folder containing supported audio files.

      -----------------------------------------------------------------

03. Step 1: Verify Audio Files

```bash

#!/usr/bin/env bash
# Step 1: Verify Audio Files (Run from Album folder)

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_1_verify_audio.log"
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
        echo "Scan complete | Total: $total | Passed: $passed | Failed: $failed"
    } | tee -a "$LOG_FILE"

    (( failed == 0 ))
}

verify_audio

```

      -----------------------------------------------------------------
   
04A. Step 2A: Apply Album & Track Gain (FLAC, MP3, OGG, OPUS, etc.)

```bash

#!/usr/bin/env bash
# Step 2A: Apply Album & Track Gain (FLAC, MP3, OGG, OPUS, etc.)

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_2A_album_gain.log"

shopt -s nullglob nocaseglob
files=( *.flac *.mp3 *.ogg *.opus *.aac *.ape *.wv *.mpc *.spx )

# Safe exit whether script is executed or sourced
if [ ${#files[@]} -eq 0 ]; then
    return 0 2>/dev/null || exit 0
fi

if ! loudgain -a -k -s e -L -- "${files[@]}" 2>&1 | tee -a "$LOG_FILE"; then
    echo "ERROR: Loudgain failed in $(pwd). Details saved to $LOG_FILE" >&2
    return 1 2>/dev/null || exit 1
fi

```
Note: Wav & Aiff are not supported by Loudgain. 

      -----------------------------------------------------------------
   
04B. Step 2B: Apply Track Gain Only (M4A / MP4 Workaround)

Note: M4A/MP4 Files: `loudgain` suffers from an upstream C-level memory corruption bug (segmentation fault) when writing album-wide metadata into MP4/M4A atom headers. To prevent crashes while keeping standard EBU R128 tags (-s e), M4A containers are intentionally processed using Track Gain only (by omitting the -a flag).

`loudgain` relies on a legacy MP4 atom parser that frequently encounters memory corruption bugs (segmentation faults) when reading or writing tags on non-standard M4A containers. 

To guarantee stability:
   1. Files are pre-processed through `ffmpeg` using stream copy (`-c copy -movflags +faststart`) to rewrite and sanitize the MP4 container structure without re-encoding the audio.
   2. `loudgain` is run in **Track Gain mode** (omitting `-a`) to prevent album-level tag aggregation crashes while preserving standard EBU R128 values (`-s e`).

```bash

#!/usr/bin/env bash
# Step 2B: Apply Track Gain to M4A/MP4 Files (Container Repair & Workaround)

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_2B_track_gain_m4a.log"

shopt -s nullglob nocaseglob
files=( *.m4a *.mp4 )

[ ${#files[@]} -eq 0 ] && exit 0

echo "Sanitizing MP4 containers..." | tee -a "$LOG_FILE"
for f in "${files[@]}"; do
    tmp="._fixed_${f}"
    if ffmpeg -v error -i "$f" -map 0 -map_metadata 0 -c copy -movflags +faststart "$tmp" 2>>"$LOG_FILE"; then
        mv "$tmp" "$f"
    else
        echo "Warning: FFmpeg container fix failed for $f" | tee -a "$LOG_FILE"
        rm -f "$tmp"
    fi
done

if ! loudgain -k -s e -L -- "${files[@]}" 2>&1 | tee -a "$LOG_FILE"; then
    echo "ERROR: Loudgain failed in $(pwd). Details saved to $LOG_FILE" >&2
    exit 1
fi


```

      -----------------------------------------------------------------

05. Step 3: Create and Verify Album Checksum

* Run from the album folder.
* Create a checksum file for the current folder.
* Verify that the generated hashes match immediately.
* Non-recursive: operates only on the current album folder.

```bash

#!/usr/bin/env bash
# Step 3: Create and Verify Album Checksum

set -o pipefail

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_3_album_checksum.log"

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
        echo "VERIFIED: ALBUM CHECKSUM OK" | tee -a "$LOG_FILE"
    else
        echo "FAILED: ALBUM CHECKSUM ERROR" | tee -a "$LOG_FILE"
        exit 1
    fi
else
    echo "FAILED: NO SUPPORTED AUDIO FILES FOUND" | tee -a "$LOG_FILE"
    exit 1
fi

```

---

06. Artist Folder

---

Run these commands from the Artist directory.

      -----------------------------------------------------------------

07. Step 4: Recursive Artist Audio Validation

* Recursively inspect only supported audio files beneath that folder.
* Verify audio integrity before any Artist-level checksum is created.
* Does not modify files.
* Produces a clear pass/fail report.

      -----------------------------------------------------------------

Step 4A: Recursive Artist Audio Validation Process

```bash

#!/usr/bin/env bash
# Step 4A – Recursive Artist Audio Validation (Run from Artist folder)

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"

LOG="$LOG_DIR/STEP_4A_artist_audio_validation.log"
PASSED="$LOG_DIR/STEP_4A_artist_audio_passed.log"
FAILED="$LOG_DIR/STEP_4A_artist_audio_failed.log"
ERRORS="$LOG_DIR/STEP_4A_artist_audio_errors.log"

: > "$LOG"
: > "$PASSED"
: > "$FAILED"
: > "$ERRORS"

total=0; passed=0; failed=0

echo "Starting recursive validation for: $(pwd)" | tee -a "$LOG"

while IFS= read -r -d '' file; do
    total=$((total + 1))
    clean_file=$(printf '%s' "$file" | tr -d '\r')

    if ffmpeg -v error -i "$clean_file" -f null - 2>>"$ERRORS"; then
        passed=$((passed + 1))
        echo "OK: $clean_file" >> "$PASSED"
        echo "OK: $clean_file" >> "$LOG"
    else
        failed=$((failed + 1))
        echo "FAILED: $clean_file" >> "$FAILED"
        echo "FAILED: $clean_file" | tee -a "$LOG"
    fi
done < <(
    find . -type f \( \
        -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
        -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
        -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif"  \
    \) -print0
)

echo "----------------------------------------" | tee -a "$LOG"
echo "Total Processed: $total | Passed: $passed | Failed: $failed" | tee -a "$LOG"

if (( failed > 0 )); then
    echo "Validation failed for $failed file(s). Review logs in: $LOG_DIR" >&2
    exit 1
fi

```

      -----------------------------------------------------------------

Step 4B: Separate Results

Step 4A generates four distinct log files inside $HOME/.logs/Linux_Audio_Folder_Level/:

    STEP_4A_artist_audio_validation.log

        Complete validation report summary.

    STEP_4A_artist_audio_passed.log

        List of all audio files that passed integrity checks.

    STEP_4A_artist_audio_failed.log

        List of audio files that failed integrity checks.

    STEP_4A_artist_audio_errors.log

        Raw ffmpeg decoder output recorded during failure inspection.

     -----------------------------------------------------------------

Step 4C: Review Results
```bash

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"

# View raw FFmpeg error output
cat "$LOG_DIR/STEP_4A_artist_audio_errors.log"

# View overall summary report
cat "$LOG_DIR/STEP_4A_artist_audio_validation.log"

# View list of passed files
cat "$LOG_DIR/STEP_4A_artist_audio_passed.log"

# View list of failed files
cat "$LOG_DIR/STEP_4A_artist_audio_failed.log"

```

      -----------------------------------------------------------------

08. Step 5: Create and Verify Artist Checksum

* Run from the artist folder only.
* Each immediate child folder is treated as an album folder.
* Requires completed ALBUM.sha512sums.txt files.
* Does not recursively inspect audio files.

```bash

#!/usr/bin/env bash
# Step 5 – Create and Verify Artist Checksum (Run from Artist folder)
#
# Run from the artist folder only.
# Each immediate child folder is treated as an album folder.
# Requires completed ALBUM.sha512sums.txt files.

set -o pipefail

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_5_artist_checksum.log"

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
    find . -mindepth 1 -maxdepth 1 -type d -print0 |
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

# Automatic log directory cleanup upon successful final pass
if [ "$MISMATCH_COUNT" -eq 0 ]; then
    echo "----------------------------------------"
    echo "ALL PROCESSES PASSED. Purging log directory: $LOG_DIR"
    rm -rf "$LOG_DIR"
else
    echo "----------------------------------------" >&2
    echo "ARTIST CHECKSUM ERRORS DETECTED. Logs retained in $LOG_DIR" >&2
    exit 1
fi

```

---

---

---
9. Step 6: Log Cleanup

```bash
#!/usr/bin/env bash
# Step 6: Log Cleanup
#
# Finds existing log files, waits for user confirmation,
# removes them, and verifies cleanup.

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"

echo "Log directory:"
echo "$LOG_DIR"
echo

echo "Step 6A: Finding Log Files"
echo "----------------------------------------"

LOG_COUNT=$(find "$LOG_DIR" -type f -name "*.log" | wc -l)

if [ "$LOG_COUNT" -eq 0 ]; then
    echo "No log files found."
    exit 0
fi

find "$LOG_DIR" -type f -name "*.log"

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

find "$LOG_DIR" -type f -name "*.log" -delete

echo "Log deletion complete."

echo
echo "Step 6C: Verify Log Deletion"
echo "----------------------------------------"

REMAINING=$(find "$LOG_DIR" -type f -name "*.log" | wc -l)

if [ "$REMAINING" -eq 0 ]; then
    echo "SUCCESS: No log files remain."
else
    echo "WARNING: $REMAINING log file(s) remain."
    find "$LOG_DIR" -type f -name "*.log"
fi

echo
echo "Log cleanup finished."
```

---
