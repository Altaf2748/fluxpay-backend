# FluxPay e-KYC Backend
### Multi-Modal Face + Voice Identity Verification Server

Built with **InsightFace** (ArcFace buffalo_l) + **SpeechBrain ECAPA-TDNN** + **FastAPI**.  
Powers real-time biometric verification for the FluxPay payment wallet.

---

## What This Does

This server accepts a face image and voice recording, extracts biometric embeddings from both, fuses the scores using a weighted algorithm, and returns an `ACCEPT` or `REJECT` decision. It is called automatically by the FluxPay frontend during:

- **User onboarding** — enrolls face + voice once at signup
- **High-value payments** — verifies identity before any transaction above ₹1000

---

## Requirements

| Requirement | Version |
|---|---|
| Python | 3.10 exactly (not 3.11, not 3.12) |
| pip | any recent version |
| ffmpeg | must be installed and on PATH |
| Windows | 10 or 11 (Developer Mode recommended) |
| RAM | minimum 4GB free |
| Disk | minimum 2GB free (models download on first run) |

> **Python 3.10 is mandatory.** Other versions cause package conflicts with insightface and speechbrain.

---

## Step 1 — Install Python 3.10

1. Go to: https://www.python.org/downloads/release/python-31011/
2. Scroll down to **Files** → download **Windows installer (64-bit)**
3. Run the installer
4. **IMPORTANT:** Check the box **"Add Python to PATH"** before clicking Install
5. Verify: open PowerShell and run:
   ```
   python --version
   ```
   Must show `Python 3.10.x`

---

## Step 2 — Install ffmpeg

ffmpeg is required for audio processing.

1. Go to: https://www.gyan.dev/ffmpeg/builds/
2. Download **ffmpeg-release-essentials.zip**
3. Extract it to `C:\ffmpeg`
4. Add to PATH:
   - Press Windows key → search "Environment Variables" → open it
   - Under System Variables → find **Path** → click Edit
   - Click New → type `C:\ffmpeg\bin`
   - Click OK on all windows
5. Verify: open a NEW PowerShell window and run:
   ```
   ffmpeg -version
   ```
   Must show version info, not an error

---

## Step 3 — Enable Windows Developer Mode

This is required for model downloading to work correctly.

1. Press Windows key → search **"Developer settings"** → open it
2. Toggle **Developer Mode → ON**
3. Click Yes on the confirmation dialog
4. You do not need to restart

---

## Step 4 — Clone the Repository

```
git clone https://github.com/Altaf2748/fluxpay-backend.git
cd fluxpay-backend
```

---

## Step 5 — Create Virtual Environment

```
python -m venv .venv
```

Activate it — run this every time you open a new terminal:

```
# Windows PowerShell:
.venv\Scripts\activate

# Windows Command Prompt:
.venv\Scripts\activate.bat

# Mac / Linux:
source .venv/bin/activate
```

You must see `(.venv)` at the start of your prompt before continuing.

---

## Step 6 — Install Dependencies

Run these commands one by one. Do not skip any.

```
pip install -U pip
```

```
pip install numpy<2
```

```
pip install torch==2.1.2 torchaudio==2.1.2 --index-url https://download.pytorch.org/whl/cpu
```

```
pip install speechbrain==1.0.0
```

```
pip install insightface onnxruntime
```

```
pip install fastapi uvicorn python-multipart
```

```
pip install opencv-python requests tqdm soundfile
```

```
pip install huggingface_hub==0.23.4
```

> **Why specific versions?**  
> `numpy<2` — insightface and speechbrain were compiled against NumPy 1.x  
> `torch==2.1.2` — must match torchaudio exactly  
> `speechbrain==1.0.0` — older versions don't have `speechbrain.inference`  
> `huggingface_hub==0.23.4` — newer versions break the model download API

---

## Step 7 — Verify Installation

Run this to confirm all packages loaded correctly:

```
python -c "import insightface, speechbrain, fastapi; print('All packages OK')"
```

Expected output:
```
All packages OK
```

If you see warnings about torchaudio backend or torchvision — those are harmless, ignore them.

If you see any `ModuleNotFoundError` or `ImportError` — go to the Troubleshooting section at the bottom.

---

## Step 8 — Set Up the Database Files

The database files must exist and contain valid JSON before starting the server.

```
[System.IO.File]::WriteAllText("db\teachers.json", '{"version": 1, "teachers": []}')
```

```
[System.IO.File]::WriteAllText("db\pending.json", '{"version": 1, "pending": []}')
```

```
New-Item -ItemType Directory -Force -Path logs | Out-Null
New-Item -ItemType File -Force -Path logs\attempts.jsonl | Out-Null
```

> **Why not use `echo` or `Out-File`?**  
> PowerShell's `Out-File` and `echo` add a BOM (Byte Order Mark) to the file which breaks Python's JSON parser. `WriteAllText` writes clean UTF-8 without BOM.

---

## Step 9 — Start the Server

```
python main.py
```

**First run only:** The server will automatically download two AI models:
- InsightFace buffalo_l (~250MB) — face recognition
- SpeechBrain ECAPA-TDNN (~90MB) — voice recognition

This takes 3-5 minutes depending on internet speed. Subsequent starts are instant.

**Expected output when ready:**
```
Loading AI Models...
find model: ...buffalo_l\w600k_r50.onnx recognition...
Models Loaded Successfully.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

---

## Step 10 — Confirm It's Working

Open a browser and go to:
```
http://localhost:8000/api/health
```

Expected response:
```json
{"status":"ok","teachers":0,"version":"1.0"}
```

Also open the interactive API docs:
```
http://localhost:8000/docs
```

---

## Configuration

All settings are in `config.json`. The most important ones:

```json
{
  "thresholds": {
    "face": 0.30,
    "voice": 0.25,
    "alpha": 0.6,
    "require_both": true
  }
}
```

| Setting | Description | Recommended |
|---|---|---|
| `face` | Face similarity threshold (0-1) | 0.30 for testing, 0.46 for production |
| `voice` | Voice similarity threshold (0-1) | 0.25 for testing, 0.40 for production |
| `alpha` | Face weight in fusion (voice gets 1-alpha) | 0.6 |
| `require_both` | Both face AND voice must match same person | true |

> Start with low thresholds (0.30 / 0.25) during initial setup and testing. Increase to production values (0.46 / 0.40) once enrollment quality is confirmed good.

---

## API Endpoints

### Health Check
```
GET /api/health
```

### Enroll a User
```
POST /api/register_teacher
Content-Type: application/json

{
  "teacher_id": "user-uuid",
  "name": "user@email.com",
  "image": "base64-encoded-jpeg",
  "audio": "base64-encoded-wav",
  "robot_captured": false,
  "pending_approval": false
}
```

### Verify Identity (Fusion)
```
POST /api/verify_fusion
Content-Type: application/json

{
  "image": "base64-encoded-jpeg",
  "audio": "base64-encoded-wav",
  "source": "fluxpay_web"
}
```

Returns:
```json
{
  "decision": "ACCEPT",
  "face": { "score": 0.72, "recognized": true },
  "voice": { "score": 0.61, "recognized": true },
  "fusion": { "fused_score": 0.68 }
}
```

### List Enrolled Users
```
GET /api/teachers
```

---

## CORS Configuration

The server allows requests from these origins by default:
- `http://localhost:5173`
- `http://localhost:8080`
- `http://localhost:3000`
- `http://127.0.0.1:8080`
- `https://fluxpay-smart-flow.vercel.app`

To add your own frontend URL, open `main.py` and add it to the `allow_origins` list in the `CORSMiddleware` block near the top of the file.

---

## Troubleshooting

### `ModuleNotFoundError: No module named 'speechbrain.inference'`
```
pip uninstall speechbrain -y
pip install speechbrain==1.0.0
```

### `AttributeError: module 'torchaudio' has no attribute 'list_audio_backends'`
```
pip uninstall torch torchaudio speechbrain -y
pip install torch==2.1.2 torchaudio==2.1.2 --index-url https://download.pytorch.org/whl/cpu
pip install speechbrain==1.0.0
```

### `TypeError: hf_hub_download() got an unexpected keyword argument 'use_auth_token'`
```
pip install huggingface_hub==0.23.4
```

### `OSError: [WinError 1314] A required privilege is not held by the client`
Enable Windows Developer Mode (Step 3 above) then restart the terminal.

### `json.decoder.JSONDecodeError: Unexpected UTF-8 BOM`
Your db files were created with PowerShell Out-File which adds a BOM. Fix:
```
[System.IO.File]::WriteAllText("db\teachers.json", '{"version": 1, "teachers": []}')
[System.IO.File]::WriteAllText("db\pending.json", '{"version": 1, "pending": []}')
```

### `json.decoder.JSONDecodeError: Extra data`
Your db file has two JSON objects. Fix it with the same WriteAllText commands above.

### `numpy: A module compiled with NumPy 1.x cannot run in NumPy 2.x`
```
pip install "numpy<2"
```

### `OPTIONS /api/verify_fusion HTTP/1.1" 400 Bad Request`
Your frontend origin is not in the CORS allowed list. Add it to `allow_origins` in `main.py`.

### `POST /api/register_teacher HTTP/1.1" 404 Not Found`
The route exists. This usually means the request body is malformed. Check that image and audio are base64 strings without the `data:image/jpeg;base64,` prefix.

### `face<thresh,voice<thresh` — verification always fails
Two possible causes:
1. User was never enrolled — check `GET /api/teachers` returns count > 0
2. Thresholds too high — lower them in config.json to `face: 0.20, voice: 0.15` for testing

### `Unable to connect to the remote server`
The backend is not running. Start it with `python main.py` in a terminal that stays open.

---

## Daily Usage

Every time you use FluxPay:

**Terminal 1 — keep open the entire session:**
```
cd path\to\fluxpay-backend
.venv\Scripts\activate
python main.py
```

**Terminal 2 — frontend:**
```
cd path\to\fluxpay-smart-flow
npm run dev
```

The backend must be running at port 8000 whenever the frontend needs to enroll or verify a user.

---

## Project Structure

```
fluxpay-backend/
├── main.py                    ← FastAPI server (all endpoints)
├── verify_fusion.py           ← Face + voice fusion decision logic
├── face_model_insightface.py  ← InsightFace ArcFace embeddings
├── voice_model.py             ← SpeechBrain ECAPA-TDNN embeddings
├── db_utils.py                ← JSON database read/write helpers
├── server_utils.py            ← Logging, file locking, utilities
├── config.json                ← All configuration and thresholds
├── db/
│   ├── teachers.json          ← Enrolled users (auto-managed)
│   └── pending.json           ← Pending approvals (auto-managed)
├── logs/
│   └── attempts.jsonl         ← Audit log of every attempt
└── pretrained_models/         ← Downloaded automatically on first run
```

---

## Related Repository

FluxPay Frontend: https://github.com/Altaf2748/fluxpay-smart-flow