# Reproduction log — baseline (Nitsche & Øyulvstad, 2025)

***Parts of this log is AI-generated**.

## What this log is

My thesis builds on Nitsche & Øyulvstad (2025), *Utilizing LSH to compute trajectory similarity*
(`docs/thomas-og-eivind-LSH-similarity.pdf`, called "the thesis" below). Their idea:
- Hash every trajectory with an LSH scheme (a grid or a set of disks) and group trajectories into buckets.
- Run exact DTW only between trajectories that share a bucket, instead of between all pairs.

Before I build my own filter (cell sequences → shingles → MinHash-LSH), I re-ran their code on my laptop.
This checks three things:
1. The code still works after the Python 3.14 upgrade and the preprocessing fixes.
2. My numbers agree with theirs, so I can trust the code as a baseline.
3. Which concrete numbers my method has to beat.

All experiments use **n = 500 trajectories** per city. The thesis uses 500 for its parameter studies
and reports runtimes for 500–2000.

## Terms used in this log

| Term | Meaning |
|---|---|
| **Ground truth** | Exact DTW between all pairs of trajectories. A pair is *truly similar* if DTW < θ. |
| **θ (threshold)** | DTW cut-off for "similar". The thesis uses **θ = 0.1** (sec. 4.3.2). The code also reports 0.2–0.5. |
| **Grid scheme** | Covers the area with square cells of side *resolution* (km). Each *layer* is one grid with a random offset. |
| **Disk scheme** | Places *disks* random disks of a given *diameter* (km). Each *layer* is one random set of disks. |
| **Strict bucketing** (`original` in code) | Per layer, the trajectory's whole cell sequence is hashed into one bucket. Two trajectories are candidates if their **full sequences are identical** in at least one layer. |
| **Loose bucketing** (`loose`) | One bucket per cell. Two trajectories are candidates if they **visit any common cell** in any layer. |
| **Candidate pairs** | Pairs that share at least one bucket, i.e. the pairs that get an exact DTW check. |
| **Recall** | Share of truly similar pairs that became candidates. 1.0 = nothing missed. |
| **Precision** | Share of candidates that are truly similar (measured *before* the DTW check). |
| **Candidate ratio** | Candidate pairs ÷ all pairs (124 750 for n = 500). This is what determines runtime. |
| **24 runs** | Grid and disk placement is random, so every config is run 24 times and averaged (8 parallel jobs × 3 iterations here; 24 jobs × 1 iteration in the original). |

**How candidate pairs are derived:** the code does not report the candidate count directly. I compute it as
`recall × (number of true pairs) / precision`. This is exact for the micro-averaged precision the code uses,
but approximate here because the CSVs round to 3 decimals.

## Environment

- Laptop: AMD Ryzen AI 9 365 (10 cores / 20 threads), 31 GB RAM, Windows 11
- Python 3.14.7, packages pinned in `requirements.txt` (includes `h3` and `datasketch` for my own work)
- Smart App Control turned off (it blocked compiled packages)
- The original authors ran on NTNU IDUN: Xeon Gold 6348, 24 cores, 24 parallel processes (thesis sec. 4.1)

## Settings shared by all experiments

Unless a section says otherwise:
- Size: **n = 500** (set automatically when `MEASURE = "dtw"`)
- Thresholds reported: 0.1, 0.2, 0.3, 0.4, 0.5. **θ = 0.1 is the one that matters**; it is the thesis's choice.
- Randomness: `iterations=3`, `parallell_jobs=8` (24 runs). The original notebooks have `iterations=1, parallell_jobs=24`.
- DTW: traj-dist Cython, Euclidean distance on raw (lat, lon) degrees
- All result CSVs are local only. `*.csv` is in `.gitignore`, so they are not committed.
- The notebook edits used for these runs were temporary and were reverted with `git restore computation/`.
  The settings below are enough to recreate them.

---

## 1. Preprocessing (2026-11-10)

**What and why:** the raw Porto and Rome datasets are filtered and cut to a bounding box. The cleaned
trajectories are then written to one file each. The thesis uses fixed subsets listed in `META-n.txt` files,
so I must end up with exactly the same trajectories, or nothing is comparable.

**Configuration**
- Notebooks: `dataset/preprocessing/porto/porto_clean_data.ipynb`, `dataset/preprocessing/rome/rome_clean_data.ipynb`
- Fixed in commit 4ed7949: paths are built from the project root, and `create_meta_files()` is disabled so the
  committed META files are kept. The filtering logic is unchanged.
- Raw data: `dataset/porto/porto_raw.csv`, `dataset/rome/rome_raw.txt`

**Result**
- Porto: 3000 trajectories. Rome: 3059 trajectories.
- Every name in META-3000 (Porto) and META-3050 (Rome) exists → **same data as the original authors**.

---

## 2. DTW ground truth (2026-11-10)

**What and why:** to judge any filter, you need to know which pairs are *really* similar. The thesis computes
exact DTW for all pairs and calls a pair similar if DTW < 0.1 (sec. 4.4.1). Every recall and precision number
below is measured against this matrix.

**Configuration**
- Notebook: `computation/similarity_values/true/compute_true_dtw_similarity_values.ipynb`
- Changed: `SIZES = [500]` in both the Rome cell and the Porto cell. Note: *not* `np.arange(500)`, which looks for META-0.
- Output: `results_true/similarity_values/{porto,rome}/dtw/{city}-dtw-500.csv`
  (lower-triangular matrix; the consumers make it symmetric with `df + df.T`)

**Sanity checks passed:** 500 × 500, names match META-500, zero diagonal, lower-triangular storage.

**Result**

| | Porto | Rome |
|---|---|---|
| DTW min / median / max | 0.0120 / 1.0256 / 34.4564 | 0.0277 / 2.1393 / 28.9422 |
| Pairs < 0.1 | **581** (0.466 %) | **46** (0.037 %) |
| Pairs < 0.2 | 2196 (1.760 %) | 310 (0.248 %) |
| Pairs < 0.3 | 5232 (4.194 %) | 921 (0.738 %) |
| Pairs < 0.4 | 9940 (7.968 %) | 1862 (1.493 %) |
| Pairs < 0.5 | 16155 (12.950 %) | 3175 (2.545 %) |

Total pairs: 124 750 per city.

**What it shows**
- Rome has ~13× fewer similar pairs at θ = 0.1. One pair is ~2 percentage points of recall, so Rome results are noisy.
- DTW is a *sum* over matched points, and Rome trajectories are ~1.9× longer on average (n = 500: mean 110 vs 58
  points; median 79.5 vs 50).
  The same θ is therefore effectively stricter in Rome. **θ in raw degrees does not mean the same thing across datasets.**
- Similar pairs grow ~n², so larger n gives more stable Rome numbers.

---

## 3. Grid scheme, strict bucketing (2026-11-10 / 2026-11-12)

**What and why:** grid + strict is the thesis's best combination (sec. 5.1.1, 5.2.1). This sweep checks that
resolution and layers affect recall and precision the way the thesis describes. Their full sweep covers
res 0.01–3.0 and up to 50 layers; mine is a small subset.

**Configuration**
- Notebook: `computation/similarity_values/hashed/bucketing_compute_grid_similarity_values.ipynb`
- `BUCKETING_METHOD = "original"`, `MEASURE = "dtw"`
- Porto: `CITY = "porto"`, `RESOLUTION_VALUES = [0.5, 1.0, 1.6]`, `LAYERS_VALUES = [1, 3, 5]`, then a separate run with
  `RESOLUTION_VALUES = [1.6]`, `LAYERS_VALUES = [25]` (see section 6)
- Rome: `CITY = "rome"`, `RESOLUTION_VALUES = [0.5, 1.0, 1.4, 1.6]`, `LAYERS_VALUES = [1, 3, 5, 25]`
- Output: `results_hashed/bucket_evaluation/original/`
  - Porto sweep: `porto_dtw_grid_500_original_sweep.csv` (renamed)
  - Porto 25 layers: `porto_dtw_grid_500_original.csv`
  - Rome: `rome_dtw_grid_500_original.csv`

**Result — Porto, θ = 0.1** (581 true pairs; selected rows)

| Res (km) | Layers | Precision | Recall | F1 | ≈ Candidate pairs (ratio) |
|---|---|---|---|---|---|
| 0.5 | 1 | 0.938 | 0.069 | 0.128 | ~40 (0.03 %) |
| 0.5 | 5 | 0.943 | 0.192 | 0.319 | ~120 (0.09 %) |
| 1.0 | 1 | 0.480 | 0.325 | 0.365 | ~390 (0.3 %) |
| 1.0 | 3 | 0.365 | 0.661 | **0.458** | ~1 050 (0.8 %) |
| 1.0 | 5 | 0.315 | 0.781 | 0.444 | ~1 440 (1.2 %) |
| 1.6 | 3 | 0.089 | 0.919 | 0.162 | ~6 000 (4.8 %) |
| 1.6 | 5 | 0.074 | 0.973 | 0.138 | ~7 600 (6.1 %) |
| 1.6 | 25 | 0.046 | **1.000** | 0.088 | ~12 600 (10.1 %) |

**Result — Rome, θ = 0.1** (46 true pairs; selected rows)

| Res (km) | Layers | Precision | Recall | F1 | ≈ Candidate pairs (ratio) |
|---|---|---|---|---|---|
| 0.5 | 25 | 0.983 | 0.187 | **0.313** | ~9 (0.01 %) |
| 1.0 | 3 | 0.183 | 0.492 | 0.266 | ~120 (0.1 %) |
| 1.0 | 25 | 0.112 | 0.831 | 0.197 | ~340 (0.3 %) |
| 1.4 | 5 | 0.024 | 0.878 | 0.046 | ~1 700 (1.3 %) |
| 1.4 | 25 | 0.016 | 0.999 | 0.031 | ~2 900 (2.3 %) |
| 1.6 | 25 | 0.011 | **1.000** | 0.022 | ~4 200 (3.4 %) |

**What it shows**
- **Same trends as the thesis:** larger cells → recall up, precision down; more layers → recall up, precision down.
  Layers act as an OR: a pair only needs to match in one layer.
- Porto does better than Rome, as the thesis also reports.
- **Best F1 at θ = 0.1:** Porto 0.458, Rome 0.313. The thesis reports a best layer-averaged F1 of 0.54 (Porto) / 0.33 (Rome),
  and a best single config of 0.65 (Porto, res 0.6, 50 layers). Those configs were not in my sweep.
- **Recall 1 is expensive:** it takes 25 layers at coarse resolution and keeps ~10 % (Porto) / ~2–3 % (Rome) of all pairs.
- The "Avg Correlation Coefficient" column is 1.000 everywhere; it is not meaningful here.

---

## 4. Grid scheme, loose bucketing (2026-11-10 / 2026-11-12)

**What and why:** loose bucketing is the thesis's alternative strategy (sec. 5.1.1, 5.4). It is the opposite
extreme of strict: one shared cell is enough to become candidates.

**Configuration**
- Same notebook as section 3, with `BUCKETING_METHOD = "loose"`
- Both cities: `RESOLUTION_VALUES = [0.5, 1.0, 1.6]`, `LAYERS_VALUES = [1, 3, 5]`
- Output: `results_hashed/bucket_evaluation/loose/{porto,rome}_dtw_grid_500_loose.csv`

**Result — θ = 0.1**

| | Recall | Precision | F1 | ≈ Candidate ratio |
|---|---|---|---|---|
| Porto (all configs) | 1.000 | 0.006–0.010 | ≤ 0.020 | ~47–78 % of all pairs |
| Rome (all configs) | 1.000 | 0.000–0.001 | ≤ 0.002 | at least ~25 % (too rounded to compute exactly) |

**What it shows**
- Loose never misses a pair, but it keeps most pairs, so it hardly filters at all.
- The thesis gets slightly better loose precision (Porto 0.018 at res 0.04, Tab 5.4.2) by using much finer cells.
  That is consistent with my numbers.

---

## 5. Disk scheme, strict bucketing (2026-11-10 / 2026-11-12)

**What and why:** the disk scheme is the thesis's second LSH scheme (sec. 5.1.2). The thesis finds it worse
than the grid. I only ran a tiny test to check that the code works and that the ranking holds.

**Configuration**
- Notebook: `computation/similarity_values/hashed/bucketing_compute_disk_similarity_values.ipynb`
- `BUCKETING_METHOD = "original"`, `CITY = "porto"` / `"rome"`
- `bulk_groups` replaced by one group: `{"name": "TEST", "DIAMETER_VALUES": [0.6, 1.0], "LAYERS_VALUES": [3], "DISKS_VALUES": [50]}`
- Output: `results_hashed/bucket_evaluation/original/{porto,rome}_dtw_disk_500_original.csv`

**Result — θ = 0.1** (3 layers, 50 disks)

| City | Diameter (km) | Precision | Recall | F1 | ≈ Candidate pairs (ratio) |
|---|---|---|---|---|---|
| Porto | 0.6 | 0.098 | 0.693 | 0.169 | ~4 100 (3.3 %) |
| Porto | 1.0 | 0.148 | 0.640 | 0.232 | ~2 500 (2.0 %) |
| Rome | 0.6 | 0.019 | 0.701 | 0.036 | ~1 700 (1.4 %) |
| Rome | 1.0 | 0.065 | 0.505 | 0.112 | ~360 (0.3 %) |

**What it shows**
- **Disk is worse than grid**, as in the thesis. For similar recall, disk needs 2–4× more candidate pairs than grid 1.0 km / 3 layers
  (e.g. Porto disk 0.6 km: ~4 100 candidates at recall 0.693, vs grid: ~1 050 at recall 0.661).
- In both cities the larger disk gave *lower* recall. A likely cause: with 50 overlapping disks, a trajectory
  falls in more disks, so its hash sequence is longer and exact matches are rarer. Not verified.
- Too small a sweep to compare the schemes fairly. Disk is not my main baseline anyway.

---

## 6. The thesis's "recall = 1" configurations (2026-11-12)

**What and why:** the thesis only measures runtime for configurations that reach **recall = 1**, so that no
similar pair is lost. Among those, it picks the fastest (sec. 4.4.2, 5.5). These configurations are their
headline results, so I reproduced them exactly.

**Configuration**
- Grid strict, θ = 0.1, same notebook as section 3
- Porto: `RESOLUTION_VALUES = [1.6]`, `LAYERS_VALUES = [25]` (thesis Tab 5.5.2)
- Rome: res 1.4, 25 layers, included in the Rome sweep of section 3 (thesis Tab 5.5.5)

**Result**

| | Porto mine | Porto thesis | Rome mine | Rome thesis |
|---|---|---|---|---|
| Recall | 1.000 | 1 | 0.999 | 1 |
| Precision | 0.046 | 0.045 | 0.016 | 0.015 |
| # Buckets | 4 297 | 4 276 | 7 948 | 7 902 |
| # Buckets > 1 trajectory | 1 747 | 1 743 | 2 060 | 2 057 |
| Largest bucket | 72.7 | 75 | 23.5 | 24 |
| ≈ Candidate ratio | 10.1 % | 10.3 % | 2.3 % | 2.5 % |

**What it shows**
- **Reproduced:** all values agree within ~1–3 %, except Rome's precision (0.016 vs 0.015) and candidate ratio
  (2.3 % vs 2.5 %), which differ by ~7–8 % on paper. Both precisions are rounded to 3 decimals, so this is within
  rounding. The small differences come from random grid offsets.
- Rome's 0.999 is about one missed pair in one of the 24 runs. That is normal randomness with only 46 true pairs.

---

## 7. Runtime: all-to-all DTW vs. grid strict (2026-11-10 / 2026-11-12)

**What and why:** the point of bucketing is speed. The thesis compares the total time of
hashing + bucketing + DTW on candidates with exact DTW on all pairs, and reports the **speedup**
(sec. 5.5, Tab 5.5.1 / 5.5.4 / 5.5.7 / 5.5.8).

**Configuration — all-to-all DTW**
- Notebook: `computation/runtimes/true/compute_true_runtimes.ipynb`
- `CITY = "porto"` / `"rome"`, `MEASURE = "true_dtw_cy"` (default is Fréchet!), `DATA_SIZES = [500]`, `PARALLEL_JOBS = 8`, `ITERATIONS = 3`
- Each process computes the full 500 × 500 matrix on one core; the time is the average of 24 runs.
- Output: `results_true/runtimes/{porto,rome}/{city}_true_similarity_runtimes_measure_true_dtw_cy_sizes_500_500.csv`

**Configuration — grid strict with bucketing**
- Notebook: `computation/runtimes/hashed/compute_hashed_bucketing_runtimes.ipynb`, **Grid section only**
  (run cell 1, then cells 9, 11, 12; skip the Disk cells 4–7)
- Cell 9: `MEASURE = "grid_dtw_cy"`, `CITY = "porto"` / `"rome"`, `DATA_SIZE = [500]`, `BUCKETING_METHOD = "original"`,
  `TRUE_TRAJECTORIES = True`, `PARALLEL_JOBS = 8`, `ITERATIONS = 3`
- Cell 12: the filter picks the recall = 1 config from section 6:
  - Porto: `bucket_evaluation_df[(bucket_evaluation_df["Threshold"] == 0.1) & (bucket_evaluation_df["Layers"] == 25)]`
  - Rome: the same, plus `& (bucket_evaluation_df["Resolution"] == 1.4)`
- Output: `results_hashed/runtimes/bucketing/{city}/true_trajectories/original/dtw/runtimes_bucketing(original)_(true_trajectories)_{city}_dtw_[500]_grid.csv`
  (the Porto file also has two earlier test rows, res 1.6 with 5 and 3 layers, and a repeated header line)

**Result** (seconds)

| | Porto mine | Porto thesis | Rome mine | Rome thesis |
|---|---|---|---|---|
| Config | res 1.6, 25 layers | same | res 1.4, 25 layers | same |
| DTW on candidates | 3.904 | 3.449 | 4.475 | 3.463 |
| Hashing | 0.943 | 1.054 | 2.120 | 1.803 |
| Bucket distribution | 0.036 | 0.049 | 0.057 | 0.067 |
| **Total** | **4.883** | **4.552** | **6.652** | **5.333** |
| All-to-all DTW | 40.998 | 34.38 | 176.115 | 122.90 |
| **Speedup** | **8.4×** | 7.55× | **26.5×** | 23.05× |

Earlier Porto test (not a thesis config): res 1.6 with 5 layers took 2.371 s (recall 0.973); with 3 layers, 2.059 s (recall 0.919).

**What it shows**
- **Reproduced:** same time breakdown and similar speedups.
- My laptop is 1.1–1.4× slower than IDUN in absolute terms (Porto: 1.07× total, 1.19× all-to-all; Rome: 1.25× total,
  1.43× all-to-all). Speedups are therefore only
  valid when both times come from the same machine. Never divide my times by thesis times.
- DTW on candidates is 67–80 % of total time and hashing 19–32 %; bucket distribution is negligible.
- Cost per DTW pair:
  - on candidate pairs (DTW time ÷ derived candidate count): Porto ≈ 0.31 ms, Rome ≈ 1.56 ms (5.0×);
  - on all pairs (all-to-all time ÷ 124 750): Porto ≈ 0.33 ms, Rome ≈ 1.41 ms (4.3×). The plan uses these.
  - The code computes each candidate pair only once, even if it shares several buckets, so both are valid.
  - DTW cost per pair ∝ length_i × length_j, so the mean lengths matter: (110 / 58)² ≈ 3.6× explains most of the
    difference. Hashing per layer: Porto ≈ 0.04 s, Rome ≈ 0.085 s.
- Rome's speedup is larger because it keeps far fewer candidates (2.3 % vs 10 %) and each avoided pair is more expensive.

---

## Overview: thesis vs. my reproduction

| What | Thesis | Mine | Verdict |
|---|---|---|---|
| Same trajectories (META files) | — | all names present | ✅ |
| Grid strict trends (resolution, layers) | yes | yes | ✅ |
| Porto better than Rome | yes | yes | ✅ |
| Loose: recall 1, precision ≈ 0.01 | yes | yes | ✅ |
| Disk worse than grid | yes | yes | ✅ |
| Recall = 1 config, Porto (res 1.6, 25 layers) | P 0.045 | P 0.046 | ✅ |
| Recall = 1 config, Rome (res 1.4, 25 layers) | P 0.015 | P 0.016 | ✅ |
| Speedup, Porto | 7.55× | 8.4× | ✅ similar |
| Speedup, Rome | 23.05× | 26.5× | ✅ similar |
| Best single strict grid F1 (Porto res 0.6, 50 layers: 0.65) | 0.65 | — | not tested |
| Results for n = 1000–2000 | reported | — | not tested |

---

## Summary: what the reproduction shows and what it means for my thesis

**The baseline is reproduced.** On my laptop, with Python 3.14 and the fixed preprocessing, the code produces:
- the same trajectory subsets as the original authors;
- recall, precision and bucket statistics within a few percent of the thesis;
- the same speedups for the configurations the thesis presents as its main results.

I can therefore use this code and these numbers as the baseline for my own method. Two things were not checked:
- the best-F1 configuration with 50 layers;
- dataset sizes above 500.

Both are optional and easy to add later with the same notebooks.

**What the baseline actually does well and badly**
- **Strict bucketing** is precise but brittle. Two trajectories must produce *exactly* the same cell sequence
  in some layer. Near-identical trips that differ by a single cell are missed. The only way to recover
  them is many layers and large cells, which also pulls in many dissimilar pairs.
- **Loose bucketing** has the opposite problem. Sharing one cell is enough, so almost every pair in a city becomes
  a candidate, and the filtering effect is close to zero.
- There is nothing in between: either all of the sequence must match, or just one cell. **This gap is exactly what my
  method targets.** Shingles over the collapsed cell sequence + MinHash measure *how much* of the route is shared
  (Jaccard similarity), and LSH turns that into a tunable threshold.

**Targets to beat** (n = 500, θ = 0.1, grid strict, fastest recall = 1 config, measured on my laptop):

| | Porto | Rome |
|---|---|---|
| Recall | 1.000 | 0.999 |
| Candidate ratio | ~10 % | ~2.3 % |
| Total time (hash + bucket + DTW) | 4.88 s | 6.65 s |
| All-to-all DTW | 41.0 s | 176.1 s |
| Speedup | 8.4× | 26.5× |

My method is better than the baseline if, at recall ≈ 1:
- it keeps clearly fewer candidates than ~10 % (Porto) / ~2.3 % (Rome); and
- its total time (including shingling and MinHash) is lower than 4.88 s / 6.65 s.

The candidate ratio is the key number: DTW on candidates dominates runtime, so fewer candidates means faster.

**Things to keep in mind for my own experiments**
- **Use the same evaluation rules as the thesis**, so the comparison is fair:
  - θ = 0.1;
  - recall = 1 as the requirement for runtime results;
  - fastest qualifying config;
  - 24 runs;
  - the same machine and the same parallel setup (8 jobs × 3 iterations).
- **Report the candidate ratio directly.** The original code only reports precision and recall, so I had to
  derive it. A pair-based evaluator should output recall, precision, F1, candidate ratio and the runtime of
  each stage.
- **Rome is statistically weak at n = 500**, with only 46 true pairs. Consider also reporting n = 1000–2000,
  or more thresholds.
- **θ = 0.1 in raw degrees does not transfer across datasets**, because DTW grows with trajectory length. GeoLife will need its
  own justified threshold, e.g. chosen by visual inspection as in the thesis, or a percentile of DTW values.
  Discuss this in the thesis.
- **Hashing is not free.** It is 20–30 % of the baseline's total time, and Rome's longer trajectories double it.
  MinHash with many permutations could cost more than grid hashing, so measure it separately.

---

# Pilot: do DTW-similar pairs have higher shingle similarity? (2026-11-12)

**What and why:** my method assumes that trajectories that are close in DTW also share many ordered cell
shingles. If that is false, MinHash-LSH on shingles cannot work as a filter. This pilot tests the assumption
*before* the semester. It uses the n = 500 ground truth from section 2 and computes the **exact Jaccard
similarity** of the shingle sets for all 124 750 pairs. There is no MinHash or LSH yet, so this is the best case
that MinHash-LSH can only approximate. It is the "pilot test" from my project plan.

**Configuration**
- Notebook: `pilot/shingle_pilot.ipynb`; only `CITY` in cell 3 changes between runs
- `SIZE = 500`, `THETA = 0.1`, ground truth `results_true/similarity_values/{city}/dtw/{city}-dtw-500.csv`
- Pipeline per trajectory: GPS points → cell ids → collapse consecutive repeats → set of ordered k-shingles
  (a sequence shorter than k becomes a single shingle)
- Cells:
  - H3 resolutions 7, 8, 9 (edge ≈ 1.4 / 0.5 / 0.2 km), via `h3.latlng_to_cell`
  - Square cells of 1.0 / 0.5 / 0.25 km, using a local flat-earth projection around the mean latitude
  - **Fixed** cell borders: no random offsets and no layers
- k = 1 (just the set of visited cells), 2, 3
- Metrics per configuration:
  - **AUC:** chance that a random similar pair has a higher Jaccard than a random other pair
  - **Spearman(J, DTW)**
  - **Similar with J=0:** similar pairs that share no shingle
  - **Cand. ratio recall=1:** share of all pairs with Jaccard ≥ the lowest Jaccard among similar pairs
  - **Cand. ratio recall=0.95:** the same, using the 5th percentile instead of the minimum
- Output (local): `results_pilot/shingle_pilot_{porto,rome}_500.csv` and `.png` (histogram + DTW-vs-Jaccard scatter
  for the configuration with the lowest candidate ratio at recall 1)

**Result — Porto** (581 similar pairs; median 50, mean 58 points per trajectory)

| Cells | k | Cells/traj | AUC | Spearman | Similar with J=0 | Cand. ratio recall=1 | Cand. ratio recall=0.95 |
|---|---|---|---|---|---|---|---|
| H3 7 | 1 | 2 | 0.886 | −0.64 | 0 | 59.0 % | 59.0 % |
| H3 8 | 1 | 5 | 0.983 | −0.69 | 0 | 22.9 % | 8.2 % |
| H3 8 | 2 | 5 | 0.981 | −0.43 | 7 | 100 % | 5.3 % |
| H3 9 | 1 | 11 | 0.996 | −0.64 | 0 | 13.1 % | 2.0 % |
| H3 9 | 2 | 11 | 0.994 | −0.47 | 0 | 12.3 % | 3.2 % |
| H3 9 | 3 | 11 | 0.985 | −0.37 | 8 | 100 % | 5.4 % |
| Square 1.0 | 1 | 5 | 0.983 | −0.70 | 0 | 14.6 % | 9.6 % |
| Square 0.5 | 1 | 8 | 0.994 | −0.65 | 0 | 9.4 % | 2.9 % |
| Square 0.5 | 2 | 8 | 0.993 | −0.46 | 1 | 100 % | 2.7 % |
| **Square 0.25** | **1** | 14 | **0.997** | −0.62 | 0 | **8.1 %** | 1.9 % |
| Square 0.25 | 2 | 14 | 0.995 | −0.48 | 1 | 100 % | **1.8 %** |
| Square 0.25 | 3 | 14 | 0.990 | −0.38 | 5 | 100 % | 2.9 % |

All 18 rows are in the CSV. With H3 7, k = 2 and 3 lose 127 and 265 similar pairs completely.

**Result — Rome** (46 similar pairs; median 79.5, mean 110 points per trajectory)

| Cells | k | Cells/traj | AUC | Spearman | Similar with J=0 | Cand. ratio recall=1 | Cand. ratio recall=0.95 |
|---|---|---|---|---|---|---|---|
| H3 7 | 1 | 2 | 0.959 | −0.48 | 0 | 23.7 % | 14.5 % |
| H3 8 | 1 | 6 | 0.996 | −0.57 | 0 | 4.3 % | 1.7 % |
| H3 8 | 2 | 6 | 0.997 | −0.39 | 0 | 2.2 % | 1.6 % |
| H3 9 | 1 | 15 | 0.999 | −0.56 | 0 | 0.77 % | 0.63 % |
| **H3 9** | **2** | 15 | **0.999** | −0.43 | 0 | **0.53 %** | **0.37 %** |
| H3 9 | 3 | 15 | 0.999 | −0.33 | 0 | 0.99 % | 0.40 % |
| Square 1.0 | 1 | 6 | 0.987 | −0.57 | 0 | 9.2 % | 6.2 % |
| Square 0.5 | 1 | 11 | 0.997 | −0.57 | 0 | 2.0 % | 1.4 % |
| Square 0.25 | 1 | 19 | 0.999 | −0.55 | 0 | 0.63 % | 0.55 % |
| Square 0.25 | 2 | 19 | 0.999 | −0.44 | 0 | 0.55 % | 0.49 % |
| Square 0.25 | 3 | 19 | 0.997 | −0.34 | 0 | 1.4 % | 0.99 % |

Computing cells and exact Jaccard for all pairs took < 0.05 s per configuration.

**Comparison with the baseline** (candidate ratio at recall 1, n = 500, θ = 0.1)

| | Baseline: grid strict, fastest recall = 1 config (section 6) | Pilot: best exact-Jaccard config |
|---|---|---|
| Porto | ~10.1 % (res 1.6 km, 25 layers) | **8.1 %** (square 0.25 km, k = 1); 1.8–1.9 % at recall 0.95 |
| Rome | ~2.3 % (res 1.4 km, 25 layers) | **0.53 %** (H3 9, k = 2), about 4× fewer candidates |

**What it shows**
- **The assumption holds.** With fine cells (H3 9 or 0.25 km squares) and k ≤ 2, AUC is ≥ 0.99 in both cities
  (Porto H3 9 with k = 3 is 0.985): similar
  pairs almost always have a higher Jaccard than other pairs. The plots show two clearly separated groups. Most other
  pairs have Jaccard ≈ 0, and the similar pairs lie mostly between 0.35 and 1.
- **Finer cells are better**, and the finest cells tested were the best in both cities. The optimum may be finer still
  (H3 10, 0.1 km squares). Coarse cells (H3 7, 1 km) give only 2–6 cells per trajectory, which is too few to tell
  routes apart.
- **Order (k ≥ 2) helps only when sequences are long enough.**
  - In Rome (15–19 cells per trajectory), k = 2 is best.
  - In Porto (shorter trips), k ≥ 2 often leaves some similar pairs with **no shared shingle at all**. Then recall 1 is
    impossible without keeping every pair.
  - k = 3 is worse than k = 2 almost everywhere. The one exception is Rome with 0.5 km squares at recall 1
    (5.2 % vs 5.6 %), where k = 1 is best anyway (2.0 %).
  - So k should match trajectory length, or several k should be combined.
- **In Porto, recall = 1 is decided by a few outlier pairs.** Keeping 95 % of similar pairs needs only ~1.8 % of all
  pairs, but the last 5 % pushes the ratio to 8 %. In the scatter plot these are pairs just under DTW 0.1 with
  Jaccard ≈ 0.2. Likely causes are fixed cell borders (a route running along a border flips between cells) and the
  hard θ cut-off. Random offsets, several layers or multi-resolution tokens (variants in my plan) may rescue them.
  Worth inspecting those pairs on a map.
- Spearman is only moderate (−0.3 to −0.7 in the tables) because most dissimilar pairs share nothing (Jaccard = 0), so their order
  is arbitrary. That does not matter for filtering. AUC and candidate ratio are the relevant metrics.

**Caveats**
- **Exact Jaccard is the best case.** MinHash-LSH has an S-shaped collision curve, so to keep pairs with Jaccard as low
  as ~0.2 (Porto) or ~0.33 (Rome) it needs a low threshold and a margin. The real candidate ratio at recall ≈ 1 will
  be higher than these numbers.
- These are single fixed-grid runs, not averages over 24 random offsets as in the baseline.
- n = 500 only; Rome has 46 similar pairs.

**Conclusion for the thesis:** the core idea is supported. Ordered shingles over fine cells separate DTW-similar pairs
well. Even the exact-Jaccard best case beats the baseline's candidate ratio by ~4× in Rome but only slightly in Porto at
recall 1. The Porto gap comes from a handful of hard pairs, which makes them an interesting target for the variants
(random offsets, multi-resolution, combined k). Things to test in the semester:
- finer cells;
- whether MinHash-LSH can get close to these exact numbers;
- whether the variants close the gap at recall 1.
