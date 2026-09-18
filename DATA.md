# DATA.md — data card for ForeSight-EK

Task: short-term action anticipation on EPIC-KITCHENS-100 (EK-100). Given egocentric video
observed up to τ_a seconds before an action starts, predict the (verb, noun) pair.

**No dataset file is stored in this repository.** Everything below is downloaded locally into
`data/`, which is git-ignored.

---

## 1. Sources

| What | Where | Local path |
|---|---|---|
| EK-100 annotations | https://github.com/epic-kitchens/epic-kitchens-100-annotations | `data/epic-annotations/` |
| RULSTM EK-100 CSVs and action ids | https://github.com/fpv-iplab/rulstm | `data/rulstm/RULSTM/data/ek100/` |
| Precomputed TSN features (not yet downloaded, see A-04) | `https://iplab.dmi.unict.it/sharing/rulstm/features/ek100/{rgb,flow,obj}_full/data.mdb` | `data/ek100/{rgb,flow,obj}/data.mdb` |

Commands used:

```bash
mkdir data
git clone --depth 1 https://github.com/epic-kitchens/epic-kitchens-100-annotations.git data/epic-annotations
git clone --depth 1 --filter=blob:none --sparse https://github.com/fpv-iplab/rulstm.git data/rulstm
cd data/rulstm && git sparse-checkout set RULSTM/data/ek100
```

The feature URLs come from `RULSTM/scripts/download_data_ek100_full.sh`.

---

## 2. Files and sizes

### RULSTM CSVs — `data/rulstm/RULSTM/data/ek100/` (about 3.5 MB total)

| File | Lines | Header? | Content |
|---|---|---|---|
| `training.csv` | 67,217 | no | one action segment per line |
| `validation.csv` | 9,668 | no | same format |
| `actions.csv` | 3,807 | yes | 3,806 action classes: `id,verb,noun,action` |
| `training_videos.csv` | 495 | no | video ids in train |
| `validation_videos.csv` | 138 | no | video ids in validation |
| `validation_unseen_participants_ids.csv` | 1,065 | no | segment ids from unseen participants |
| `validation_tail_actions_ids.csv` | 3,105 | no | tail-class segment ids |
| `validation_tail_nouns_ids.csv` | 1,900 | no | tail-noun segment ids |
| `validation_tail_verbs_ids.csv` | 1,760 | no | tail-verb segment ids |
| `test_timestamps.csv` | 13,092 | no | test segments, labels withheld |

### Annotations — `data/epic-annotations/` (about 77 MB without `.git`)

| File | Rows | Content |
|---|---|---|
| `EPIC_100_train.csv` | 67,217 | full annotations with narrations and timestamps (8.4 MB) |
| `EPIC_100_validation.csv` | 9,668 | same (1.2 MB) |
| `EPIC_100_verb_classes.csv` | 97 + header | verb classes with synonyms |
| `EPIC_100_noun_classes.csv` | 300 + header | noun classes with categories |
| `EPIC_100_video_info.csv` | one row per video | duration, resolution, real fps |
| `license.txt` | — | CC BY-NC 4.0 |

Row counts in the two sources match exactly, so the RULSTM CSVs are a reformatting of the official
annotations, not a subset.

### Features (to be filled in during A-04)

| File | Size | Modality | Feature dim |
|---|---|---|---|
| `rgb/data.mdb` | TBD | TSN RGB | TBD (expected 1024) |
| `obj/data.mdb` | TBD | object scores | TBD |
| `flow/data.mdb` | TBD | TSN optical flow | TBD |

Plan: download RGB first, and treat `obj` and `flow` as optional.

---

## 3. Columns

### RULSTM `training.csv` / `validation.csv` — 7 columns, no header

| # | Column | Example | Meaning |
|---|---|---|---|
| 0 | segment id | `P01_01_0` | `<participant>_<video>_<index>` |
| 1 | video id | `P01_01` | video the segment comes from |
| 2 | start frame | `4` | action start, **30 fps** numbering |
| 3 | stop frame | `101` | action end, 30 fps numbering |
| 4 | verb class | `3` | 0–96 |
| 5 | noun class | `3` | 0–299 |
| 6 | action class | `2413` | 0–3805, index into `actions.csv` |

Checked: row `P01_01_0` has verb 3, noun 3, action 2413, and `actions.csv` line 2413 is
`2413,3,3,open cupboard`. The column order is confirmed.

### EPIC annotation CSVs — 15 columns with header

`narration_id`, `participant_id`, `video_id`, `narration_timestamp`, `start_timestamp`,
`stop_timestamp`, `start_frame`, `stop_frame`, `narration`, `verb`, `verb_class`, `noun`,
`noun_class`, `all_nouns`, `all_noun_classes`.

Used in this project for: the human-readable narration, verb and noun names for the demo, and
timestamps for cutting the demo clips. Training uses the RULSTM CSVs.

---

## 4. Frame rate — important

The two sources number frames differently:

| Source | Frame numbering | Seconds → frames |
|---|---|---|
| RULSTM CSVs | **30 fps** | `frame = seconds * 30` |
| EPIC annotation CSVs | **60 fps** | `frame = seconds * 60` |

So `EPIC start_frame = 2 × RULSTM start_frame`, for every video. This holds no matter the video's
real frame rate, which ranges from 29.97 to 90 fps across EK-100 (see `EPIC_100_video_info.csv`);
all frame numbers were resampled to a fixed rate.

Verified on two segments from different participants:

| Segment | Start timestamp | EPIC frame | Implied fps | RULSTM frame | Implied fps |
|---|---|---|---|---|---|
| `P01_11_10` | 00:00:49.15 | 2949 | 60.00 | 1474 | 29.99 |
| `P09_07_10` | 00:00:32.59 | 1955 | 59.99 | 977 | 29.98 |

**Rule for this project:** all windowing and τ_a arithmetic uses the RULSTM frames at 30 fps, so
τ_a = 1 s is 30 frames before the action start. Features are stored at about 4 fps, meaning one
feature roughly every 0.25 s, so a 0.25 s step is 7.5 frames in this numbering; use RULSTM's own
sampling function rather than inventing frame arithmetic.

---

## 5. Label spaces and statistics

| | Train | Validation |
|---|---|---|
| Segments | 67,217 | 9,668 |
| Videos | 495 | 138 |
| Participants | 32 | 32 |
| Distinct verbs present | 97 | 78 |
| Distinct nouns present | 289 | 211 |
| Distinct actions present | 3,568 | 1,352 |
| Median segment duration | 1.57 s | 1.97 s |
| Mean segment duration | 3.12 s | 3.68 s |

Full label spaces: **97 verbs, 300 nouns, 3,806 actions**. Not every class appears in each split,
which is the long-tail problem this project measures with Mean Top-5 Recall.

Validation subsets shipped with RULSTM, used for the tail and unseen-participant analysis:
tail actions (3,105 segments), tail nouns (1,900), tail verbs (1,760), unseen participants (1,065).

---

## 6. Splits used in this project

- **train′** — the official train split minus 3 held-out participants.
- **dev** — those 3 participants. All tuning and early stopping happen here.
- **val** — the official validation split, used **only** for final numbers, as a test set.

Exact participant ids and the resulting counts are recorded in A-07, once `splits.py` exists.

---

## 7. License and attribution

- **EPIC-KITCHENS-100 annotations and data**: Creative Commons Attribution-NonCommercial 4.0
  International (CC BY-NC 4.0). Use is **non-commercial only**, with attribution.
  Cite: Damen et al., *Rescaling Egocentric Vision: Collection, Pipeline and Challenges for
  EPIC-KITCHENS-100*, IJCV 2022.
- **Precomputed features and CSV format**: from RULSTM, Furnari & Farinella,
  *Rolling-Unrolling LSTMs for Action Anticipation from First-Person Video*, TPAMI 2020.
- **This repository's code**: MIT (see `LICENSE`). The license applies to the code only, not to
  the data, which stays under CC BY-NC 4.0.
- No dataset file, feature file or video is committed to this repository. `data/` is git-ignored.
