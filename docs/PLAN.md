# MetaAnalysis: development plan

Contents

1. [Goals and scope](#1-goals-and-scope)
2. [Users and what they do](#2-users-and-what-they-do)
3. [Architecture](#3-architecture)
4. [Data model](#4-data-model)
5. [Dataset pipelines and comparison experiments](#5-dataset-pipelines-and-comparison-experiments)
6. [Statistics](#6-statistics)
7. [Multimodal integration](#7-multimodal-integration)
8. [GUI](#8-gui)
9. [Milestones and tasks](#9-milestones-and-tasks)
10. [Risks](#10-risks)
11. [Open decisions](#11-open-decisions)

The V&V program is in [VV_PLAN.md](VV_PLAN.md). What this plan needs from IBIF napari and
vtea-core is in [UPSTREAM.md](UPSTREAM.md).

---

## 1. Goals and scope

### Goals

| ID | Goal |
|---|---|
| G1 | **One catalogue for the whole atlas.** Every cohort, donor, specimen, tissue region and dataset, in every modality, is in one place, with its metadata, files, checksums and provenance. |
| G2 | **Two linked levels.** From any summary (a cohort, a donor, a cell type in a tissue region) you can reach the datasets behind it and open them in IBIF napari at the object level. From a dataset you can get back to its donor and cohort. |
| G3 | **Pipelines apply across cohorts.** A pipeline defined on one dataset (IBIF napari or a vtea-core protocol) runs on every matching dataset in chosen cohorts: locally, on the facility GPU server or on a SLURM cluster. It resumes after a failure, and it re-runs only what changed. |
| G4 | **Sound comparisons.** Cohorts are compared with the donor as the unit of replication. Covariates and batch (site, scanner, run) are modelled, multiple testing is corrected, and every result reports an effect size with a confidence interval. |
| G5 | **Multimodal integration.** Cell types are harmonised to one ontology across modalities, each donor gets one row of results across all modalities, and abundances measured in different modalities are checked against each other. |
| G6 | **Reproducible and citable.** Any figure can be traced back to its data, pipeline, parameters, software versions and random seeds. Results can be exported as a publication bundle (RO-Crate) that is readable without the tool. |
| G7 | **Trusted with evidence.** Every requirement has numeric acceptance criteria and is tested against synthetic truth or an independent calculation. Each release ships a V&V report, as IBIF napari does. |

### Not in scope for 1.0

- Re-implementing segmentation, measurement or cytometry. These come from IBIF napari
  and vtea-core.
- Primary single-cell RNA-seq processing (alignment, ambient RNA removal, doublet calls).
  The tool ingests processed AnnData / SpatialData objects and their cell type labels.
- A public web portal with anonymous access. The data model and API are designed so a web
  front end can be added later (M9).
- Clinical decision support.

---

## 2. Users and what they do

| User | Typical tasks |
|---|---|
| **Atlas curator** | Registers cohorts, donors, specimens and datasets from a manifest spreadsheet. Fixes metadata, checks completeness, freezes an atlas release. |
| **Imaging analyst** | Builds and tunes a pipeline on a few representative datasets in IBIF napari, then runs it across cohorts. Inspects outliers at the object level. |
| **Biologist / PI** | Asks questions such as "are injured proximal tubule cells more abundant in AKI than in reference tissue, adjusted for age and site?" Reads results, opens example donors and exports figures. |
| **Data scientist** | Uses the Python API and notebooks on the same catalogue. Adds statistical models or integration methods as plug-ins. |
| **Facility manager** | Runs the server and its executors, and manages access, storage and quotas. |

### Main user journeys (they become V&V acceptance scenarios)

- **J1 Ingest.** Import a manifest of 3 cohorts, 60 donors and 200 datasets in 4 modalities.
  See what failed validation and why, fix it, and re-import.
- **J2 Browse and drill down.** Atlas → cohort "AKI" → donor D-017 → specimen → cortex →
  3D cytometry dataset → open in IBIF napari with its segmentation and phenotypes.
- **J3 Define a pipeline.** In IBIF napari, build segmentation → secondary objects →
  measurements → phenotype tree on dataset X. Export it as a pipeline, and register it
  with a name and version.
- **J4 Run a comparison experiment.** Pipeline P v2 × cohorts {Reference, AKI, CKD}, only
  3D cytometry of cortex. Outcome: phenotype proportions and mean marker intensity per
  phenotype. Covariates: age, sex and site. Run on the cluster, watch progress, and see
  results arrive per donor.
- **J5 Inspect and iterate.** Find an outlier donor on a forest plot and open its datasets.
  The segmentation turns out to be wrong. Change one parameter (P v3) and re-run: only the
  affected steps run again. Compare the v2 and v3 results.
- **J6 Integrate.** Compare the proportion of a cell type from imaging with the proportion
  from snRNA-seq, per donor. Build a donor × modality feature table.
- **J7 Publish.** Freeze the experiment and export the figures, tables, pipeline, data
  dictionary and provenance as an RO-Crate.

---

## 3. Architecture

### Layers

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ GUI (Qt; docks in napari or runs alone)                                       │
│  Atlas browser · Cohort builder · Pipeline registry · Experiment designer     │
│  Run monitor · Results explorer · Integration view · "Open in IBIF napari"    │
├──────────────────────────────── Python API (metaatlas) ───────────────────────┤
│ Service layer: catalogue queries, experiment orchestration, statistics, export│
├──────────────┬───────────────┬────────────────┬───────────────┬───────────────┤
│ Catalogue    │ Pipeline      │ Experiment     │ Statistics    │ Integration   │
│ (metadata DB,│ adapters      │ engine (plan,  │ (composition, │ (ontology,    │
│  schema,     │ (ibif-napari, │  cache, run,   │  mixed models,│  cross-modal  │
│  validation) │  vtea-core,   │  aggregate,    │  spatial,     │  concordance, │
│              │  external)    │  provenance)   │  power)       │  donor tables)│
├──────────────┴───────────────┴───────┬────────┴───────────────┴───────────────┤
│ Storage: SQL metadata · Parquet tables (DuckDB queries) · OME-Zarr / original │
│ files (referenced by checksum) · AnnData / SpatialData · content-addressed    │
│ result cache                                                                  │
├──────────────────────────────────────┴────────────────────────────────────────┤
│ Executors: local process pool · IBIF facility GPU server · SLURM cluster      │
└───────────────────────────────────────────────────────────────────────────────┘
          REST API (FastAPI) over the same service layer for server mode
```

### Principles

1. **Headless first.** Everything the GUI does is a call to the Python API, which can also
   run from scripts, notebooks, the CLI (`metaatlas …`) or a cluster job. This is how
   vtea-core and IBIF napari are already split, and it makes the GUI testable.
2. **Reference large data, never copy it.** Images and omics objects stay where they are.
   The catalogue stores their URI, size and SHA-256. Derived tables are Parquet.
3. **Everything derived is content-addressed.** A result's key is the hash of its input
   checksums, the pipeline step's canonical JSON and the relevant software versions. A
   matching key means the result is reused. Any change gives a new key, so stale results
   are never mixed with new ones.
4. **Immutable releases.** An atlas *release* (a frozen catalogue snapshot) and an
   experiment *run* never change once frozen. Corrections create new versions.
5. **Plug-ins at the seams.** Pipeline adapters, modalities, statistical models,
   integration methods and executors are entry-point plug-ins with a small interface.
6. **Two deployment modes, one code base.**
   - *Local / lab*: SQLite + files on disk, single user, no server.
   - *Facility / consortium*: PostgreSQL + object store or shared file system,
     FastAPI service, users and roles, cluster executor.

### Technology choices (proposed; final choices in M0, see §11)

| Concern | Choice | Why |
|---|---|---|
| Language | Python ≥ 3.10 | Same as IBIF napari and vtea-core |
| Metadata store | SQLAlchemy 2 + Alembic; SQLite (local) / PostgreSQL (server) | Schema migrations, one ORM for both modes |
| Object tables | Apache Parquet, partitioned by dataset; queried with DuckDB | Columnar storage for 10⁸+ cells; SQL across datasets without loading them all |
| Images | Original files (ND2 / CZI / LIF / TIFF) via IBIF napari readers; OME-Zarr for derived images | Reuses existing readers; Zarr allows chunked and remote access |
| Omics | AnnData (`.h5ad`), SpatialData (Zarr) | The community standards |
| Validation | pydantic models + JSON Schema for manifests, pipelines and experiments | Clear errors, and the schemas are versioned |
| Statistics | statsmodels, scipy, scikit-learn; optional R bridge (lme4, propeller) for cross-checks | Mixed models in Python; R as the V&V reference |
| Provenance | W3C PROV-style run records; RO-Crate export | Standard and readable without the tool |
| Executors | `concurrent.futures`; IBIF napari's `_remote` and `_cluster` (SSH, SLURM) | Already built and tested |
| GUI | Qt (qtpy), matplotlib / pyqtgraph; docks into napari | Same toolkit as IBIF napari, so drill-down is in-process |
| Server API | FastAPI + OpenAPI | Typed, testable, and ready for a later web front end |
| Packaging | `pyproject.toml`, `metaatlas` package, `metaatlas` CLI, `metaatlas-vv` V&V CLI | Same pattern as `ibif-napari` and `ibif-napari-vv` |

### Proposed package layout

```
src/metaatlas/
  catalog/        schema (ORM), queries, manifest import, validation, releases
  ontology/       Cell Ontology (CL) / UBERON terms, cell-type mapping tables
  io/             adapters to read datasets: ibif workspace, vtea export, AnnData, SpatialData, CSV/Parquet
  pipelines/      PipelineSpec, adapters (ibif_napari, vtea_core, external command), registry
  engine/         experiment planner, cache, executors, scheduler, provenance
  stats/          aggregation, composition, mixed models, spatial, multiple testing, power
  integrate/      harmonisation, cross-modal concordance, donor feature tables
  export/         figures, tables, RO-Crate bundle, data dictionary
  api/            FastAPI app (server mode)
  gui/            Qt widgets and the napari bridge
  vv/             V&V suite: synthetic atlas, requirements, checks, report
  cli.py
tests/
docs/
```

---

## 4. Data model

### Entities

| Entity | Key fields | Notes |
|---|---|---|
| **Atlas** | id, name, description, release tag | Top-level container. Several atlases can share one store. |
| **Release** | id, atlas, frozen_at, catalogue hash | An immutable snapshot that experiments refer to |
| **Cohort** | id, name, definition (static list or saved query), description | A donor can be in several cohorts (e.g. "AKI" and "Site 2"). Dynamic cohorts are queries, e.g. `diagnosis == 'CKD' and eGFR < 45`. |
| **Donor** (participant) | id (pseudonymised), cohort memberships, covariates (age band, sex, diagnosis, clinical variables), site | No direct identifiers, ever (see §10, data governance) |
| **Specimen** | id, donor, type (biopsy, nephrectomy, block, section), collection info, preservation | Several per donor |
| **TissueRegion** | id, specimen, UBERON term (e.g. renal cortex, glomerulus), how it was defined (annotation, segmentation, whole specimen) | Lets analyses be restricted to a compartment |
| **Dataset** | id, specimen / region, modality, assay, instrument, acquisition parameters, files (URI, SHA-256, size), voxel size, channels / markers, QC status, batch keys (site, run, scanner, date) | One acquisition. The unit a pipeline runs on. |
| **Modality** | name, plug-in, required metadata, object-table schema | e.g. `3d-cytometry`, `multiplex-if`, `codex`, `visium`, `xenium`, `snrnaseq`, `scrnaseq`, `lmd-proteomics`, `histology` |
| **Marker / Channel map** | dataset channel → marker name → canonical marker (e.g. HGNC symbol) | Harmonises channel naming between datasets |
| **PipelineSpec** | id, name, version, adapter, canonical JSON, hash, input contract, outputs, author, notes | See §5 |
| **Experiment** | id, release, pipeline version, dataset selection, cohorts, design (outcomes, contrasts, covariates, random effects), status | See §5 |
| **Run / Task** | id, experiment, dataset, step, cache key, executor, start / end, status, logs, environment | One per dataset × pipeline step |
| **Result** | cache key, kind (object table, labels, summary, statistics), URI, schema, checksum | Content-addressed |
| **CellType** | atlas label, CL term, parent, synonyms per modality | The harmonised hierarchy (§7) |
| **Annotation / Note** | target (any entity), author, text, time | Curation notes and QC flags |

### Object tables

Each dataset's objects are one Parquet table with a fixed core and free feature columns:

```
dataset_id, object_id, t, position, region_id,
centroid_z_um, centroid_y_um, centroid_x_um, volume_um3,
cell_type (atlas label), cell_type_cl (CL term), cell_type_source, confidence,
<feature columns: "<objects>: <channel> <stat>" as written by IBIF napari, or vtea-core names>
```

A **data dictionary** (vtea-core's `FeatureCatalog`) describes each feature column: unit,
channel, marker, objects layer and statistic. Features are matched across datasets through
the dictionary and the channel → marker map, never by column position.

### Metadata import

Curators import a **manifest** (CSV / Excel, one sheet per entity, or a single long
sheet). It is validated against the JSON Schema, and every error is listed with its row
and column. The import is idempotent: importing the same manifest twice changes nothing,
and a changed row creates a new version of that entity.

---

## 5. Dataset pipelines and comparison experiments

### PipelineSpec

A pipeline is defined once on a dataset, then applied to many. A `PipelineSpec` is:

```yaml
metaatlas_pipeline_version: 1
name: cortex-immune-3d
version: 3
adapter: ibif-napari          # or vtea-core, external-command, notebook
requires:                      # the input contract, checked before anything runs
  modality: 3d-cytometry
  channels: {nuclei: DAPI, membrane: [WGA, Phalloidin], markers: [CD3, CD68, LRP2]}
  voxel_size_um: {z: [0.3, 2.0], xy: [0.1, 0.6]}
steps:                         # adapter-specific, canonicalised and hashed
  - segment:   {method: cellpose, model: cpsam_v2, channel: nuclei, mode: 3d, diameter: auto}
  - secondary: {kind: whole_cells, seeds: segment, channel: membrane, threshold: otsu}
  - measure:   {objects: [segment, secondary], channels: all}
  - phenotype: {tree: phenotypes/immune.v2.json}   # vtea-core classes / gates
  - classify:  {model: models/nephnet3d-v1, objects: segment}   # optional
outputs:
  objects: parquet             # object table (with the core columns)
  labels: ome-zarr             # optional; needed for drill-down
  summary: [counts_by_type, mean_by_type]
```

**Where pipelines come from.**

- **IBIF napari**: "Export as pipeline…" saves the steps that made the current workspace
  (segmentation and region settings, secondary objects, measurements, cytometry gates,
  phenotypes, classification model) as a spec. This needs an upstream change (U1 in
  [UPSTREAM.md](UPSTREAM.md)).
- **vtea-core**: a `*.vtea.json` protocol is wrapped as it is. Its `processing` /
  `analysis` steps map directly to `steps`.
- **External command**: any container or script that reads a dataset and writes an
  object table in the core schema. This allows other tools, such as a QuPath or
  MCMICRO workflow.
- **Precomputed**: datasets that already have object tables (e.g. published cytometry
  tables, AnnData with cell type labels) join an experiment without a pipeline step.

**Input mapping.** Before a pipeline runs, each dataset's channels are matched to the
spec's `requires` through the marker map. Datasets that do not match are listed with the
reason (missing marker, voxel size out of range, failed QC). They are excluded, never
silently run with the wrong channel.

### Comparison experiment

```yaml
experiment: aki-vs-reference-immune
release: atlas-2026.11
pipeline: cortex-immune-3d@3
select:
  cohorts: [Reference, AKI, CKD]
  modality: 3d-cytometry
  region: UBERON:0001225          # renal cortex
  qc: passed
design:
  unit: donor                     # results are aggregated to donors before testing
  outcomes:
    - composition: {level: phenotype, parent: Immune}         # proportions
    - feature: {name: "cells: CD68 mean", by: phenotype}     # per type
    - spatial: {metric: neighbourhood_enrichment, pairs: [[Macrophage, PT-injured]]}
  contrasts: [[AKI, Reference], [CKD, Reference]]
  covariates: [age, sex]
  batch: [site]
  random: [donor]                 # for repeated specimens per donor
sweep:                            # optional: run several pipeline variants
  segment.diameter: [auto, 12, 16]
executor: {kind: slurm, profile: facility-a100, max_parallel: 40}
seed: 20261114
```

### What the engine does

1. **Plan.** Resolve the selection against the frozen release, check each dataset against
   the input contract, and expand the sweep. The result is a task graph (dataset × step ×
   variant) with a cache key for every node. Before anything runs, the plan reports what is
   cached, what will run, what is excluded and why, and an estimated cost (built on
   vtea-core's `workflow.cost`).
2. **Execute.** Send the tasks to the executor. Tasks are idempotent and write atomically,
   as IBIF napari's batch jobs do (`tmp` then `os.replace`). A killed run resumes with only
   the unfinished tasks. Progress goes to a run record that the GUI and CLI follow.
3. **Collect.** Validate each dataset's object table against its schema and attach the
   dataset, specimen, region and donor IDs.
4. **Aggregate.** Turn cells into donor-level summaries: counts and proportions per type,
   per-type feature means and medians, and spatial statistics, weighted by cell count and
   carrying the number of datasets and cells behind each value.
5. **Model.** Fit the statistical models (§6) for each outcome and contrast.
6. **Report.** Write tables, figures and an HTML / Markdown report, each figure linked to
   the donors and datasets behind it. Each run also writes its provenance record.

**Re-running.** Changing one step's parameters changes its cache key and the keys of the
steps after it, so only those re-run. Two runs of the same experiment can be compared in a
**run diff**: which donors changed, which effects moved, and by how much.

---

## 6. Statistics

The main statistical risk in atlas meta-analysis is **pseudo-replication**: treating
10⁶ cells from 20 donors as 10⁶ independent observations. The engine therefore always
aggregates to the donor (or uses a model with donor as a random effect) before testing.
The V&V program checks that type I error is calibrated (§VV-STAT).

| Outcome | Default method | Alternatives (plug-ins) |
|---|---|---|
| **Composition** (cell type proportions) | Proportions per donor, logit or arcsine-square-root transform, linear model with covariates and batch (as in `propeller`); reports the fold change and its CI | Dirichlet-multinomial regression; CLR + linear model; Bayesian compositional model (scCODA-like) |
| **Feature per type** (e.g. marker intensity in macrophages) | Linear mixed model: `feature ~ group + covariates + batch + (1 \| donor)`, on cell-level data or on donor-level means weighted by cell count | Donor pseudo-bulk + Welch / Mann–Whitney; quantile comparisons |
| **Spatial** (neighbourhood enrichment, distances, niche frequency) | Per-dataset statistic against a label-permutation null (from vtea-core `spatial` / `neighborhoods`), then a donor-level model of the z-scores | Niche composition as a compositional outcome |
| **Cross-modal agreement** | Per donor: proportion in modality A vs B; Lin's concordance, Bland–Altman bias and limits | Deming regression |
| **Multiple testing** | Benjamini–Hochberg across the outcomes of one experiment, reported per family | Holm; hierarchical FDR along the phenotype tree |
| **Power / design** | Simulation from the synthetic atlas generator (§VV) with effect sizes and donor variance estimated from pilot data | |

Every reported result includes: estimate, 95% CI, p and q values, the number of donors
per group, the number of datasets and cells behind them, the model formula, any
convergence warnings, and diagnostics (residual plot, influence of each donor).

**Batch.** Site, scanner and run are covariates by default. A "batch check" view shows
whether batch is confounded with group (e.g. all AKI donors from one site) and warns before
the experiment runs.

---

## 7. Multimodal integration

1. **Harmonised cell types.** An atlas cell type hierarchy with a Cell Ontology (CL) term
   for every node. Each modality maps its labels to it: phenotype names from the IBIF
   napari / vtea-core phenotype trees, cluster or reference-mapping labels from snRNA-seq,
   and spatial transcriptomics labels. A mapping can be 1:1, coarsen a label (map to a
   parent) or leave it unmapped (with a reason). Unmapped labels are reported.
2. **Harmonised markers.** Channel → marker → gene symbol, so protein (imaging) and RNA
   features of the same gene can be shown side by side.
3. **Regions.** UBERON terms for tissue regions, so "cortex" means the same thing in every
   modality.
4. **Donor × modality table.** One row per donor, with columns per modality × cell type ×
   measure (proportion, feature means, spatial statistics) and missingness recorded
   explicitly. This is the input for cross-modal analyses and the export most users want.
5. **Concordance.** For cell types present in two modalities, compare donor-level
   abundance, with the expected biases stated (dissociation loss in scRNA-seq, section
   depth in 2D imaging compared with 3D cytometry).
6. **Plug-ins** for deeper integration later (e.g. MOFA+ factor models, reference transfer
   such as STELLAR via vtea-core classification). They use the donor table and object
   tables as inputs and do not change the core.

---

## 8. GUI

A Qt application (`metaatlas`) that runs on its own or docks into napari. It uses the same
look, icons and help conventions as IBIF napari: one panel with icon buttons, and hover
text naming each button and what it does.

| Section | What it shows and does |
|---|---|
| **Atlas** | A tree: Atlas → Cohorts → Donors → Specimens → Regions → Datasets, with search and filters (modality, region, QC, site). Counts at every node. A coverage grid of donors × modalities that shows what exists, what is missing and what failed QC. |
| **Cohorts** | Build static or query-defined cohorts with a filter builder. Shows the size and covariate balance (age, sex and site distribution per cohort). |
| **Datasets** | Metadata, files and checksums, channel → marker map, QC, a thumbnail, and every result computed for the dataset. **Open in IBIF napari** opens the dataset (and its labels and phenotypes from any experiment) in the same process. |
| **Pipelines** | The registry: versions, diffs between versions, input contract, origin (which dataset and workspace it came from) and test datasets. Import from IBIF napari / `.vtea.json`. "Try on…" runs it on a few datasets and opens them side by side. |
| **Experiment designer** | A step-by-step form: pipeline → cohorts and filters → outcomes → contrasts, covariates, batch → executor. Shows the plan (to run / cached / excluded with reasons, cost estimate) and the batch-confounding check before **Run**. Saves the experiment as YAML. |
| **Runs** | Progress per dataset and step, logs, retry of failed tasks, cancellation. Works for runs started from the CLI too. |
| **Results** | Forest plots per contrast, composition bar charts per donor, volcano plots, box or strip plots of donor values coloured by covariate. **Every point is clickable**: donor → datasets → open in napari at that object type. Run diff between two runs. |
| **Integration** | Cell type mapping editor (with CL lookup), donor × modality table, concordance plots. |
| **Export** | Figures (SVG / PNG / PDF), tables, report, RO-Crate bundle. |

---

## 9. Milestones and tasks

Each milestone ends with a release tag and a **V&V gate**: the requirements listed for it
in [VV_PLAN.md](VV_PLAN.md) pass in CI, with the report attached to the release. The
durations assume one or two developers and are for planning only.

### M0: Foundations (0.0, ~3 weeks)

| Task | Description |
|---|---|
| M0.1 | Repository scaffold: `pyproject.toml`, `src/metaatlas`, pytest, ruff, pre-commit, GitHub Actions (lint, test, matrix Python 3.10 to 3.12, Linux / macOS / Windows) |
| M0.2 | Architecture decision records (ADR) for the decisions in §11 |
| M0.3 | Inventory of the existing atlas data: modalities, file formats, metadata spreadsheets, object tables and cell type label sets from previous studies. Choose 2 pilot cohorts. |
| M0.4 | Requirements baseline: freeze the first version of the requirements in VV_PLAN.md and set up `docs/vv/requirements.md`, generated from code |
| M0.5 | Data governance: confirm the de-identification rules, access model and where data may be stored (IRB / DUA constraints) |
| M0.6 | Agree the upstream changes (UPSTREAM.md) with the IBIF napari and vtea-core maintainers |

**Exit:** CI green on an empty package; ADRs accepted; pilot data identified.

### M1: Catalogue and data model (0.1, ~6 weeks)

| Task | Description |
|---|---|
| M1.1 | ORM models for all §4 entities; Alembic migrations; SQLite and PostgreSQL |
| M1.2 | pydantic / JSON Schema for manifests; validation with row / column error messages |
| M1.3 | Manifest import (CSV / Excel), idempotent, with versioned changes |
| M1.4 | File registration: URI, size, SHA-256 (streamed, parallel), re-check command |
| M1.5 | Query API: filters over every level, dynamic cohorts, counts |
| M1.6 | Releases: freeze, catalogue hash, diff between releases |
| M1.7 | Ontology module: CL / UBERON term lookup (offline snapshot), marker → gene map |
| M1.8 | CLI: `metaatlas init / import / validate / query / release` |
| M1.9 | V&V: synthetic manifest generator (valid and with planted errors); requirements VV-CAT-* |

**Exit:** the pilot cohorts are imported, validated and released; VV-CAT-* pass.

### M2: Dataset adapters (0.2, ~5 weeks)

| Task | Description |
|---|---|
| M2.1 | Adapter interface: `read_objects`, `read_labels`, `channels`, `metadata`, `open_in_napari` |
| M2.2 | IBIF napari workspace (`.ibif`) and measurement tables (CSV / Excel), including joined *All objects* tables |
| M2.3 | vtea-core export (CSV / Parquet + data dictionary) and `*.vtea.json` protocols |
| M2.4 | AnnData `.h5ad` (scRNA / snRNA-seq) and SpatialData (Visium, Xenium, CODEX) |
| M2.5 | Generic object table (CSV / Parquet) with a column map |
| M2.6 | Object table normaliser: core schema, units in µm, feature dictionary, marker map |
| M2.7 | Parquet store partitioned by dataset; DuckDB views across datasets |
| M2.8 | V&V: round-trip and schema tests for each adapter (VV-IO-*) |

**Exit:** every pilot dataset can be read through an adapter and queried across datasets.

### M3: Pipelines and dataset-level execution (0.3, ~7 weeks)

| Task | Description |
|---|---|
| M3.1 | `PipelineSpec` schema, canonical JSON, hashing, versioning, registry |
| M3.2 | ibif-napari adapter (depends on U1, U2): runs segmentation → secondary → measure → phenotype → classify headless |
| M3.3 | vtea-core adapter: runs a `*.vtea.json` protocol on a dataset (including `blocked` large-image mode) |
| M3.4 | External-command adapter (container or script with a file contract) |
| M3.5 | Input contract and channel mapping, with an exclusion report |
| M3.6 | Content-addressed cache: key = inputs + step JSON + versions; atomic writes; garbage collection |
| M3.7 | Executors: local pool, IBIF facility GPU server and SLURM, reusing IBIF napari's `_remote` / `_cluster` / `_batch` |
| M3.8 | Provenance record per task (inputs, parameters, versions, host, time, seed, logs) |
| M3.9 | "Try on…": run a pipeline on N datasets and open the results in napari |
| M3.10 | V&V: the oracle pipeline, cache invalidation, resume after a kill, executor equivalence (VV-PIPE-*, VV-EXEC-*) |

**Exit:** a pipeline exported from IBIF napari runs on 20 pilot datasets on the cluster and
gives the same tables as running it in napari, within the stated tolerances.

### M4: Cohort experiment engine (0.4, ~5 weeks)

| Task | Description |
|---|---|
| M4.1 | Experiment schema (YAML), validation, saved with its release and pipeline version |
| M4.2 | Planner: selection → task graph, sweep expansion, plan report, cost estimate |
| M4.3 | Scheduler: priority, maximum parallel tasks, retries with backoff, cancellation, resume |
| M4.4 | Collection and donor-level aggregation (counts, proportions, weighted means, spatial summaries) |
| M4.5 | Run records, run diff |
| M4.6 | CLI: `metaatlas experiment plan / run / status / diff` |
| M4.7 | V&V: synthetic atlas v1 (cohorts and donors with known compositions); aggregation checks against an independent NumPy calculation (VV-EXP-*, VV-AGG-*) |

**Exit:** J4 runs end to end from the CLI on the synthetic atlas and on the pilot cohorts.

### M5: Statistics and reporting (0.5, ~6 weeks)

| Task | Description |
|---|---|
| M5.1 | Composition models (propeller-style default, Dirichlet-multinomial, CLR) |
| M5.2 | Mixed models for features, with covariates, batch and donor random effects; diagnostics |
| M5.3 | Spatial outcome: per-dataset permutation z-scores → donor model |
| M5.4 | Multiple testing (BH, Holm, hierarchical along the phenotype tree) |
| M5.5 | Batch confounding check; power analysis by simulation |
| M5.6 | Report: HTML / Markdown, figures, tables, every number traceable to donors and datasets |
| M5.7 | Reference cross-checks against R (lme4, propeller) on fixed fixtures |
| M5.8 | V&V: null calibration, power, CI coverage and pseudo-replication guard on the synthetic atlas (VV-STAT-*) |

**Exit:** VV-STAT-* pass; the pilot comparison report has been reviewed by a statistician.

### M6: Multimodal integration (0.6, ~5 weeks)

| Task | Description |
|---|---|
| M6.1 | Atlas cell type hierarchy with CL terms; mapping tables per modality; editor API |
| M6.2 | Donor × modality feature table with explicit missingness |
| M6.3 | Cross-modal concordance (Lin's CCC, Bland–Altman), with known biases documented |
| M6.4 | Synthetic atlas v2: the same donors rendered in several modalities (imaging via IBIF napari's V&V generator; count matrices for sn / scRNA-seq) with known cross-modal relationships |
| M6.5 | Integration plug-in interface (e.g. MOFA+, reference transfer) |
| M6.6 | V&V: VV-INT-* |

**Exit:** J6 runs on the synthetic atlas v2 and on the pilot cohorts.

### M7: GUI (0.7 to 0.8, ~10 weeks, can start after M2 in parallel)

| Task | Description |
|---|---|
| M7.1 | Application shell, icon panel, settings, help links (same conventions as IBIF napari) |
| M7.2 | Atlas browser and coverage grid |
| M7.3 | Cohort builder with covariate balance |
| M7.4 | Dataset view and the napari bridge ("Open in IBIF napari" with labels and phenotypes) |
| M7.5 | Pipeline registry, import, version diff, "Try on…" |
| M7.6 | Experiment designer with plan preview and batch check |
| M7.7 | Run monitor |
| M7.8 | Results explorer with click-through to donors and datasets; run diff |
| M7.9 | Integration view and mapping editor |
| M7.10 | Export dialogs |
| M7.11 | User manual with screenshots (like IBIF napari's manual), generated screenshots in CI |
| M7.12 | V&V: pytest-qt tests for every section, end-to-end GUI journeys J1 to J7 on the synthetic atlas, a usability study (VV-GUI-*, VV-UX-*) |

**Exit:** J1 to J7 are done in the GUI by test users without help from the developers.

### M8: Hardening and release 1.0 (~6 weeks)

| Task | Description |
|---|---|
| M8.1 | Performance: 500 donors, 5,000 datasets and 10⁸ objects; query and aggregation budgets (VV-PERF-*) |
| M8.2 | Server mode: FastAPI, authentication, roles (curator, analyst, viewer), audit log |
| M8.3 | Security review; PHI checks (VV-SEC-*) |
| M8.4 | RO-Crate publication bundle; data dictionary; "re-run from bundle" check |
| M8.5 | Real-data validation: reproduce selected published results from previous atlas work (VV-REAL-*) |
| M8.6 | Installers and environments (conda / pip; optional PyInstaller, as vtea-python does) |
| M8.7 | Final V&V report; release notes; known limitations |

**Exit:** all 1.0 requirements pass with `--strict`; V&V report published with the release.

### M9: After 1.0 (optional)

A web front end over the REST API, a public read-only atlas browser, more integration
plug-ins, and federated queries across institutions.

### Order and overlap

```
M0 ─ M1 ─ M2 ─ M3 ─ M4 ─ M5 ─ M6 ─────── M8
              └──────── M7 (GUI, in parallel from M2) ──┘
Upstream:     U1 U2 (needed by M3)   U3 U4 (needed by M6/M8)
```

Approximate total to 1.0: 12 to 14 months with one developer, 7 to 9 with two.

---

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| IBIF napari steps cannot yet be run headless as one recipe | Blocks M3 | U1 agreed in M0; until it exists, use the vtea-core adapter and the batch Cellpose path |
| Pseudo-replication or confounded batch gives false findings | Wrong science | Donor-level design enforced by default, confounding check before a run, null calibration in V&V |
| Channel and marker naming differs between sites and years | Silent wrong channel | Explicit marker map, input contract, exclusion report; no matching by position |
| Data volume (10⁸ cells, TB of images) | Slow, costly | Reference rather than copy, Parquet + DuckDB, cache, cluster executor, performance budgets in V&V |
| Dependency conflicts (napari, zarr 3, vtea-core pins) | Install failures | Core installs without napari; GUI and napari bridge are optional extras; CI on 3 OSes; U3 (zarr 3) |
| Protected health information in donor metadata | Compliance breach | Pseudonymous IDs only, coarse covariates (age bands), schema rejects identifier-like fields, VV-SEC checks |
| Cell type labels disagree between modalities | Misleading integration | Explicit mapping with "unmapped" reasons, coarsening to parent types, concordance reporting |
| Scope creep into a full web portal | Delay | Web is M9; API designed for it but not built before 1.0 |

---

## 11. Open decisions

These are settled in M0 as ADRs. The proposed default is listed first.

| ID | Decision | Proposed | Alternatives |
|---|---|---|---|
| D1 | GUI technology | Qt desktop, dockable in napari (in-process drill-down, team already uses Qt) | Web (React) on FastAPI from the start |
| D2 | Metadata store | SQLite local + PostgreSQL server via SQLAlchemy | PostgreSQL only; a document store |
| D3 | Ontologies | CL, UBERON, HGNC; consortium-specific terms as extensions | Local vocabulary only |
| D4 | Default composition model | propeller-style logit linear model | Dirichlet-multinomial; Bayesian |
| D5 | Package name | `metaatlas` | `tissue-atlas`, `ibif-atlas` |
| D6 | Where the IBIF napari pipeline export lives | In IBIF napari (U1), spec format owned by MetaAnalysis | In vtea-core's protocol format |
| D7 | Licence | GPL-2.0, to match vtea-python (which the tool imports) | A permissive licence, which would need the vtea-core licence to change too |
