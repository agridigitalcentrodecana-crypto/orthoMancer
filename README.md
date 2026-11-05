# Remote-sensing phenotyping pipeline — sugarcane

A modular R pipeline (orchestrated with **targets**) that starts from RGB/multispectral orthomosaics, DSMs, DTMs and the field layout (plot map), and ends with block-adjusted means per breeding material.

```
01 catalog → 02 layout+QC → 03 plot clipping → 04 segmentation → 05 masks → 06 VIs
                                      └──────────────────┬───────────────────┘
                              07 canopy closure · 08 CHM → 09 GLM/MM → 10 reports
```

## How to use
1. Edit `config/config.yml` (items marked "AJUSTAR": layout shapefile, its column names, band order).
2. Run sections 0–1 of `run.R` and check the catalog (parsed names, layers, CRS) and the alignment QC.
3. `targets::tar_make()`. A new flight only reprocesses that flight.
4. Outputs: `data/processed/` (long/wide raw tables, adjusted means, per-material summary), `reports/figuras/` (figures), `reports/qc/` (segmentation overlays).

Python (BiRefNet only): `pip install -r python/requirements.txt`. Without Python, set `segmentacao: metodo: "exg_otsu"`.

## Unique keys
| Key | Format | Example |
|---|---|---|
| `voo_id` (flight) | `<trial>_v<no.>_<YYYYMMDD>` | `ER2022_v01_20231116` |
| `parcela_uid` (plot) | `<trial>_<plot>` | `ER2022_101` |
| `chave` (plot × flight) | `<voo_id>__<parcela_uid>` | `ER2022_v01_20231116__ER2022_101` |
| trait (`trait`) | `<variable>__<statistic>` | `NDVI__media`, `chm__p90_planta`, `fechamento_dossel__prop` |

Intermediate files (in `data/interim/`): `parcelas/<trial>/<voo_id>/<type>/<parcela_uid>.tif` (type = orto, mds, mdt), `mascaras/<trial>/<voo_id>/<parcela_uid>_mask.tif`, `segmentadas/…/<parcela_uid>.tif`, `indices/<VI>/<trial>/<voo_id>/<parcela_uid>.tif` and `chm/…`.

## Methodological decisions
- **Plot pixel**: a pixel belongs to a plot if its center falls inside the polygon (same rule for clipping and counting). `cobertura` = valid pixels / polygon pixels.
- **"Vector per material"**: `vetor_por_material()` returns a list named by material with all plots × flights (file paths); `carregar_material()` opens the rasters. Works for clips, segmented clips and elevation models.
- **Segmentation**: BiRefNet receives RGB with letterboxing (plots are elongated). The mask (1 plant, 0 soil, NA outside) is applied to all bands. `gerar_overlays_qc()` samples plots for visual checking; validate against a hand-labeled sample before trusting it.
- **VIs**: computed on plant pixels only; RGB indices use chromatic coordinates (scale-independent). SAVI/EVI require reflectance.
- **Canopy closure**: `fechamento_dossel` (÷ polygon pixels) and `fechamento_dossel_valido` (÷ valid pixels).
- **CHM** = DSM − DTM per flight, negatives set to 0; statistics over all pixels and over plant pixels only (sampled by nearest neighbor).
- **Block normalization**, per trial × flight × trait: **GLM** `block + material` (quasibinomial for proportions, optional Gamma-log) and **MM** `material + (1|block)` (lmer; glmmTMB beta-binomial for proportions). Both return adjusted means (emmeans) and a material-effect test. With `material_aleatorio: true`, the MM returns BLUPs and H² (Gaussian only).

## What was tested
`tests/testthat` (~40 s) runs modules 01–10 on a synthetic experiment (3 blocks × 4 materials × 2 flights, same file naming as the real data) and checks pixel counts, plant fraction, height, material ranking and GLM × MM agreement. **Not exercised yet:** BiRefNet (needs torch/model), `_targets.R` with dynamic branching, and your real data.
