### linux-audio-folder-recertification — Change Log

All version changes are appended to this file, newest last, one `## vX Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-folder-recertification-v4.md`) so every published version stays retrievable.

**Current version: v4** — Current version; supersedes v3.

Main guide: [linux-audio-folder-recertification.md](linux-audio-folder-recertification.md)

---

## v4 Change Log (2026-09-14)

No per-version change log was recorded for v3 and earlier (those versions are
archived locally only). This section records the v3 → v4 changes plus the
2026-09-14 review corrections; the embedded scripts are the corrected,
tested versions.

* **Scope sections added** — the Introduction now opens with "What This
  Guide Is For" (the quick, folder-scoped path) and "What This Guide
  Bypasses" (explicitly naming the safety checks this quick path omits),
  plus a Supported Audio Formats list and a pointer to the centralized
  log directory.
* **Section 04 wrapper added** — Steps 2A/2B (renumbered from 04A/04B)
  are grouped under "Step 2: Apply Gain" with a stated purpose;
  M4A/MP4 track-gain-only workaround documented in Step 2B.
* **Troubleshooting & Reference section added (Section 10)** — Common
  Issues and Fixes (M4A/MP4 handling, "MISSING ALBUM CHECKSUM", Permission
  Denied fix), a Log File Reference covering every step's log file, and a
  General Cleanup note.
* **Script corrections (2026-09-14 review):**
  * Steps 2A/2B now run with `set -o pipefail` — previously the
    `loudgain | tee` pipeline always reported success because the `if`
    tested tee's exit status, not loudgain's.
  * Step 2A now uses `nullglob`/`nocaseglob` — previously unmatched
    extension patterns were passed to loudgain as literal arguments,
    causing spurious failures in folders that did not contain every
    listed format, and uppercase extensions were missed.
  * Step 4A's ffmpeg now runs with `-nostdin` (the same loop
    stdin-consumption bug documented and fixed in the moode cleanup
    guide).
  * Step 5 no longer purges the entire log directory on success. The
    auto-purge moved to the START of the workflow (Step 1), so logs from
    a completed run remain reviewable until the next run replaces them;
    Step 6 (manual, confirmed cleanup) is unchanged.
  * Steps 4A/5 now skip `Ignore` folders, matching the rest of the suite.
  * Step 2B's container-repair temp file is now created in the log
    directory via `mktemp` instead of inside the album folder, so moOde
    never transiently indexes it.
* **Divider standardization** — long dash dividers unified to the suite's
  87-dash convention.
* **Requirements section added (2026-09-14)** — tool list matching the
  suite standard (flac, ffmpeg/ffprobe, loudgain, sha512sum, core
  utilities), placed in the Introduction.
* **Folder-structure example aligned (2026-09-14)** — the example now uses
  the suite's `YYYY Album Name / NN Title` naming convention (it was
  `Track 01.ext` style), with a note that any naming works but the
  convention keeps manifests and moOde display consistent.
