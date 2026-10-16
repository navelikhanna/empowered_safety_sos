# Empowered Safety

A browser-based personal safety prototype that combines three features in one web page:

- **SOS dispatch**: send an emergency SMS with your location to saved contacts.
- **Voice activation**: trigger an alert by speaking a keyword.
- **Camera-based heart-rate screening**: estimate heart rate from a webcam (remote photoplethysmography, rPPG) and raise an alert on abnormal readings.

There is nothing to install on the client and no dedicated hardware is needed. SMS dispatch requires the small self-hosted Flask/Twilio backend included here.


## Contents

- [How it works](#how-it-works)
- [Quick start](#quick-start)
- [Repository structure](#repository-structure)
- [Heart-rate validation](#heart-rate-validation)
- [Data export](#data-export)
- [Tests](#tests)
- [Configuration](#configuration)
- [Limitations](#limitations)
- [Citation and license](#citation-and-license)

---

## How it works

| Module | What it does | Runs in |
|---|---|---|
| Contacts and SOS | Stores emergency contacts (E.164 phone numbers) and sends an SMS with a location link | Browser + Flask backend |
| Voice activation | Listens for a keyword using the browser's `SpeechRecognition` API | Browser |
| Heart-rate monitor | Estimates BPM from the face using the POS method, with calibration, dynamic thresholds and sensor fusion | Browser (`script.js`) |
| Backend | Receives SOS requests, sends SMS through Twilio, logs every attempt | `sos_server.py` |

**Heart-rate method (POS):** central-face RGB signal, 15 s window stepped every 3 s, linear detrend, Hann window, spectral peak in the 0.7-3.0 Hz band.

---

## Quick start

### Requirements

- Python 3.9 or newer
- A Twilio account (Account SID, Auth Token and a phone number) for SMS dispatch
- A Chromium-based browser (voice recognition uses the browser's speech service)
- Node.js, only for the optional JS-vs-Python cross-check test

### 1. Backend

```bash
pip install -r requirements.txt
cp .env.example .env
# edit .env with your Twilio Account SID, Auth Token and phone number
python sos_server.py
```

The server listens on `http://127.0.0.1:5000`.

### 2. Frontend

Serve the page over HTTP. Opening `index.html` as a `file://` URL will not work, because camera and microphone access need a secure context.

```bash
python -m http.server 8000
```

Open `http://127.0.0.1:8000/index.html`.

### 3. Add a contact

Add at least one emergency contact in E.164 format (for example `+919876543210`) before testing SOS dispatch.

---

## Repository structure

```text
empowered_safety/
├── index.html              Frontend page
├── script.js               Frontend logic (contacts, SOS, voice, heart-rate monitor)
├── styles.css              Frontend styles
├── sos_server.py           Flask backend (Twilio SMS + dispatch logging)
├── ppg_detector.py         Standalone finger-PPG prototype (reference only, not used by the app)
├── requirements.txt        Backend dependencies
├── .env.example            Template for Twilio credentials and options
├── CHANGES.md              Change log
├── validation/             Offline rPPG evaluation (see validation/README.md)
├── tests/                  Backend tests and JS-vs-Python cross-check
├── data/                   Where datasets are placed (see data/README.md)
└── archive/                Earlier prototypes, not part of the running system
```

---

## Heart-rate validation

The estimator is evaluated offline on the public **UBFC-rPPG** dataset (DATASET_2), not on volunteer recordings. The method was developed on 10 subjects and then evaluated once, frozen, on 32 held-out test subjects.

- `validation/rppg_core.py` is the Python port of the estimator shipped in the app.
- `validation/evaluate_ubfc.py` is the main evaluation script:

```bash
python validation/evaluate_ubfc.py --method pos --split dev
python validation/evaluate_ubfc.py --method pos --split test
```

- Scripts marked LEGACY (`evaluate_ubfc_legacy.py`, `evaluate_mcd.py`, `synthetic_motion_experiment.py`) evaluate the earlier red-channel method that the app no longer uses.

Datasets are **not** included in this repository. Download them from their official pages and follow `data/README.md` for the folder layout. Check each dataset's terms before redistributing any files.

Outputs (CSV files and a Bland-Altman plot) are written to `results/`. For the full set of options and sanity checks, see `validation/README.md`.

---

## Data export

The frontend logs trials in memory and exports CSV files:

- **Heart rate:** every BPM attempt, including failed estimates, motion score, frame rate, calibration baseline, alert events and any manually entered reference readings.
- **Voice:** every trial with session ID, speaker ID, noise condition, transcript, recognizer confidence and matched keyword. Recognition errors are logged, not dropped.
- **SOS:** per dispatch, the trigger source (manual, voice or heart-rate based), client and server latency, geolocation accuracy radius and network information where available.

The server also writes `sessions/sos_dispatch_log.csv` and `sessions/run_metadata.json`.

Note that `geo_accuracy_m` is the radius the browser *claims*, not a measured error.

To analyse exported logs:

```bash
cd validation
python analyze_logs.py --sos sos_dispatch_log.csv --voice voice_trial_log.csv --hr hr_session_data.csv
```

---

## Tests

```bash
# Backend tests with Twilio mocked (no SMS is sent)
python -m unittest tests.test_server -v

# Check that the browser estimator and the Python port agree (needs Node.js)
python tests/crosscheck_js_python.py
```

---

## Configuration

Set these in `.env` (see `.env.example`):

| Variable | Purpose |
|---|---|
| `TWILIO_ACCOUNT_SID` | Twilio account identifier |
| `TWILIO_AUTH_TOKEN` | Twilio auth token |
| `TWILIO_PHONE_NUMBER` | Sending number, E.164 format |
| `SERVER_LOCATION`, `NETWORK_NOTE` | Optional, recorded in `run_metadata.json` |
| `ALLOWED_ORIGINS` | Optional, comma-separated origins allowed to call `/sos` |

By default the backend only accepts requests from `127.0.0.1` and `localhost` on ports 8000 and 5500. **Never commit your `.env` file.**

---

## Limitations

- The backend must be self-hosted; there is no public deployment.
- Voice recognition uses the browser's cloud speech service in Chromium browsers, not on-device processing.
- Heart-rate estimation is evaluated offline on laboratory video with a pulse-oximeter reference. It has not been tested for real emergency detection, for motion or distress, or under real in-browser webcam conditions.
- The 130 / 60 BPM and dynamic-band thresholds are prototype heuristics, not clinically validated criteria.
- "Alert Nearest Police" is a placeholder; no emergency-services API is called.

---
