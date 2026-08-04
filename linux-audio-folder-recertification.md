# Linux: Audio Folder-Level 

---

Use these commands after making changes such as:

- Replacing a bad track
- Updating album artwork
- Correcting tags
- Adding or removing tracks

---

## Album Folder

---

Run these commands from the album directory.

### Step 1: Verify FLAC Files

```bash
flac -t *.flac
```

### Step 2: Apply ReplayGain

```bash
shopt -s nullglob nocaseglob

loudgain -a -k -s e \
    *.flac \
    *.mp3 \
    *.ogg \
    *.opus \
    *.m4a \
    *.mp4 \
    *.aac \
    *.ape \
    *.wv \
    *.mpc \
    *.spx
```

### Step 3: Create and Verify Album Checksum

```bash
find . -type f \
    \( \
        -iname "*.flac" -o \
        -iname "*.mp3"  -o \
        -iname "*.m4a"  -o \
        -iname "*.wav"  -o \
        -iname "*.ogg"  -o \
        -iname "*.opus" -o \
        -iname "*.aac"  -o \
        -iname "*.alac" -o \
        -iname "*.aiff" \
    \) \
    -print0 |
LC_ALL=C sort -z |
xargs -0 -r sha512sum > ALBUM.sha512sums.txt &&
[ -s ALBUM.sha512sums.txt ] &&
echo "CREATED: ALBUM.sha512sums.txt ($(wc -l < ALBUM.sha512sums.txt) files)" &&
sha512sum -c ALBUM.sha512sums.txt &&
echo "VERIFIED: ALBUM CHECKSUM OK" ||
echo "FAILED: ALBUM CHECKSUM ERROR"
```

---

## Artist Folder

---

Run this command from the artist directory.

### Step 4: Create and Verify Artist Checksum

```bash
if [ -z "$(find . -mindepth 2 -maxdepth 2 -type d)" ]; then

    > ARTIST.sha512sums.txt

    find . -mindepth 1 -maxdepth 1 -type d -print0 |
    LC_ALL=C sort -z |
    while IFS= read -r -d '' album; do
        name=$(basename "$album")

        hash=$(
            cd "$album" &&
            find . -type f ! -name ALBUM.sha512sums.txt -print0 |
            LC_ALL=C sort -z |
            xargs -0 sha512sum |
            sha512sum |
            cut -d' ' -f1
        )

        printf "%s  %s\n" "$hash" "$name" >> ARTIST.sha512sums.txt
    done

    echo "CREATED: $(basename "$PWD") ARTIST.sha512sums.txt"

    while IFS=" " read -r stored_hash album; do
        actual_hash=$(
            cd "$album" 2>/dev/null &&
            find . -type f ! -name ALBUM.sha512sums.txt -print0 |
            LC_ALL=C sort -z |
            xargs -0 sha512sum |
            sha512sum |
            cut -d' ' -f1
        )

        if [ "$stored_hash" = "$actual_hash" ]; then
            echo "VERIFIED: $album"
        else
            echo "FAILED: $album"
        fi
    done < ARTIST.sha512sums.txt

else

    total=$(find . -mindepth 1 -maxdepth 1 -type d | wc -l)
    i=0

    find . -mindepth 1 -maxdepth 1 -type d -print0 |
    LC_ALL=C sort -z |
    while IFS= read -r -d '' artist; do

        i=$((i + 1))

        (
            cd "$artist"

            > ARTIST.sha512sums.txt

            find . -mindepth 1 -maxdepth 1 -type d -print0 |
            LC_ALL=C sort -z |
            while IFS= read -r -d '' album; do

                name=$(basename "$album")

                hash=$(
                    cd "$album" &&
                    find . -type f ! -name ALBUM.sha512sums.txt -print0 |
                    LC_ALL=C sort -z |
                    xargs -0 sha512sum |
                    sha512sum |
                    cut -d' ' -f1
                )

                printf "%s  %s\n" "$hash" "$name" >> ARTIST.sha512sums.txt

            done
        )

        echo "[$i/$total] CREATED: $(basename "$artist")"

        (
            cd "$artist"

            while IFS=" " read -r stored_hash album; do

                actual_hash=$(
                    cd "$album" 2>/dev/null &&
                    find . -type f ! -name ALBUM.sha512sums.txt -print0 |
                    LC_ALL=C sort -z |
                    xargs -0 sha512sum |
                    sha512sum |
                    cut -d' ' -f1
                )

                if [ "$stored_hash" = "$actual_hash" ]; then
                    echo "VERIFIED: $album"
                else
                    echo "FAILED: $album"
                fi

            done < ARTIST.sha512sums.txt
        )

    done

fi
```

---



