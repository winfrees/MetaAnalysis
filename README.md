# MetaAnalysis: a multimodal tissue atlas workbench

**Status: planning.** This repository holds the plan for the tool. There is no code yet.

MetaAnalysis is a desktop and server tool for meta-analysis across many tissue datasets,
in the style of a multimodal tissue atlas. Its levels are:

```
Atlas ─┬─ Cohort ─── Donor ─── Specimen ─── Tissue region ─── Dataset ─── Objects (cells, structures)
       │  (disease,   (participant,  (biopsy,     (compartment,    (one acquisition    (one row per object:
       │   control,    clinical       block,       e.g. cortex,     in one modality:     features, class,
       │   site)       covariates)    section)     glomerulus)      3D cytometry,        position)
       │                                                            multiplex IF,
       │                                                            spatial / sn RNA-seq,
       │                                                            proteomics …)
       └─ Experiments: one dataset-level pipeline × selected cohorts → donor-level comparisons
```

The tool works at two levels:

- **High level.** Browse and query the atlas by cohort, donor, tissue and modality.
  Combine modalities per donor and per cell type, and compare cohorts.
- **Dataset level.** Open any dataset in [IBIF napari](https://github.com/CU-IBIF/Napari_Cellpose)
  (segmentation, secondary objects, measurements, cytometry, classification), which is
  built on [vtea-core](https://github.com/winfrees/vtea-python).

The main function is the **comparison experiment**. You take a dataset-level pipeline
defined once in IBIF napari or vtea-core, apply it to every dataset in a set of cohorts,
collect the results by donor, and compare the cohorts with statistics that treat the
donor, not the cell, as the unit of replication.

## Documents

| Document | What it covers |
|---|---|
| [docs/PLAN.md](docs/PLAN.md) | Goals, architecture, data model, pipeline and experiment engine, GUI, milestones and tasks, risks and open decisions |
| [docs/VV_PLAN.md](docs/VV_PLAN.md) | Verification and validation program: requirements with acceptance criteria, the synthetic atlas with planted effects, statistical calibration, test levels, traceability and release gates |
| [docs/UPSTREAM.md](docs/UPSTREAM.md) | What IBIF napari and vtea-core need to provide, and in which milestone |
