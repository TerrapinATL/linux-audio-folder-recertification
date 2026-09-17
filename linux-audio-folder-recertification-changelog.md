### linux-audio-folder-recertification — Change Log

All version changes are appended to this file, newest last, one `## vX Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-folder-recertification-v4.md`) so every published version stays retrievable.

**Current version: v5** — supersedes v4. Error fixes (MP4 scanning, loudgain -L flag, FLAC-specific testing at artist level, error-message accuracy, stale Step 6 reference, real changelog file) and script formatting aligned to the moOde cleanup guide standard. See the v5 entry below.

Main guide: [linux-audio-folder-recertification.md](linux-audio-folder-recertification.md)

---

## v1–v4 Change Logs (recovered summary)

No per-version change log was recorded for v1 through v4 (this file was
accidentally a duplicate of the guide instead of a changelog — fixed in
v5). Archived copies of every version live in `Old/`:

* **v1–v2** — initial folder-level recertification workflow (album Steps
  1–3, artist Steps 1–2, log cleanup).
* **v3** — manifest conventions aligned with the SHA-512 guide
  (`ARTIST.sha512sums.txt` / `ALBUM.sha512sums.txt` generic names;
  album manifest covers all files except the manifests; artist aggregate
  computed the same way as the SHA-512 guide's Step 4 so the Nemo
  "Verify ARTIST SHA512" action and the SHA-512 guide can verify it).
* **v3 → v4** — loudgain MP4/M4A workaround documented (Step 2B:
  ffmpeg stream-copy sanitize + Track Gain only), auto-purge comment in
  Album Step 1, log-file reference expanded.

---

## v5 Change Log (2026-09-17)

Full review against the moOde cleanup guide standard, plus error fixes:

* **Error fix — MP4 files were skipped by audio verification.** Album
  Step 1 and Artist Step 1 did not scan `*.mp4` files (MP4 is in the
  supported-formats list), so MP4 files were never integrity-tested.
  Both scans now include `*.mp4`.
* **Error fix: Step 2A loudgain flags.** Now uses the moOde-standard
  flag set `-a -k -s e -L` (album, noclip, EBU R128 + extra tags,
  lowercase tag names) — matching the cleanup guide's Step 5 and the
  Nemo Apply ReplayGain action. The `-L` flag was missing.
* **Error fix: Artist Step 1 now uses FLAC-specific testing.** FLAC
  files are tested with `flac -s -t` (the more precise FLAC decoder
  check, as Steps 1/4/10 of the cleanup guide); all other formats use
  the ffmpeg null-decode. Previously everything went through ffmpeg.
* **Error fix: Step 3 error-message accuracy.** The "empty checksum
  file" failure path no longer claims "NO SUPPORTED AUDIO FILES FOUND";
  it now reports the sha512sum failure accurately.
* **Error fix: stale cross-reference.** Section 10 pointed to a
  nonexistent "Step 6" for log cleanup; now correctly references
  Artist Step 3 (Log Cleanup).
* **Changelog file rebuilt.** This file had accidentally become a
  duplicate of the guide itself (685 identical lines, no Change Log
  sections); it is now a real changelog and the suite convention is
  restored.
* **Formatting aligned to the moOde cleanup guide standard** (all six
  scripts): standard `========== Step N: Title ==========` headers with
  `Root:`/`Started:` lines; strictly-last footers with `----` dividers
  and step-title banners; per-file `[i/total]` OK/FAIL lines (log-only
  OK chatter at artist level); album headers on folder change for the
  recursive artist pass and the artist checksum loop; `set -u`; and the
  keep-terminal-open EXIT trap so failure causes stay visible.
* **Step numbering convention unchanged:** steps restart at 1 within
  each folder-level section (Album / Artist), per the suite's stated
  numbering rule for this guide.
* All embedded scripts re-verified with `bash -n`.
