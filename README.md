# AI-GARA — Eye Tracking for Cognitive Difficulty Detection
  
A research-grade Chrome MV3 extension that uses gaze-tracking to detect reading difficulty in real time. Built for academic research, this system analyzes eye movement patterns to classify cognitive states during reading and delivers targeted interventions.

**Research Focus:** Can gaze tracking reliably predict cognitive difficulty? This extension implements the full gaze-first pipeline to investigate that question.

---

## Quick Start

```bash
# Clone and setup
git clone https://github.com/yourusername/AI-GARA.git
cd AI-GARA

# Install dependencies
npm install
cd server && npm install && cd ..

# Set up your Groq API key (free tier at console.groq.com)
echo "GROQ_API_KEY=your_key_here" > server/.env

# Start the backend
node server/index.js

# Load extension in Chrome
# 1. chrome://extensions/
# 2. Enable Developer mode
# 3. Load unpacked → select this folder
```

Enable the camera, run calibration, and start reading. The system detects cognitive states via gaze.

---

## Overview

AI-GARA is a gaze-first reading analysis system designed to investigate whether eye movement patterns can reliably predict reading difficulty. The three-layer architecture:

1. **Gaze Tracking** — WebGazer.js + local TensorFlow FaceMesh extract real-time eye position
2. **Cognitive State Detection** — Decision tree classifier maps 9 gaze features → 5 cognitive states
3. **Non-Intrusive Intervention** — Popup suggestions without disrupting the reading experience

Works on:
- Web articles, documentation, any text-heavy page
- Local PDFs (via bundled PDF.js viewer)
- Local PowerPoint presentations (via bundled PPTX parser)

**Zero transmission of video or raw gaze data** — all processing is local to your browser.

---

## How It Works

```
Webcam Feed
    ↓
WebGazer.js (TensorFlow FaceMesh) — local gaze estimation (x, y) every ~33ms
    ↓
gaze-features.js — 2.5-second rolling window; extract 9 features
    (fixation duration, regression rate, saccade metrics, drift, quality)
    ↓
lang-detect.js — script-aware patching for RTL/CJK languages
    ↓
classifier.js — decision tree: 9 features → cognitive state
    ↓
Five Cognitive States:
  • focused      → reading normally (no action)
  • skimming     → fast scanning (no action)
  • struggling   → high regressions + long fixations → AI explanation
  • overloaded   → dense text, short fixations → simplified summary
  • zoning_out   → gaze drifting, off-screen → gentle nudge
    ↓
Non-intrusive popup: question or summary
(only when confident; never interrupt on uncertain states)
```

**Key Research Features:**

- **No video storage** — WebGazer processes frames locally and discards immediately
- **9 gaze features** — fixation duration, regression rate, saccade length/variance, gaze drift, velocity, line re-reads, quality score
- **Personal baseline normalization** — Each reader's features are calibrated against their own reading profile, not global norms
- **Language-aware patching** — RTL and CJK scripts require feature adjustment before classification
- **Synthetic-data classifier** — Trained on controlled data (known limitation; marked for real-world validation)
- **State smoothing** — 3-sample modal ring buffer suppresses single-frame noise

---

## Research Capabilities

### Gaze Feature Extraction
| Feature | Description | Interpretation |
|---|---|---|
| **Fixation Duration** | Average time eyes hold on one point (ms) | Long fixations ↔ cognitive load |
| **Regression Rate** | Proportion of eye movements going backward | High regression ↔ re-reading / confusion |
| **Saccade Length** | Average distance of eye jumps | Short saccades ↔ close reading |
| **Saccade Variance** | Std dev of saccade distances | Low variance ↔ consistent scanning |
| **Gaze Drift** | Dispersion of samples in a fixation (pixels) | High drift ↔ noise / poor calibration |
| **Velocity Mean** | Average eye speed (px/ms) | *Note: unreliable due to tracker noise* |
| **Line Re-reads** | Detected backward movement within line | High count ↔ difficulty with line |
| **Quality Score** | Confidence in tracker output (0–1) | <0.25 triggers gating (no classification) |
| **On-Page Fraction** | % of samples within text area | Detects reader attention drift |

**Known Limitation:** Saccade metrics (length, variance, velocity) sit within WebGazer's ~180px error margin and largely measure tracker noise rather than eye movement. Documented for transparency; retraining recommended before production use.

### Calibration & Personalization
- **Dot calibration** — 9-point grid (2 passes) trains WebGazer's ridge regression model
- **Reading calibration** — Natural-pace word highlighting captures ~80 real reading samples
- **Personal baseline** — Each user's feature distributions become the norm; readers above/below their own baseline trigger states
- **Continuous improvement** — Every mouse click is logged as a training sample for WebGazer
- **Gaze quality gate** — Classification skipped when quality <25% (poor lighting, occlusion)

### Intervention System
- **Non-intrusive popups** — Suggestions appear below or beside text, not overlaying it
- **Reading map** — Sidebar visualization of article progress and confusion events (`Alt+M`)
- **Focus ruler** — Horizontal dim-band following gaze Y, aids line tracking (`Alt+F`)
- **Session tracking** — Per-session state durations, event log, average WPM
- **Paragraph highlighting** — Source paragraphs marked for review after session ends

### Document Support
| Format | Method |
|---|---|
| Web articles | Direct DOM injection; gaze maps to `<p>`, `<div>`, etc. |
| PDFs | `file://*.pdf` intercepted; bundled PDF.js viewer with gaze overlay |
| PowerPoint (PPTX) | `file://*.pptx` intercepted; JSZip parses slides; gaze tracking over text |

---

## Cognitive States (Gaze-Based Classification)

The classifier maps 9 normalized gaze features to 5 states:

| State | Gaze Pattern | Typical Features | Interpretation |
|---|---|---|---|
| **focused** | Steady fixations, forward saccades | Low regression, normal fixation, high velocity | Normal reading pace |
| **skimming** | Fast scanning, long saccades | High velocity, short fixations, low fixation time | Rapid document scanning |
| **struggling** | High regressions, prolonged fixations | High regression rate, long fixation duration, low saccade variance | Re-reading due to difficulty or confusion |
| **overloaded** | Shallow scanning despite density | Short fixations with high text complexity | Cognitive overload; reduced attention |
| **drifting** | Off-screen, low fixation count | Low on-page fraction, gaze outside text area | Attention has left the page |

Classification runs every 3 seconds; state smoothing (3-sample modal filter) suppresses single-frame noise.

**Intervention Policy:**
- Max 1 intervention per 3 minutes per paragraph
- Max 5 per session
- Never interrupt on `unknown` (insufficient confidence)

---

## Language Handling & Script-Aware Feature Patching

The classifier was trained on English LTR reading data. To generalize across scripts and languages, feature values are patched before classification based on detected script:

### RTL Languages (Arabic, Hebrew, Persian, Urdu, Syriac, etc.)

**Problem:** In LTR, leftward eye movement = regression. In RTL, leftward = forward motion.

**Solution:** `lang-detect.js` inverts `regression_rate` for RTL pages:
```
patched_regression_rate = 1 − raw_regression_rate
```

Result: A focused RTL reader (mostly leftward saccades) no longer triggers false `struggling` classification.

### CJK Languages (Chinese, Japanese, Korean)

**Problem:** Dense glyphs require 30–40% longer fixations per character.

**Solution:** Fixation features scaled by 0.78 before classification:
```
avg_fixation_ms ← avg_fixation_ms × 0.78
fixation_std    ← fixation_std     × 0.78
```

Result: Normal-speed Japanese reading (500ms fixations) no longer falsely triggers `overloaded` state.

### Script Detection

Three-layer detection (runs once per page load):
1. `<html lang="...">` attribute
2. `dir="rtl"` or computed CSS `direction`
3. Unicode character sampling (>15% threshold in RTL or CJK blocks)

**Known Limitation:** Single-page apps that navigate between languages without reload will not re-detect. This is documented for future research improvements.

---

## Architecture: Three-Layer System

### Layer 1: Gaze Acquisition (webgazer-bootstrap.js)
Injects WebGazer into the page's MAIN world (where `window.webgazer` is accessible). Runs TensorFlow FaceMesh to extract face landmarks and estimate gaze (x, y) on screen. Uses postMessage bridge to relay gaze samples back to the isolated content script.

### Layer 2: Feature Extraction & Classification (gaze-features.js → classifier.js)
Buffers gaze samples over 2.5-second rolling window. Computes 9 features: fixation duration, regression rate, saccade metrics, gaze drift, quality score. Applies language-aware patching (RTL inversion, CJK fixation scaling). Feeds normalized features to decision tree classifier → cognitive state.

### Layer 3: Intervention & UI (ui-controller.js → popup)
Non-intrusive popups display targeted suggestions. Session tracking records state durations, event log. Reading map sidebar visualizes progress and confusion moments.

### Supporting Systems
- **reading-calibration.js** — Dot calibration (9-point) and reading calibration (word-by-word) to build personal baselines
- **session-tracker.js** — Per-session statistics (time per state, WPM, confusion count)
- **comprehension-monitor.js** — Telemetry-based checks (reading too fast for difficulty, scroll backtrack)
- **lang-detect.js** — Script detection and feature patching for multilingual support
- **pdf-handler.js**, **pptx-handler.js** — Document parsing for PDF and PowerPoint

### Cross-World Communication
Chrome content scripts run in an isolated world; WebGazer runs in the page context (MAIN world):

```
Content script (isolated)
    ↓ postMessage({ gaze: {x, y} })
webgazer-bootstrap.js (MAIN world)
    ↓ recordScreenPosition(x, y) [trains WebGazer]
    ↓ postMessage({ gaze: estimated{x, y} })
Content script
    ↓ gaze-features.js → classifier.js → state
```

---

## Installation & Testing

### Prerequisites
- Google Chrome (or Chromium) version 87+ (MV3 support)
- Node.js 18+
- A webcam
- Groq API key (free tier: console.groq.com)

### Quick Start (5 minutes)

```bash
# 1. Clone and install
git clone https://github.com/yourusername/AI-GARA.git
cd AI-GARA
npm install

# 2. Set up backend
cd server
npm install
echo "GROQ_API_KEY=gsk_your_key_here" > .env
node index.js &
cd ..

# 3. Load in Chrome
# → chrome://extensions/
# → Developer mode (top right)
# → Load unpacked → select this folder
# → Click Details → "Allow access to file URLs" (for PDF/PPTX)

# 4. Test on a page
# Open any article → click AI-GARA icon → Start Camera
# Follow calibration prompts → read
```

### Calibration (Required)

1. **Dot calibration** (~1 minute)
   - Opens automatically on first use
   - Click green dots as they appear (2 passes × 9 points = 18 clicks)
   - Trains WebGazer's ridge regression offset

2. **Reading calibration** (~1 minute, optional)
   - Words highlight one at a time; read at natural pace
   - Captures ~80 training samples from your actual reading position
   - Builds personal WPM baseline (used for too-fast/too-slow detection)

**Run both calibrations once; they persist across all pages.**

---

## Backend Server (`server/index.js`)

Proxies paragraph text to Groq API for AI summaries. Runs on `http://localhost:3000` by default.

**Environment setup:**
```bash
cd server
npm install
echo "GROQ_API_KEY=gsk_your_key_here" > .env
node index.js
```

**Endpoint:**
```
POST /api/summarize
Content-Type: application/json

{
  "text": "paragraph text...",
  "mode": "tldr" | "simplify" | "explain",
  "context": "previous paragraph (optional)"
}
```

**Security:** CORS restricted to `localhost` + `chrome-extension://` origins. 30 req/min rate limit. API key never exposed to extension.

---

## Usage

### Typical Reading Session

1. Open any article/PDF/PPTX
2. Click AI-GARA icon → **Start Camera** (allow permission)
3. Follow calibration prompts (dot calibration, optional reading calibration)
4. Read normally — system runs silently in background
5. When `struggling` or `overloaded` is detected, popup appears with AI summary
6. After session, view **Session Report** to see:
   - Time in each cognitive state
   - Paragraphs where difficulty was detected
   - Average WPM
   - Event log (confusion moments, regressions)

### Keyboard Shortcuts

| Key | Action |
|---|---|
| `Alt+S` | Summarise paragraph at current gaze |
| `Alt+T` | Toggle text-to-speech (read aloud) |
| `Alt+F` | Toggle focus ruler (dim-band) |
| `Alt+M` | Toggle reading map sidebar |
| `Alt+1` to `Alt+5` | Simulate cognitive states (for testing) |
| `Esc` | Close popup |

### Testing & Debugging

**Simulate cognitive states** (useful for testing interventions):
- `Alt+1` → Simulate `struggling` (high regression)
- `Alt+2` → Simulate `overloaded` (short fixations)
- `Alt+3` → Simulate `drifting` (off-page)
- `Alt+4` → Simulate `skimming` (fast saccades)

**Toggle debug mode** in popup Settings to see real-time gaze coordinates and feature values.

---

## Configuration & Storage

Settings are stored in `chrome.storage.local` (persistent across restarts):

| Key | Default | Purpose |
|---|---|---|
| `sra_eye` | `false` | Enable/disable eye tracking |
| `sra_calibration` | `{dx:0, dy:0}` | Gaze offset (set during dot calibration) |
| `sra_personal_baseline` | `null` | Per-reader feature thresholds (from reading calibration) |
| `sra_baseline_wpm` | `null` | Personal words-per-minute baseline |
| `sra_backend_url` | `http://localhost:3000/api/summarize` | AI backend endpoint |
| `sra_sessions` | `[]` | Session data (last 20) |
| `sra_debug` | `false` | Show real-time gaze coordinates |
| `sra_tts` | `false` | Enable text-to-speech on interventions |

**Research note:** All data stays local. No telemetry is sent unless you explicitly send session data to an external service.

---

## Accessibility Features

- **Focus Ruler** (`Alt+F`) — Horizontal dim-band follows gaze Y; aids line tracking and place-keeping
- **Text-to-Speech** (`Alt+T`) — Web Speech API; reads triggered paragraphs word-by-word with highlighting
- **Dyslexia Mode** — Font, spacing, line-height adjustments; optional colour overlay
- **Dark Mode** — Theme extension UI and in-page overlays
- **Bionic Reading** — Bolds first ~45% of each word for visual anchors

All features are optional and can be toggled independently.

---

## Privacy & Data Handling

- **No video stored or transmitted** — WebGazer.js processes webcam locally via TensorFlow.js; frames discarded immediately after landmark extraction
- **Gaze data stays local** — Raw coordinates, computed features, cognitive state labels never leave your browser
- **Only text is sent to AI** — Paragraph snippets go to your local backend (`localhost:3000`), which proxies to Groq
- **Session data is local** — Stored in `chrome.storage.local`; never synced to cloud
- **Full transparency** — Enable `sra_debug` to see real-time feature values and gaze dots

---

## Security

- **API keys isolated** — `GROQ_API_KEY` in `server/.env` only; never exposed to extension
- **CORS restricted** — Backend accepts only `chrome-extension://` and `localhost`
- **Rate limited** — 30 requests/min on `/api/summarize`
- **Bundled libraries** — WebGazer, PDF.js, JSZip included; no CDN calls
- **Input sanitized** — All text HTML-escaped before rendering

---

## Development & Research Notes

### Known Limitations (Transparent for Researchers)

1. **Classifier trained on synthetic data** — 5000 rows (500/class + Gaussian noise), 0.851 synthetic accuracy. Real-world validation needed before production claims.

2. **Saccade metrics unreliable** — `saccade_length`, `saccade_std`, `velocity_mean` sit inside WebGazer's ~180px error; they measure tracker noise more than eye movement. Best gaze features: `gaze_drift`, `on_page_fraction`, `fixation_duration`, `regression_rate`.

3. **Feature extraction improvements** — `gaze-features.js` line 84 uses raw viewport Y (no scroll offset), so scrolling changes line band with zero eye movement. DBSCAN eps=80 is below tracker noise. Documented for future work.

4. **RTL/CJK patching is workaround** — Feature inversion/scaling compensates for LTR-trained classifier on non-LTR scripts. True multilingual training would be superior.

### Extending the System

**Collect Real Reading Data:**
1. Enable feature export in `session-tracker.js` (return raw 9-feature vectors)
2. Label sessions by cognitive state (use state smoothing output)
3. Retrain decision tree (scikit-learn, XGBoost, etc.)
4. Export as JavaScript if/else; replace `src/content/classifier.js`

**Add New Cognitive State:**
1. Add leaf to decision tree in `classifier.js`
2. Add color mapping in `popup.html` + `overlay.css`
3. Define action in UI handler

**Change AI Model:**
Edit `server/index.js` — change `model` param in Groq call. System is model-agnostic.

### Permissions

| Permission | Reason |
|---|---|
| `storage` | Calibration, settings, sessions |
| `scripting` | Inject WebGazer (MAIN world) |
| `activeTab` | Send messages to current tab |
| `tabs` | URL inspection, tab creation |
| `webNavigation` | Monitor `file://` navigation |
| `file:///*` | Fetch local PDFs/PPTXs |

---

## Known Issues & Future Work

- **Single-page app language switching** — Detection runs once on load; SPA navigation to different language contexts won't re-detect. Add `MutationObserver` on `<html lang>` / `dir`.
- **PPTX image slides** — Current viewer renders text only. Image-heavy slides show empty.
- **Real-world classifier validation** — Synthetic accuracy is not real-world validation. Collect reading data from diverse subjects.
- **Multilingual retrain** — Collect RTL/CJK reading samples to eliminate feature patching workaround.

---

## Citation

If using this system for academic research:

```
AI-GARA: Eye Tracking for Cognitive Difficulty Detection
Research extension implementing gaze-first classification pipeline
WebGazer (Papoutsaki et al., 2016) + decision tree classifier
AGPL-3.0 License
```

Acknowledge synthetic-data limitation and ~180px gaze error margin in your results.

---

**Status:** Research variant, pre-release (v0.2.0)
**License:** AGPL-3.0 (due to WebGazer.js GPLv3 dependency)
**Note:** This extension is maintained for educational and research purposes. For production reading-assistance systems, see related work. Alcoia used in contrast and files, send for review. old context. edit readme. explore older version of the codebase deeply
