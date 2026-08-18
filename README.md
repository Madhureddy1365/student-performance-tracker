# Student Performance Tracker

A full-stack web application that tracks student academic performance and uses a
trained Machine Learning model to predict each student's final score from their
academic and attendance data — built as an ML internship project for **CODTECH**.

The app has three parts, all included in this repo:

- **Frontend** — React.js dashboard (search/filter, charts, forms)
- **Backend** — Python Flask REST API + SQLite database
- **Machine Learning** — Pandas / NumPy / Scikit-learn pipeline that trains and
  compares Linear Regression, Decision Tree, and Random Forest models and
  serves the best one through the API

---

## 1. Features

| Area | What it does |
|---|---|
| **Dashboard** | Total students, average predicted score, average attendance, count of high performers vs. students needing improvement, performance distribution charts |
| **Add Student** | Form for the 8 academic/attendance inputs; on submit, calls the ML API and shows the predicted score, category, and recommendations immediately |
| **Prediction** | Random Forest / Linear Regression / Decision Tree comparison; best model auto-selected and used for every prediction; classifies students as Excellent / Good / Average / Needs Improvement |
| **Student Records** | Searchable, filterable table of all students with View / Edit / Delete |
| **Analytics** | Attendance vs. Marks, Study Hours vs. Marks, Assignment Score vs. Marks, performance distribution, and model comparison (MAE/MSE/RMSE/R²) charts |
| **Recommendations** | Rule-based tips generated from each student's weakest inputs (attendance, study hours, assignments, exam prep, etc.) |

---

## 2. Tech Stack

- **Frontend:** React 18, React Router 6, Recharts, Axios
- **Backend:** Python 3, Flask
- **Machine Learning:** Pandas, NumPy, Scikit-learn, Joblib
- **Database:** SQLite (file-based, zero setup)

---

## 3. Project Structure

```
student-performance-tracker/
├── README.md
├── .gitignore
├── backend/
│   ├── app.py                 # Flask API (all endpoints)
│   ├── database.py            # SQLite schema + CRUD helpers
│   ├── requirements.txt
│   ├── student_tracker.db     # created automatically on first run
│   ├── data/
│   │   └── student_data.csv   # sample dataset (auto-generated if missing)
│   └── ml/
│       ├── train_model.py     # training / evaluation / model-selection script
│       ├── predict.py         # loads saved model, runs predictions
│       └── models/
│           ├── best_model.pkl
│           ├── scaler.pkl
│           └── metrics.json   # MAE/MSE/RMSE/R² for all 3 models
└── frontend/
    ├── package.json
    ├── .env.example
    ├── public/
    │   └── index.html
    └── src/
        ├── index.js
        ├── App.jsx
        ├── api/api.js
        ├── components/        # Sidebar, StatCard, Loader, Toast, etc.
        ├── pages/              # Dashboard, AddStudent, StudentRecords, StudentDetail, Analytics
        └── styles/App.css
```

---

## 4. How the ML pipeline works

`backend/ml/train_model.py`:

1. Loads `backend/data/student_data.csv`. **If it doesn't exist**, it generates a
   realistic 400-row synthetic dataset (with a few missing values and one
   categorical column) and saves it there.
2. **Preprocessing:** fills missing numeric values with the column median, and
   label-encodes any categorical columns.
3. **Splits** the data 80/20 into train/test sets and standardizes features
   with `StandardScaler`.
4. **Trains three regression models:** Linear Regression, Decision Tree
   Regressor, and Random Forest Regressor.
5. **Evaluates** each with MAE, MSE, RMSE, and R² on the held-out test set.
6. **Selects the best model** (highest R²) and saves it, the fitted scaler,
   and the comparison metrics to `backend/ml/models/`.

The Flask API loads these saved artifacts at startup and uses them for every
`/api/predict` and `/api/students` (POST/PUT) call — so predictions are
instant, with no retraining needed per request.

Re-run training any time (e.g. after replacing the dataset with real data):

```bash
cd backend
python ml/train_model.py
```

To use your **own dataset** instead of the generated one, just drop a CSV with
the same column names at `backend/data/student_data.csv` before running the
script (it will use your file since it already exists).

---

## 5. Setup & Run Locally

### Prerequisites
- Python 3.9+
- Node.js 16+ and npm

### 5.1 Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

# Train the ML model (generates the sample dataset the first time)
python ml/train_model.py

# Start the API (runs on http://localhost:5000)
python app.py
```

The SQLite database (`student_tracker.db`) and its `students` table are
created automatically the first time `app.py` runs.

### 5.2 Frontend

In a **second terminal**:

```bash
cd frontend
npm install
npm start
```

This opens the app at `http://localhost:3000`. The dev server proxies
`/api/*` requests to `http://localhost:5000` (configured via the `proxy`
field in `frontend/package.json`), so no extra configuration is needed.

> Deploying separately (e.g. frontend on Vercel, backend elsewhere)? Copy
> `frontend/.env.example` to `frontend/.env` and set
> `REACT_APP_API_BASE_URL` to your backend's URL before running `npm run build`.

### 5.3 Quick smoke test

With both servers running:

```bash
curl http://localhost:5000/api/health
```

should return `{"status": "ok", ...}`. Then open `http://localhost:3000`,
add a student, and confirm a predicted score and recommendations appear.

---

## 6. API Reference

Base URL: `http://localhost:5000`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/health` | Liveness check |
| GET | `/api/students?search=&category=` | List students (optional search / category filter) |
| POST | `/api/students` | Create a student — predicts and stores the score |
| GET | `/api/students/<id>` | Get one student + recommendations |
| PUT | `/api/students/<id>` | Update a student — re-predicts the score |
| DELETE | `/api/students/<id>` | Delete a student |
| POST | `/api/predict` | Predict without saving (what-if scoring) |
| GET | `/api/dashboard/stats` | Aggregated dashboard statistics |
| GET | `/api/analytics/data` | Per-student data points for charts |
| GET | `/api/model/metrics` | MAE / MSE / RMSE / R² for all trained models |

**Example — create a student:**

```bash
curl -X POST http://localhost:5000/api/students \
  -H "Content-Type: application/json" \
  -d '{
    "student_id": "STU1042",
    "name": "Priya Sharma",
    "attendance_percentage": 88,
    "study_hours": 5,
    "previous_marks": 78,
    "assignment_score": 82,
    "internal_exam_score": 75,
    "assignments_completed": 16,
    "participation_score": 7
  }'
```

Response includes the saved student record (with `predicted_score` and
`performance_category`), a list of `recommendations`, and the `model_used`.

---

## 7. Notes for the demo / write-up

- The comparison table in `backend/ml/models/metrics.json` (and the
  **Analytics → Model Performance Comparison** chart in the UI) shows exactly
  which of the three models was selected and why — useful for explaining the
  model-selection step during a project demo or viva.
- The sample dataset is synthetic but formula-driven with realistic noise and
  a couple of non-linear effects, so the three models produce genuinely
  different, explainable metrics rather than arbitrary numbers.
- Every "Add Student" and "Edit Student" submission re-runs the trained model
  live through the Flask API — this is a real prediction, not a mock value.
