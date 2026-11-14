# What MetaAnalysis needs from IBIF napari and vtea-core

MetaAnalysis does not re-implement dataset-level analysis. It calls IBIF napari
([CU-IBIF/Napari_Cellpose](https://github.com/CU-IBIF/Napari_Cellpose)) and vtea-core
([winfrees/vtea-python](https://github.com/winfrees/vtea-python)). These are the changes it
needs from them, in the order they are needed. Each becomes an issue in the upstream
repository, agreed with its maintainers in M0 (task M0.6).

| ID | Repository | Change | Why | Needed by |
|---|---|---|---|---|
| U1 | IBIF napari | **Headless pipeline export and run.** "Export as pipeline…" writes the steps that made a workspace (segmentation with its region and Z settings, secondary objects, measurements, cytometry gates and phenotype tree, classification model) as one JSON recipe. A function `run_recipe(recipe, dataset) -> (labels, tables)` runs it without a viewer, using the same functions the panels call. Today only Cellpose segmentation can run without napari (`_batch.run`). | The ibif-napari pipeline adapter (M3.2) | M3 |
| U2 | IBIF napari | **Stable measurement table schema and data dictionary.** Column names, units and the join rule for *All objects* tables are versioned, and a dictionary is written with each table (as vtea-core's `FeatureCatalog` does). | Matching features across datasets by meaning rather than by name (M2.6) | M2 |
| U3 | vtea-core | **zarr 3 support.** vtea-core pins `zarr<3`, which caps napari below 0.8 in IBIF napari and blocks current SpatialData and OME-Zarr releases. | SpatialData adapter (M2.4); dependency risk | M2 / M6 |
| U4 | vtea-core | **Spatial statistics with a permutation null, per dataset**, returning z-scores and the null distribution (building on `spatial` and `neighborhoods`). | Spatial outcomes (M5.3) | M5 |
| U5 | IBIF napari | **Open a dataset programmatically with given labels and phenotype columns** (an API around the workspace loader and the Cytometry section). | "Open in IBIF napari" drill-down (M7.4) | M7 |
| U6 | IBIF napari + vtea-core | **Release tags and a changelog of behaviour changes**, so MetaAnalysis can pin versions, and the VV-PIPE checks can tell an intended change from a regression. | Change control (VV_PLAN §7) | M3 |
| U7 | IBIF napari | **Expose the V&V scene generator as a library** (`ibif_napari.vv.synth`) with a stable signature for rendering scenes from given objects and marker intensities. | Synthetic atlas image tier S2 | M3 |

Until U1 exists, the vtea-core adapter and IBIF napari's batch Cellpose path are enough
to start M3.
