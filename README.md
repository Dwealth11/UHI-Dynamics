# NDVI, LST, and UHI Analysis — Southwest Nigeria

**Author:** Olaleye Daniel Abimbola  
**Project:** NDVI, Land Surface Temperature (LST) and Urban Heat Island (UHI) analysis for selected LGAs in Southwest Nigeria using Google Earth Engine.

## Overview
This repository contains a Google Earth Engine (GEE) script that:
- defines ROIs for Akure, Ibadan, Osogbo, and Ondo,
- computes NDVI and LST for dry-season composites (Nov–Feb),
- computes UHI (pixel LST minus region mean LST),
- plots NDVI and LST time series and UHI intensity across years.

## Files
- `gee/ndvi-lst-uhi.js` — Main GEE script (paste into Earth Engine Code Editor).
- `docs/figures/` — Suggested folder to store exported maps/charts.
- `.github/ISSUE_TEMPLATE.md` — Issue template.
- `.github/PULL_REQUEST_TEMPLATE.md` — PR template.
- `LICENSE` — MIT license.
- `.gitignore` — Git ignore file.

## How to run
1. Open [Google Earth Engine Code Editor](https://code.earthengine.google.com).
2. Create a new script and paste the contents of `gee/ndvi-lst-uhi.js`.
3. Click **Run**. The map and charts will appear in the Code Editor console & map.

> **Note:** The script uses Landsat Collection level-2 products. Ensure your GEE account is active and has access to public datasets.

## Adding outputs
- Export map images / charts from the UI to `docs/figures/` and commit them to the repo for portfolio use.
- Optionally add a `notebooks/` folder with analysis notes or exported CSVs.

## License
MIT — see `LICENSE`.

## Contributing
Contributions, issues and feature requests are welcome. Please check `.github/` templates before opening issues.

