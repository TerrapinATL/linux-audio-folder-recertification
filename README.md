# linux-audio-folder-recertification

**Guide version: v5** — Current version; supersedes v4. Error fixes (MP4 files skipped by audio verification, missing loudgain `-L` flag, FLAC-specific testing at artist level, error-message accuracy, stale Step 6 reference, changelog file rebuilt as a real changelog) and script formatting aligned to the moOde cleanup guide standard (2026-09-17, see the change log).

* Full guide: [linux-audio-folder-recertification.md](linux-audio-folder-recertification.md)
* Change log: [linux-audio-folder-recertification-changelog.md](linux-audio-folder-recertification-changelog.md)

---

This series provides quick maintenance procedures for verifying and recertifying music libraries after making changes to existing audio files or metadata. Rather than rescanning an entire collection, these commands operate at the album or artist level, making it easy to update checksums and ReplayGain information following routine maintenance.

---

Typical use cases include:

* Replacing a damaged or corrected audio file
* Updating album artwork
* Correcting or improving metadata tags
* Adding or removing tracks
* Re-encoding or remastering an album
* Verifying file integrity after copying or restoring data

---

The workflow performs four primary tasks:

1. **Verify audio integrity** by testing supported audio files for corruption.
2. **Recalculate ReplayGain** values for all supported audio formats in the current album.
3. **Generate and verify album-level checksums** to create a cryptographic fingerprint of the album's contents.
4. **Generate and verify artist-level checksums** to provide a single fingerprint for each album within an artist's directory, allowing rapid verification that no album has changed unexpectedly.

---

## Recommended Workflow

A four part series to clean, verify, and lockdown securely the integrity of an audio file library.

1. linux-audio-moode-prep: https://github.com/TerrapinATL/linux-audio-moode-prep

2. linux-audio-sha512-checksums: https://github.com/TerrapinATL/linux-audio-sha512-checksums

3. linux-os-nemo-sha512-shortcut: https://github.com/TerrapinATL/linux-os-nemo-sha512-shortcut

4. linux-audio-folder-recertification: https://github.com/TerrapinATL/linux-audio-folder-recertification

---

These procedures are designed to be run from either an **album** or **artist** directory, depending on the task being performed. They are intended as lightweight maintenance tools for users who want to quickly recertify a portion of their music library without rebuilding verification data for the entire collection.

## IMPORTANT

Your Original Library should be treated as immutable.

You should only work on a COPY of your Original Library when processing these scripts. The workflow is designed around creating a validated secondary copy, testing the results, and only then promoting that copy to become a replacement.

Before promotion, files should be cleaned, verified with `flac -t`, and protected with two layers of SHA-512 checksums.

NOTE: The artist checksum (Artist Step 2) stores hashes of album contents, not individual files. It must be verified using the Artist verification routine, which recreates each album hash and compares it. Do not use `sha512sum -c` on Artist manifests. The album and artist manifests use the same algorithms as the SHA-512 guide, so manifests are verifiable across all three tools (this guide, the SHA-512 guide, and the Nemo actions).

The purpose is to ensure you have a verifiable library that can be copied, backed up, and restored repeatedly while still matching the validated cleaned copy.

## Disclaimer

This file was created as a mix of AI generated content, user input, and user editing. It was a cooperative effort between Claude, Gemini, ChatGPT, Mistral, and the user, built and polished with the OpenCode project: https://opencode.ai/
