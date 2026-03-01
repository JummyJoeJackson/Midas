# Midas - QHacks 2026

Theme: "The Golden Age"

Authors: Daniel Kwan, Diego Gonzalez, Ryan Li, Siddharth Tiwari

**Touch Broken Tech. Restore to Gold.**

MIDAS is an AR-powered repair assistant that helps anyone diagnose and fix broken technology. Available as a web application (**optimized for mobile**), simply point your camera at a device and MIDAS identifies components, detects faults, and provides step-by-step, voice-guided, interactive repair instructions. Midas uses AI to bring neglected devices to their former "Golden Age".

---

## How It Works

1. **Scan** -- Point your camera at a broken device
2. **Detect** -- YOLOv11 identifies and labels objects in real-time with AR bounding boxes
3. **Diagnose** -- GPT-4o Vision analyzes the image and identifies damage
4. **Repair** -- Step-by-step voice-guided instructions, one step at a time
5. **Verify** -- Mark steps complete and get the next instruction

Users interact via **hold-to-talk** (Whisper STT) or **text input**. Midas' responses are read aloud via OpenAI TTS.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 19, Vite 7, Tailwind CSS v4, Framer Motion |
| **Backend** | FastAPI, Uvicorn |
| **Object Detection** | YOLOv11n (Ultralytics), trained on 80 COCO classes inc. cellphone, computer mouse, keyboard, person, etc. |
| **Image Processing** | OpenCV (CLAHE contrast enhancement, sharpening, resize) |
| **Vision AI** | GPT-4o Vision (damage analysis + repair instructions) |
| **Speech-to-Text** | OpenAI Whisper API |
| **Text-to-Speech** | OpenAI TTS (`tts-1`, `nova` voice) |

---

## Project Structure

```
midas/
├── frontend/                        # React + Vite mobile web app
│   ├── index.html                   # Entry HTML (Google Fonts, PWA meta)
│   ├── vite.config.js               # Vite config (React + Tailwind plugins)
│   ├── package.json
│   ├── public/
│   │   └── manifest.json            # PWA manifest
│   └── src/
│       ├── main.jsx                 # React root
│       ├── App.jsx                  # Main app -- camera, AR overlay, session routing
│       ├── index.css                # Tailwind v4 imports + custom styles
│       ├── api/
│       │   └── midas.js             # Backend API client (all fetch calls)
│       ├── components/
│       │   ├── CameraFeed.jsx       # <video> element with camera stream
│       │   ├── AROverlayLayer.jsx   # Renders AR detection markers over camera
│       │   ├── ARMarker.jsx         # Individual AR bounding box with label
│       │   ├── ActiveSession.jsx    # Main session UI (buttons, steps, transcript)
│       │   ├── ControlBar.jsx       # Bottom bar (mic, torch, end session)
│       │   ├── TranscriptPanel.jsx  # Chat history + text input
│       │   ├── PermissionGate.jsx   # Camera permission request screen
│       │   ├── StartScreen.jsx      # "Start Session" landing screen
│       │   ├── SessionComplete.jsx  # Session summary screen
│       │   └── StepIndicator.jsx    # Repair step progress indicator
│       └── hooks/
│           ├── useCamera.js         # Camera stream, torch, front/back toggle
│           ├── useMicrophone.js     # MediaRecorder for hold-to-talk audio capture
│           └── useRepairSession.js  # Session state, chat, TTS, step tracking
│
├── backend/                         # Python backend (FastAPI + AI/CV)
│   ├── server.py                    # FastAPI server -- all endpoints
│   ├── __init__.py
│   └── utils/
│       ├── vision_advisor.py        # GPT-4o Vision wrapper (analyze + chat)
│       └── __init__.py
│
├── requirements.txt                 
├── .env                         
└── .gitignore
```

---

## Backend

### Endpoints

| Method | Endpoint | Description | Cost |
|--------|----------|-------------|------|
| `POST` | `/preview` | OpenCV preprocessing + YOLOv11 object detection. Returns base64 enhanced image + labeled bounding boxes. Called continuously by frontend. | Free |
| `POST` | `/chat` | Multi-turn conversation with GPT-4o Vision. Accepts message history + camera frame. Returns concise repair guidance. | $ |
| `POST` | `/analyze` | Single-shot structured analysis. Returns JSON with device, damage, severity, tools, steps. | $ |
| `POST` | `/transcribe` | Speech-to-text via OpenAI Whisper. Accepts audio blob (WebM/Opus). | $ |
| `POST` | `/tts` | Text-to-speech via OpenAI TTS. Returns MP3 audio bytes. | $ |
| `GET` | `/health` | Health check -- reports API key and YOLO model status. | Free |

### Processing Pipeline

```
Camera Frame --> OpenCV Preprocessing --> YOLOv11 Detection --> Frontend AR Overlay
                     |
                     |-- Resize (max 1280px)
                     |-- CLAHE contrast enhancement
                     '-- Sharpen filter

User Action --> Whisper STT --> GPT-4o Vision --> Stream Response --> OpenAI TTS --> Audio Playback
```

### Vision Advisor (`ml/utils/vision_advisor.py`)

Two modes of operation:

- **`analyze_image()`** -- Single-turn structured JSON analysis (device, damage, severity, tools, steps)
- **`chat()`** -- Multi-turn conversational repair guidance with system prompt enforcing:
  - Max 2 sentences per response
  - 1 numbered step at a time (`Step n: [action]`)
  - References to specific visible parts in the image
  - Safety warnings inline
  - Plain spoken English (read aloud via TTS)

---

## Frontend

### App Flow

```
PermissionGate --> StartScreen --> ActiveSession --> SessionComplete
   (camera)         (ready)          (active)          (complete)
```

### Key Hooks

| Hook | Purpose |
|------|---------|
| `useCamera` | Manages `getUserMedia`, torch control, front/back camera |
| `useMicrophone` | `MediaRecorder` for hold-to-talk, returns audio `Blob` |
| `useRepairSession` | Core session logic -- chat, TTS, step tracking |

### Session Actions

- **`scan()`** -- Hold-to-talk: Whisper STT --> GPT-4o chat --> TTS playback
- **`scanPhoto()`** -- Tap camera button: frame --> GPT-4o chat --> TTS
- **`completeStep()`** -- Mark current step done, request next step
- **`sendText()`** -- Type a message to chat with the AI

### AR Detection Loop

Runs continuously during active sessions (as fast as round-trips allow, no fixed interval):

1. Capture JPEG frame from `<video>` element
2. `POST` to `/preview` endpoint
3. Receive YOLOv11 detections (max 3, labeled with class name + confidence %)
4. Render as gold AR bounding boxes over the live camera feed

---

## Getting Started

### Prerequisites

- OpenAI API key

### Setup

```bash
# Clone
git clone <repo-url>
cd midas

# Environment
echo "OPENAI_API_KEY=sk-..." > .env

# Backend
pip install -r requirements.txt
python ml/server.py

# Frontend (new terminal)
cd frontend
npm install
echo "VITE_API_URL=http://localhost:8000" > .env
npm run dev
```

Frontend runs at `http://localhost:5173`, backend at `http://localhost:8000`.

### Accessing on Mobile

To test on your phone (same Wi-Fi network required), expose both servers on your local IP:

```bash
# Backend (already binds 0.0.0.0 by default)
python ml/server.py

# Frontend — expose on network
cd frontend
echo "VITE_API_URL=http://<your-local-ip>:8000" > .env
npm run dev -- --host
```

Find your local IP with `ipconfig getifaddr en0` (macOS) or `hostname -I` (Linux). Then open `http://<your-local-ip>:5173` on your phone's browser.

> **Note:** Camera access requires a secure context. `localhost` works automatically, but on a LAN IP you may need HTTPS. Use the single server mode below with a reverse proxy, or use a tunneling tool like `ngrok`.

> **Tip:** On mobile, the app includes a **flashlight toggle** button in the control bar for illuminating devices in low-light conditions (requires a device with torch support).

### Single Server Mode

Build the frontend and serve everything from FastAPI:

```bash
cd frontend && npm run build && cd ..
python ml/server.py
```
