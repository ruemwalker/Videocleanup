# Video Cleanup App Implementation Plan

**For agentic workers:** Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to build this plan task by task. Steps use checkbox syntax for tracking.

**Goal:** Build a Windows app that downloads iCloud videos in batches, flags bad clips, groups repeats, saves the good scenes from long videos, lets you approve every decision on a local web page, and hands verified keepers to Google Drive.

**Architecture:** A Python package with one module per pipeline stage. A SQLite file records each clip's stage, so any run can stop and resume. A command line runs the stages in order, one batch at a time.

**Tech stack:** Python 3.12, ffmpeg, PySceneDetect, videohash, icloudpd, Flask, pytest, SQLite, and rclone for the final Google Drive check.

**Spec:** docs/superpowers/specs/2026-11-17-video-cleanup-design.md

## Global constraints

- Platform: Windows with ffmpeg on the PATH, and the flash drive on D: formatted NTFS at 256 GB.
- Library: 5,589 videos.
- Batches: about 500 videos or one year, whichever is smaller, with at least 20% of the drive free during a batch.
- Deletion: nothing is deleted or replaced without approval in the review screen, and originals stay on D: until Google Drive holds a verified copy.
- Download: icloudpd in plain download mode only. The user enters the Apple ID password and two-factor code, and the app stores neither.
- iCloud: the app never changes iCloud. The user deletes from iCloud after the final batch.
- Trial: the first run covers about 20 videos.
- Cost: scanning and sorting call no Claude tokens at runtime.
- Review screen: a local web page opened in a browser.
- Thresholds: starting values live in one config object and get tuned after the trial.

## Review focus

1. A truncated or corrupt download is flagged as unreadable and the run continues (tests in Tasks 2 and 3).
2. Portrait iPhone videos report their displayed width and height, and HDR clips are measured after tone mapping so bright footage avoids a false dark flag (Task 2 pins rotation, and tone mapping is the builder's call).
3. The same filename in two batches, such as IMG_0001.MOV, stays two separate records (Task 1).
4. A run interrupted halfway through apply finishes on rerun with no error and no duplicate files (Task 8).
5. Google Drive for desktop shows a file locally before its upload finishes, so verification compares against Drive itself using rclone check (Task 9).

## File structure

| File | Responsibility |
| --- | --- |
| pyproject.toml | Dependencies and the videocleanup command |
| videocleanup/config.py | Thresholds, folder paths and the batch settings |
| videocleanup/state.py | SQLite record of each clip's stage, verdict and decision, so runs resume |
| videocleanup/scan.py | Measures one clip: black frames, brightness, shake, size and capture time |
| videocleanup/classify.py | Turns measurements into Keep, Flagged or Borderline, and scans a whole batch |
| videocleanup/scenes.py | Splits long videos into scenes with PySceneDetect and cuts out scenes |
| videocleanup/group.py | Groups near-identical clips, picks the clearest, and finds back-to-back clips to combine |
| videocleanup/batches.py | Splits the library into batches and checks the drive has room |
| videocleanup/download.py | Builds and runs the icloudpd command in plain download mode |
| videocleanup/thumbs.py | Makes one thumbnail image per clip for the review page |
| videocleanup/review/app.py | Flask routes for the local review page |
| videocleanup/review/static/swirl.js and style.css | The animation that swirls thumbnails into the Keep, Hold and Combine folders |
| videocleanup/review/templates/index.html | The review page layout |
| videocleanup/apply.py | Applies approved decisions with no deletions |
| videocleanup/upload.py | Copies keepers to Google Drive, verifies them, and frees a batch's space |
| videocleanup/cli.py | The commands that run each stage |
| tests/ | One test file per module, plus shared video fixtures in conftest.py |

## Task 1: Project skeleton, config and resumable state

**Files:** Create pyproject.toml, videocleanup/__init__.py, videocleanup/config.py, videocleanup/state.py, tests/test_state.py

**Interfaces, produced for every later task:**

```python
class Verdict(StrEnum): KEEP = "keep"; FLAGGED = "flagged"; BORDERLINE = "borderline"
class Decision(StrEnum): KEEP = "keep"; HOLD = "hold"
class Stage(StrEnum): DOWNLOADED; SCANNED; GROUPED; REVIEWED; APPLIED; UPLOADED; VERIFIED; FREED

@dataclass(frozen=True)
class Thresholds:
    black_ratio_flag: float = 0.95
    dark_flag: float = 35.0         # mean luma on a 0 to 255 scale
    dark_borderline: float = 55.0
    shake_flag: float = 8.0         # pixels of frame-to-frame jitter
    shake_borderline: float = 4.0

class State:
    def __init__(self, db_path: Path) -> None
    def upsert(self, path: Path, batch: str, stage: Stage) -> None
    def stage_of(self, path: Path, batch: str) -> Stage | None
    def paths_at(self, batch: str, stage: Stage) -> list[Path]
    def set_verdict(self, path: Path, batch: str, verdict: Verdict, reason: str = "") -> None
    def verdict_of(self, path: Path, batch: str) -> Verdict | None
    def set_decision(self, path: Path, batch: str, decision: Decision) -> None
    def decision_of(self, path: Path, batch: str) -> Decision | None
```

- [ ] **Step 1: Write the failing tests** in tests/test_state.py: test_stage_survives_reopen (open a second State on the same file and read the same stage), test_paths_at_filters_by_stage_and_batch, test_same_filename_in_two_batches_is_two_records (IMG_0001.MOV under batch-01 and batch-02 keeps separate stages and verdicts).
- [ ] **Step 2: Run** `pytest tests/test_state.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement** the interfaces above. One SQLite table clips with primary key (batch, path) and columns stage, verdict, reason, decision. Starting threshold values come from this task and get tuned after the trial run.
- [ ] **Step 4: Run** the same command. Expected: 3 passed.
- [ ] **Step 5: Commit** after `git init`: `git add . && git commit -m "feat: config and resumable state"`

## Task 2: Measure one clip

**Files:** Create videocleanup/scan.py, tests/conftest.py, tests/test_scan.py

**Interfaces, produced:**

```python
class UnreadableVideo(Exception): ...

@dataclass(frozen=True)
class ClipMetrics:
    path: Path; duration_s: float; black_ratio: float; mean_luma: float
    shake_score: float; width: int; height: int; capture_time: datetime | None

def scan_clip(path: Path) -> ClipMetrics
```

conftest.py provides the fixture make_video(kind: Literal["black", "dark", "normal", "portrait"], seconds: float = 3.0) -> Path, built with ffmpeg lavfi sources. The portrait clip uses ffmpeg's -display_rotation 90 input option.

- [ ] **Step 1: Write the failing tests** in tests/test_scan.py:

```python
def test_black_clip_has_high_black_ratio(make_video):
    assert scan_clip(make_video("black")).black_ratio >= 0.95
def test_dark_clip_has_low_luma(make_video):
    assert scan_clip(make_video("dark")).mean_luma < 35
def test_normal_clip_is_clean(make_video):
    m = scan_clip(make_video("normal")); assert m.mean_luma > 55 and m.black_ratio < 0.05
def test_truncated_file_raises(make_video, tmp_path):
    cut = tmp_path / "cut.mp4"; cut.write_bytes(make_video("normal").read_bytes()[:1024])
    with pytest.raises(UnreadableVideo): scan_clip(cut)
def test_portrait_clip_reports_displayed_size(make_video):
    m = scan_clip(make_video("portrait")); assert m.height > m.width
```

- [ ] **Step 2: Run** `pytest tests/test_scan.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement scan_clip.** ffprobe supplies duration, size, rotation and the creation_time tag. ffmpeg blackdetect and signalstats supply the black ratio and mean luma, with HDR clips tone mapped first. Shake is the spread of frame-to-frame translation on frames sampled at 5 per second, and OpenCV phase correlation is the suggested method. Any ffprobe or ffmpeg failure raises UnreadableVideo.
- [ ] **Step 4: Run** the same command. Expected: 5 passed.
- [ ] **Step 5: Commit:** `git add videocleanup/scan.py tests && git commit -m "feat: scan one clip"`

## Task 3: Classify clips and scan a batch

**Files:** Create videocleanup/classify.py, tests/test_classify.py

**Interfaces, consumed:** ClipMetrics, scan_clip, Thresholds, Verdict, Stage, State. **Produced:**

```python
def classify(m: ClipMetrics, t: Thresholds = Thresholds()) -> Verdict
def scan_batch(state: State, batch: str, t: Thresholds = Thresholds()) -> None
```

- [ ] **Step 1: Write the failing tests.** For classify, with a helper metrics(**overrides) that starts from black_ratio 0, mean_luma 120, shake_score 1:

```python
assert classify(metrics(black_ratio=0.96)) == Verdict.FLAGGED
assert classify(metrics(mean_luma=20)) == Verdict.FLAGGED
assert classify(metrics(mean_luma=45)) == Verdict.BORDERLINE
assert classify(metrics(shake_score=9)) == Verdict.FLAGGED
assert classify(metrics(shake_score=5)) == Verdict.BORDERLINE
assert classify(metrics()) == Verdict.KEEP
```

For scan_batch: test_unreadable_clip_is_flagged_and_run_continues (one good clip and one truncated clip end with verdicts KEEP and FLAGGED, reason "unreadable", both at Stage.SCANNED) and test_scan_batch_skips_clips_already_scanned (a monkeypatched scan_clip counter stays at 0 on the second call).
- [ ] **Step 2: Run** `pytest tests/test_classify.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement.** classify returns the most severe verdict across the black, dark and shake checks. scan_batch handles every path at Stage.DOWNLOADED, catches UnreadableVideo, writes verdicts, and advances the stage.
- [ ] **Step 4: Run** the same command. Expected: all tests pass.
- [ ] **Step 5: Commit:** `git add videocleanup/classify.py tests && git commit -m "feat: classify and scan batches"`

## Task 4: Split long videos into scenes

**Files:** Create videocleanup/scenes.py, tests/test_scenes.py. Add make_long_video(minutes_good: float, minutes_black: float) -> Path to conftest.py.

**Interfaces, consumed:** scan_clip, classify, Thresholds, Verdict. **Produced:**

```python
@dataclass(frozen=True)
class SceneVerdict: start_s: float; end_s: float; verdict: Verdict

def split_scenes(path: Path, min_duration_s: float = 180.0) -> list[tuple[float, float]]
def extract_scene(src: Path, start_s: float, end_s: float, dest: Path) -> Path
def score_scenes(path: Path, workdir: Path, t: Thresholds = Thresholds()) -> list[SceneVerdict]
```

- [ ] **Step 1: Write the failing tests,** marked slow: a 4-minute fixture of 2 good minutes then 2 black minutes gives at least 2 ranges from split_scenes that cover the full duration within 1 second; a 30-second clip returns []; extract_scene output duration is within 0.2 seconds of the requested span; score_scenes marks the black scene FLAGGED and the good scene KEEP.
- [ ] **Step 2: Run** `pytest tests/test_scenes.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement.** split_scenes uses PySceneDetect's ContentDetector and returns [] under min_duration_s. extract_scene re-encodes with ffmpeg for frame-exact cuts. score_scenes extracts each scene into workdir, then runs scan_clip and classify on it.
- [ ] **Step 4: Run** the same command. Expected: 4 passed.
- [ ] **Step 5: Commit:** `git add videocleanup/scenes.py tests && git commit -m "feat: scene splitting"`

## Task 5: Group repeats

**Files:** Create videocleanup/group.py, tests/test_group.py

**Interfaces, consumed:** ClipMetrics. **Produced:**

```python
def group_clips(items: list[ClipMetrics], hash_distance: int = 10, window_s: float = 600.0) -> list[list[Path]]
def pick_clearest(group: list[ClipMetrics]) -> Path
def combine_plan(group: list[ClipMetrics], max_gap_s: float = 180.0) -> list[Path] | None
```

- [ ] **Step 1: Write the failing tests:** test_reencoded_copy_groups_with_original, test_different_content_makes_no_group, test_identical_content_two_hours_apart_stays_separate, test_missing_capture_time_falls_back_to_file_mtime, test_pick_clearest_prefers_lowest_shake_then_resolution, and test_combine_plan_orders_back_to_back_clips_and_rejects_big_gaps (a 30-second gap returns the clips in capture order, a 10-minute gap returns None).
- [ ] **Step 2: Run** `pytest tests/test_group.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement.** videohash gives each clip a 64-bit hash, and two clips match when their Hamming distance is at most hash_distance and their capture times fall within window_s. group_clips returns groups of 2 or more only. combine_plan treats a clip as back to back when it starts within max_gap_s of the previous clip's end.
- [ ] **Step 4: Run** the same command. Expected: 6 passed.
- [ ] **Step 5: Commit:** `git add videocleanup/group.py tests && git commit -m "feat: group repeats"`

## Task 6: Batches, drive headroom and download

**Files:** Create videocleanup/batches.py, videocleanup/download.py, tests/test_batches.py, tests/test_download.py

**Interfaces, consumed:** State, Stage. **Produced:**

```python
@dataclass(frozen=True)
class Batch: name: str; since: date; until: date; expected: int
class InsufficientSpace(Exception): ...

def plan_batches(counts_by_month: dict[tuple[int, int], int], max_videos: int = 500) -> list[Batch]
def has_headroom(drive: Path, incoming_bytes: int, min_free_ratio: float = 0.20) -> bool
def build_icloudpd_command(dest: Path, username: str, since: date, until: date) -> list[str]
def run_download(batch: Batch, dest: Path, username: str, estimated_bytes: int,
                 runner: Callable[[list[str]], int] = subprocess_runner) -> None
def register_downloads(state: State, batch: str, dest: Path) -> int
```

- [ ] **Step 1: Write the failing tests.**

```python
def test_batches_follow_whole_months_and_never_span_years():
    b = plan_batches({(2021,1):300, (2021,2):300, (2021,3):100, (2022,1):50})
    assert [x.expected for x in b] == [300, 400, 50]
def test_single_month_above_limit_becomes_its_own_batch():
    assert [x.expected for x in plan_batches({(2021,5):700})] == [700]
def test_headroom_respects_twenty_percent_free(monkeypatch):
    # disk_usage patched to total 256 GB, free 156 GB
    assert has_headroom(D, 100 * GB) is True   # 21.9% would remain
    assert has_headroom(D, 110 * GB) is False  # 18.0% would remain
def test_icloudpd_command_never_deletes_or_stores_passwords():
    cmd = build_icloudpd_command(dest, "me@example.com", date(2021,1,1), date(2021,1,31))
    assert not {"--delete-after-download", "--auto-delete", "--keep-icloud-recent-days", "--password"} & set(cmd)
    assert str(dest) in cmd and "me@example.com" in cmd
def test_run_download_refuses_without_room(): # runner never called, InsufficientSpace raised
def test_register_downloads_adds_videos_only(tmp_path): # 3 .MOV/.mp4 files registered at Stage.DOWNLOADED, a .txt ignored
```

- [ ] **Step 2: Run** `pytest tests/test_batches.py tests/test_download.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement.** plan_batches fills a batch month by month until the next month would pass max_videos or cross a year. build_icloudpd_command takes its date-range and videos-only flag names from icloudpd --help and always omits the three delete options. The month counts come from an initial listing pass, and the builder picks the listing option from icloudpd --help. run_download checks has_headroom first, then runs the command so icloudpd prompts the user for the password and two-factor code.
- [ ] **Step 4: Run** the same command. Expected: 6 passed.
- [ ] **Step 5: Commit:** `git add videocleanup/batches.py videocleanup/download.py tests && git commit -m "feat: batches and download"`

## Task 7: Thumbnails and the review page with the swirl animation

**Files:** Create videocleanup/thumbs.py, videocleanup/review/app.py, videocleanup/review/templates/index.html, videocleanup/review/static/swirl.js, videocleanup/review/static/style.css, tests/test_thumbs.py, tests/test_review.py, tests/swirl.test.js

**Interfaces, consumed:** State, Decision, Verdict, Stage, ClipMetrics, group_clips, pick_clearest, combine_plan. **Produced:**

```python
def make_thumbnail(video: Path, dest: Path, at_s: float | None = None) -> Path   # JPEG, 320 px wide, midpoint frame by default
def seed_decisions(state: State, batch: str, metrics: list[ClipMetrics]) -> None
def create_app(state: State, batch: str, batch_dir: Path, thumbs_dir: Path) -> Flask
```

```js
export function spiralPoint(t, from, to) // returns {x, y}; t runs 0 to 1
```

seed_decisions sets Flagged clips to HOLD, Keep clips to KEEP, leaves Borderline clips undecided, and sets every non-clearest member of a repeat group to HOLD.

Routes: GET / (a card per Flagged or Borderline clip, plus three drop folders marked data-folder="keep", "hold" and "combine"), GET /thumb/<name>, GET /video/<name> (serves only files inside batch_dir), POST /decide with {name, decision}, GET /combines (proposals from combine_plan), POST /combine with {names} (appends the list to batch_dir/combines.json), GET /summary, and POST /approve (sets Stage.REVIEWED, and returns 409 while any Flagged or Borderline clip is undecided).

The animation: when a clip is decided, its thumbnail travels along a shrinking spiral into its folder, scaling down and fading as it lands. Approve all swirls the remaining cards into their folders in a staggered sequence. When the system asks for reduced motion, the card jumps straight to its folder.

- [ ] **Step 1: Write the failing tests.**

```python
def test_thumbnail_is_320_wide_jpeg(make_video, tmp_path)
def test_seed_decisions_holds_flagged_and_non_clearest_repeats(state, metrics)
def test_index_lists_flagged_and_borderline_with_three_folders(client)  # data-folder keep, hold, combine present
def test_video_route_rejects_paths_outside_batch_dir(client)           # GET /video/..%2Fsecret -> 404
def test_decide_persists_and_summary_counts_it(client, state)
def test_approve_refuses_with_undecided_borderline_clips(client)       # 409, stage unchanged
def test_approve_sets_reviewed_when_all_decided(client, state)
def test_combine_appends_to_combines_json(client, batch_dir)
def test_swirl_assets_are_served(client)                               # /static/swirl.js and style.css return 200
```

In tests/swirl.test.js, using node:test: spiralPoint(0, a, b) equals a, spiralPoint(1, a, b) equals b, and the distance from the centre of the spiral to the point shrinks at every step of t from 0 to 1.

- [ ] **Step 2: Run** `pytest tests/test_thumbs.py tests/test_review.py -v` and `node --test tests/swirl.test.js`. Expected: FAIL.
- [ ] **Step 3: Implement.** make_thumbnail grabs one frame with ffmpeg. The page loads vanilla JavaScript and plain CSS with no build step. swirl.js animates with requestAnimationFrame and a transform per card. The videos play in a native video element for previews.
- [ ] **Step 4: Run** the same commands. Expected: pytest 9 passed and the Node test 3 passed.
- [ ] **Step 5: Look at it.** Start the page on 5 fixture clips and open it in a browser. Decide 3 cards and confirm each card swirls into its folder and the summary counts update. Run Approve all and confirm the rest swirl in.
- [ ] **Step 6: Commit:** `git add videocleanup/thumbs.py videocleanup/review tests && git commit -m "feat: review page with swirl animation"`

## Task 8: Apply approved decisions with no deletions

**Files:** Create videocleanup/apply.py, tests/test_apply.py

**Interfaces, consumed:** State, Decision, Stage, score_scenes, extract_scene, split_scenes. **Produced:**

```python
class NotApproved(Exception): ...

@dataclass(frozen=True)
class ApplyReport: kept: int; held: int; combined: int; scenes: int

def combine(paths: list[Path], dest: Path) -> Path
def apply_batch(state: State, batch: str, batch_dir: Path, holding_dir: Path, out_dir: Path) -> ApplyReport
```

apply_batch writes out_dir/upload/ holding a hard link to every keeper (kept originals, combined files from combines.json, and saved good scenes from long videos), so no extra disk space is used.

- [ ] **Step 1: Write the failing tests:** test_apply_refuses_before_approval (raises NotApproved), test_apply_moves_held_clips_and_links_keepers, test_apply_never_deletes (os.remove, Path.unlink and shutil.rmtree are monkeypatched to raise AssertionError, and apply still succeeds), test_apply_twice_is_safe (a second call returns the same report with no duplicates, and a run where one held clip was already moved still completes), test_combined_duration_matches_sum_within_half_second, and test_long_video_saves_only_good_scenes_as_separate_clips.
- [ ] **Step 2: Run** `pytest tests/test_apply.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement.** combine uses the ffmpeg concat demuxer. apply_batch checks every clip has reached Stage.REVIEWED, moves HOLD clips into holding_dir/<batch>, and skips any step whose output already exists so a rerun resumes. It sets Stage.APPLIED when finished.
- [ ] **Step 4: Run** the same command. Expected: 6 passed.
- [ ] **Step 5: Commit:** `git add videocleanup/apply.py tests && git commit -m "feat: apply approved decisions"`

## Task 9: Upload, verify against Google Drive, free space

**Files:** Create videocleanup/upload.py, tests/test_upload.py

**Interfaces, consumed:** State, Stage. **Produced:**

```python
def file_sha256(path: Path) -> str
def copy_to_drive(upload_dir: Path, drive_dir: Path) -> list[Path]
def rclone_check_command(upload_dir: Path, remote: str) -> list[str]
def verify_batch(upload_dir: Path, remote: str, runner: Callable[[list[str]], int] = subprocess_runner) -> bool
def free_batch(state: State, batch: str, batch_dir: Path, holding_dir: Path, upload_dir: Path,
               verified: bool, approved: bool) -> None
```

- [ ] **Step 1: Write the failing tests:** test_copy_to_drive_copies_every_file_with_matching_sha256, test_rclone_check_uses_one_way_check_and_no_write_flags (the command contains check and --one-way and omits sync, copy, move and delete), test_verify_batch_is_false_on_nonzero_exit, test_free_batch_refuses_without_verification_or_approval (both branches leave every file in place), and test_free_batch_removes_batch_upload_and_holding_then_sets_freed.
- [ ] **Step 2: Run** `pytest tests/test_upload.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement.** copy_to_drive copies into the Google Drive for desktop folder. verify_batch runs rclone check against the Drive remote, because Drive for desktop lists a file locally before its upload finishes and rclone compares Drive's own checksums. free_batch is the only function in the project allowed to delete, and it deletes only when verified and approved are both true. The originals stay in iCloud until the user clears them, which is a second safety net.
- [ ] **Step 4: Run** the same command. Expected: 5 passed.
- [ ] **Step 5: Commit:** `git add videocleanup/upload.py tests && git commit -m "feat: upload, verify and free"`

## Task 10: Command line and the 20-video trial

**Files:** Create videocleanup/cli.py, tests/test_cli.py

**Interfaces, consumed:** every earlier module. **Produced:**

```python
def main(argv: list[str] | None = None) -> int   # subcommands: download, scan, review, apply, upload, verify, free, trial
```

- [ ] **Step 1: Write the failing tests:** test_help_lists_all_subcommands, test_scan_command_scans_batch_fixtures (returns 0 and leaves every clip at Stage.SCANNED), test_free_requires_approve_flag (returns nonzero and every file remains), and test_trial_stops_before_apply (trial --limit 20 scans and opens review, and a monkeypatched apply_batch that raises is never called).
- [ ] **Step 2: Run** `pytest tests/test_cli.py -v`. Expected: FAIL with ImportError.
- [ ] **Step 3: Implement main with argparse.** The trial subcommand downloads, scans and starts the review page for the first 20 videos and stops there.
- [ ] **Step 4: Run the whole suite:** `pytest -v`. Expected: every test passes. Then run `videocleanup trial --limit 20` on the real iCloud videos. Expected: a printed count of Keep, Flagged and Borderline clips, the review page open at http://127.0.0.1:5000, and nothing moved or deleted yet. The user checks the results, and the thresholds get tuned before the full library.
- [ ] **Step 5: Commit:** `git add videocleanup/cli.py tests && git commit -m "feat: command line and trial run"`
