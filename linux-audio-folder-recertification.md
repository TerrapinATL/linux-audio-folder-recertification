# Linux: Audio Folder-Level 

---

01. Introduction

These procedures are based on a FOLDER LEVEL approach. Each bash script is designed to be executed manually from the appropriate target folder and is not intended to operate recursively.

Use these commands after making intentional library changes such as:

- Replacing a bad track
- Updating album artwork
- Correcting tags
- Adding or removing tracks

\      -----------------------------------------------------------------

-- Anticipated folder structure for this process:

`
Artist/
└── Album/
    ├── Track 01.flac
    ├── Track 02.mp3
    ├── Track 03.m4a    
`

\      -----------------------------------------------------------------

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

\      -----------------------------------------------------------------

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
        local base_name
        base_name=$(basename "$file")

        case "${file,,}" in
            *.flac)
                if flac -s -t -- "$file" >/dev/null 2>&1; then
                    printf "[OK]     %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((passed++))
                else
                    printf "[FAILED] %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((failed++))
                fi
                ;;
            *.mp3|*.m4a|*.wav|*.ogg|*.aac|*.opus|*.aiff|*.aif)
                if ffmpeg -nostdin -v error -hide_banner -nostats \
                    -i "$file" -f null - >/dev/null 2>&1; then
                    printf "[OK]     %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((passed++))
                else
                    printf "[FAILED] %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((failed++))
                fi
                ;;
        esac
    done < <(
        find . -maxdepth 1 -type f \( \
            -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
            -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
            -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif"    \
        \) -print0 | LC_ALL=C sort -z -V
    )

    {
        echo "----------------------------------------"
        echo "Folder Recertification: Step 1"
        echo "----------------------------------------"
        echo "Scan complete | Total: $total | Passed: $passed | Failed: $failed"
    } | tee -a "$LOG_FILE"

    (( failed == 0 ))
}

verify_audio

```

\      -----------------------------------------------------------------
   
04. Step 2: Apply Album & Track Gain (With Pre-Sanitization for M4A/MP4)

```bash

#!/usr/bin/env bash
# Step 2: Apply Album & Track Gain (With Pre-Sanitization for M4A/MP4/MP3)

set -o pipefail

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_2_album_gain.log"

EXTS=(flac m4a mp3 ogg opus mp4 aac ape wv mpc spx)

shopt -s nullglob nocaseglob

# Collect all matching audio files and run pre-sanitization
all_files=()
for ext in "${EXTS[@]}"; do
    group=( *."$ext" )
    if [ ${#group[@]} -gt 0 ]; then
        all_files+=( "${group[@]}" )

        # Container / Bitstream Pre-Sanitization prior to Loudgain
        if [[ "$ext" == "m4a" || "$ext" == "mp4" ]]; then
            echo "Sanitizing ${#group[@]} .$ext container(s) with FFmpeg..." | tee -a "$LOG_FILE"
            for f in "${group[@]}"; do
                tmp="._fixed_${f}"
                if ffmpeg -v error -hide_banner -i "$f" -map 0 -map_metadata 0 -c copy -movflags +faststart "$tmp" 2>>"$LOG_FILE"; then
                    mv -f "$tmp" "$f"
                else
                    echo "Warning: FFmpeg container repair failed for $f" | tee -a "$LOG_FILE"
                    rm -f "$tmp"
                fi
            done
        elif [[ "$ext" == "mp3" ]]; then
            if command -v mp3val &>/devnull; then
                echo "Sanitizing ${#group[@]} .mp3 bitstream(s) with mp3val..." | tee -a "$LOG_FILE"
                mp3val -f -t "${group[@]}" >>"$LOG_FILE" 2>&1
            else
                echo "Notice: mp3val not found. Skipping MP3 bitstream repair." | tee -a "$LOG_FILE"
            fi
        fi
    fi
done

if [ ${#all_files[@]} -eq 0 ]; then
    echo "No supported audio files found in $(pwd)." | tee -a "$LOG_FILE"
    exit 0
fi

# Run Loudgain ONCE across ALL files together for accurate Album Gain
echo "Processing ${#all_files[@]} total file(s) in $(pwd)..." | tee -a "$LOG_FILE"
if ! loudgain -a -k -s e -L -- "${all_files[@]}" 2>&1 | tee -a "$LOG_FILE"; then
    echo "ERROR: Loudgain failed in $(pwd)." >&2 | tee -a "$LOG_FILE"
fi

echo "----------------------------------------"
echo "Folder Recertification: Step 2"
echo "----------------------------------------"

```

Note: Wav & Aiff are not supported by Loudgain. 

\      -----------------------------------------------------------------

05. Step 3: Re-Verify Audio Files

```bash

#!/usr/bin/env bash
# Step 3: Re-Verify Audio Files (Run from Album folder)

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_3_verify_audio.log"
: > "$LOG_FILE"

verify_audio() {
    local passed=0 failed=0 total=0

    echo "Scanning audio files in $(pwd)..." | tee -a "$LOG_FILE"

    while IFS= read -r -d '' file; do
        ((total++))
        local base_name
        base_name=$(basename "$file")

        case "${file,,}" in
            *.flac)
                if flac -s -t -- "$file" >/dev/null 2>&1; then
                    printf "[OK]     %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((passed++))
                else
                    printf "[FAILED] %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((failed++))
                fi
                ;;
            *.mp3|*.m4a|*.wav|*.ogg|*.aac|*.opus|*.aiff|*.aif)
                if ffmpeg -nostdin -v error -hide_banner -nostats \
                    -i "$file" -f null - >/dev/null 2>&1; then
                    printf "[OK]     %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((passed++))
                else
                    printf "[FAILED] %s\n" "$base_name" | tee -a "$LOG_FILE"
                    ((failed++))
                fi
                ;;
        esac
    done < <(
        find . -maxdepth 1 -type f \( \
            -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
            -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
            -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif"    \
        \) -print0 | LC_ALL=C sort -z -V
    )

    {

        echo "Scan complete | Total: $total | Passed: $passed | Failed: $failed"
    } | tee -a "$LOG_FILE"

    (( failed == 0 ))
}

verify_audio
        echo "----------------------------------------"
        echo "Folder Recertification: Step 3"
        echo "----------------------------------------"
```

\      -----------------------------------------------------------------
            
06. Step 4: Create and Verify Album Checksum

* Run from the album folder.
* Create a checksum file for the current folder.
* Verify that the generated hashes match immediately.
* Non-recursive: operates only on the current album folder.

```bash

#!/usr/bin/env bash
# Step 4: Create and Verify Album Checksum (Run from Album folder)

set -o pipefail

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_4_album_checksum.log"

CHECKSUM="ALBUM.sha512sums.txt"

find . -maxdepth 1 -type f ! -name "$CHECKSUM" \( \
    -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a" -o \
    -iname "*.wav"  -o -iname "*.ogg"  -o -iname "*.aac" -o \
    -iname "*.opus" -o -iname "*.aiff" -o -iname "*.aif"    \
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

        echo "----------------------------------------"
        echo "Folder Recertification: Step 4"
        echo "----------------------------------------"

```

---

07. Artist Folder

---

Run these commands from the Artist directory.


08. Step 5: Verification of Album Checksums

```bash

#!/usr/bin/env bash
# Step 5: Verification of Album Checksums (Run from Artist folder)

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
RUN_LOG="$LOG_DIR/STEP_5_run.log"
ERR_LOG="$LOG_DIR/STEP_5_errors.log"

# Permission & Ownership Verification
if [ ! -w "$PWD" ]; then
    echo "CRITICAL ERROR: Write permission denied on $PWD."
    exit 1
fi

if find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    echo "Notice: Some files/folders are not owned by $USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi

mapfile -d '' manifests < <(find "$PWD" -maxdepth 2 -type f -name "ALBUM.sha512sums.txt" -print0 | LC_ALL=C sort -z)

total=${#manifests[@]}
if [ "$total" -eq 0 ]; then
    echo "ALERT: No ALBUM.sha512sums.txt files found under $PWD."
    exit 1
fi

i=0
for m in "${manifests[@]}"; do
    ((i++))
    dir_path=$(dirname "$m")
    album_name=$(basename "$dir_path")
    artist_name=$(basename "$(dirname "$dir_path")")
    label="$artist_name - $album_name"

    temp_err="$LOG_DIR/temp_err.log"
    (
        cd "$dir_path" || exit 1
        sha512sum -c --quiet --strict "ALBUM.sha512sums.txt" 2>&1
    ) > "$temp_err"

    if [ $? -ne 0 ]; then
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG"
        sed "s/^/[$i\/$total] ERROR: $label :: /" "$temp_err" >> "$ERR_LOG"
    else
        echo "OK [$i/$total] $label" | tee -a "$RUN_LOG"
    fi
    rm -f "$temp_err"

done

        echo "----------------------------------------"
        echo "Folder Recertification: Step 5"
        echo "----------------------------------------"

```

\     -----------------------------------------------------------------

09. Step 6: Create and Verify Artist Checksum

* Run from the artist folder only.
* Each immediate child folder is treated as an album folder.
* Requires completed ALBUM.sha512sums.txt files.
* Does not recursively inspect audio files.

```bash

#!/usr/bin/env bash
# Step 6: Create and Verify Artist Checksum (Run from Artist folder)
#
# Run from the artist folder only.
# Each immediate child folder is treated as an album folder.
# Requires completed ALBUM.sha512sums.txt files.

set -o pipefail

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/STEP_6_artist_checksum.log"

CHECKSUM="ARTIST.sha512sums.txt"
TEMP_CHECKSUM="${CHECKSUM}.tmp"
FAILED=0

: > "$TEMP_CHECKSUM"

# 1. Creation Phase
while IFS= read -r -d '' album; do
    name=$(basename "$album")

    if [ ! -f "$album/ALBUM.sha512sums.txt" ]; then
        echo "FAILED: MISSING ALBUM CHECKSUM: $name" | tee -a "$LOG_FILE"
        FAILED=1
        continue
    fi

    # Hash calculation matching Step 4 definition
    hash=$(cd "$album" 2>/dev/null && find . -type f ! -name ALBUM.sha512sums.txt -print0 | LC_ALL=C sort -z | xargs -0 sha512sum | sha512sum | cut -d" " -f1)

    if [ -n "$hash" ]; then
        printf "%s  %s\n" "$hash" "$name" >> "$TEMP_CHECKSUM"
    else
        echo "FAILED: COULD NOT CALCULATE HASH FOR $name" | tee -a "$LOG_FILE"
        FAILED=1
    fi
done < <(
    find . -mindepth 1 -maxdepth 1 -type d -print0 | LC_ALL=C sort -z
)

if [ "$FAILED" -ne 0 ] || [ ! -s "$TEMP_CHECKSUM" ]; then
    rm -f "$TEMP_CHECKSUM"
    echo "FAILED: ARTIST CHECKSUM CREATION ABORTED" | tee -a "$LOG_FILE"
    exit 1
fi

mv "$TEMP_CHECKSUM" "$CHECKSUM"
echo "CREATED: $CHECKSUM ($(wc -l < "$CHECKSUM") albums)" | tee -a "$LOG_FILE"

# 2. Verification Phase
MISMATCH_COUNT=0
while read -r stored_hash album; do
    actual_hash=$(cd "$album" 2>/dev/null && find . -type f ! -name ALBUM.sha512sums.txt -print0 | LC_ALL=C sort -z | xargs -0 sha512sum | sha512sum | cut -d" " -f1)

    if [ -n "$actual_hash" ] && [ "$stored_hash" = "$actual_hash" ]; then
        printf "%-10s %s\n" "OK" "$album" | tee -a "$LOG_FILE"
    else
        printf "%-10s %s\n" "MISMATCH" "$album" | tee -a "$LOG_FILE"
        ((MISMATCH_COUNT++))
    fi
done < "$CHECKSUM"

# Log Retention / Cleanup Condition
if [ "$MISMATCH_COUNT" -eq 0 ]; then
    echo "----------------------------------------" | tee -a "$LOG_FILE"
    echo "ALL CHECKSUMS PASSED." | tee -a "$LOG_FILE"
else
    echo "----------------------------------------" >&2
    echo "ARTIST CHECKSUM ERRORS DETECTED. Logs stored in $LOG_DIR" >&2
    exit 1
fi

    echo "----------------------------------------"
    echo "Folder Recertification: Step 6"
    echo "----------------------------------------"

```

---

10. Step 7: Log Cleanup

---

```bash

#!/usr/bin/env bash
# Step 7: Manual Log Cleanup
#
# Finds existing log files, waits for user confirmation,
# removes them, and verifies cleanup.

LOG_DIR="$HOME/.logs/Linux_Audio_Folder_Level"

echo "Log directory: $LOG_DIR"
echo

LOG_COUNT=$(find "$LOG_DIR" -type f -name "*.log" 2>/dev/null | wc -l)

if [ "$LOG_COUNT" -eq 0 ]; then
    echo "No log files found."
    exit 0
fi

echo "Found $LOG_COUNT log file(s):"
echo "----------------------------------------"
find "$LOG_DIR" -type f -name "*.log"
echo "----------------------------------------"

read -rp "Delete these log files? (y/N): " CONFIRM

if [[ "$CONFIRM" =~ ^[Yy]$ ]]; then
    find "$LOG_DIR" -type f -name "*.log" -delete
    echo "Log deletion complete."
else
    echo "Cleanup cancelled."
fi

    echo "----------------------------------------"
    echo "Folder Recertification: Step 7"
    echo "----------------------------------------"

```

---



