# Drowsy Driver Detection 😴🚘
Real-time computer-vision app that detects driver drowsiness from a webcam feed and triggers **on-screen**, **audio**, and optional **SMS alerts**. Built with **Python**, **OpenCV**, and **facial landmarks**. Designed to be CPU-friendly, easy to run, and easy to tweak.

> TL;DR: We compute an **Eye Aspect Ratio (EAR)** from facial landmarks each frame. If **EAR** stays below a threshold for **N consecutive frames**, we raise an alert—avoiding false positives from normal blinks.

---

## 📸 Demo Screenshot

![Demo](assets/demo.png "Live detection with AWAKE/DROWSY badge")

---

## ✨ Features
- **Real-time** detection on CPU (≥15–20 FPS on a typical laptop)
- **Blink filtering** using consecutive-frame logic (reduces false positives)
- **Visual status badge** (AWAKE/DROWSY) overlayed on the video
- **Audio alarm** for local feedback (optional)
- **Twilio SMS alert** to a phone number with cooldown (optional)
- **Configurable thresholds**: EAR and consecutive frames
- **Privacy-friendly**: Processes frames in memory; doesn’t save video by default

---

## 🧠 How It Works (Overview)
**Pipeline**  
```
Webcam → Preprocess (resize, grayscale)
      → Face detection + facial landmarks (68-pt)
      → Extract 6 eye points per eye
      → Compute EAR per frame
      → If EAR < θ for N frames → Alert (UI + sound + SMS)
```

**Eye Aspect Ratio (EAR)**  
For each eye (six landmark points p1..p6), we compute:
\[
EAR = \frac{\lVert p_2 - p_6 \rVert + \lVert p_3 - p_5 \rVert}{2\,\lVert p_1 - p_4 \rVert}
\]
- **Intuition:** When eyes close, vertical distances shrink faster than the horizontal distance → EAR drops.
- We average **left** and **right** EAR and smooth slightly to reduce jitter.

**Default thresholds** (tune per person/camera):
- `EAR_THRESH = 0.25`
- `CONSEC_FRAMES = 20`  (≈1 second at 20 FPS)

---

## 🏗️ Architecture
```
┌───────────┐   ┌──────────────┐   ┌─────────────────────┐   ┌─────────────┐   ┌───────────────┐
│  Webcam   │→→ │  Preprocess  │→→ │ Face + Landmarks    │→→ │  EAR Logic  │→→ │ Alerts (UI/   │
│ (cv2)     │   │(resize, gray)│   │ (dlib/Haar + 68pts) │   │(θ & N rules)│   │ Sound/SMS)    │
└───────────┘   └──────────────┘   └─────────────────────┘   └─────────────┘   └───────────────┘
```

**Notes**
- **Landmarks:** Use `dlib` 68-point predictor for robust eye localization (preferred). A Haar-cascade fallback is possible but less stable.
- **Performance:** Resize frames (e.g., width=640). Optionally detect faces every few frames, track in-between to boost FPS.

---

## 🧰 Tech Stack
- **Language:** Python 3.9+
- **Core:** OpenCV (`opencv-python`), NumPy
- **Landmarks:** `dlib` (preferred) or Haar cascades (fallback)
- **Helpers:** `imutils` for convenience
- **Alerts:** `playsound` or platform sound; `twilio` (optional SMS)
- **Config:** `.env` via `python-dotenv`

---

## 🚀 Quickstart
> If you don’t need SMS, you can skip the Twilio steps.

### 1) Set up Python
```bash
# clone your repo
git clone <your-repo-url> drowsy-driver
cd drowsy-driver

# (recommended) create a virtual environment
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

# install dependencies
pip install --upgrade pip
pip install -r requirements.txt
```

**requirements.txt (example)**
```
opencv-python
numpy
imutils
dlib           # optional, preferred; if build fails, use Haar-only mode
playsound      # optional
twilio         # optional
python-dotenv
```

### 2) Models
- **dlib landmarks (recommended):** place `shape_predictor_68_face_landmarks.dat` in `models/`  
- **Haar fallback:** ensure cascade XMLs exist in `models/` (e.g., `haarcascade_frontalface_default.xml`, `haarcascade_eye_tree_eyeglasses.xml`)

### 3) Environment config
Create a `.env` file in the project root (only required if using SMS or to override defaults):
```
# --- app thresholds ---
EAR_THRESH=0.25
CONSEC_FRAMES=20
SMS_COOLDOWN_SEC=60

# --- twilio (optional) ---
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_FROM=+1XXXXXXXXXX
ALERT_TO=+1XXXXXXXXXX
```

### 4) Run the app
```bash
python main.py   --camera 0   --width 640   --no-audio false   --sms false   --use-dlib true
```
Common flags:
- `--camera`: index or IP stream URL
- `--width`: frame width (affects FPS)
- `--no-audio`: disable/enable beep
- `--sms`: toggle SMS alerts
- `--use-dlib`: toggle dlib vs Haar

---

## ⚙️ Configuration
You can set **defaults** via `.env` or CLI flags. At runtime the app typically resolves config as:
1. CLI flags (highest priority)
2. `.env` values
3. Built-in defaults

**Example: change sensitivity**  
- Increase `EAR_THRESH` slightly (e.g., 0.27) if you’re getting too few alerts.
- Increase `CONSEC_FRAMES` (e.g., 24–28) if long blinks are triggering false positives.

---

## 🧪 Validating It Works
- Watch the **status badge**: should say **AWAKE** when looking ahead, switch to **DROWSY** if eyes close for ~1s.
- Open the console log for EAR values; see them **drop** when you blink.
- If SMS is enabled, ensure you get a single text per event (cooldown avoids spam).

---

## 🛠️ Troubleshooting
**Dlib won’t install**  
- Ensure you have a compiler toolchain and CMake installed. If you can’t, run in **Haar-only** mode (`--use-dlib false`).

**No camera found**  
- Try `--camera 1`, or check OS camera permissions.

**Low FPS**  
- Use `--width 480` or detect faces every 3–5 frames and track in-between.

**SMS not sending**  
- Verify `.env` credentials and phone numbers (E.164 format, e.g., `+1...`). Check Twilio trial restrictions.

**False alarms in low light**  
- Turn on a light, enable histogram equalization in code, increase `CONSEC_FRAMES` slightly.

---

## 🔒 Privacy
- Frames are processed **in-memory only**. No video is saved or uploaded. SMS contains **only** the alert text, not images.

---

## 🗺️ Roadmap
- Hybrid **EAR + CNN** eye-state classifier for robustness
- Head-pose/yawn detection
- Embedded builds (Jetson / Raspberry Pi)
- Optional dashboard with aggregated stats (no video uploads)

---

## 📂 Suggested Repo Structure
```
.
├── assets/
│   └── demo.png                 # ← add your screenshot here
├── models/
│   ├── shape_predictor_68_face_landmarks.dat   # (recommended)
│   └── haarcascade_frontalface_default.xml     # (fallback)
├── src/
│   ├── detector.py              # face/landmarks
│   ├── ear.py                   # EAR computation + smoothing
│   ├── alerts.py                # UI, sound, SMS (Twilio)
│   ├── config.py                # env + CLI handling
│   └── main.py                  # entrypoint
├── requirements.txt
└── README.md
```

---

## ❓ FAQ
**Why EAR instead of a neural net?**  
It’s **fast, explainable, and works on CPU**. You can later add a tiny CNN for eye-state to improve robustness.

**Do I need Twilio?**  
No. It’s optional. Local UI + sound is enough for many scenarios.

**Will it work with glasses?**  
Usually yes, but accuracy can vary with reflections. Use dlib landmarks and decent lighting.

---

## 📝 License
MIT (or your preferred license).

---

## 🙌 Acknowledgements
- OpenCV (cv2) and the dlib community
- Twilio for the developer-friendly SMS API
- Everyone who tested the app across different lighting and cameras
