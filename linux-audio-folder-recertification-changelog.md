### linux-audio-folder-recertification — Change Log

All version changes are appended to this file, newest last, one `## vX Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-folder-recertification-v4.md`) so every published version stays retrievable.

**Current version: v6** — supersedes v5. Adds Section 11 (Library-Wide
Verification & Repair). See the v6 entry below. Error fixes (MP4 scanning, loudgain -L flag, FLAC-specific testing at artist level, error-message accuracy, stale Step 6 reference, real changelog file) and script formatting aligned to the moOde cleanup guide standard. See the v5 entry below.

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

---

## v6 Change Log (2026-09-20)

* **New Section 11 — Library-Wide Verification & Repair.** Documents the
  whole-drive verification tool (`~/.local/bin/verify-mastercopy`, 182
  artists / 763 albums per pass, live progress with ETA, resume-safe)
  and its alert companion (`alert-mastercopy`: beep + desktop
  notification on MISMATCH or completion). Both are extensionless
  Python, per the no-`.sh`-files convention.
* **Documents the repair protocol** validated on 4 real corruptions
  (2026-09-19): album-level pinpoint before replacing, local-backup
  verification first, bad copies stashed in /tmp/opencode (never left in
  album directories), album-level and artist-level recheck, log update.
  Records that all 4 failures were truncated server files from an
  interrupted Sep 18 transfer, repaired and re-verified.
* **Versioned copy** — the prior guide (v5) was archived as
  `linux-audio-folder-recertification-v5.md` before editing, per the
  update rule.

## v7 Change Log (2026-09-26)

* **Manifest convention change: audio files only.** Step 3 (album
  checksum) now hashes AUDIO FILES ONLY — cover art, `.mpdignore`, and
  other non-audio files are excluded. Artist Step 2's aggregate digest
  (creation and verification) recomputes over the same audio-only file
  set, matching the SHA-512 guide's v16 and the Nemo actions' v8.
  Artwork changes no longer invalidate artist manifests.
* Owner decision (2026-09-26): checksums protect the audio; artwork is
  freely replaceable and not part of the cryptographic baseline.
* Manifests generated under the old all-files convention must be
  regenerated to match.
* **Versioned copy** — the prior guide (v6) was archived as
  `Old/linux-audio-folder-recertification-v6.md` before editing, per the
  update rule.

## v8 Change Log (2026-09-27)

* **Artist-digest convention change: everything, no exceptions.** Owner
  standard (2026-09-27), matching the SHA-512 guide's v17: the artist
  aggregate digest covers **EVERYTHING in each album folder except
  `ALBUM.sha512sums.txt`** — audio files AND cover art. ALBUM manifests
  remain AUDIO FILES ONLY. This supersedes the v7 audio-only artist
  wording and resolves the guide's internal contradiction (the artist
  Step 2 prose already described the all-files algorithm).
* Artist Step 2 embedded script: the audio-extension `find` filter was
  removed from both the creation and verification pipelines (decode-test
  file discovery remains audio-only by design — it tests music files).
* **Versioned copy** — the prior guide (v7) was archived as
  `Old/linux-audio-folder-recertification-v7.md` before editing, per the
  update rule.

## v9 Change Log (2026-09-27)

* **Artist-digest scope change: include the ALBUM manifest.** Owner
  decision (2026-09-27): the artist aggregate digest now covers
  **EVERYTHING in each album folder, INCLUDING `ALBUM.sha512sums.txt`** —
  audio files, cover art, and the album manifest itself, no exceptions,
  matching the SHA-512 guide's v18. Rationale: modifications or corruption
  of an ALBUM manifest must be caught at the artist tier. ALBUM manifests
  remain AUDIO FILES ONLY.
* Artist Step 2 embedded script: the `! -name "ALBUM.sha512sums.txt"`
  exclusion was removed from both the creation and verification pipelines.
* **Versioned copy** — the prior guide (v8) was archived as
  `Old/linux-audio-folder-recertification-v8.md` before editing, per the
  update rule.
