# Lateral_Movement_Detection




## Demo 2

# Insider Threat App

A lightweight demo for insider‑threat detection using synthetic data, an LSTM anomaly model, NetworkX graphs, and a Flask dashboard.



---

## 1. Data Generation (`generate_synthetic_data.py`)

**Purpose:**  
Produce a lightweight “CERT‑style” dataset so you can develop without downloading 80 GB.

**What it does:**

- Defines **30 users** (`user01`–`user30`) and ~**15 hosts** (workstations, file servers, DC).
- Simulates and writes to `data/synthetic/`:
  - **Logon events** (`logon.csv`)
  - **USB device inserts** (`device.csv`)
  - **File transfers** with random names (`file.csv`)
  - **User metadata** (`users.csv`: dept & role)
  - **Psychometric profiles** (`psychometric.csv`: O/C/E/A/N traits)

---

## 2. Model Definition & Training (`model.py`)

### Key Constants & Paths

- `SEQ_LEN = 10`  
  Sliding‑window length for sequences.  
- `DATA_PATH`  
  Points at `data/synthetic/` CSVs.  
- `MODEL_PATH`  
  Location to save/load the trained LSTM (`.keras`).

### Main Methods

1. **`load_data()`**  
   - Reads `logon.csv`, merges date & time into a datetime column, sorts by user/time.  
   - Label‑encodes each PC into integers.

2. **`prepare_sequences()`**  
   - Slides a window of length 10 over each user’s PC sequence.  
   - Builds `(X, y)` pairs: first 10 events → 11th event.  
   - Pads, scales (0–1), reshapes to `(samples, 10, 1)`.

3. **`build_model()`**  
   - Constructs an LSTM network:  
     ```text
     LSTM(64) → Dropout(0.3) → Dense(32, relu) → Dense(1, linear)
     ```  
   - Compiled with Adam optimizer & MSE loss.

4. **`train()`**  
   - Runs `load_data()`, `prepare_sequences()`, builds the model, fits for N epochs, then saves to `MODEL_PATH`.

5. **`load_model()`**  
   - If a `.keras` model exists, loads it (no legacy HDF5 compile), then re‑compiles with MSE.

6. **`predict_score(seq)`**  
   - Given a list of last 10 encoded PCs: scales them, predicts next PC index, returns absolute error as an anomaly score.

---

## 3. Graph Construction (`graph_utils.py`)

### `build_network(logon_df, file_df, device_df) → nx.Graph`

- **Nodes**  
  - All unique **users** (`type='user'`)  
  - All unique **hosts** across logs (`type='host'`)
- **Edges**  
  - Between each user ↔ host  
  - Weight = summed event counts  
  - Tagged by interaction type: `logon`, `file`, `usb` (or combinations)

_Result:_ A NetworkX graph capturing how frequently each user touched each host.

---

## 4. Flask App & Routes (`app.py`)

### Startup

1. Instantiate `InsiderThreatModel`.
2. Try `load_model()`.  
   - If none, run `train()` on synthetic data (seconds).
3. Read all 5 CSVs into pandas DataFrames.

### In‑Memory State

- **`user_sequences`**: maps user → last up‑to‑10 encoded PC events  
- **`anomalies`**: list of detected suspicious events

### Routes

- **`GET /` → `index()`**  
  Renders `index.html` with:
  - `graph_html`: embedded Plotly network graph (threat‑colored)
  - `users`: list of all user IDs for sidebar
- **`GET /status`**  
  Health‑check JSON (`{"status":"ok"}`)
- **`GET /prompt`**  
  Returns original system prompt for docs
- **`POST /log` → `ingest_log()`**  
  - JSON `{user, pc, timestamp}`  
  - Appends encoded PC to user’s window  
  - If full (10 events), calls `predict_score()`  
  - If `score > threshold` (default 5), records anomaly
- **`GET /alerts`**  
  Top 10 highest‑scoring anomalies as JSON
- **`GET /graph`**  
  Rebuilds network graph (marks anomalous users in red), returns Plotly HTML
- **`GET /user/<user_id>` → `user_detail()`**  
  Renders `user.html` showing:
  - Full logon history (from CSV)  
  - That user’s in‑memory anomaly events

### Server Launch

```bash
python app.py
