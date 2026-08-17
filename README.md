# TigerSpoter — Automated Camera Trap Triage & Tiger Movement Intelligence

**Target reserve:** Pench Tiger Reserve, Madhya Pradesh, India  
**Architecture:** Zero-shot / pretrained models only. No training from scratch.  
**Status:** MVP — functional end-to-end pipeline, documented simplifications below.

---

## System Overview

```
Raw camera-trap frames
        │
        ▼
┌──────────────────────────────────┐
│  BOX 1                           │  Stage A: MegaDetector (MDv5a)
│  Detection + Tiger Species Filter│           Detects all animals / people / vehicles
│  box1_species_filter_lite.py     │  Stage B: EfficientNet-B4 (ImageNet-pretrained)
│                                  │           Keeps ONLY tiger detections
└──────────────┬───────────────────┘
               │ results_tigers_only.json
               ▼
┌──────────────────────────────────┐
│  BOX 2                           │  MegaDescriptor-T-224 (wildlife_tools + timm)
│  Individual Tiger ID             │  Assigns Tiger_NN IDs, handles multi-tiger frames
│  individual_id_v3.py             │
└──────────────┬───────────────────┘
               │ sightings_real table in tigers.db
               ▼
┌──────────────────────────────────┐
│  BOX 3                           │  shapely (no HTML — all maps live in dashboard)
│  Occupancy Mapping + Paths       │  Directed movement paths, convex-hull home ranges,
│  occupancy_mapping_v3.py  ◄MAIN  │  activity centroids → dashboard/movement_paths.json
└──────────────┬───────────────────┘
               │ tiger_history table in tigers.db
               ▼
┌──────────────────────────────────┐
│  BOX 4                           │  Pure Python (no extra geo deps)
│  Deviation Alerting              │  Flags range shifts, new stations,
│  deviation_alerting.py           │  village proximity, prolonged absences
└──────────────┬───────────────────┘
               │ Folders/alerts.log + stdout
               ▼
┌──────────────────────────────────┐
│  DASHBOARD                       │  Static HTML + Leaflet.js (OpenStreetMap)
│  dashboard/index.html            │  Reads dashboard_data.json + movement_paths.json
│  dashboard/export_dashboard.py   │  Interactive map, alerts, proximity, movement paths
└──────────────────────────────────┘

┌──────────────────────────────────┐
│  VILLAGE WEBCAM MONITOR          │  OpenCV (cv2) — laptop camera live demo
│  village_video_monitor.py        │  Tiger-ONLY warnings; demo mode built in
└──────────────────────────────────┘
```

---

## Script Reference

| Script | Box | Input | Output |
|--------|-----|-------|--------|
| `Folders/box1_species_filter_lite.py` | Box 1 | `Sample_image/Rawdata/` | `Folders/json/results_tigers_only.json` |
| `Folders/individual_id_v3.py` | Box 2 | `results_tigers_only.json` | `tigers.db → sightings_real` |
| `Folders/occupancy_mapping_v3.py` | Box 3 (**main**) | `sightings_real` + `stations.csv` | `dashboard/movement_paths.json` + `tigers.db → tiger_history` |
| `Folders/deviation_alerting.py` | Box 4 | `tiger_history` | `Folders/alerts.log` + stdout |
| `dashboard/export_dashboard.py` | Dashboard | `tigers.db` + `alerts.log` | `dashboard/dashboard_data.json` |
| `Folders/village_video_monitor.py` | Live Monitor | Laptop webcam (camera 0) | `Folders/json/village_video_results.json` + `Folders/village_alerts.log` |

---

## How to Run (End to End)

```powershell
# Activate the venv (Windows)
.\venv\Scripts\Activate.ps1

# Box 1 — detect animals, filter to tigers only
python Folders\box1_species_filter_lite.py

# Box 2 — individual ID
python Folders\individual_id_v3.py

# Box 3 — movement-path occupancy mapping (JSON output, no HTML)
python Folders\occupancy_mapping_v3.py
# → writes dashboard/movement_paths.json

# Box 4 — deviation alerts
python Folders\deviation_alerting.py
# → check Folders\alerts.log

# Dashboard exporter — bake all data into dashboard_data.json
python dashboard\export_dashboard.py

# Open the dashboard (no server needed — works as file://)
start dashboard\index.html
```

### Village Webcam Monitor (live demo)

```powershell
# Make sure opencv-python is installed
pip install opencv-python

# Run the live monitor (uses your laptop camera)
python Folders\village_video_monitor.py
```

- Press **q** in the camera window or **Ctrl-C** to stop.
- `DEMO_MODE = True` (default) fires a simulated tiger alert every 30 s.
- To use a real YOLOv8 detector: `pip install ultralytics` then set `DEMO_MODE = False`.
- Tiger detections appear instantly in the **Village Video Monitor** panel of the dashboard.
- **Only tiger detections raise warnings** — other motion is silently ignored.

> **First run only:** Box 1 downloads MegaDetector weights (~600 MB) and
> EfficientNet-B4 weights (~74 MB) automatically and caches them locally.
> Subsequent runs use the cache and start immediately.

---

## Data Layout

```
TigerSpoter/
├── stations.csv                          GPS lookup (see note below)
├── villages.csv                          Village coords for proximity alerts
├── requirements.txt
├── Folders/
│   ├── box1_species_filter_lite.py       Box 1 — detection + species filter
│   ├── individual_id_v3.py               Box 2 — individual ID
│   ├── occupancy_mapping_v3.py           Box 3 — movement-path occupancy mapping ◄ MAIN
│   ├── deviation_alerting.py             Box 4 — deviation alerting
│   ├── village_video_monitor.py          Live webcam tiger monitor
│   ├── tigers.db                         SQLite database
│   │     ├── sightings_real             (real Rawdata run — tiger sightings)
│   │     └── tiger_history              (per-run occupancy summaries)
│   ├── json/
│   │   ├── results_megadetector_raw.json  Box 1 Stage A output
│   │   ├── results_tigers_only.json       Box 1 Stage B output → Box 2 input
│   │   ├── species_report.txt             Per-detection species audit log
│   │   └── village_video_results.json     Webcam monitor output → dashboard
│   ├── crops_real/                       Cropped tiger detections (real Rawdata)
│   ├── alerts.log                        Box 4 alert history
│   └── village_alerts.log                Webcam tiger alert log
├── dashboard/
│   ├── index.html                        Interactive web dashboard (Leaflet/OSM)
│   ├── export_dashboard.py               Writes dashboard_data.json from tigers.db
│   ├── dashboard_data.json               Main dashboard data (sightings, alerts…)
│   └── movement_paths.json               Movement-path data from Box 3
└── Sample_image/
    ├── Rawdata/
    │   ├── Station_01/ … Station_10/    Real camera-trap photos
    └── tiger/                            Stock-photo test set
```

---

## Key Configuration

### Box 1 — Detection + Species Filter (`box1_species_filter_lite.py`)

| Parameter | Value | Notes |
|---|---|---|
| `DETECTOR_MODEL` | `MDV5A` | Auto-downloaded ~600 MB on first run |
| `DETECTION_CONFIDENCE_THRESHOLD` | `0.4` | Raised from MegaDetector default 0.2 to suppress vegetation false-positives |
| `MIN_CROP_AREA_FRACTION` | `0.01` | Skips crops < 1% of frame area (sensor noise / tiny blobs) |
| `TIGER_CLASS_IDX` | `292` | ImageNet class 292 = "tiger, Panthera tigris" |
| `TIGER_TOP1_THRESHOLD` | `0.15` | Top-1 prediction must be tiger at this confidence or above |
| `TIGER_TOP5_THRESHOLD` | `0.10` | Tiger anywhere in top-5 at this confidence also passes |

### Box 2 — Individual ID (`individual_id_v3.py`)

| Parameter | Value | Notes |
|---|---|---|
| `RESULTS_JSON` | `results_tigers_only.json` | Species-filtered Box 1 output |
| `AUTO_MATCH_THRESHOLD` | `0.85` | Confident same individual — auto-accepted |
| `REVIEW_THRESHOLD` | `0.55` | Probable match — flagged `needs_review` for biologist |
| Below review threshold | — | New individual enrolled |

### Box 3 — Occupancy Mapping (`occupancy_mapping_v3.py`)

| Parameter | Value | Notes |
|---|---|---|
| `TABLE_PRIMARY` | `sightings_real` | Preferred table (has `station_id` + timestamp) |
| `TABLE_FALLBACK` | `sightings_v3` | Older schema (image_file column) |
| `OUTPUT_JSON` | `dashboard/movement_paths.json` | **No HTML file is generated** |
| `RESERVE_CENTER_LAT/LON` | `22.33`, `80.61` | Pench / Kanha placeholder centre |

### Village Webcam Monitor (`village_video_monitor.py`)

| Parameter | Value | Notes |
|---|---|---|
| `CAMERA_INDEX` | `0` | Default laptop webcam |
| `DEMO_MODE` | `True` | Simulates tiger alert every 30 s for live demo |
| `DEMO_INTERVAL_SECONDS` | `30` | Interval between simulated alerts |
| `CONFIDENCE_THRESHOLD` | `0.50` | Min confidence for real detector |
| `TIGER_CLASS_NAMES` | `{"cat"}` | COCO proxy — replace with custom model class |

### Box 4 — Alerts (`deviation_alerting.py`)
- Core range shift threshold: **15 km**
- Buffer range shift threshold: **5 km**
- Village/border stations: `Station_09`, `Station_10` (update with real metadata)

---

## Dashboard Panels

| Panel | Data source | Notes |
|---|---|---|
| 🗺️ Tiger Movement Map | `dashboard_data.json` | Leaflet/OSM, villages, danger zones, paths |
| 🐯 Individual Tiger Sightings | `dashboard_data.json` | Searchable + filterable table |
| 🔔 Active Alerts | `dashboard_data.json` | Village proximity, range shifts, absences |
| ⚠️ Tiger Near Village | `dashboard_data.json` | Critical (<2 km) / Warning (<5 km) tabs |
| 📹 Village Video Monitor | `village_video_results.json` | Tiger-only detections from webcam |
| 🗺️ Movement Paths | `movement_paths.json` | Per-tiger path list; click to fly-to on map |

---

## Known Simplifications (read as competence, not weakness)

### 1. GPS Coordinates Are Kanha NP Placeholders
`stations.csv` contains coordinates from **Kanha National Park** (~100 km from Pench),
used as a geographically realistic stand-in.  
→ *Action required:* replace with real Pench station GPS when available.

### 2. Species Filter Is a General ImageNet Classifier
EfficientNet-B4 pretrained on ImageNet-1k reliably distinguishes tigers from
Pench's other common fauna (deer, boar, langur, peacock, jackal). It may
occasionally pass a leopard under very poor lighting — an acceptable MVP tradeoff.

### 3. MegaDescriptor Not Fine-Tuned on Pench Tigers
The individual ID model (`MegaDescriptor-T-224`) is pretrained but not fine-tuned
on Pench-specific individuals. ID assignments should be validated by a field
biologist before use in official records.

### 4. Timestamp Fallback Chain
Most images in `Rawdata` do not follow the `YYYYMMDD_HHMMSS` filename pattern.
The pipeline falls back to: (1) filename pattern, (2) EXIF `DateTimeOriginal`,
(3) file modification time (least accurate).

### 5. Gallery Not Persisted Between Runs
The MegaDescriptor in-memory gallery is rebuilt each run. For production, persist
embeddings per individual in `tigers.db`.

### 6. Convex Hull Area Approximation
Home-range area uses a flat-earth approximation: `hull.area × 111² km²`.
Replace with a proper equal-area projection for publication-quality estimates.

### 7. Box 4 Alerts on First Run
Deviation alerting requires ≥ 2 runs to compare. On the first run, only
`VILLAGE_PROXIMITY` and `NEW_STATION` alerts will fire.

### 8. Webcam Tiger Detection Is Demo-Mode by Default
`village_video_monitor.py` ships with `DEMO_MODE = True`, which simulates
tiger detections. For real detection, install `ultralytics` and set
`DEMO_MODE = False` — the script auto-detects YOLOv8 and switches modes.

---

## Dependencies

```
megadetector      # MegaDetector v5 (MDv5a) — animal detection
torch             # PyTorch backend
torchvision       # EfficientNet-B4 species classifier + image transforms
timm              # MegaDescriptor-T-224 model
Pillow            # Image I/O, cropping, EXIF extraction
wildlife-tools    # MegaDescriptor embedding extractor
shapely           # Convex-hull geometry (Box 3)
opencv-python     # Webcam capture (village_video_monitor.py)
# ultralytics     # Optional — YOLOv8 for real tiger detection (pip install ultralytics)
```

Install all:
```powershell
pip install -r requirements.txt
```

---

## Reviewers / Field Staff Notes

- Individual IDs (`Tiger_01`, `Tiger_02`, …) are assigned sequentially within a run.
  They do **not** correspond to named individuals in field records without a manual
  cross-reference step.
- Sightings with `status = needs_review` in `sightings_real` should be checked by eye.
  Crop images are in `Folders/crops_real/`.
- The per-detection species audit log is at `Folders/json/species_report.txt`.
- All alert logic and thresholds are in `deviation_alerting.py` and can be adjusted
  without touching any other component.
- Movement path data is in `dashboard/movement_paths.json` — regenerate after each
  Box 3 run with `python Folders/occupancy_mapping_v3.py`.
#   T i g e r S p o t e r v 2  
 