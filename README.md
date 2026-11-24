# CAMS Natural Dust Service Tool

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/rgaratac/natural_dust_service_tool/blob/main/Dust_deduction_tool.ipynb)

The tool estimates how much of the PM10 and PM2.5 measured at European air-quality stations comes from natural desert dust, and subtracts that contribution from the measurements.

It supports the natural-source assessment of **Article 16 of the Ambient Air Quality Directive (EU) 2024/2881**: identifying where exceedances of the limit values are attributable to natural sources. The method follows the European Commission guidance on natural dust contributions. Dust days are identified with the **CAMS European air quality interim reanalysis** (dust). The natural load is quantified at a reference station, following the guidance:

* stations are classified as regional background (**RB**), urban background (**UB**) or traffic/industrial (**TH**);
* every station is paired with a suitable reference station;
* the natural load is quantified at the paired RB (**Option 1**) or by self-calibration of a UB station (**Option 2**);
* the pairing can be reviewed and edited on a map or as a CSV file.

Created by Jessie Zhang and Roger Garatachea Solé, TNO.

## Quick start

1. Open [`Dust_deduction_tool.ipynb`](Dust_deduction_tool.ipynb) in Google Colab (badge above).
2. Run section 1. It installs the missing packages and downloads the helper modules from this repository.
3. In section 2, choose the countries, year, pollutant, dataset and time resolution. For a first run, set the download flags to `True` and `LOAD_PARQUET_DATA = False`.
4. Download the station metadata from the [EEA Air Quality Viewer](https://discomap.eea.europa.eu/App/AQViewer/index.html?fqn=Airquality_Dissem.b2g.measurements). Save it as `DataExtract.csv` in the output folder (`MyDrive/CAMS_Tool_output` in Colab). A copy is included in `Data/DataExtract.csv`.
5. For the CAMS download, create a free account on the [Copernicus Atmosphere Data Store](https://ads.atmosphere.copernicus.eu/). Accept the licence of `cams-europe-air-quality-reanalyses` and paste your API key when the notebook asks for it.
6. Run the notebook from top to bottom. On later runs, set the download flags back to `False` and `LOAD_PARQUET_DATA = True` to reload the results and redraw the figures.

The code cells are hidden; double-click a cell to show the code. Settings are in the cells marked *Edit here*.

## Notebook sections

| Section | What it does |
|---|---|
| 1 Import packages | Installs missing packages, downloads and imports the helper modules |
| 2 Settings | Countries, year, pollutant, dataset, time resolution, run flags, limit values, pairing and method parameters |
| 3 Download data | EEA observations (or your own observations, 3.1.1) and CAMS dust. Stored files are reused; only missing or damaged files are downloaded |
| 4 Data preparation | Validity / verification filter, coverage filter, station metadata, daily CAMS dust at each station |
| 5 Station pairing | RB / UB / TH classification, default pairing, interactive map editor, CSV export / import |
| 6 Flag dust days | CAMS dust above `CAMS_dust_threshold` (default 5 µg/m³) |
| 7 Calculate the dust contribution | Natural load (Option 1 / Option 2) and the adjusted daily series |
| 8 Save or load results | Parquet file, station summary (CSV), optional threshold sweep, optional hourly + daily combination |
| 9 Visualisation | Interactive map, clickable time series, static maps, exceedance and compliance report |

## Method

**Dust days.** A day is a natural-event (dust) day when CAMS dust at the station is above `CAMS_dust_threshold`. Dust days are excluded from every reference window.

**Station classes** (from `Air Quality Station Type` and `Air Quality Station Area` in `DataExtract.csv`):

| Class | Rule |
|---|---|
| RB, regional background | rural area and background type |
| UB, urban background | urban or suburban area and background type |
| TH, hotspot | traffic or industrial type |

**Pairing.** Each non-RB station is paired with the nearest RB within `MAX_RB_DISTANCE_KM` (100 km) and `MAX_RB_ALTITUDE_DIFF_M` (500 m), in the same country by default. A TH station without a suitable RB is paired with the nearest UB within `MAX_UB_DISTANCE_KM` (50 km). An RB station is its own reference. With `PAIRING_MODE = 'none'`, every station is its own reference.

**Natural load.**

| Method | When | Natural load |
|---|---|---|
| `option1` | suitable RB | At the RB: reference = 50th percentile of days *d*−15 … *d*+15 without dust; load = max(RB(*d*) − reference, 0). The same load is subtracted at every station paired to that RB |
| `option2` | UB without suitable RB | Self-calibration at the station: the 3 days before and after the dust episode, without dust days |
| `option2_ub_paired` | TH without suitable RB | Option 2 load of the paired UB |
| `none` | no valid reference | nothing is subtracted |

A TH station can be set to `option2` by hand, as a manual exception to the guidance.

The load is subtracted on every dust day, with or without an exceedance: adjusted value = measured − load, never below 0. Days with too few valid reference days (`MIN_REFERENCE_DAYS_OPTION1` = 10, `MIN_REFERENCE_DAYS_OPTION2` = 3) are flagged `insufficient_reference` and keep the measured value. A dust day with a load > 0 is a `quantified_dust_day`.

**Limit values** (µg/m³, Directive (EU) 2024/2881, editable in section 2):

| | Daily | Annual |
|---|---|---|
| PM10 | 45 | 20 |
| PM2.5 | 25 | 10 |

## Time zones

* EEA hourly time stamps are in CET (fixed UTC+1, no daylight saving).
* EEA daily values use the station's local day.
* CAMS is in UTC.

With `HOURLY_DAY_DEFINITION = 'local'` (default), hourly observations are shifted to the station's local standard time. The shift uses the fixed `Timezone` offset in `DataExtract.csv`. CAMS dust is then averaged over the same local day (00–24 h), so hourly and daily results use the same day. With `'utc'`, the observations and CAMS are averaged over the UTC day.

For a station with offset *h* hours, the CAMS mean of local day D is the mean of the UTC hours in `[D 00:00 − h, D+1 00:00 − h)`. A day with fewer than 18 hours of CAMS data is left missing.

## Outputs

Files are written to the output folder: `MyDrive/CAMS_Tool_output` in Colab, or the project folder when `USE_GOOGLE_DRIVE = False`. `{countries}` is the sorted list of country codes, e.g. `ES_PT`.

| Product | Name |
|---|---|
| EEA download | `{dataset}_{POLLUTANT}_{hour/day}_{YEAR}_{countries}.zip` |
| CAMS download | `IRA_dust/CAMS_IRA_*.zip` (December of the previous year + four quarters) |
| Daily results | `CAMS_dust_{POLLUTANT}_deduction_{dataset}_{hour/day}_{YEAR}_{countries}.parquet` (hourly data on the local day end in `_localday`) |
| Station summary | `{countries}_station_annual_{POLLUTANT}_{dataset}_{hour/day}_{YEAR}.csv` |
| Station pairing | `station_pairing_default.csv`, `station_pairing_user.csv` (edited) |

**Daily columns** include `station_class`, `method`, `reference_station_id`, `reference_value`, `natural_contribution`, `cams_dust_flag`, `quantified_dust_day`, `adjusted_value`, `insufficient_reference` and `pairing_source`.

**Station summary columns** include:
* `cams_dust_days`, `quantified_dust_days`, `insufficient_reference_days`;
* `exceedances_measured`, `exceedances_adjusted`, `exceedances_natural`;
* `annual_mean_measured`, `annual_mean_adjusted`;
* `annual_limit_exceeded_measured`, `annual_limit_exceeded_adjusted`.

**Exceedance report** (section 9.4). Per station, it gives the annual mean and the exceedance days before and after the deduction, with a compliance status for the daily and the annual limit value:
* green: complies without the deduction;
* orange: complies only with the deduction;
* red: still exceeds after the deduction.

## Repository contents

| File | Role |
|---|---|
| `Dust_deduction_tool.ipynb` | The tool (notebook) |
| `util.py` | Shared helpers: coverage filter, CAMS interpolation, time series and map plots |
| `cams_data_to_eea_daily.py` | EEA / CAMS download helpers, daily EEA path, CAMS local-day means at the stations, own observations |
| `pairing.py` | Station classification, default pairing, manual edits, pairing CSV round trip |
| `natural_contribution.py` | Option 1 / Option 2 natural load, adjusted series, station indicators |
| `pairing_viz.py` | Pairing maps (Plotly, Cartopy) and the map editor |
| `dust_threshold_sensitivity.py` | Sweep of the CAMS dust threshold (section 8.2.1) |
| `exceedance_report.py` | Compliance table and maps (section 9.4) |
| `cams_data_download.py` | Standalone CAMS download script |
| `Data/DataExtract.csv` | Example EEA station metadata |
| `tests/` | Unit tests |

## Running locally

1. Clone the repository and install the requirements below.
2. Leave `USE_GOOGLE_DRIVE = False` (default outside Colab); outputs go to `project_dir`.
3. Place `DataExtract.csv` in the project folder (or keep `Data/DataExtract.csv`).
4. The interactive maps are designed for Colab and may not work in VS Code or on an HPC system; use the static maps in section 9.3.

**Requirements:** `numpy`, `pandas`, `xarray`, `netCDF4`, `scipy`, `matplotlib`, `cartopy`, `plotly` (≥ 6), `anywidget`, `ipywidgets`, `ipyleaflet`, `pyarrow`, `requests`, `tqdm`, `dask`, `psutil`, `cdsapi` (≥ 0.7.7).


