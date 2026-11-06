# Fine-Tuning a Multimodal Model on FAA Sectional Charts

This project fine-tunes an open-source vision-language model to find and identify symbols on FAA sectional charts.

## Objective

The FAA database gives the position of each airport and navaid. Thus, on an official chart file with coordinates, a database lookup gives the correct answer. The model is for images with no coordinates, for example photos and screenshots of charts. On these images, a lookup cannot operate.

## Status

| Step | Description | Status |
|---|---|---|
| 1 | Convert FAA coordinates to chart pixels | Done |
| 2 | Make the tile dataset for all charts | Done |
| 3 | Measure the base model (zero-shot) | Next |
| 4 | Fine-tune with LoRA | Not started |
| 5 | Compare with baselines | Not started |
| 6 | Deploy and publish | Not started |

## Data Sources

- **Sectional charts:** 34 charts of the lower 48 states, GeoTIFF, edition of October 29, 2026. [FAA VFR Raster Charts](https://www.faa.gov/air_traffic/flight_info/aeronav/digital_products/vfr/)
- **NASR data:** APT and NAV CSV files, effective October 29, 2026. [FAA 28 Day NASR Subscription](https://www.faa.gov/air_traffic/flight_info/aeronav/aero_data/NASR_Subscription/)
- **Main map outlines:** From the [pirep](https://github.com/shanet/pirep/tree/2fe3868b215edab347e6a59511ef7d4d1017f4ee) project. See [Credits](#credits).

The charts and the NASR data must have the same edition date.

## Setup

Python 3.13 on macOS.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

If pip shows a build error for a package, install it with `--only-binary=:all:`.

## Folder Structure

```
configs/
  charts.csv                # Chart name, file name, and split
  chart_areas.json          # Main map outline of each chart
data/                       # Not in Git
  raw/
    charts/                 # FAA chart GeoTIFF files
    29_Oct_2026_APT_CSV/
    29_Oct_2026_NAV_CSV/
  processed/all/
    tiles/<chart>/          # Tile images
    checks/                 # One check image for each chart
    train.jsonl
    val.jsonl
    test.jsonl
notebooks/
  build_la_dataset.ipynb    # Steps 1 and 2 on Los Angeles only
  set_map_areas.ipynb       # Makes configs/charts.csv
  build_dataset.ipynb       # Step 2 on all charts
```

## How the Dataset Is Made

The notebook `build_dataset.ipynb` does these steps for each chart:

1. Opens the chart and converts its color table to RGB.
2. Keeps the airports (type "A" only) and the navaids inside the main map outline.
3. Converts each latitude and longitude to a pixel position on the chart.
4. Cuts the chart into tiles of 1024 x 1024 pixels with 128 pixels of overlap.
5. Keeps a tile only if:
   - Its four corners are inside the main map outline. This removes the legend panels and the border.
   - It does not touch an exclusion box. This removes insets, for example the Los Angeles Basin inset.
   - It has at least one object.
6. Writes one label for each tile: a JSON list with the type, the ID, and the pixel position of each object.
7. Saves a check image for each chart, with the kept tiles drawn on it.

The FAA georeferences only the main map. The legend panels and insets have no correct coordinates, so the dataset does not use them.

## Data Split

Each chart is fully in one split. Thus, the model never sees the test areas in training.

| Split | Charts |
|---|---|
| Train | 28 charts |
| Validation | Charlotte, Chicago, Phoenix |
| Test | Denver, Miami, Seattle |

## Label Format

Each line in `train.jsonl`, `val.jsonl`, and `test.jsonl` is one tile:

```json
{
  "chart": "Los Angeles",
  "image": "tiles/Los Angeles/r1792_c8960.png",
  "row": 1792,
  "col": 8960,
  "question": "List all airports and navaids in this image with their pixel positions.",
  "answer": "[{\"type\": \"airport\", \"id\": \"PSP\", \"x\": 412, \"y\": 305}]"
}
```

Each label is a point at the center of the symbol, not a box.

## Known Limits

- Some airports and navaids in the FAA data have no symbol on the chart. These give wrong labels.
- Areas of Mexico and Canada inside the outlines have no FAA objects. Tiles there can have missing labels.
- Exclusion boxes for insets are set by hand.
- The map outlines are from January 2024. Check the check images after each new chart edition.

## Credits

The main map outlines in `configs/chart_areas.json` come from the [pirep](https://github.com/shanet/pirep/tree/2fe3868b215edab347e6a59511ef7d4d1017f4ee) project by shanet, folder `lib/faa/charts_crop_shapefiles/sectional`. The pirep project uses the [AGPL-3.0 license](https://github.com/shanet/pirep/blob/2fe3868b215edab347e6a59511ef7d4d1017f4ee/LICENSE). The outlines were converted from shapefiles to one JSON file. Four outlines (Denver, Klamath Falls, Seattle, and Twin Cities) were converted from Lambert Conformal Conic meters to longitude and latitude.

The method is described in the blog post [Generating map tiles for FAA sectional charts with GDAL](https://www.ephemeral.cx/2023/03/generating-map-tiles-for-faa-sectional-charts-with-gdal/).

## Disclaimer

This project is not for real flight navigation.