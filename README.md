linux-audio-folder-recertification

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

3. linux-os-nemo-sha512-shortcut:https://github.com/TerrapinATL/linux-os-nemo-sha512-shortcut

4. linux-audio-folder-recertification: https://github.com/TerrapinATL/linux-audio-folder-recertification

---

These procedures are designed to be run from either an **album** or **artist** directory, depending on the task being performed. They are intended as lightweight maintenance tools for users who want to quickly recertify a portion of their music library without rebuilding verification data for the entire collection.





