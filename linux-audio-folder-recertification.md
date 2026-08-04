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
# Step 1: Verify Audio Files

passed=0
failed=0

while IFS= read -r -d '' file; do

    case "${file,,}" in

        *.flac)
            if flac -s -t "$file" 2>/dev/null; then
                echo "[OK]     $file"
                ((passed++))
            else
                echo "[FAILED] $file"
                ((failed++))
            fi
            ;;

        *)
            if ffmpeg -v error -i "$file" -f null - 2>&1 | grep -q .; then
                echo "[FAILED] $file"
                ((failed++))
            else
                echo "[OK]     $file"
                ((passed++))
            fi
            ;;

    esac

done < <(
    find . -maxdepth 1 -type f \
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" \
        \) \
        -print0 | sort -z -V
)

echo "----------------------------------------"
echo "Scan complete. Passed: $passed | Failed: $failed"

```

      -----------------------------------------------------------------
   
04. Step 2: Apply ReplayGain

```bash

#!/usr/bin/env bash

LOG_FILE="loudgain_error.log"

shopt -s nullglob nocaseglob

files=(
    *.flac
    *.mp3
    *.ogg
    *.opus
    *.m4a
)

if [ ${#files[@]} -eq 0 ]; then
    echo "No matching audio files found."
    exit 0
fi

# Run loudgain and capture output while streaming it live to stdout
if ! loudgain -a -k -s e "${files[@]}"; then
    status=$?
    {
        echo "----------------------------------------"
        echo "Timestamp : $(date '+%Y-%m-%d %H:%M:%S')"
        echo "Directory : $(pwd)"
        echo "Exit Code : $status"
        echo "Files Targeted (${#files[@]}):"
        printf '  - %s\n' "${files[@]}"
        echo "----------------------------------------"
        echo ""
    } >> "$LOG_FILE"

    echo "Error encountered! Details appended to $LOG_FILE" >&2
    exit "$status"
fi

```
Note: Wav & Aiff are not supported by Loudgain. 

      -----------------------------------------------------------------

05. Step 3: Create and Verify Album Checksum

* Run from the album folder.
* Create a checksum file for the current folder.
* Verify that the generated hashes match immediately.
* Non-recursive: operates only on the current album folder.

```bash

#!/usr/bin/env bash
# Step 3 – Create and Verify Album Checksum

CHECKSUM="ALBUM.sha512sums.txt"

find . -maxdepth 1 -type f \
    ! -name "$CHECKSUM" \
    \( \
        -iname "*.flac" -o \
        -iname "*.mp3"  -o \
        -iname "*.m4a"  -o \
        -iname "*.ogg"  -o \
        -iname "*.opus" -o \
        -iname "*.wav"  -o \
        -iname "*.aiff" \
    \) \
    -print0 |
LC_ALL=C sort -z |
xargs -0 -r sha512sum > "$CHECKSUM"

if [ -s "$CHECKSUM" ]; then

    echo "CREATED: $CHECKSUM ($(wc -l < "$CHECKSUM") files)"
    echo "VERIFYING: $CHECKSUM"

    # Swap output from "filename: OK" to "OK: filename"
    if sha512sum -c "$CHECKSUM" | awk -F': ' '{printf "%-6s %s\n", $2, $1}'; then
        echo "VERIFIED: ALBUM CHECKSUM OK"
    else
        echo "FAILED: ALBUM CHECKSUM ERROR"
    fi

else

    echo "FAILED: NO SUPPORTED AUDIO FILES FOUND"

fi

```

---

06. Artist Folder

---

Run these commands from the Artist directory.

      -----------------------------------------------------------------

07. Step 5: Recursive Artist Audio Validation

* Recursively inspect only supported audio files beneath that folder.
* Verify audio integrity before any Artist-level checksum is created.
* Does not modify files.
* Produces a clear pass/fail report.

      -----------------------------------------------------------------

Step 5A: Recursive Artist Audio Validation Process

```bash

#!/usr/bin/env bash
# Step 5A – Recursive Artist Audio Validation

LOG="artist_audio_validation.log"
PASSED="artist_audio_passed.log"
FAILED="artist_audio_failed.log"
ERRORS="artist_audio_errors.log"

: > "$LOG"
: > "$PASSED"
: > "$FAILED"
: > "$ERRORS"

total=0
passed=0
failed=0

while IFS= read -r -d '' file; do

    total=$((total + 1))

    case "${file,,}" in

        *.flac)

            if flac -t "$file" >/dev/null 2>>"$ERRORS"; then

                echo "OK: $file" | tee -a "$LOG"
                passed=$((passed + 1))

            else

                echo "FAILED: $file" | tee -a "$LOG"
                failed=$((failed + 1))

            fi
            ;;

        *.mp3|*.m4a|*.ogg|*.opus|*.wav|*.aiff)

            if ffmpeg \
                -v error \
                -i "$file" \
                -f null - \
                >/dev/null \
                2>>"$ERRORS"; then

                echo "OK: $file" | tee -a "$LOG"
                passed=$((passed + 1))

            else

                echo "FAILED: $file" | tee -a "$LOG"
                failed=$((failed + 1))

            fi
            ;;

    esac

done < <(
    find . -type f \
    \( \
        -iname "*.flac" -o \
        -iname "*.mp3"  -o \
        -iname "*.m4a"  -o \
        -iname "*.ogg"  -o \
        -iname "*.opus" -o \
        -iname "*.wav"  -o \
        -iname "*.aiff" \
    \) \
    -print0 |
    LC_ALL=C sort -z
)

grep '^OK' "$LOG" > "$PASSED" || true
grep '^FAILED' "$LOG" > "$FAILED" || true

echo
echo "RESULTS"
echo "-------"
echo "Files Tested: $total"
echo "Passed:       $passed"
echo "Failed:       $failed"

if [ "$failed" -eq 0 ]; then

    echo "VALIDATION PASSED"

else

    echo "VALIDATION FAILED"
    echo "Review: $FAILED"

fi


```

      -----------------------------------------------------------------

Step 5B: Separate Results

The following files are created by Step 5A:

   1. artist_audio_validation.log
      * Complete validation report.
   2. artist_audio_passed.log
      * Files that passed integrity testing.
   3. artist_audio_failed.log
      * Files that failed integrity testing.
   4. artist_audio_errors.log
      * Tool output generated during validation.

     -----------------------------------------------------------------

Step 5C: Review Results
```bash

cat artist_audio_errors.log

cat artist_audio_validation.log

cat artist_audio_passed.log

cat artist_audio_failed.log

```

      -----------------------------------------------------------------

08. Step 6: Create and Verify Artist Checksum

* Run from the artist folder only.
* Each immediate child folder is treated as an album folder.
* Requires completed ALBUM.sha512sums.txt files.
* Does not recursively inspect audio files.

```bash

#!/usr/bin/env bash
# Step 6 – Create and Verify Artist Checksum
#
# Run from the artist folder only.
# Each immediate child folder is treated as an album folder.
# Requires completed ALBUM.sha512sums.txt files.

rm -f ./*.log

CHECKSUM="ARTIST.sha512sums.txt"
TEMP_CHECKSUM="${CHECKSUM}.tmp"
FAILED=0

: > "$TEMP_CHECKSUM"

while IFS= read -r -d '' album; do

    name=$(basename "$album")

    if [ ! -f "$album/ALBUM.sha512sums.txt" ]; then

        echo "FAILED: MISSING ALBUM CHECKSUM: $name"
        FAILED=1
        continue

    fi

    # Calculates composite hash of all non-checksum files in the album directory
    hash=$(
        cd "$album" &&
        find . -type f ! -name "ALBUM.sha512sums.txt" -print0 |
        LC_ALL=C sort -z |
        xargs -0 sha512sum |
        sha512sum |
        cut -d' ' -f1
    )

    printf "%s  %s\n" "$hash" "$name" >> "$TEMP_CHECKSUM"

done < <(
    find . -mindepth 1 -maxdepth 1 -type d -print0 |
    LC_ALL=C sort -z
)

if [ "$FAILED" -ne 0 ]; then

    rm -f "$TEMP_CHECKSUM"
    echo "FAILED: ARTIST CHECKSUM NOT CREATED"
    exit 1

fi

if [ -s "$TEMP_CHECKSUM" ]; then

    mv "$TEMP_CHECKSUM" "$CHECKSUM"
    echo "CREATED: $CHECKSUM ($(wc -l < "$CHECKSUM") albums)"

else

    rm -f "$TEMP_CHECKSUM"
    echo "FAILED: NO ALBUM CHECKSUMS FOUND"
    exit 1

fi


while read -r stored_hash album; do

    actual_hash=$(
        cd "$album" 2>/dev/null &&
        find . -type f ! -name "ALBUM.sha512sums.txt" -print0 |
        LC_ALL=C sort -z |
        xargs -0 sha512sum |
        sha512sum |
        cut -d' ' -f1
    )

    if [ "$stored_hash" = "$actual_hash" ]; then

        printf "%-10s %s\n" "OK" "$album"

    else

        printf "%-10s %s\n" "MISMATCH" "$album"

    fi

done < "$CHECKSUM"

```

---



