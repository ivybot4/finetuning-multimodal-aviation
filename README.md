# Fine-Tuning a Multimodal Model on FAA Sectional Charts

This project fine-tunes an open-source vision-language model to find and identify symbols on FAA sectional charts.

## Objective

The FAA database gives the position of each airport and navaid. Thus, on an official chart file with coordinates, a database lookup gives the correct answer. The model is for images with no coordinates, for example photos and screenshots of charts. On these images, a lookup cannot operate.

## Status

| Step | Description | 
|---|---|
| 1 | Convert FAA coordinates to chart pixels | 
| 2 | Make the Los Angeles tile dataset | 
| 3 | Measure the base model (zero-shot) | 
| 4 | Fine-tune with LoRA |
| 5 | Compare with baselines |
| 6 | Deploy and publish | 

## Data Sources

- **Sectional chart:** Los Angeles, GeoTIFF, edition of October 29, 2026. [FAA VFR Raster Charts](https://www.faa.gov/air_traffic/flight_info/aeronav/digital_products/vfr/)
- **NASR data:** APT and NAV CSV files, effective October 29, 2026. [FAA 28 Day NASR Subscription](https://www.faa.gov/air_traffic/flight_info/aeronav/aero_data/NASR_Subscription/)

The chart and the data must have the same edition date.

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
data/
  raw/                      # FAA downloads (not in Git)
  processed/los_angeles/
    tiles/                  # Tile images
    train.jsonl             # Training labels
    test.jsonl              # Test labels
notebooks/
  build_la_dataset.ipynb    # Steps 1 and 2
```

## How the Dataset Is Made

The notebook `build_la_dataset.ipynb` does these steps:

1. Opens the chart and converts its color table to RGB.
2. Loads the airports (type "A" only) and the navaids from the NASR files.
3. Converts each latitude and longitude to a pixel position on the chart.
4. Keeps only the objects inside the main map area. The legend panels and the Los Angeles Basin inset are not used, because the FAA georeferences only the main map.
5. Cuts the map area into tiles of 1024 x 1024 pixels with 128 pixels of overlap. It keeps only the tiles with at least one object.
6. Writes one label for each tile: a JSON list with the type, the ID, and the pixel position of each object.
7. Splits the tiles by area: the left 80% of the map for training and the right 20% for testing. Tiles that cross the split line are not used.

Result for Los Angeles: 88 tiles.

## Label Format

Each line in `train.jsonl` and `test.jsonl` is one tile:

```json
{
  "image": "tiles/la_r1500_c4700.png",
  "row": 1500,
  "col": 4700,
  "question": "List all airports and navaids in this image with their pixel positions.",
  "answer": "[{\"type\": \"airport\", \"id\": \"PSP\", \"x\": 412, \"y\": 305}]"
}
```

## Known Limits

- The dataset has only one chart. It is for testing the pipeline, not for training a good model.
- Some airports and navaids in the FAA data have no symbol on the chart. These give wrong labels.
- The map area limits are set by hand for each chart.

## Disclaimer

This project is not for real flight navigation.