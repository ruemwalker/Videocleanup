# Video Cleanup App Design

Approved by Rue on 2026-11-17. Source doc: https://claude.ai/artifact/YHpTUh6uaBTKDxJ4xeYHjZ

## Intent and why

The app cleans up 5,589 iCloud videos by finding the bad ones, merging or trimming repeats, and moving the keepers to Google Drive, where Claude can manage them afterward. It runs on Rue's Windows computer, so scanning and sorting use no Claude tokens. Success means every keeper sits in Google Drive, verified, and iCloud can be emptied with no risk of losing a good video.

## Goals, scope and constraints

| Goal | What it means |
| --- | --- |
| Find bad videos | Too dark, shaky, or accidental clips with no picture, plus the weak stretches inside long videos |
| Handle repeats | Group near-identical clips, then keep the clearest one or combine back-to-back clips into one |
| Protect originals | Nothing is deleted until approved |
| Hand off to Google Drive | Keepers upload and are verified before iCloud is emptied |
| Cost no tokens to run | Scanning and sorting run locally; Claude only helps on request |

Out of scope: creative edits such as music, titles or color grading, and deleting from iCloud, which Rue does after the final check. A later editing stage for Instagram posts is parked and gets its own project.

| Constraint | Detail |
| --- | --- |
| Library | 5,589 videos inside a 410 GB iCloud library that also holds photos |
| Flash drive | D:, 256 GB, NTFS, empty |
| Google Drive | Paid storage plan in place |
| Computer | Windows, using icloudpd to download, with Rue entering the Apple ID password and two-factor code |
| Duration | Downloading and scanning take hours, so the app works in batches and can pause and resume |

## Approaches and the one chosen

Rue chose option C.

| Option | How it works | Trade-off |
| --- | --- | --- |
| A. Scripts do the sorting | ffmpeg and Python flag black or empty clips, dark clips, shaky clips and near-identical groups, then Rue reviews the list | Fast and free to run, but a script can misjudge a borderline clip |
| B. Claude watches every clip | Claude views sample frames from all 5,589 videos | Best judgment, but slow and token-heavy |
| C. Scripts first, borderline clips checked (chosen) | Option A runs on everything, and only clips the scripts cannot decide get a closer look | Close to B's accuracy at a small share of the cost |

The closer look happens in the app's review screen, where Rue decides. Claude reviews frames from borderline clips only on request.

## How it works

Each batch of about 500 videos, or one year if that is smaller, runs through nine steps.

1. Download the batch with icloudpd into a batch folder on D:, using plain download mode so iCloud stays untouched, and leave at least 20% of the drive free.
2. Split long videos into scenes with PySceneDetect, then scan every video and scene with ffmpeg for empty or black frames, low brightness and camera shake.
3. Group near-identical clips by capture time and visual similarity, using the videohash package.
4. Sort each clip into Keep, Flagged or Borderline.
5. Open the review screen, a local web page. Rue sees the Flagged list, previews the Borderline clips, and approves or changes each decision. Each decided thumbnail swirls into its Keep, Hold or Combine folder.
6. Apply the decisions. Flagged clips move to a holding folder, repeats reduce to the clearest clip, combined clips are written as new files, and good scenes from long videos are saved as separate clips. Originals stay untouched until step 8.
7. Copy the keepers to Google Drive using Google Drive for desktop, then verify them against Drive with rclone check.
8. After Rue approves the match, free the batch's space on D: and start the next batch.
9. When the last batch is done, Rue deletes the videos from iCloud.

Combining is a separate pass Rue approves. The app combines only clips shot back to back within a few minutes of each other, and keeps the clearest single clip for every other group of repeats.

## Safety rules and open items

| Safety rule | Detail |
| --- | --- |
| Approval first | No file is deleted or replaced until Rue approves it in the review screen |
| Originals kept | Source videos stay on D: until Google Drive holds a verified copy |
| Verified upload | File sizes and checksums are compared before a batch is cleared |
| Resumable | A log records every step, so a stopped run picks up where it left off |
| Test run | The first run covers about 20 videos before the full library |
| iCloud last | Rue deletes from iCloud only after the final batch is verified |

Calls made by Claude, open to change: Python with ffmpeg, PySceneDetect, videohash and icloudpd does the downloading, scanning and editing; the review screen is a local web page; batches are about 500 videos or one year, whichever is smaller; rclone check does the final verification because Drive for desktop lists files locally before upload finishes.

Left to the builder: the exact darkness and shake thresholds, tuned after the 20-video trial run, plus folder names and the ffmpeg install.
