# Stroke Prediction — MLOps Project

An end-to-end MLOps pipeline for predicting stroke risk. The project covers data preprocessing, model training with experiment tracking (MLflow), evaluation, model promotion, a FastAPI serving layer, data drift monitoring (Evidently AI), and a batch inference pipeline.

---

## Project Structure

```
someml/
├── data/
│   ├── raw/                    # Raw dataset (stroke-data.csv)
│   ├── processed/              # Generated: train/test/reference splits
│   └── incoming/               # New data for batch inference / drift check
├── models/
│   └── champion/               # Generated: exported champion model
├── output/                     # Generated: scored Excel reports
├── reports/                    # Generated: Evidently drift HTML reports
├── mlruns/                     # Generated: MLflow experiment tracking
├── src/
│   ├── config.py               # Central project configuration
│   ├── training/
│   │   ├── preprocess.py       # Data cleaning and train/test split
│   │   ├── train.py            # Model training with MLflow tracking
│   │   ├── evaluate.py         # Model evaluation and champion promotion
│   │   └── export_model.py     # Export champion model to disk
│   ├── serving/
│   │   └── app.py              # FastAPI prediction server
│   ├── inference/
│   │   └── pipeline.py         # Batch inference with drift check
│   └── monitoring/
│       ├── drift_check.py      # Evidently data drift detection
│       └── generate_drifted_data.py  # Demo: generate drifted dataset
├── ui/
│   ├── index.html              # Web UI
│   ├── app.js
│   └── styles.css
└── requirements.txt
```

---

## Prerequisites

- Python 3.9+
- The raw dataset at `data/raw/stroke-data.csv` (available on [Kaggle](https://www.kaggle.com/datasets/fedesoriano/stroke-prediction-dataset))

---

## Setup

**1. Create and activate a virtual environment**

```bash
python -m venv .venv
source .venv/bin/activate        # Linux / macOS
# .venv\Scripts\activate         # Windows
```

**2. Install dependencies**

```bash
pip install -r requirements.txt
```

---

## Running the Project

All commands are run from the **project root directory**.

### Step 1 — Preprocess Data

Cleans the raw dataset and creates train / test / reference splits.

```bash
python src/training/preprocess.py
```

Outputs: `data/processed/train.csv`, `data/processed/test.csv`, `data/processed/reference.csv`

---

### Step 2 — Train Models

Trains Logistic Regression, Random Forest, and Gradient Boosting variants with 5-fold cross-validation. All runs are tracked in MLflow and the best model is registered in the MLflow Model Registry.

```bash
python src/training/train.py
```

---

### Step 3 — Evaluate & Promote

Evaluates the latest registered model on the held-out test set and promotes it to the `champion` alias if F1 ≥ threshold (default 0.15).

```bash
python src/training/evaluate.py
```

---

### Step 4 — Export Champion Model

Exports the promoted champion model to `models/champion/` for use by the API and inference pipeline.

```bash
python src/training/export_model.py
```

---

### Step 5 — Start the API Server

Launches the FastAPI prediction server at `http://localhost:8000`. The web UI is served at `http://localhost:8000/ui/`.

```bash
python src/serving/app.py
```

Or with uvicorn directly:

```bash
uvicorn src.serving.app:app --host 0.0.0.0 --port 8000 --reload
```

**Available endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Web UI |
| GET | `/health` | Health check |
| GET | `/model-info` | Model metadata and training metrics |
| POST | `/predict` | Single patient prediction (JSON body) |
| POST | `/predict/batch` | Batch prediction from CSV upload |

API docs available at `http://localhost:8000/docs`

---

### Step 6 — Data Drift Monitoring

**Check drift on incoming data:**

```bash
python src/monitoring/drift_check.py --incoming data/incoming/new_patients.csv
```

Outputs an HTML drift report to `reports/drift_report_<timestamp>.html`.

**Generate a synthetic drifted dataset (for demo / testing):**

```bash
python src/monitoring/generate_drifted_data.py
```

Outputs `data/incoming/drifted_patients.csv`. Run the drift check against it to see a FAIL result:

```bash
python src/monitoring/drift_check.py --incoming data/incoming/drifted_patients.csv
```

---

### Step 7 — Batch Inference Pipeline

Runs the full inference flow: load CSV → drift check → score → write Excel report.

```bash
python src/inference/pipeline.py --incoming data/incoming/new_patients.csv
```

To skip the drift check and score anyway:

```bash
python src/inference/pipeline.py --incoming data/incoming/new_patients.csv --skip-drift
```

Scored results are saved to `output/scored_<timestamp>.xlsx`.

---

## View MLflow Experiments

```bash
mlflow ui --backend-store-uri file:./mlruns
```

Opens at `http://localhost:5000` — browse runs, compare metrics, and inspect registered models.

---

## Full Pipeline (Quick Reference)

```bash
# 1. Install
pip install -r requirements.txt

# 2. Preprocess
python src/training/preprocess.py

# 3. Train
python src/training/train.py

# 4. Evaluate & promote
python src/training/evaluate.py

# 5. Export champion model
python src/training/export_model.py

# 6. Start API
python src/serving/app.py

# 7. (Optional) Check drift
python src/monitoring/drift_check.py --incoming data/incoming/new_patients.csv

# 8. (Optional) Batch inference
python src/inference/pipeline.py --incoming data/incoming/new_patients.csv
```

---

## Key Configuration

All paths, feature definitions, and hyperparameters are centralized in `src/config.py`. Notable settings:

| Setting | Default | Description |
|---------|---------|-------------|
| `TEST_SIZE` | `0.2` | Train/test split ratio |
| `CV_FOLDS` | `5` | Cross-validation folds |
| `MODEL_PROMOTION_THRESHOLD` | `0.15` | Minimum F1 to promote model to champion |
| `API_PORT` | `8000` | FastAPI server port |
| `MLFLOW_EXPERIMENT_NAME` | `stroke-prediction` | MLflow experiment name |

Secrets (if any) go in a `.env` file at the project root — this file is gitignored.
