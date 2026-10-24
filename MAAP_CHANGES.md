# Changes from upstream

Upstream: https://github.com/NISAR-Science-Algorithms/NISAR_Biomass, branch `main` at commit 9aff4e2b3be1a6c734e408bc88094139ef29685a (2026-05-18), by Naveen Ramachandran, KC Cushman, Alexandra Christensen and Sassan Saatchi (JPL), Apache License 2.0. Upstream has no tags or releases.

This copy starts from upstream's file tree at 9aff4e2 without its git history. It is unchanged except for the items below. The model (`algorithm/bin/model_radar.py`), the calibrated parameters (`config/config.ini`) and the LICENSE are untouched. `.github/` was left out: its only workflow mirrors the repository to the team's GitLab, which a copy should not do.

## 2026-10-24: the notebook can run without prompts

`NISAR_L3_AGB_ProductGeneration.ipynb` asked for the database at the credentials cell and, through `functions.filter_data`, for the rows to exclude, so it could only be run by hand; papermill and other batch tools stop at the first `input()` with `StdinNotImplementedError`.

A cell tagged `parameters` (the papermill convention) now sets `DATABASE_NAME` and `EXCLUDE_ROWS`, just before the credentials cell. Both default to `None`, which keeps the prompts exactly as before. `filter_data(df, user_input=None)` asks only when no selection is given; the selection uses the same format as the prompt (`'3, 8, 14'` or `'3-5'`, empty for all rows). To run the demo unattended, from `algorithm/notebooks/`:

```sh
papermill NISAR_L3_AGB_ProductGeneration.ipynb out.ipynb -p DATABASE_NAME DEMO -p EXCLUDE_ROWS ''
```

The working directory matters: the notebook sets `MAIN_DIR` two levels above it and writes everything under `data_repo/`.

Not changed: the `earthaccess` and `asf_search` branches still ask for a user name and password.

## 2026-10-24: per-scene AGB maps are named by acquisition date

The L3 cell passed `ab_range=[54, 62]` to `colormap_plot_agb_prediction`, and the per-scene plot sliced `basename[54:62]`. In the GCOV file names that range is part of the acquisition time (for example `4757_202`), not the date, so scenes taken at the same time of day got the same file name and overwrote each other: with the 20 demo scenes only 2 per-scene files survived, and the per-scene plot showed the wrong maps. Both now use `[43, 51]`, the date, which is also the function's own default. The mean L3 map does not change.

## 2026-10-24: product version numbers work on Linux and macOS

`functions.get_version_number` searched `DIRECTORY + '\*_FILTERED_DATA.csv'`, a Windows path separator, so outside Windows it never found earlier runs: every product was version `000` and a rerun on the same day overwrote the previous one. It now uses `os.path.join`. A first run still gets `000`.

## 2026-10-24: requirements

`requirements.yml` now asks for `python>=3.12` (the file-name cell nests quotes inside an f-string, which older Python rejects) and lists `s3fs`, which the notebook imports for the demo data but which only arrived through `earthaccess`.

## 2026-10-24: README

How to run the notebook without prompts.

## Notes for running it

- Demo inputs: 20 simulated NISAR L2 GCOV granules for the LENO site (3.77 GB) and a land-cover raster, streamed anonymously from the public Wasabi bucket `s3://nisar-public-ebd` (endpoint `https://s3.us-west-1.wasabisys.com`). No login.
- All 20 scenes take many hours on 4 cores (the per-pixel inversion is about 25 minutes per scene). `EXCLUDE_ROWS 2-19` keeps the first two scenes, about an hour.
- The L3 product is written to `data_repo/output/nisar/LENO/<FILE_NAME>/output/<FILE_NAME>.tif` (Cloud-Optimized GeoTIFF, Mg/ha, 100 m).

## Check

On 2026-10-24, in a fresh conda env built from `requirements.yml` (Python 3.14.9, GDAL 3.13.3, rasterio 1.5.2, numpy 2.5.4, pandas 3.1.0, scipy 1.18.1) plus papermill, from `algorithm/notebooks/`:

```sh
papermill NISAR_L3_AGB_ProductGeneration.ipynb out.ipynb -p DATABASE_NAME DEMO -p EXCLUDE_ROWS 2-19
```

It ran to the end with no prompts: 72 minutes on 4 cores, 4.9 GB peak for the largest process. The L3 mean AGB over the 480,382 scene-mask pixels is the same as from the unchanged code (mean 113.31 Mg/ha, median 116.0, 5th to 95th percentile 17.2 to 203.1). The per-scene maps now come out one per scene, `_agb_predictions_20210806.tif` (mean 100.23) and `_agb_predictions_20210820.tif` (mean 126.39); before the fix only one file survived and it held the second scene.
