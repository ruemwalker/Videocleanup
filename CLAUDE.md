# Video Cleanup App

Read these before changing anything:

- Design: docs/superpowers/specs/2026-11-17-video-cleanup-design.md
- Plan: docs/superpowers/plans/2026-11-17-video-cleanup.md

## How to work

- Build the plan task by task in order, using test-driven steps: failing test, run it, implement, run it, commit.
- Use the exact file names, signatures and test names from the plan. Keep each file to one responsibility.
- Commit after every task with the message the plan gives.
- This machine runs Windows. Use PowerShell commands and Windows paths.

## Hard rules

- The app never deletes or overwrites a user video except in free_batch in videocleanup/upload.py, and only when verified and approved are both true.
- Never pass delete options to icloudpd: no --delete-after-download, --auto-delete or --keep-icloud-recent-days.
- Never store or log the Apple ID password, two-factor code or Google credentials.
- The flash drive is D:. Keep at least 20% of it free during a batch.
- Scanning and sorting call no Claude tokens at runtime.

## Before you start

Check that Python 3.12, ffmpeg, Node and git are installed, and tell the user what is missing instead of installing silently.
