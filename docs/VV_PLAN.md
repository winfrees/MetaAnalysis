# MetaAnalysis: verification and validation program

This program shows, with evidence, that MetaAnalysis computes what it claims to, and where
it cannot be trusted. It follows the approach of the IBIF napari V&V suite
([docs/vv](https://github.com/CU-IBIF/Napari_Cellpose/tree/main/docs/vv)). Data with known
truth is generated, the tool's own code runs on it, results are compared with the truth or
with an independent calculation, and a report explains every requirement with its evidence.
Two things are new here:

- The truth is at the **atlas level**: cohorts, donors and effects planted between them, not
  only objects in one image.
- **Statistical behaviour** is verified (false positive rate, power and confidence interval
  coverage over many simulated studies), not only single numbers.

*Verification* checks the tool against an independent calculation of the same thing
("built right"). *Validation* checks it against the known truth or a real-world reference,
within a tolerance ("right thing built").

Contents

1. [Organisation](#1-organisation)
2. [The synthetic atlas (truth)](#2-the-synthetic-atlas-truth)
3. [Requirements and acceptance criteria](#3-requirements-and-acceptance-criteria)
4. [Test levels](#4-test-levels)
5. [Real-data validation](#5-real-data-validation)
6. [Traceability, reports and release gates](#6-traceability-reports-and-release-gates)
7. [Change control and regression](#7-change-control-and-regression)
8. [V&V schedule by milestone](#8-vv-schedule-by-milestone)

---

## 1. Organisation

| Item | Where |
|---|---|
| Requirements (the single source) | `src/metaatlas/vv/requirements.py`; `docs/vv/requirements.md` is generated from it (`metaatlas-vv requirements`) |
| Truth generators | `src/metaatlas/vv/synth/` (atlas, manifests, object tables, images via `ibif_napari.vv.synth`, omics matrices) |
| Checks | `src/metaatlas/vv/checks/` (one function per requirement, returns metrics + pass/fail per criterion) |
| CLI | `metaatlas-vv generate / run / report / all / requirements`, with `--quick`, `--only`, `--limits`, `--strict`, `--backend`, `--seed` (same meaning as in `ibif-napari-vv`) |
| Tests of the suite itself | `tests/test_vv.py` |
| CI | `.github/workflows/vv.yml`: quick suite on every pull request; full suite nightly and before every release |
| Results | `results/report.html`, `report.md`, `results.json`, `tables/*.csv`, `figures/*.png` |

**Roles.** The developer writes the requirement and its check together with the feature.
A second person (developer or statistician) reviews each requirement's criteria before it
is accepted. The statistician signs off on VV-STAT and VV-INT. Test users are recruited for
VV-UX.

**Risk-based depth.** Each requirement has a risk class. Higher classes get more
independent evidence:

| Class | Failure would… | Evidence required |
|---|---|---|
| **A** | silently produce a wrong scientific result (wrong donor, wrong channel, wrong statistic) | Independent re-implementation or reference tool, synthetic truth, and real-data check |
| **B** | cause visible failure or lost work (crash, failed run, wrong display) | Automated test with known truth |
| **C** | inconvenience (layout, wording) | Test or inspection |

---

## 2. The synthetic atlas (truth)

`metaatlas-vv generate` writes a complete atlas from fixed seeds, so it is identical on
every computer and needs no storage. It has three tiers, used at different speeds:

### Tier S1: metadata and object tables (fast; seconds)

1. **Design.** Cohorts (e.g. Reference, Disease A, Disease B), donors per cohort,
   covariates (age, sex) and sites, with a controllable **confounding** between site and
   cohort.
2. **Donor biology.** Each donor's true cell type proportions are drawn from a
   Dirichlet with cohort-specific means. Planted effects: for example, type T is ×1.5 in
   Disease A, age has a slope on type U, and site shifts marker M by +10%. The donor random
   effect variance is set.
3. **Specimens and datasets.** 1 to 3 specimens per donor, 1 to 4 regions, and datasets in
   several modalities. The number of cells per dataset is drawn from realistic ranges.
4. **Objects.** Every cell has a type, a position (from a spatial process with planted
   neighbourhood preferences, e.g. macrophages near injured PT) and features drawn from
   type × donor × site distributions.
5. **Manifests.** A valid manifest, and variants with planted errors (missing donor,
   duplicate ID, wrong unit, unknown ontology term, checksum mismatch, identifier-like
   field).
6. **Truth files.** `truth/atlas.json` (every parameter and planted effect),
   `truth/donors.csv` (true proportions and means per donor), `truth/effects.csv` (true
   contrasts per outcome).

### Tier S2: images (slow; minutes)

A subset of datasets are rendered as microscope images with `ibif_napari.vv.synth`
(phantom channels, PSF, noise and anisotropic voxels), with marker channels whose
intensities follow the S1 cell types. The truth labels are saved too. This tests the
**full chain**: image → pipeline → objects → donor aggregates → statistics, and shows
how much segmentation error weakens the detection of planted effects.

### Tier S3: other modalities

The same donors as count matrices (AnnData) for sn / scRNA-seq, with known
modality-specific biases (e.g. a type lost in dissociation at a known rate), and spot-level
mixtures for spatial transcriptomics. This tests integration and concordance.

### Back-ends (as in ibif-napari-vv)

| Back-end | What runs | Purpose |
|---|---|---|
| `truth` | An oracle pipeline that returns the true objects | Self-check: everything downstream must recover the planted values exactly or within sampling error |
| `classic` | IBIF napari threshold + watershed | Runs anywhere, no GPU or weights |
| `cellpose` | IBIF napari Cellpose | The real default pipeline |
| `vtea` | A vtea-core protocol | The vtea-core adapter |

---

## 3. Requirements and acceptance criteria

These are the first requirements. Limits are defaults and can be overridden with
`--limits`; the names are kept in `LIMITS` in `requirements.py`. "R reps" means the check
is repeated over R independently simulated atlases.

### Harness (VV-H)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-H-01 | The synthetic atlas generator produces what its parameters say | verification | A | Realised cohort sizes exact; mean donor proportions within 3 SE of the specified Dirichlet means; identical output for the same seed on Linux, macOS and Windows (SHA-256 of all files) |
| VV-H-02 | The suite's own metrics are correct | verification | A | Known-answer tests for coverage, FPR, power and CCC calculations match hand-computed values |

### Catalogue and data integrity (VV-CAT)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-CAT-01 | A valid manifest imports completely and correctly | verification | A | Every entity and field in the DB equals the manifest (100%); entity counts equal |
| VV-CAT-02 | Invalid manifests are rejected with the right reason | verification | A | Each of the planted errors is detected, with the correct row, column and error type (100%); no valid row rejected |
| VV-CAT-03 | Import is idempotent and versioned | verification | B | Re-importing the same manifest changes 0 rows; changing k rows creates exactly k new versions |
| VV-CAT-04 | File checksums detect corruption | verification | A | A single changed byte in any registered file is reported by `validate`; unchanged files never are |
| VV-CAT-05 | Queries and cohorts return exactly the right donors and datasets | verification | A | 200 random filter queries give the same IDs as an independent pandas evaluation (100%) |
| VV-CAT-06 | Releases are immutable | verification | A | Any write to a frozen release is refused; its catalogue hash is unchanged after later imports |
| VV-CAT-07 | Ontology terms are valid | verification | B | Every stored CL / UBERON term exists in the pinned ontology snapshot; unknown terms are rejected |

### Dataset adapters (VV-IO)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-IO-01 | IBIF napari workspaces and measurement tables are read correctly | verification | A | Object table values identical to the tables written by IBIF napari; units and voxel size identical to 1e-9 |
| VV-IO-02 | vtea-core exports and protocols are read correctly | verification | A | Values and data dictionary identical to vtea-core's export |
| VV-IO-03 | AnnData and SpatialData are read correctly | verification | A | Cell type labels, counts and coordinates identical to the source |
| VV-IO-04 | Features are matched by marker, not by position | verification | A | Datasets with permuted channel orders give identical harmonised tables |
| VV-IO-05 | Units are normalised | verification | A | Tables in px, nm and µm give the same µm values (to 1e-9 relative) |

### Pipelines and execution (VV-PIPE, VV-EXEC)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-PIPE-01 | A pipeline gives the same result in MetaAnalysis as in IBIF napari | verification | A | On S2 datasets: labels identical (same back-end, same seed); object tables equal to 1e-6 relative |
| VV-PIPE-02 | A pipeline gives the same result in MetaAnalysis as in vtea-core | verification | A | As VV-PIPE-01 for a `.vtea.json` protocol |
| VV-PIPE-03 | The full chain recovers the truth | validation | A | With the `cellpose` back-end on S2: per-dataset object count error ≤ 10%; type proportion error ≤ 0.05 absolute (limits follow IBIF napari's VV-SEG results) |
| VV-PIPE-04 | The input contract excludes datasets that do not fit | verification | A | Datasets with a missing marker, out-of-range voxel size or failed QC are excluded with the correct reason (100%); no fitting dataset excluded |
| VV-PIPE-05 | Pipeline hashing is canonical | verification | A | Reordered keys and equivalent number formats give the same hash; any changed parameter gives a different hash |
| VV-EXEC-01 | The cache re-runs exactly what changed | verification | A | Changing step k re-runs exactly steps ≥ k for affected datasets (task list equals expected); no change re-runs 0 tasks |
| VV-EXEC-02 | Killed runs resume correctly | verification | B | Killing workers at random points (20 trials): resumed results identical to an uninterrupted run; no partial file is ever read |
| VV-EXEC-03 | Executors give the same results | verification | A | Local, GPU-server and SLURM (fake SLURM in CI, real cluster before release) give identical tables for CPU back-ends; GPU vs CPU within documented tolerance |
| VV-EXEC-04 | Runs are reproducible | verification | A | Same experiment, seed and environment run twice → identical results.json (excluding timings) |
| VV-EXEC-05 | Provenance is complete | verification | A | Every result has inputs (with checksums), parameters, software versions, host, seed and time; a result can be re-created from its provenance record alone |

### Experiments and aggregation (VV-EXP, VV-AGG)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-EXP-01 | The plan selects exactly the right datasets | verification | A | Selected IDs equal the independent evaluation of the selection over the release (100%) |
| VV-EXP-02 | Sweeps expand correctly | verification | B | Number of variants and parameters per variant equal the expected Cartesian product |
| VV-EXP-03 | Batch confounding is detected | verification | A | Warning given when Cramér's V(site, cohort) ≥ 0.5 in planted designs, and not given when it is ≤ 0.1 |
| VV-AGG-01 | Donor-level aggregation is exact | verification | A | Counts, proportions and weighted means per donor equal an independent NumPy calculation (to 1e-12) |
| VV-AGG-02 | Each object is attributed to the right donor, specimen and region | verification | A | 100% on S1 with deliberately interleaved dataset IDs |
| VV-AGG-03 | Aggregation recovers the truth (oracle back-end) | validation | A | Donor proportions within the multinomial sampling 99% interval for ≥ 98% of donor × type pairs |

### Statistics (VV-STAT)

Evaluated over many simulated atlases (R reps), at the design sizes users will use (e.g.
8, 15 and 30 donors per cohort).

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-STAT-01 | Type I error is controlled under the null | validation | A | With no planted effect, R = 1000: rejection rate at α = 0.05 within [0.035, 0.065] for each default model (composition, feature, spatial) |
| VV-STAT-02 | No pseudo-replication | validation | A | With many cells per donor and donor variance > 0, the false positive rate stays within VV-STAT-01's bounds, while a deliberately naive cell-level test (run as a control) exceeds 0.2 |
| VV-STAT-03 | Confidence intervals have nominal coverage | validation | A | 95% CIs contain the true effect in [93%, 97%] of R = 1000 reps |
| VV-STAT-04 | Planted effects are detected with the expected power | validation | A | Power within ±0.07 of the analytically or simulation-derived target for the design; effect estimate bias < 10% of the effect |
| VV-STAT-05 | Batch adjustment works | validation | A | With a site effect and partial confounding, the adjusted estimate's bias < 10%, while the unadjusted estimate shows the planted bias |
| VV-STAT-06 | FDR is controlled | validation | A | With 50 outcomes, 10 truly non-null: empirical FDR ≤ 0.05 + 0.02 at q = 0.05 |
| VV-STAT-07 | Models agree with reference implementations | verification | A | On fixed fixtures: fixed effects within 1e-4 relative and p-values within 1e-3 of R lme4 / propeller; any disagreement explained |
| VV-STAT-08 | Model problems are reported | verification | B | Singular fits, non-convergence and < 3 donors per group produce a visible warning (100% of planted cases) |

### Integration (VV-INT)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-INT-01 | Cell type mapping is applied exactly | verification | A | Mapped labels equal the mapping table (100%); unmapped labels listed with reasons |
| VV-INT-02 | The donor × modality table is correct | verification | A | Every cell equals the independent calculation; missing modality is NA, never 0 |
| VV-INT-03 | Concordance recovers planted agreement and bias | validation | A | On S3: estimated Bland–Altman bias within 0.02 of the planted dissociation bias; CCC within 0.05 of its expected value |
| VV-INT-04 | Hierarchy coarsening is consistent | verification | A | Proportions at a parent equal the sum of its children's, for every donor |

### GUI and usability (VV-GUI, VV-UX)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-GUI-01 | Each GUI section calls the same API as the CLI | verification | B | pytest-qt: the results of each GUI action equal those of the corresponding API call |
| VV-GUI-02 | Drill-down opens the right data | verification | A | Clicking a donor point opens exactly that donor's datasets; "Open in IBIF napari" shows the dataset with the experiment's labels, at the right voxel size |
| VV-GUI-03 | Journeys J1 to J7 work end to end | validation | B | Scripted GUI runs of J1 to J7 on the synthetic atlas complete with the expected outputs |
| VV-UX-01 | Users can do the main tasks | validation | B | Usability study, ≥ 5 users (curator, analyst, biologist): ≥ 80% of tasks completed without help; SUS score ≥ 70; every severe issue fixed or documented |

### Performance, security and publication (VV-PERF, VV-SEC, VV-PUB)

| ID | Requirement | Kind | Class | Acceptance criteria |
|---|---|---|---|---|
| VV-PERF-01 | The catalogue is fast at atlas scale | verification | B | 500 donors / 5,000 datasets: any filter query < 1 s; coverage grid < 2 s |
| VV-PERF-02 | Aggregation scales | verification | B | 10⁸ objects aggregated to donors in < 10 min on one 16-core node, with memory < 32 GB |
| VV-PERF-03 | Planning is fast | verification | B | Planning a 5,000-dataset experiment with a 3-variant sweep < 30 s |
| VV-SEC-01 | No direct identifiers are stored | verification | A | Schema rejects fields that look like names, dates of birth, MRNs or addresses (planted cases, 100%); exports contain only pseudonymous IDs |
| VV-SEC-02 | Access control works (server mode) | verification | A | Each role can do exactly what its permissions allow (full matrix of role × endpoint) |
| VV-PUB-01 | A publication bundle is complete and re-runnable | verification | A | RO-Crate validates; results re-created from the bundle equal the originals |

---

## 4. Test levels

| Level | What | Tools | When |
|---|---|---|---|
| **Unit** | Functions and classes; property-based tests for hashing, aggregation and schema (Hypothesis) | pytest, Hypothesis | Every commit |
| **Integration** | Adapters against real IBIF napari / vtea-core code; DB on SQLite and PostgreSQL; fake SLURM (as IBIF napari's tests do) | pytest, testcontainers | Every pull request |
| **System (V&V suite)** | `metaatlas-vv` requirements on the synthetic atlas | metaatlas-vv | Quick on every PR; full nightly |
| **Statistical calibration** | VV-STAT with R = 1000 | metaatlas-vv on a cluster | Weekly and before every release |
| **GUI** | pytest-qt for each section; scripted journeys | pytest-qt, xvfb | Every PR (quick), nightly (journeys) |
| **Acceptance** | Journeys with real users; real-data validation (§5) | Study protocol, report | At M5, M7 and 1.0 |
| **Performance** | VV-PERF benchmarks, tracked over time | pytest-benchmark, asv | Nightly; regressions > 20% fail |
| **Fault injection** | Killed workers, full disk, network loss, corrupted files | Custom fixtures | Nightly |

Test coverage target: ≥ 90% lines for `catalog`, `engine`, `stats` and `integrate`;
≥ 75% overall.

---

## 5. Real-data validation

Synthetic truth cannot show that the tool works on real tissue. Before 1.0:

1. **Reproduce published results.** Choose 3 to 5 quantitative results from previous atlas
   publications (e.g. a cell type proportion difference between cohorts, a 3D cytometry
   abundance, an imaging vs snRNA-seq comparison). Re-derive them with MetaAnalysis from the
   original data. Acceptance: the same direction and significance, and effect sizes within
   the published CIs. Every difference is explained in the report.
2. **Independent re-analysis.** A pilot comparison is analysed independently by a
   statistician with their own scripts, without seeing the tool's output. Acceptance: the
   same conclusions; estimates within 5%.
3. **Expert review of drill-down.** For 20 randomly chosen donors, an expert opens the
   datasets behind the donor-level numbers and confirms that the segmentation and
   phenotypes are acceptable. The rate of unacceptable datasets is reported, with the
   effect of excluding them.
4. **Cross-site check.** The same pipeline run on data from two sites: the batch view and
   adjustment are reviewed with the site leads.

---

## 6. Traceability, reports and release gates

**Traceability matrix.** Generated from `requirements.py` and the test markers
(`@pytest.mark.req("VV-STAT-01")`):

```
Goal (G1–G7) → Requirement (VV-…) → Check / tests → Evidence (results.json, figures) → Release
```

Every goal has at least one requirement, every requirement has at least one check, and the
generator fails CI if any link is missing.

**The V&V report** (`report.html`, self-contained) contains:

- a summary table: requirement, class, PASS / FAIL / NOT RUN, with the key number;
- for each requirement: the statement, method, criteria, the measured values, figures (e.g.
  the FPR across reps with its binomial interval, CI coverage plots, power curves,
  agreement with R) and the per-row tables behind them;
- the environment (versions of MetaAnalysis, IBIF napari, vtea-core, Python, OS, GPU),
  seeds and limits used;
- known limitations: what the requirements do not cover.

**Release gates.**

| Release | Gate |
|---|---|
| Every merge to `main` | Unit + integration green; quick V&V suite: no class A failure |
| Milestone tag (0.x) | The milestone's requirements PASS in the full suite (`--strict` on those IDs); report attached to the GitHub release |
| 1.0 | All requirements PASS with `--strict`; VV-STAT calibration passed within the last 7 days; §5 real-data validation report signed off by the PI and the statistician |

A requirement may be **waived** only with a written justification in the report, an
issue to fix it, and sign-off. Class A requirements cannot be waived for 1.0.

---

## 7. Change control and regression

- Requirements and limits are code. Changing a limit needs a pull request with the reason
  and a reviewer, and the report records the limits used.
- The **upstream pins** (IBIF napari, vtea-core commits) are updated by pull request, and
  the full V&V suite must pass on the new pins. VV-PIPE-01/02 catch changes in their
  behaviour.
- **Golden results**: the full results.json for the synthetic atlas at each release is kept.
  Nightly runs are compared with it; a change in any class A number beyond tolerance opens
  an issue automatically.
- **Bugs found in use** get a regression test, and a new requirement where the bug shows a
  gap in the requirements.

---

## 8. V&V schedule by milestone

| Milestone | Requirements that must pass | V&V work in the milestone |
|---|---|---|
| M0 | — | Harness skeleton, requirements format, CI workflow, report template |
| M1 | VV-H-01, VV-CAT-01 to 07, VV-SEC-01 | Synthetic manifests (S1), planted-error manifests |
| M2 | VV-IO-01 to 05 | Fixtures written by real IBIF napari / vtea-core code |
| M3 | VV-PIPE-01, 02, 04, 05; VV-EXEC-01 to 05 | S2 image tier; oracle back-end; fake SLURM; fault injection |
| M4 | VV-EXP-01 to 03; VV-AGG-01 to 03 | Synthetic atlas S1 complete with planted effects |
| M5 | VV-H-02; VV-STAT-01 to 08; VV-PIPE-03 | Calibration runs on a cluster; R reference fixtures |
| M6 | VV-INT-01 to 04 | S3 tier (omics) |
| M7 | VV-GUI-01 to 03; VV-UX-01 | GUI journeys; usability study |
| M8 | VV-PERF-01 to 03; VV-SEC-02; VV-PUB-01; all with `--strict` | Scale data; real-data validation (§5); final report |
