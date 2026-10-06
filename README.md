# Automatic-Number-Plate-Detection-and-Storage-System
An end-to-end Automatic Number Plate Recognition (ANPR) system developed using Python, OpenCV, Tesseract OCR, and MySQL. The system captures vehicle frames through a webcam, detects and localizes license plates using contour detection, extracts alphanumeric characters using OCR, and stores vehicle records in a MySQL database.
# Autonomous Number Plate Recognition (ANPR)

A Python system that watches a camera, video file or network stream, detects vehicle number plates, reads them with OCR, and stores every confirmed plate in a SQLite database together with a cropped image of the plate. It runs unattended and avoids saving the same vehicle over and over.

---

## Contents

1. [Features](#features)
2. [How it works](#how-it-works)
3. [Project files](#project-files)
4. [Installation](#installation)
5. [Quick start](#quick-start)
6. [Command-line options](#command-line-options)
7. [Example output](#example-output)
8. [Database schema](#database-schema)
9. [Querying and exporting data](#querying-and-exporting-data)
10. [Improving accuracy](#improving-accuracy)
11. [Running on Raspberry Pi / embedded boards](#running-on-raspberry-pi--embedded-boards)
12. [Troubleshooting](#troubleshooting)
13. [Limitations](#limitations)
14. [Privacy and legal notes](#privacy-and-legal-notes)

---

## Features

- **Multiple inputs**: webcam index, video file, or RTSP/HTTP stream.
- **Two detection modes**: classical OpenCV (contours + Haar cascade fallback) that needs no training, or an optional YOLO model for higher accuracy.
- **OCR with EasyOCR**, with image preprocessing (upscaling, CLAHE contrast, blur) and support for two-line plates.
- **Indian plate rules**: position-based correction of common OCR mistakes (`O`/`0`, `I`/`1`, `S`/`5`, `B`/`8`) and format validation for standard plates and BH-series plates.
- **Multi-frame voting**: a plate is saved only after it has been read several times, which removes one-off misreads.
- **Duplicate suppression**: a cooldown period stops the same parked or slow-moving vehicle from filling the database.
- **Persistent storage**: SQLite database plus JPEG crops of each plate.
- **Built-in search and CSV export**.
- **Headless friendly**: the live preview window is optional.

---

## How it works

```
 camera / video / stream
          |
          v
  +----------------+     every Nth frame (default 3)
  | Frame grabber  |
  +----------------+
          |
          v
  +----------------+     YOLO model if --weights is given,
  | Plate detector |     otherwise contours -> Haar cascade
  +----------------+
          |  plate crops
          v
  +----------------+     grayscale, upscale, CLAHE, blur
  |  EasyOCR read  |     letters and digits only
  +----------------+
          |  raw text + confidence
          v
  +----------------+     clean text, fix letter/digit positions,
  |  Validation    |     check Indian plate format, min confidence
  +----------------+
          |
          v
  +----------------+     must be seen >= min-hits times within ~2 s,
  | Voting/cooldown|     then not re-saved for `cooldown` seconds
  +----------------+
          |
          v
  +----------------+
  | SQLite + JPEG  |     plates.db and plate_images/
  +----------------+
```

### Detection

- **Contour method** (default): bilateral filter, Canny edges, then keeps four-sided shapes whose aspect ratio is between 1.2 and 6.0 (covers two-wheeler plates at about 1.5 up to car plates at about 4 to 5). Overlapping boxes are merged by IoU and at most 3 plates are processed per frame.
- **Haar fallback**: if no contour candidate is found, OpenCV's bundled `haarcascade_russian_plate_number.xml` is tried.
- **YOLO** (optional): pass `--weights your_plate_model.pt` and the model's boxes are used instead.

### Text cleanup

Indian plates follow `SS DD L{1,3} DDDD` (state, district, series, number), for example `TN09BQ1234`. After OCR, the first two characters are forced to letters, the next two to digits, the middle characters to letters, and the last four to digits. This fixes mistakes such as `TN0980 1234` style confusions without any retraining. BH-series plates (`22BH1234AA`) are also recognised.

### Voting and cooldown

All timing values are converted to frames using the source's FPS, so the logic behaves the same on live cameras and on video files processed faster than real time.

---

## Project files

| File               | Purpose                                              |
|--------------------|------------------------------------------------------|
| `anpr.py`          | The complete application (detector, OCR, storage, CLI) |
| `requirements.txt` | Python dependencies                                  |
| `README.md`        | This document                                        |

Files created at run time:

| Path             | Purpose                                         |
|------------------|-------------------------------------------------|
| `plates.db`      | SQLite database of saved detections             |
| `plate_images/`  | Cropped JPEG of each saved plate                |

---

## Installation

Requires Python 3.9 or newer.

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

The first run downloads the EasyOCR English model (roughly 100 MB), so an internet connection is needed once. EasyOCR installs PyTorch; on a machine with an NVIDIA GPU, install the CUDA build of PyTorch first and run with `--gpu`.

For YOLO support, additionally:

```bash
pip install ultralytics
```

---

## Quick start

```bash
# Webcam 0 with a live preview window (press q to quit)
python anpr.py --source 0 --show

# Process a recorded video
python anpr.py --source traffic.mp4

# IP camera
python anpr.py --source "rtsp://user:pass@192.168.1.20:554/stream1"

# Use a trained YOLO plate detector
python anpr.py --source traffic.mp4 --weights plate.pt
```

Stop at any time with `Ctrl+C` (or `q` in the preview window). The database is closed cleanly.

---

## Command-line options

| Option          | Default          | Description                                                        |
|-----------------|------------------|--------------------------------------------------------------------|
| `--source`      | `0`              | Camera index, video file path, or stream URL                       |
| `--db`          | `plates.db`      | SQLite database path                                               |
| `--images`      | `plate_images`   | Folder for plate crops                                             |
| `--weights`     | none             | YOLO `.pt` file for plate detection                                |
| `--every`       | `3`              | Process every Nth frame (higher = faster, may miss fast vehicles)  |
| `--min-conf`    | `0.4`            | Minimum OCR confidence (0 to 1)                                    |
| `--min-hits`    | `2`              | Reads required within about 2 seconds before a plate is saved      |
| `--cooldown`    | `30`             | Seconds before the same plate can be saved again                   |
| `--no-validate` | off              | Also store reads that do not match Indian plate formats            |
| `--gpu`         | off              | Run EasyOCR on the GPU                                             |
| `--show`        | off              | Display live preview with boxes and plate text                     |
| `--search TEXT` | none             | Search stored plates (partial match) and exit                      |
| `--export FILE` | none             | Export the whole database to CSV and exit                          |

---

## Example output

> The sample outputs below are **illustrative**, showing the format the program produces. Plate numbers, times and confidences will differ on your own footage.

### Running on a video

```
$ python anpr.py --source traffic.mp4
19:52:03 Source opened (30 fps). Loading OCR model...
19:52:09 SAVED  TN09BQ1234   conf=0.87
19:52:14 SAVED  KA01AB4321   conf=0.79
19:52:21 SAVED  TN22CD5678   conf=0.91
19:52:26 SAVED  MH12DE9876   conf=0.66
19:52:40 Stream ended.
```

Each `SAVED` line means that plate passed validation, was read at least `--min-hits` times, and has been written to the database and `plate_images/`.

### Live preview (`--show`)

A window opens with each recognised plate outlined in green and labelled with its text and confidence, for example `TN09BQ1234 0.87`.

### Searching the database

```
$ python anpr.py --search TN09
(1, 'TN09BQ1234', 0.87, '2026-10-06T19:52:09', 'traffic.mp4')
```

Each row is `(id, plate, confidence, timestamp, source)`, newest first.

### Exporting to CSV

```
$ python anpr.py --export plates.csv
Exported to plates.csv
```

`plates.csv`:

```
id,plate,confidence,valid,timestamp,source,image_path
1,TN09BQ1234,0.87,1,2026-10-06T19:52:09,traffic.mp4,plate_images/TN09BQ1234_20261006_195209_412873.jpg
2,KA01AB4321,0.79,1,2026-10-06T19:52:14,traffic.mp4,plate_images/KA01AB4321_20261006_195214_108344.jpg
3,TN22CD5678,0.91,1,2026-10-06T19:52:21,traffic.mp4,plate_images/TN22CD5678_20261006_195221_730915.jpg
4,MH12DE9876,0.66,1,2026-10-06T19:52:26,traffic.mp4,plate_images/MH12DE9876_20261006_195226_265102.jpg
```

### Folder after a run

```
.
├── anpr.py
├── requirements.txt
├── plates.db
└── plate_images/
    ├── TN09BQ1234_20261006_195209_412873.jpg
    ├── KA01AB4321_20261006_195214_108344.jpg
    ├── TN22CD5678_20261006_195221_730915.jpg
    └── MH12DE9876_20261006_195226_265102.jpg
```

---

## Database schema

Table `detections`:

| Column       | Type    | Meaning                                                    |
|--------------|---------|------------------------------------------------------------|
| `id`         | INTEGER | Auto-increment primary key                                 |
| `plate`      | TEXT    | Recognised plate text                                      |
| `confidence` | REAL    | Best OCR confidence among the voting reads                 |
| `valid`      | INTEGER | `1` if it matches a known plate format, otherwise `0`      |
| `timestamp`  | TEXT    | ISO 8601 local time of saving                              |
| `source`     | TEXT    | Camera index, file path or URL                             |
| `image_path` | TEXT    | Path to the saved crop of the best read                    |

An index on `plate` keeps lookups fast.

---

## Querying and exporting data

Besides `--search` and `--export`, the database is plain SQLite and can be opened with any tool:

```bash
sqlite3 plates.db "SELECT plate, COUNT(*) FROM detections GROUP BY plate ORDER BY 2 DESC LIMIT 10;"
```

```python
import sqlite3
con = sqlite3.connect("plates.db")
for row in con.execute("SELECT plate, timestamp FROM detections WHERE valid = 1"):
    print(row)
```

---

## Improving accuracy

1. **Camera placement**: mount it so the plate is roughly 150 to 300 pixels wide in the frame, with minimal angle.
2. **Lighting and shutter**: a fast shutter speed (1/500 s or quicker) avoids motion blur; IR or supplementary lighting helps at night.
3. **Use YOLO**: train a plate detector (for example a small YOLOv8 model on a public licence-plate dataset) and pass it with `--weights`. This is the largest single improvement over the contour method.
4. **Tune thresholds**: raise `--min-conf` and `--min-hits` to reduce false saves; lower them if real plates are being missed.
5. **Process more frames**: lower `--every` (for example `1` or `2`) for fast traffic, if your hardware keeps up.
6. **Review `valid = 0` rows** if you run with `--no-validate`; they are likely misreads or non-standard plates.

---

## Running on Raspberry Pi / embedded boards

- Use a Pi 4 or 5 (4 GB or more) with a USB or CSI camera.
- Raise `--every` to 5 or more and lower the camera resolution to keep the frame rate usable.
- EasyOCR (PyTorch) is heavy on small boards. If it is too slow, a lighter OCR such as Tesseract, or an exported lightweight detector, can be swapped into the `PlateReader` class, since the rest of the pipeline only needs a `read(crop) -> (text, confidence)` method.
- For a deployed unit, run it as a `systemd` service so it restarts on boot and after crashes.

---

## Troubleshooting

| Problem                                   | Likely cause and fix                                                                 |
|-------------------------------------------|--------------------------------------------------------------------------------------|
| `Cannot open source`                      | Wrong camera index, path or URL; try `--source 1`, or check camera permissions       |
| No plates are saved                       | Plate too small or blurry, or validation too strict; try `--show`, `--no-validate`, a lower `--min-conf` |
| Many wrong plates                         | Raise `--min-conf` and `--min-hits`; use a YOLO model                                |
| Same car saved repeatedly                 | Increase `--cooldown`                                                                |
| Very slow                                 | Raise `--every`, use `--gpu`, or reduce the camera resolution                        |
| `ModuleNotFoundError: ultralytics`        | Only needed with `--weights`; run `pip install ultralytics`                          |
| `--show` crashes on a server              | No display available; run without `--show`, or install `opencv-python` (not headless) on a desktop |
| First run is slow to start                | EasyOCR is downloading its model; this happens once                                  |

---

## Limitations

- The default contour detector is a simple classical method; it struggles with strong angles, glare, dirty or damaged plates and cluttered backgrounds.
- Validation and position correction are written for Indian plate formats. For other countries, edit `STATE_RE`, `BH_RE` and `fix_positions` in `anpr.py`, or use `--no-validate`.
- Stylised, non-Latin or heavily decorated plates are not supported by the English OCR model.
- Accuracy figures depend entirely on your camera, lighting and traffic speed; test on your own footage before relying on it.

---

## Privacy and legal notes

Number plates are personal data in many jurisdictions. Before deploying this on a public road or a private premises, check the local laws on surveillance and data retention, secure the database and image folder, and delete records you no longer need. Use it only for lawful purposes such as parking, gate access or traffic studies you are authorised to run.
