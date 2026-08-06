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

`
Artist/
└── Album/
├── Track 01.flac
├── Track 02.mp3
├── Track 03.m4a
`

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

verify_audio() {
    local passed=0 failed=0 total=0

    echo "Scanning audio files..."
    echo

    while IFS= read -r -d '' file; do
        ((total++))

        case "${file,,}" in
            *.flac)
                if flac -s -t -- "$file" >/dev/null 2>&1; then
                    echo "[OK]     $(basename "$file")"
                    ((passed++))
                else
                    echo "[FAILED] $(basename "$file")"
                    ((failed++))
                fi
                ;;

            *.mp3|*.m4a|*.wav|*.ogg|*.aac|*.opus)
                if ffmpeg -nostdin -v error -hide_banner -nostats \
                    -i "$file" -f null - >/dev/null 2>&1; then
                    echo "[OK]     $(basename "$file")"
                    ((passed++))
                else
                    echo "[FAILED] $(basename "$file")"
                    ((failed++))
                fi
                ;;
        esac

    done < <(
        find . -type f \( \             -iname "*.flac" -o \             -iname "*.mp3"  -o \             -iname "*.m4a"  -o \             -iname "*.wav"  -o \             -iname "*.ogg"  -o \             -iname "*.aac"  -o \             -iname "*.opus" \         \) -print0 | sort -z -V
    )

    echo
    echo "----------------------------------------"
    echo "Scan complete"
    echo "Files checked: $total"
    echo "Passed:        $passed"
    echo "Failed:        $failed"

    if (( failed > 0 )); then
        return 1
    else
        return 0
    fi
}

verify_audio

> -----------------------------------------------------------------

