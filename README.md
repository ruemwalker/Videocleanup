# Video Cleanup App

A Windows app that downloads iCloud videos in batches, flags bad clips (too dark, shaky, no picture), groups repeats, saves the good scenes from long videos, and lets you approve every decision on a local web page where thumbnails swirl into Keep, Hold and Combine folders. Approved keepers go to Google Drive and are verified before any space is freed.

## Requirements

- Windows with Python 3.12, ffmpeg on the PATH, and Node 20 or later (for one small test)
- A flash drive (D:, NTFS) used as batch staging space
- icloudpd (installed by the project), and rclone for the final Google Drive check
- Your Apple ID password and two-factor code, which you enter yourself when prompted. The app stores neither.

## Setup

```
python -m venv .venv
.venv\Scripts\activate
pip install -e ".[dev]"
pytest -v
```

## Commands

```
videocleanup download   # one batch from iCloud into the staging folder, plain download mode
videocleanup scan       # measure and classify every clip in the batch
videocleanup review     # open the review page at http://127.0.0.1:5000
videocleanup apply      # apply approved decisions, never deletes
videocleanup upload     # copy keepers to Google Drive for desktop
videocleanup verify     # compare against Google Drive with rclone check
videocleanup free       # clear the batch from the flash drive, requires --approve
videocleanup trial --limit 20   # first run on about 20 videos, stops before apply
```

## Safety rules

- Nothing is deleted or replaced without your approval in the review page.
- Originals stay on the flash drive until Google Drive holds a verified copy.
- Only the free command deletes anything, and only after verification and your approval.
- The app never changes iCloud. You delete from iCloud yourself after the final batch.

## Docs

- Design: docs/superpowers/specs/2026-11-17-video-cleanup-design.md
- Plan: docs/superpowers/plans/2026-11-17-video-cleanup.md
