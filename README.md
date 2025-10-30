

# =============================
# Xenium Dataflow
# Inputs → Processes → Outputs
# =============================

[INPUTS]
  ├─ IF morphology images                # DAPI ± membrane/cytoplasm (if Multimodal Cell Segmentation, MCS)
  ├─ RNA chemistry images + codebook     # Per-cycle, per-channel images + panel/codewords
  └─ Run metadata                        # Instrument run, FOVs, panel JSON

# ---------- CORE PIPELINE ----------

# 1) Decoding: convert RNA chemistry images into decoded molecules
[PROCESS] Decode
  # uses: RNA chemistry images + codebook/panel JSON
  └──▶ transcripts.parquet               # per-molecule table: x,y,(z), gene, Q-score, cycle, channel, intensity
  └──▶ transcripts.zarr.zip              # same as above, but in zarr bundle for compatibility with Xenium Explorer

# 2) Segmentation: derive nuclei/cell masks from morphology IF
[PROCESS] Segmentation
  # uses: IF morphology images (DAPI ± membrane/cytoplasm)
  └──▶ cells.zarr.zip                    # authoritative segmentation: masks, centroids, areas, per-cell geometry
      ├──▶ cell_boundaries.parquet       # simplified membrane polygons (derived from cells.zarr.zip)
      └──▶ nucleus_boundaries.parquet    # simplified nucleus polygons (derived from cells.zarr.zip)

# 3) Assignment: place decoded molecules into segmented cells
[PROCESS] Assign transcripts to cells
  # uses: transcripts.parquet + cells.zarr.zip
  └──▶ cells.parquet                     # per-cell summaries (counts, genes detected, area, %nuclear, QC)
      └──▶ cell_feature_matrix.*         # gene×cell count matrix (Q≥threshold); formats: MEX/, .h5, .zarr.zip

# 4) Secondary analysis: run on the cell×gene matrix
[PROCESS] Analyze matrix
  # uses: cell_feature_matrix.*
  └──▶ analysis/                         # CSV/TSV: PCA loadings/scores, neighbors, clustering, UMAP, DE/markers
      └──▶ analysis.zarr.zip             # packaged analysis for Xenium Explorer

# 5) QC roll-up / reporting
[PROCESS] Summarize QC
  # uses: metrics from decoding + segmentation + matrix + key images
  ├──▶ metrics_summary.csv               # run-wide KPIs (assign rate, median genes/cell, %Q20, etc.)
  └──▶ analysis_summary.html             # interactive HTML report; links thumbnails/overlays/plots


# ---------- AUXILIARY (PROVENANCE & QA) ----------

[AUX] per_cycle_channel_images/          # FROM: RNA chemistry images; QA frames used to inspect decoding inputs
[AUX] background_qc_images/              # FROM: autofluorescence frames; used to make morphology_focus/*
[AUX] morphology_focus/                  # FROM: IF morphology – background subtracted, multi-focus 2D OME-TIFFs
[AUX] overview_scan/ + *_fov_locations   # FROM: instrument overview & tiling; slide/FOV context for viewers
[AUX] decoding_qc/                       # FROM: decoder; codeword confusion, Q-score histograms, FDR vs Q
[AUX] segmentation_qc/                   # FROM: cells.zarr.zip + IF; mask overlays, size/shape distributions
[AUX] assignment_qc/                     # FROM: transcripts + cells; unassigned density, boundary distances
[AUX] morphology_thumbnails/             # FROM: morphology.ome.tif; downsampled PNGs for HTML/report
[AUX] runtime_logs/                      # FROM: analyzer; versions, CLI args, timing breakdown
[AUX] gene_panel.json                    # FROM: input design; codewords/targets used in decoding


# ---------- FILE → PROCESS LINK SUMMARY (at a glance) ----------
- RNA images + codebook   ──Decode────▶ transcripts.parquet
- IF images               ──Segment───▶ cells.zarr.zip ──simplify──▶ cell_/nucleus_boundaries.parquet
- transcripts + cells     ──Assign────▶ cells.parquet ──filter_tx──▶ cell_feature_matrix.*
- matrix                  ──Analyze───▶ analysis/ ──pack──▶ analysis.zarr.zip
- all metrics + images    ──Summarize─▶ metrics_summary.csv, analysis_summary.html

# ---------- EXPORTS FOR EXPLORER ----------
# Preferred: transcripts.zarr.zip, cell_feature_matrix.zarr.zip, analysis.zarr.zip (+ experiment.xenium manifest)
