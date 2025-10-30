Xenium Dataflow: Inputs → Processes → Outputs

A concise, practical overview of the Xenium output bundle: what goes in, which processes run, and which files come out. Includes aux/QA artifacts, Explorer round-trip notes, and quick R/Python loaders.

Table of Contents

Overview

Inputs

Core Pipeline (Process-Linked DAG)

Auxiliary Outputs (Provenance & QA)

File → Process Links (At a Glance)

Exports for Xenium Explorer

Quick Start: Loaders

Python (SpatialData/Scanpy)

R (Seurat/SpatialExperiment)

Space & Format Tips

Version Notes

License

Overview

Xenium downstream artifacts are emitted in multiple formats (Zarr, Parquet, CSV/MEX/HDF5) to support:

Interoperability across Python/R and Xenium Explorer

Performance/scale (chunked Zarr for random access; columnar Parquet for tall tables)

Longevity (legacy MEX/HDF5 compatibility)

Inputs
[INPUTS]
  ├─ IF morphology images                # DAPI ± membrane/cytoplasm (if Multimodal Cell Segmentation, MCS)
  ├─ RNA chemistry images + codebook     # Per-cycle, per-channel images + panel/codewords
  └─ Run metadata                        # Instrument run, FOVs, panel JSON

Core Pipeline (Process-Linked DAG)
# 1) Decoding: convert RNA chemistry images into decoded molecules
[PROCESS] Decode
  # uses: RNA chemistry images + codebook/panel JSON
  ├──▶ transcripts.parquet               # per-molecule table: x,y,(z), gene, Q-score, cycle, channel, intensity
  └──▶ transcripts.zarr.zip              # same as above, Zarr bundle compatible with Xenium Explorer

# 2) Segmentation: derive nuclei/cell masks from morphology IF
[PROCESS] Segmentation
  # uses: IF morphology images (DAPI ± membrane/cytoplasm)
  └──▶ cells.zarr.zip                    # authoritative segmentation: masks, centroids, areas, geometry
      ├──▶ cell_boundaries.parquet       # simplified cell polygons (derived from cells.zarr.zip)
      └──▶ nucleus_boundaries.parquet    # simplified nucleus polygons (derived from cells.zarr.zip)

# 3) Assignment: place decoded molecules into segmented cells
[PROCESS] Assign transcripts to cells
  # uses: transcripts.parquet + cells.zarr.zip
  └──▶ cells.parquet                     # per-cell summaries (counts, genes detected, area, %nuclear, QC)
      └──▶ cell_feature_matrix.*         # gene×cell count matrix (Q≥threshold); formats: MEX/, .h5, .zarr.zip

# 4) Secondary analysis: run on the cell×gene matrix
[PROCESS] Analyze matrix
  # uses: cell_feature_matrix.*
  └──▶ analysis/                         # CSV/TSV: PCA, neighbors, clustering, UMAP, DE/markers
      └──▶ analysis.zarr.zip             # packaged analysis for Xenium Explorer

# 5) QC roll-up / reporting
[PROCESS] Summarize QC
  # uses: metrics from decoding + segmentation + matrix + key images
  ├──▶ metrics_summary.csv               # run-wide KPIs (assign rate, median genes/cell, %Q20, etc.)
  └──▶ analysis_summary.html             # interactive HTML report; links thumbnails/overlays/plots


Authoritative segmentation lives in cells.zarr.zip. The *_boundaries.parquet files are simplified polygons for visualization.

Auxiliary Outputs (Provenance & QA)
[AUX] per_cycle_channel_images/          # FROM: RNA chemistry images; QA frames to inspect decoding inputs
[AUX] background_qc_images/              # FROM: autofluorescence frames; used to make morphology_focus/*
[AUX] morphology_focus/                  # FROM: IF morphology – background-subtracted, multi-focus 2D OME-TIFFs
[AUX] overview_scan/ + *_fov_locations   # FROM: instrument overview & tiling; slide/FOV context for viewers
[AUX] decoding_qc/                       # FROM: decoder; codeword confusion, Q-score histograms, FDR vs Q
[AUX] segmentation_qc/                   # FROM: cells.zarr.zip + IF; mask overlays, size/shape distributions
[AUX] assignment_qc/                     # FROM: transcripts + cells; unassigned density, boundary distances
[AUX] morphology_thumbnails/             # FROM: morphology.ome.tif; downsampled PNGs for HTML/report
[AUX] runtime_logs/                      # FROM: analyzer; versions, CLI args, timing breakdown
[AUX] gene_panel.json                    # FROM: input design; codewords/targets used in decoding

File → Process Links (At a Glance)
- RNA images + codebook   ──Decode────▶ transcripts.parquet
- IF images               ──Segment───▶ cells.zarr.zip ──simplify──▶ cell_/nucleus_boundaries.parquet
- transcripts + cells     ──Assign────▶ cells.parquet ──filter_tx──▶ cell_feature_matrix.*
- matrix                  ──Analyze───▶ analysis/ ──pack──▶ analysis.zarr.zip
- all metrics + images    ──Summarize─▶ metrics_summary.csv, analysis_summary.html

Exports for Xenium Explorer
Preferred Zarr bundles: transcripts.zarr.zip, cell_feature_matrix.zarr.zip, analysis.zarr.zip
Required manifest: experiment.xenium
Optional overlays: CSV with columns (e.g.) cell_id,cluster,cell_type

Quick Start: Loaders
Python (SpatialData/Scanpy)
# pip install 'spatialdata-io' spatialdata-plot scanpy squidpy
import spatialdata_io as sio
import spatialdata as sd

# Read a Xenium bundle directly
sdata = sio.read_xenium("/path/to/xenium_output_bundle")

# Access the table (AnnData)
adata = sdata["table"]  # genes × cells

# Example: plot cells/transcripts (requires spatialdata-plot)
import spatialdata_plot  # registers .pl accessor
sdata.pl.render_images().pl.render_points().pl.show()

R (Seurat/SpatialExperiment)
# install.packages("Seurat"); BiocManager::install(c("SpatialExperiment", "SpatialExperimentIO"))
library(Seurat)
# Read Xenium bundle to Seurat object (Seurat v5)
obj <- ReadXenium(data.dir = "/path/to/xenium_output_bundle")

# Quick spatial plot
SpatialDimPlot(obj, group.by = "cluster")


Prefer Zarr for Explorer and Python interactive work; prefer Parquet/CSV for tables and cross-language exchange.

Space & Format Tips

Transcripts (tall table): keep Parquet; convert to CSV only for human inspection.

Matrices: keep Zarr (Explorer/Python) and optionally HDF5 (broad I/O); export MEX only if a legacy pipeline requires it.

Boundaries: Parquet is compact and typed; CSV works if you need plain text.

Version Notes

File names and folder structure can vary by Xenium Analyzer version.

If you import custom segmentation (e.g., Baysor/Cellpose via Ranger), downstream artifacts are recomputed: cells.parquet, cell_feature_matrix.*, analysis/*, analysis.zarr.zip, and metrics.

License

Add your project license here (e.g., MIT). Replace this section with your lab’s standard license text.
