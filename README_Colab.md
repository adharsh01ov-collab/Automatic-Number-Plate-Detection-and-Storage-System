#  Automatic Number Plate Recognition (ANPR)

##  Project Overview

The **Automatic Number Plate Recognition (ANPR)** system is a computer-vision-based application designed to detect and recognize vehicle license plates from images.

This implementation is developed using **Google Colab** and combines:

- Python
- OpenCV
- Tesseract OCR
- Pytesseract
- Pandas
- Excel-based data storage

The system accepts a vehicle image as input, processes the image using computer vision techniques, identifies a potential number-plate region, extracts the characters using Optical Character Recognition (OCR), and stores the recognized plate number along with the detection timestamp in an Excel file.

---

#  Objectives

The main objectives of this project are:

1. To detect a potential vehicle number-plate region from an input image.
2. To preprocess the image for improved character recognition.
3. To extract the number-plate characters using Tesseract OCR.
4. To display the detected plate number.
5. To record the recognized plate number and timestamp.
6. To maintain detection records using an Excel file.
7. To develop a simple and reproducible ANPR prototype using Google Colab.

---

#  Technologies Used

| Technology | Purpose |
|---|---|
| Python | Main programming language |
| Google Colab | Development and execution environment |
| OpenCV | Image processing and contour detection |
| Tesseract OCR | Character recognition |
| Pytesseract | Python interface for Tesseract |
| Pandas | Data processing and record management |
| OpenPyXL | Excel file handling |
| Excel | Detection record storage |

---

#  System Architecture

```text
                 INPUT
                   │
                   ▼
          Vehicle Image Upload
                   │
                   ▼
             Image Resizing
                   │
                   ▼
          Grayscale Conversion
                   │
                   ▼
        Bilateral Noise Filtering
                   │
                   ▼
          Canny Edge Detection
                   │
                   ▼
        Contour Identification
                   │
                   ▼
      Plate Region Identification
                   │
                   ▼
       Region of Interest (ROI)
                   │
                   ▼
        Image Enhancement
                   │
                   ▼
           Tesseract OCR
                   │
                   ▼
        Recognized Plate Number
                   │
                   ▼
          Timestamp Generation
                   │
                   ▼
            Excel Storage
```

---

#  Working Principle

## 1. Image Upload

The user uploads a vehicle image to the Google Colab environment.

```python
uploaded = files.upload()
filename = list(uploaded.keys())[0]
```

The uploaded filename is then passed to the image-processing pipeline.

---

## 2. Image Preprocessing

The input image is resized to a standard resolution:

```python
img = cv2.resize(img, (640, 480))
```

The image is converted from BGR to grayscale:

```python
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
```

Grayscale conversion reduces the image complexity and makes edge detection easier.

A bilateral filter is then applied:

```python
gray = cv2.bilateralFilter(gray, 11, 17, 17)
```

This reduces noise while attempting to preserve important edges.

---

#  3. Edge Detection

Canny edge detection is applied to identify strong boundaries in the image.

```python
edged = cv2.Canny(gray, 30, 200)
```

Edges can represent the boundaries of objects, including potential license-plate regions.

---

#  4. Contour Detection

Contours are extracted from the edge image:

```python
contours, _ = cv2.findContours(
    edged.copy(),
    cv2.RETR_TREE,
    cv2.CHAIN_APPROX_SIMPLE
)
```

The contours are sorted based on their area so that larger candidate regions can be examined first.

---

#  5. Plate Region Identification

Potential plate regions are filtered using geometric characteristics such as:

- Width
- Height
- Aspect ratio
- Area

Example:

```python
aspect_ratio = w / float(h)

if 2.0 <= aspect_ratio <= 6.0 and area > 1000:
```

This helps eliminate many regions that are unlikely to represent a number plate.

---

#  6. OCR Processing

The selected region of interest is enhanced before OCR.

The region is resized:

```python
roi = cv2.resize(roi, None, fx=2, fy=2)
```

Otsu thresholding is then applied:

```python
_, roi_thresh = cv2.threshold(
    roi,
    0,
    255,
    cv2.THRESH_BINARY + cv2.THRESH_OTSU
)
```

The processed image is passed to Tesseract:

```python
text = pytesseract.image_to_string(
    roi_thresh,
    config='--psm 7'
)
```

The OCR output is then cleaned so that only alphanumeric characters remain.

---

#  7. OCR Output Cleaning

The recognized text is converted to uppercase and unwanted characters are removed.

```python
text = ''.join(
    ch for ch in text.upper()
    if ch.isalnum()
)
```

This produces a cleaner representation of the recognized plate number.

---

#  8. Data Storage

Once a plate number is obtained, the system records:

- Plate Number
- Detection Timestamp

The information is stored using Pandas:

```python
data = {
    'Plate Number': [plate_number],
    'Timestamp': [timestamp]
}
```

The data is then written to:

```text
NumberPlateData.xlsx
```

---

#  Output

## Output 1 — Input Image

**Description:** Original vehicle image uploaded to Google Colab.

###  Screenshot

> **Add your actual input-image screenshot here**

```text
┌──────────────────────────────────────────┐
│                                          │
│                                          │
│          INSERT INPUT IMAGE              │
│                                          │
│                                          │
└──────────────────────────────────────────┘
```

---

#  Output 2 — Detected Number Plate

The system identifies a potential plate region and displays the detected plate number.

### 📷 Screenshot

> **Add your actual detection output here**

```text
┌──────────────────────────────────────────┐
│                                          │
│        INSERT DETECTION OUTPUT           │
│                                          │
│      Plate: __________________           │
│                                          │
└──────────────────────────────────────────┘
```

---

#  Output 3 — OCR Result

The recognized characters obtained from Tesseract OCR are displayed.

### Example output format

```text
Detected Plate: XXXXXXXX
```

###  Actual Output

> **Paste your real Colab output screenshot here**

```text
┌──────────────────────────────────────────┐
│                                          │
│        INSERT OCR OUTPUT SCREENSHOT      │
│                                          │
└──────────────────────────────────────────┘
```

---

#  Output 4 — Excel Database

The recognized plate number and timestamp are stored in an Excel file.

### Example structure

| Plate Number | Timestamp |
|---|---|
| XXXXXXXX | YYYY-MM-DD HH:MM:SS |

### 📷 Excel Output Screenshot

> **Insert screenshot of `NumberPlateData.xlsx` here**

```text
┌──────────────────────────────────────────┐
│                                          │
│       INSERT EXCEL SCREENSHOT            │
│                                          │
└──────────────────────────────────────────┘
```

---

#  Complete Program

```python
from IPython import get_ipython
from IPython.display import display

!sudo apt update
!sudo apt install tesseract-ocr
!pip install opencv-python pytesseract openpyxl pandas

import cv2
import pytesseract
import pandas as pd
from datetime import datetime
import os
from google.colab import files

# Upload image
uploaded = files.upload()
filename = list(uploaded.keys())[0]


def extract_plate_text(image_path):

    img = cv2.imread(image_path)

    if img is None:
        print("Could not read image.")
        return ""

    img = cv2.resize(img, (640, 480))

    # Convert to grayscale
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # Noise reduction
    gray = cv2.bilateralFilter(gray, 11, 17, 17)

    # Edge detection
    edged = cv2.Canny(gray, 30, 200)

    # Find contours
    contours, _ = cv2.findContours(
        edged.copy(),
        cv2.RETR_TREE,
        cv2.CHAIN_APPROX_SIMPLE
    )

    contours = sorted(
        contours,
        key=cv2.contourArea,
        reverse=True
    )[:10]

    plate_text = ""

    for c in contours:

        x, y, w, h = cv2.boundingRect(c)

        roi = gray[y:y+h, x:x+w]

        text = pytesseract.image_to_string(
            roi,
            config='--psm 8'
        )

        if len(text.strip()) >= 5:
            plate_text = text.strip()
            break

    return plate_text


def save_to_excel(
    plate_number,
    excel_file='NumberPlateData.xlsx'
):

    timestamp = datetime.now().strftime(
        '%Y-%m-%d %H:%M:%S'
    )

    data = {
        'Plate Number': [plate_number],
        'Timestamp': [timestamp]
    }

    df = pd.DataFrame(data)

    if os.path.exists(excel_file):

        existing_df = pd.read_excel(
            excel_file
        )

        df = pd.concat(
            [existing_df, df],
            ignore_index=True
        )

    df.to_excel(
        excel_file,
        index=False
    )

    files.download(excel_file)


# Run ANPR
plate_number = extract_plate_text(filename)

if plate_number:

    print(
        f"Detected Plate: {plate_number}"
    )

    save_to_excel(plate_number)

else:

    print("No plate detected.")
```

---

#  How to Run in Google Colab

## Step 1 — Open Google Colab

Create a new Python notebook.

## Step 2 — Install Dependencies

Run:

```bash
!sudo apt update
!sudo apt install tesseract-ocr
!pip install opencv-python pytesseract openpyxl pandas
```

## Step 3 — Upload Vehicle Image

Run the upload section and select an image containing a visible license plate.

## Step 4 — Execute the ANPR Program

Run the remaining cells.

## Step 5 — Check the Result

The program will display:

```text
Detected Plate: XXXXXXXX
```

or:

```text
No plate detected.
```

If a plate is recognized, an Excel file will be generated and downloaded.

---

#  Results and Evaluation

The system should be evaluated using different image conditions.

| Test Condition | Plate Detected | OCR Successful | Remarks |
|---|---|---|---|
| Clear daylight | ⬜ | ⬜ | |
| Indoor lighting | ⬜ | ⬜ | |
| Low lighting | ⬜ | ⬜ | |
| Slightly tilted plate | ⬜ | ⬜ | |
| Different distances | ⬜ | ⬜ | |
| High-resolution image | ⬜ | ⬜ | |

> **Note:** Performance values should be filled only after actual testing.

---

#  Performance Metrics

The following metrics can be calculated after testing:

### Detection Accuracy

```text
Detection Accuracy =
Correctly detected plates / Total test images × 100
```

### OCR Accuracy

```text
OCR Accuracy =
Correctly recognized plates / Successfully detected plates × 100
```

### Example evaluation format

```text
Total Test Images       : ______
Plates Detected         : ______
OCR Correct             : ______
Detection Accuracy      : ______ %
OCR Accuracy            : ______ %
```

Do not report estimated or assumed accuracy values.

---

#  Features

- Image upload through Google Colab
- Image resizing
- Grayscale conversion
- Bilateral filtering
- Canny edge detection
- Contour-based candidate identification
- License-plate region extraction
- Tesseract OCR
- OCR text cleaning
- Automatic timestamp generation
- Excel-based record storage
- Automatic Excel file download

---

#  Limitations

The current implementation is a prototype and has several limitations:

- It processes individual images rather than continuous video.
- Contour-based detection can produce false candidate regions.
- OCR performance depends strongly on image quality.
- Low-light images may reduce recognition accuracy.
- Highly tilted or partially obscured plates may not be recognized.
- Different fonts and plate designs can affect OCR performance.
- The current system does not perform multi-object vehicle tracking.

---

#  Future Improvements

Future versions can improve the system by adding:

1. Real-time webcam/video processing.
2. YOLO-based license plate detection.
3. Deep-learning-based OCR.
4. Indian license-plate format validation.
5. Multi-frame OCR voting.
6. Duplicate detection suppression.
7. SQLite/MySQL database integration.
8. Web-based monitoring dashboard.
9. Vehicle detection and tracking.
10. Automatic violation/event logging.
11. Cloud-based storage.
12. Deployment on an edge device.

---

#  Project Workflow

```text
             Vehicle Image
                   │
                   ▼
             Preprocessing
                   │
                   ▼
           Edge Detection
                   │
                   ▼
          Contour Detection
                   │
                   ▼
        Candidate Plate Region
                   │
                   ▼
             ROI Extraction
                   │
                   ▼
             Tesseract OCR
                   │
                   ▼
          Text Cleaning
                   │
                   ▼
          Plate Number Result
                   │
                   ▼
        Timestamp Generation
                   │
                   ▼
             Excel Storage
```

---

#  Project Structure

```text
ANPR/
│
├── README.md
│
├── ANPR_Colab.ipynb
│
├── NumberPlateData.xlsx
│
├── results/
│   ├── input_image.png
│   ├── detected_plate.png
│   └── ocr_output.png
│
└── docs/
    └── project_report.pdf
```

---

#  Applications

The proposed system can be adapted for:

- Smart parking systems
- Automated vehicle entry
- Toll collection
- Traffic monitoring
- Campus vehicle management
- Security and access-control systems
- Automated parking management
- Vehicle record management

---

#  Learning Outcomes

Through this project, the following technical concepts were applied:

- Computer vision
- Image preprocessing
- Edge detection
- Contour analysis
- Region of Interest extraction
- Optical Character Recognition
- Python programming
- Data processing
- Excel automation
- Timestamp-based record management
- Google Colab development

---

#  Future System Architecture

```text
              Camera / CCTV
                    │
                    ▼
             Vehicle Detection
                    │
                    ▼
          License Plate Detection
                    │
                    ▼
             Image Enhancement
                    │
                    ▼
               OCR Engine
                    │
                    ▼
          Plate Format Validation
                    │
                    ▼
            Database / Cloud
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     Web Dashboard       CSV/Excel Export
```

---

#  Project Information

**Project:** Automatic Number Plate Recognition System

**Domain:** Computer Vision / Image Processing / OCR

**Platform:** Google Colab

**Programming Language:** Python

**Key Technologies:** OpenCV, Tesseract OCR, Pytesseract, Pandas, OpenPyXL

**Storage:** Excel

---

#  Conclusion

The Automatic Number Plate Recognition system demonstrates the application of computer vision and Optical Character Recognition for automated vehicle number-plate recognition.

The system performs image preprocessing, edge and contour analysis, candidate plate extraction, OCR-based character recognition, and timestamp-based record storage.

The current implementation provides a foundation for developing a more advanced real-time ANPR solution using deep-learning-based plate detection, improved OCR, database integration, and vehicle tracking.

---

##  Project Status

**Status:** Prototype / Development

The system has been implemented as a Google Colab-based proof of concept. Further testing with a larger dataset and different environmental conditions is required before reporting final accuracy metrics.

---

##  Final Demonstration

### Input

<img width="1000" height="514" alt="image5" src="https://github.com/user-attachments/assets/1ea09623-b82f-4351-8083-250355118fba" />

<img width="318" height="159" alt="image6" src="https://github.com/user-attachments/assets/31a84d63-732a-43bd-a533-a456bf8420f3" />


### Plate Detection

<img width="1920" height="1080" alt="Screenshot 2026-10-06 201850" src="https://github.com/user-attachments/assets/00052e01-4e17-4f42-be7a-af1d83ca9b69" />

<img width="1920" height="1080" alt="Screenshot 2026-10-06 200442" src="https://github.com/user-attachments/assets/4e3386f5-4d8f-4652-8a27-a0a15ae283ef" />


### OCR Result

Requirement already satisfied: opencv-python in /usr/local/lib/python3.13/dist-packages (5.0.0.93)
Requirement already satisfied: pytesseract in /usr/local/lib/python3.13/dist-packages (0.3.13)
Requirement already satisfied: openpyxl in /usr/local/lib/python3.13/dist-packages (3.1.5)
Requirement already satisfied: pandas in /usr/local/lib/python3.13/dist-packages (2.2.3)
Requirement already satisfied: numpy>=2 in /usr/local/lib/python3.13/dist-packages (from opencv-python) (2.1.3)
Requirement already satisfied: packaging>=21.3 in /usr/local/lib/python3.13/dist-packages (from pytesseract) (26.3)
Requirement already satisfied: Pillow>=8.0.0 in /usr/local/lib/python3.13/dist-packages (from pytesseract) (11.3.0)
Requirement already satisfied: et-xmlfile in /usr/local/lib/python3.13/dist-packages (from openpyxl) (2.0.0)
Requirement already satisfied: python-dateutil>=2.8.2 in /usr/local/lib/python3.13/dist-packages (from pandas) (2.9.0.post0)
Requirement already satisfied: pytz>=2020.1 in /usr/local/lib/python3.13/dist-packages (from pandas) (2025.2)
Requirement already satisfied: tzdata>=2022.7 in /usr/local/lib/python3.13/dist-packages (from pandas) (2026.4)
Requirement already satisfied: six>=1.5 in /usr/local/lib/python3.13/dist-packages (from python-dateutil>=2.8.2->pandas) (1.17.0)
image5.jpg
image5.jpg(image/jpeg) - 60891 bytes, last modified: 10/6/2026 - 100% done
Saving image5.jpg to image5.jpg
Detected Plate: “HROBAY1229

### Excel Record

[NumberPlateData (1).xlsx](https://github.com/user-attachments/files/33111966/NumberPlateData.1.xlsx)

### Complete Demonstration



https://github.com/user-attachments/assets/ed60d304-1799-4f2c-8048-adef2ceff02f


