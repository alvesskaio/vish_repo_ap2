# Development log

This document is built **only** from what exists in this repository: the single git commit and the markdown and code cells of the notebook. Where the repository has no information, there is an explicit `TODO` instead of a guess.

Repository history: one commit (`b27d2c1`, "Add new notebook for YOLOv26 robust training on Vish dataset"). There is no earlier history for the application or for the v1 notebook.

## 1. Stages

### 1.1 Application (capture, classification, ETL, dashboard)

Flow described by the group: photo and coordinates are persisted; GPT-4.1 mini validates the image, assigns a category and generates an explanation; an ETL loads validated records into a dimensional database; a dashboard consumes it.

TODO: dates, schema of the dimensional model, prompt versions, tech stack, difficulties. None of this is in the repository.

### 1.2 Detection baseline v1

The notebook calls v1 a "proof of concept" and says v2 replaces it. The only facts recorded about it are:

- it used `yolov8n.pt` (the notebook compares the v2 choice of `yolo26s.pt` against it);
- it had the problems listed in 2.1.

TODO: v1 data sources, size, results, and the notebook/code itself (not versioned).

### 1.3 Dataset v2 and YOLO26 baseline

Goal stated in the notebook: not only a high mAP, but a reproducible experiment that generalizes to real phone photos in Brazilian streets. Stated protocol:

1. Resumable download with progress bar and Google Drive cache.
2. Conversion of VOC/COCO/YOLO/masks into one YOLO detection format.
3. Per-image provenance manifest.
4. Exact and perceptual-hash deduplication.
5. Split by source/sequential group, avoiding neighboring frames in different splits.
6. Label, coverage and leakage audit.
7. EDA and visual inspection before training.
8. Transfer learning with YOLO26 at 960 px.
9. Precision, recall, mAP50 and mAP50-95 per class.
10. Dedicated test on hard negatives and Brazilian images.
11. Photo upload and inference with per-class thresholds.

### 1.4 v2.1 correction

The notebook records: the COCO importer now resolves the full relative image path (including TACO's `batch_*` folders), and deduplication protects files still referenced by the manifest. This fixes orphan labels caused by repeated file names across batches. The deduplication code comments describe the orphan label/image as a bug "observed in TACO".

TODO: how the bug was found, how many files were affected, date.

## 2. Decisions and difficulties

### 2.1 From v1 to v2

Problems the v2 explicitly targets (notebook, section 1):

- undetected potholes;
- people, cars, shadows and manhole covers flagged as problems (false positives);
- classes that were too broad, such as "electrical damage" mixing pole and wire;
- near-identical images in train and validation;
- classification datasets used as if they had bounding boxes;
- training classes without a minimum number of examples.

Hypothesis: dataset quality and evaluation protocol matter more than simply adding epochs.

### 2.2 Design decisions recorded in the notebook

- **Five classes** (`pothole`, `garbage`, `dangerous_wire`, `damaged_road`, `flooding`) with operational definitions and explicit exclusions. New classes only after passing the same coverage and quality gate.
- **RDD2022 mapping:** D40 becomes `pothole`; D00/D10/D20 become `damaged_road`.
- **pLitterStreet replaces TACO** as the main garbage source (more than 13k images from vehicle-mounted cameras); TACO stays as a complement.
- **QR4Change is used only for negatives**: its positives are image-level labels, and without boxes they cannot honestly become detection labels.
- **DNIT masks to boxes** by connected components, with a minimum area (`min_component_area=80`) to filter noise.
- **`dangerous_wire` requires own reviewed data** and fails the `serious` profile before training if coverage is insufficient; generic labels (`wire`, `cable`, `pole`) are rejected by default.
- **Coverage gate** minimum train boxes per class: pothole 3000, garbage 5000, dangerous_wire 500, damaged_road 3000, flooding 1000. Target of 20-40% negatives among training images.
- **Two profiles**: `smoke` (validates the pipeline) and `serious` (full experiment).
- **Model by GPU**: `yolo26m.pt` if GPU memory is at least 20 GB, otherwise `yolo26s.pt`.
- **Training settings**: seed 42, `deterministic=True`, cosine LR, mosaic 0.8 with `close_mosaic=10`, patience 20. The best checkpoint is loaded explicitly, not the last one.
- **Per-class thresholds** (initial values) instead of a single `conf=0.25`, with a higher threshold for `dangerous_wire`.
- **Brazilian test** is meant to stay isolated and be run once, after decisions are frozen.

### 2.3 Difficulties visible in the saved outputs

- pLitterStreet: 0 images imported and 500 unresolved/ambiguous in the saved run (example names are in the cell output). TODO: root cause and whether it was fixed after this run.
- Coverage gate warned of insufficient coverage and negatives were 0.0% of training (smoke profile).
- The validation metrics cell ends in `ValueError: All arrays must be of the same length` because the per-class table assumes five classes while the run only had data for `garbage`.
- Licenses: sources are not equivalent (RDD2022 is ShareAlike, TACO has per-image licenses, mirrors on Kaggle/Roboflow may have incomplete provenance). pLitterStreet's license is marked "check the record" in the notebook.

### 2.4 Results

Only a smoke run is recorded (10 epochs, 640 px, `yolo26s`, Tesla T4, 500 TACO images, 58 validation images). For `garbage`: precision 0.681, recall 0.400, mAP50 0.461, mAP50-95 0.323. This is not a result of the `serious` profile and says nothing about the other four classes.

TODO: results of the `serious` profile, hard-negative false-positive rate, and Brazilian test (none recorded). TODO: origin of the ~99% figure in the article.

## 3. Open items

- Source and history of the application code.
- v1 notebook and its results.
- Dated timeline of the project and who did what. TODO: authorship per component (git shows a single author).
