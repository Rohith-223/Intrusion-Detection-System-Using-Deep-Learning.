# Intrusion-Detection-System-Using-Deep-Learning
# backend
#main

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routes import router

app = FastAPI(title="Deep Learning IDS API")

# Allow frontend (website) to call backend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(router)

@app.get("/")
def home():
    return {"message": "IDS Backend running ✅"}

#routes
from fastapi import APIRouter, UploadFile, File, HTTPException
import pandas as pd
from app.services.predict import predict_df

router = APIRouter()

@router.post("/predict")
async def predict(file: UploadFile = File(...)):
    if not file.filename.lower().endswith(".csv"):
        raise HTTPException(status_code=400, detail="Please upload a CSV file")

    try:
        df = pd.read_csv(file.file)
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid CSV format")

    result = predict_df(df)
    return result

# Predict 
import numpy as np
import pandas as pd
import joblib
from tensorflow import keras

MODEL_PATH = os.path.join("app", "model", "ids_multiclass_model.h5")
SCALER_PATH = os.path.join("app", "model", "ids_scaler_multiclass.pkl")
ENCODER_PATH = os.path.join("app", "model", "label_encoder.pkl")

model = keras.models.load_model(MODEL_PATH)
scaler = joblib.load(SCALER_PATH)
le = joblib.load(ENCODER_PATH)

def predict_df(df: pd.DataFrame):
    df = df.copy()
    df.columns = [c.strip() for c in df.columns]

    # drop columns not used
    drop_cols = [c for c in ["Flow ID", "Source IP", "Destination IP", "Timestamp", "Label"] if c in df.columns]
    if drop_cols:
        df = df.drop(columns=drop_cols)

    X_df = df.select_dtypes(include=[np.number]).replace([np.inf, -np.inf], np.nan).dropna()

    if X_df.shape[0] == 0:
        return {"status": "error", "message": "No valid numeric rows after cleaning NaN/Inf."}

    X = scaler.transform(X_df.values)
    probs = model.predict(X, verbose=0)  # shape: (N, num_classes)
    pred_idx = np.argmax(probs, axis=1)
    pred_labels = le.inverse_transform(pred_idx)
    confidences = probs.max(axis=1)

    # Summary counts
    unique, counts = np.unique(pred_labels, return_counts=True)
    summary = dict(zip(unique.tolist(), counts.tolist()))

    # Binary summary too (for quick view)
    normal_count = int((pred_labels == "BENIGN").sum())
    attack_count = int(len(pred_labels) - normal_count)

    # Top risky rows (not BENIGN)
    risk_rows = np.where(pred_labels != "BENIGN")[0]
    top = risk_rows[np.argsort(-confidences[risk_rows])][:10] if len(risk_rows) else []

    top_risky = [
        {
            "row_index": int(i),
            "type": str(pred_labels[i]),
            "confidence": float(confidences[i])
        }
        for i in top
    ]

    first20 = [
        {
            "row_index": int(i),
            "type": str(pred_labels[i]),
            "confidence": float(confidences[i])
        }
        for i in range(min(20, len(pred_labels)))
    ]

    return {
        "status": "ok",
        "total_rows_used": int(len(pred_labels)),
        "summary_binary": {"Normal(BENIGN)": normal_count, "Attack": attack_count},
        "summary_multiclass": summary,
        "top_10_risky_rows": top_risky,
        "first_20_predictions": first20
    }
# Frontend

# index

<!DOCTYPE html>
<html>
<head>
  <title>IDS Website</title>
</head>
<body>
  <h2>Deep Learning IDS - Demo Website</h2>
  <a href="dashboard.html">Go to Dashboard</a>
</body>
</html>


#dashboard

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <title>IDS Dashboard (Multiclass)</title>

  <!-- Simple styling -->
  <style>
    body { font-family: Arial, sans-serif; background: #f6f7fb; margin: 0; }
    header { background: #111827; color: #fff; padding: 16px 22px; }
    header h2 { margin: 0; font-size: 20px; }
    .container { padding: 18px 22px; max-width: 1100px; margin: 0 auto; }

    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
    @media (max-width: 900px) { .grid { grid-template-columns: 1fr; } }

    .card { background: #fff; border-radius: 10px; padding: 14px; box-shadow: 0 3px 12px rgba(0,0,0,0.06); }
    .card h3 { margin: 0 0 10px; font-size: 16px; color: #111827; }

    .row { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; }
    input[type=file] { padding: 8px; background: #fff; border-radius: 8px; border: 1px solid #ddd; }
    button { padding: 10px 14px; border: none; border-radius: 8px; background: #2563eb; color: #fff; cursor: pointer; }
    button:disabled { background: #93c5fd; cursor: not-allowed; }

    .pill { display: inline-block; padding: 6px 10px; border-radius: 999px; font-size: 12px; }
    .pill.ok { background: #dcfce7; color: #14532d; }
    .pill.warn { background: #fee2e2; color: #7f1d1d; }
    .muted { color: #6b7280; font-size: 13px; }

    .stats { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin-top: 10px; }
    @media (max-width: 900px) { .stats { grid-template-columns: repeat(2, 1fr); } }

    .stat { background: #f9fafb; border: 1px solid #eee; border-radius: 10px; padding: 10px; }
    .stat .label { color: #6b7280; font-size: 12px; }
    .stat .value { font-size: 18px; font-weight: 700; color: #111827; margin-top: 4px; }

    table { width: 100%; border-collapse: collapse; overflow: hidden; border-radius: 10px; }
    th, td { border-bottom: 1px solid #eee; padding: 10px; font-size: 13px; text-align: left; }
    th { background: #f9fafb; color: #111827; font-weight: 700; }
    tr:hover td { background: #fafafa; }

    .footer-note { margin-top: 10px; font-size: 12px; color: #6b7280; }
    .danger { color: #b91c1c; }
    .success { color: #166534; }

    .two-col { display: grid; grid-template-columns: 1.2fr 0.8fr; gap: 14px; }
    @media (max-width: 900px) { .two-col { grid-template-columns: 1fr; } }
  </style>

  <!-- Chart.js for bar chart -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>

<body>
  <header>
    <h2>Deep Learning IDS Dashboard — Multiclass Detection</h2>
  </header>

  <div class="container">
    <div class="card">
      <h3>Upload CICIDS CSV and Detect</h3>
      <div class="row">
        <input type="file" id="fileInput" accept=".csv" />
        <button id="detectBtn" onclick="uploadAndPredict()">Detect Intrusion</button>
        <span id="statusPill" class="pill ok">Ready</span>
      </div>
      <div class="muted" style="margin-top:8px;">
        Backend API: <code>http://127.0.0.1:8001/predict</code>
      </div>
    </div>

    <div class="stats">
      <div class="stat">
        <div class="label">Total Rows Used</div>
        <div class="value" id="totalRows">—</div>
      </div>
      <div class="stat">
        <div class="label">Normal (BENIGN)</div>
        <div class="value success" id="normalCount">—</div>
      </div>
      <div class="stat">
        <div class="label">Attack (Total)</div>
        <div class="value danger" id="attackCount">—</div>
      </div>
      <div class="stat">
        <div class="label">Distinct Attack Types</div>
        <div class="value" id="attackTypes">—</div>
      </div>
    </div>

    <div class="two-col" style="margin-top:14px;">
      <div class="card">
        <h3>Attack Type Distribution (Bar Chart)</h3>
        <canvas id="attackChart" height="130"></canvas>
        <div class="footer-note">Note: BENIGN is excluded from the chart for clarity.</div>
      </div>

      <div class="card">
        <h3>Top 10 Risky Rows</h3>
        <div class="muted">Highest confidence non-BENIGN predictions</div>
        <div style="margin-top:10px; max-height: 320px; overflow:auto;">
          <table id="riskyTable">
            <thead>
              <tr>
                <th>Row</th>
                <th>Type</th>
                <th>Confidence</th>
              </tr>
            </thead>
            <tbody>
              <tr><td colspan="3" class="muted">No results yet.</td></tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div class="grid" style="margin-top:14px;">
      <div class="card">
        <h3>Multiclass Summary (Counts)</h3>
        <div style="max-height: 340px; overflow:auto;">
          <table id="summaryTable">
            <thead>
              <tr>
                <th>Class</th>
                <th>Count</th>
              </tr>
            </thead>
            <tbody>
              <tr><td colspan="2" class="muted">No results yet.</td></tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="card">
        <h3>First 20 Predictions (Preview)</h3>
        <div class="muted">Useful for quick checking in demo</div>
        <div style="margin-top:10px; max-height: 340px; overflow:auto;">
          <table id="first20Table">
            <thead>
              <tr>
                <th>Row</th>
                <th>Type</th>
                <th>Confidence</th>
              </tr>
            </thead>
            <tbody>
              <tr><td colspan="3" class="muted">No results yet.</td></tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div class="card" style="margin-top:14px;">
      <h3>Raw JSON (for debugging)</h3>
      <pre id="rawJson" style="white-space: pre-wrap; margin:0; background:#0b1220; color:#d1d5db; padding:12px; border-radius:10px; overflow:auto;">—</pre>
    </div>
  </div>

  <script>
    let chartInstance = null;

    function setStatus(text, kind) {
      const pill = document.getElementById("statusPill");
      pill.textContent = text;
      pill.className = "pill " + (kind === "warn" ? "warn" : "ok");
    }

    function toFixed3(x) {
      if (x === null || x === undefined || Number.isNaN(x)) return "—";
      return (Math.round(x * 1000) / 1000).toFixed(3);
    }

    function renderTables(data) {
      // Summary multiclass
      const summaryBody = document.querySelector("#summaryTable tbody");
      summaryBody.innerHTML = "";

      const summary = data.summary_multiclass || {};
      const entries = Object.entries(summary).sort((a, b) => b[1] - a[1]);

      if (entries.length === 0) {
        summaryBody.innerHTML = `<tr><td colspan="2" class="muted">No summary available.</td></tr>`;
      } else {
        for (const [cls, count] of entries) {
          const tr = document.createElement("tr");
          tr.innerHTML = `<td>${cls}</td><td>${count}</td>`;
          summaryBody.appendChild(tr);
        }
      }

      // Top 10 risky rows
      const riskyBody = document.querySelector("#riskyTable tbody");
      riskyBody.innerHTML = "";

      const risky = data.top_10_risky_rows || [];
      if (risky.length === 0) {
        riskyBody.innerHTML = `<tr><td colspan="3" class="muted">No risky rows detected.</td></tr>`;
      } else {
        for (const r of risky) {
          const tr = document.createElement("tr");
          tr.innerHTML = `<td>${r.row_index}</td><td>${r.type}</td><td>${toFixed3(r.confidence)}</td>`;
          riskyBody.appendChild(tr);
        }
      }

      // First 20
      const f20Body = document.querySelector("#first20Table tbody");
      f20Body.innerHTML = "";

      const first20 = data.first_20_predictions || [];
      if (first20.length === 0) {
        f20Body.innerHTML = `<tr><td colspan="3" class="muted">No preview available.</td></tr>`;
      } else {
        for (const r of first20) {
          const tr = document.createElement("tr");
          tr.innerHTML = `<td>${r.row_index}</td><td>${r.type}</td><td>${toFixed3(r.confidence)}</td>`;
          f20Body.appendChild(tr);
        }
      }
    }

    function renderStats(data) {
      document.getElementById("totalRows").textContent = data.total_rows_used ?? "—";

      const bin = data.summary_binary || {};
      const normal = bin["Normal(BENIGN)"] ?? "—";
      const attack = bin["Attack"] ?? "—";

      document.getElementById("normalCount").textContent = normal;
      document.getElementById("attackCount").textContent = attack;

      // Count distinct attack types (exclude BENIGN)
      const summary = data.summary_multiclass || {};
      const attackTypes = Object.keys(summary).filter(k => k !== "BENIGN");
      document.getElementById("attackTypes").textContent = attackTypes.length;
    }

    function renderChart(data) {
      const summary = data.summary_multiclass || {};
      const labels = Object.keys(summary).filter(k => k !== "BENIGN");
      const counts = labels.map(l => summary[l]);

      const ctx = document.getElementById("attackChart").getContext("2d");

      if (chartInstance) chartInstance.destroy();

      chartInstance = new Chart(ctx, {
        type: "bar",
        data: {
          labels: labels,
          datasets: [{
            label: "Attack Count",
            data: counts
          }]
        },
        options: {
          responsive: true,
          plugins: {
            legend: { display: true }
          },
          scales: {
            x: { ticks: { autoSkip: false } }
          }
        }
      });
    }

    async function uploadAndPredict() {
      const file = document.getElementById("fileInput").files[0];
      if (!file) {
        alert("Please choose a CSV file");
        return;
      }

      const btn = document.getElementById("detectBtn");
      btn.disabled = true;
      setStatus("Processing...", "ok");
      document.getElementById("rawJson").textContent = "Processing...";

      const formData = new FormData();
      formData.append("file", file);

      try {
        const res = await fetch("http://127.0.0.1:8001/predict", {
          method: "POST",
          body: formData
        });

        const data = await res.json();
        document.getElementById("rawJson").textContent = JSON.stringify(data, null, 2);

        if (!res.ok || data.status !== "ok") {
          setStatus("Error", "warn");
          btn.disabled = false;
          return;
        }

        renderStats(data);
        renderTables(data);
        renderChart(data);

        // Status based on attack count
        const attackCount = data.summary_binary?.["Attack"] ?? 0;
        if (attackCount > 0) setStatus("Attack Detected", "warn");
        else setStatus("All Normal", "ok");

      } catch (err) {
        document.getElementById("rawJson").textContent =
          "Error connecting to backend. Make sure FastAPI is running.\n\n" + err;
        setStatus("Backend not reachable", "warn");
      } finally {
        btn.disabled = false;
      }
    }
  </script>
</body>
</html>


  
