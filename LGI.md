# LGI — Looking Glass Interface
## Executive Supervisor Client — Architectural Blueprint

| | |
|---|---|
| **Version** | 1.2 — Her Seeing Me (webcam perception: tracking, gestures, attention gate, spatial radar) |
| **Target host** | Alienware X17 — Windows 11, NVIDIA GPU (CUDA) |
| **Runtime model** | Native Python process (**not** containerized — requires mic, webcam, screen capture, GPU, and Qt GUI access) |
| **Gateway** | Polaris Mainframe — `http://192.168.50.51:8082` (LAN, IP-allowlisted) |
| **Codebase** | 10 source files (9 Python + requirements), 2,663 Python lines + this blueprint |

---

## Table of Contents

1. [Mission & System Context](#1-mission--system-context)
2. [Design Principles](#2-design-principles)
3. [File Map](#3-file-map)
4. [Runtime Topology](#4-runtime-topology)
5. [Event Flows](#5-event-flows)
6. [Module Reference](#6-module-reference)
7. [Gateway Contract](#7-gateway-contract)
8. [Configuration Reference](#8-configuration-reference)
9. [Operator Controls](#9-operator-controls)
10. [Deployment Runbook (X17)](#10-deployment-runbook-x17)
11. [Failure Modes & Degradation](#11-failure-modes--degradation)
12. [Security & Privacy](#12-security--privacy)
13. [Extension Roadmap](#13-extension-roadmap)

---

## 1. Mission & System Context

LGI is the operator's **floating HUD + voice cockpit** for the Polaris Executive
Supervisor system. It runs natively on the Alienware X17 beside the operator and
provides five capabilities:

* **Open-mic voice chat with Polaris** — say `"Polaris <question>"`; the reply is
  spoken back through local Kokoro TTS and mirrored on the HUD.
* **Desktop vision chat** — utterances that reference what is on screen
  ("Polaris, look at this layout") attach the latest dashcam frame (rolling
  DXcam capture) plus optional local OCR text to the chat payload; the gateway
  answers through its vision LLM ladder and the reply is spoken through Kokoro.
* **Webcam perception — "Her Seeing Me" (v1.2)** — a continuous webcam stack
  tracks people (YOLOv8n + ByteTrack), reads operator attention (MediaPipe
  FaceMesh head pose), and recognizes gestures (palm = mic mute toggle,
  fist = media play/pause). Camera-directed utterances ("Polaris, analyze
  this component") are answered by `polaris-ai:latest` on the live camera
  frame, and spatial telemetry (`radar_motion` + `vision_frame`) feeds the
  Mainframe's radar room — the X17 replaces MissPi as the sole spatial sensor.
* **Continuous desktop supervision** — every 30 s (and on demand) LGI gathers a
  desktop-context snapshot (active window, clipboard, screen, webcam) and POSTs
  it to the Polaris gateway for heuristic + LLM triage. A `SUPERVISOR_ALERT`
  verdict turns the SUPERVISOR LED red, raises an alert banner, and speaks the
  category + reason.
* **Ambient awareness with privacy** — all other speech is transcribed locally
  and shown as an ambient transcript line but is **never sent to the network**.

### System context

```text
+------------------------- Alienware X17 (Windows 11) -------------------------+
|                                                                             |
|  lgi.py  (LGIApp — Qt main thread + 100 ms drain QTimer)                    |
|    |                                                                        |
|    +-- hud_ui.py             frameless always-on-top HUD                   |
|    |                         (MIC/CAM/SUP/POL LEDs, exchange, alert banner)  |
|    +-- audio_stt.py          open-mic VAD + faster-whisper     [daemon]     |
|    +-- audio_tts.py          Kokoro TTS 24 kHz, sounddevice    [daemon]     |
|    +-- vision_supervisor.py  screen/webcam/clipboard audits    [daemon]     |
|    +-- screen_capture.py     rolling DXcam dashcam -> JPEGs   [dxcam]      |
|    +-- webcam_perception.py  tracking + gestures + attention   [daemon]     |
|    +-- local_vlm.py          webcam-analyze turns -> Ollama (b64 JPEG)      |
|    +-- gateway_client.py     thread-safe HTTP client (JSON only)            |
|    +-- heartbeat             GET status every 15 s            [daemon]     |
|    +-- global hotkeys        keyboard lib -> command queue     [daemon]     |
|                                                                             |
+----------------------------- text/JSON only | LAN --------------------------+
                                              v
                          Polaris Mainframe gateway :8082
                          POST /api/chat              -> Ollama LLM (vision: multimodal ladder)
                          POST /api/supervisor/audit -> triage + vault
                          GET  /api/supervisor/status -> liveness
```

## 2. Design Principles

1. **Qt is single-threaded.** Daemon workers (STT, TTS, vision, heartbeat, chat
   executor, hotkey hook) **never touch Qt widgets**. They push events onto
   queues; a 100 ms `QTimer` in the Qt main thread drains the queues and is the
   *only* code that mutates the HUD.
2. **Never raise across threads.** Every worker converts failures into result
   dicts and guarded callbacks. Exceptions never propagate into the Qt loop.
3. **Port 8082 carries text/JSON only** — never raw audio, never video streaming
   (northstar.md rule). Images travel as base64 JPEG *inside* JSON payloads.
4. **Degrade, never crash.** CUDA falls back to CPU (STT + TTS), missing sensors
   auto-disable, hotkey failure falls back to HUD buttons, gateway errors render
   as HUD error lines.
5. **Local-first voice privacy.** STT runs entirely locally; only wake-word
   utterances ever leave the machine (default mode).
6. **Playback must never stall the mic.** TTS runs on its own thread; optional
   ducking (`LGI_DUCK_WHILE_SPEAKING`) is OFF by default.
7. **Ops-terminal chrome.** Dark high-contrast theme per northstar rules;
   frameless, always-on-top, draggable overlay.

## 3. File Map

| File | Lines | Role |
|---|---:|---|
| `lgi.py` | 631 | Entry point + orchestrator: `LGIConfig`, `LGIApp`, queue drain, voice routing, vision/webcam turns, audit routing, gesture dispatch |
| `hud_ui.py` | 259 | Frameless always-on-top PyQt6 HUD (MIC/CAM/SUP/POL LEDs, exchange, alert banner, buttons) |
| `audio_stt.py` | 232 | Continuous open-mic listener: energy VAD + Silero VAD + faster-whisper transcription |
| `audio_tts.py` | 260 | Local Kokoro TTS worker (CUDA, 24 kHz, chunked, interruptible playback) |
| `screen_capture.py` | 177 | Rolling DXcam "dashcam" (1 FPS ring buffer, mss fallback) → on-demand JPEG + OCR |
| `webcam_perception.py` | 574 | Continuous webcam perception (v1.2): shared camera owner, FaceMesh attention, Hands gestures, YOLOv8n+ByteTrack tracking, socketio spatial reporter |
| `local_vlm.py` | 99 | Webcam-analyze VLM client (Ollama `polaris-ai:latest`, single-flight, never raises) |
| `vision_supervisor.py` | 244 | Periodic desktop-context audit loop (sensors → gateway verdict; attention-gated) |
| `gateway_client.py` | 187 | Thread-safe JSON HTTP client for the Polaris gateway (retry/backoff, never raises) |
| `requirements.txt` | 47 | Dependency manifest + Windows prerequisites |
| `LGI.md` | — | This blueprint |

## 4. Runtime Topology

### Thread inventory

| Thread | Owner | Responsibility |
|---|---|---|
| Qt main thread | `QApplication.exec()` | HUD mutation **only**, via the 100 ms drain timer |
| `lgi-stt` | `MicListener` (daemon) | PyAudio capture, VAD segmentation, Whisper transcription |
| `lgi-tts` | `KokoroTTS` (daemon) | Kokoro pipeline generation + sounddevice playback |
| `lgi-vision` | `VisionSupervisor` (daemon) | Sensor gathering + audit POST every `interval_s` |
| dxcam capture thread (library-owned) | `ScreenRoller` | Rolling desktop capture at `vision_fps` (default 1 FPS) via dxcam's internal DXGI thread — LGI owns no capture thread (§6.9); JPEG served on demand |
| `lgi-perception` | `WebcamPerception` (daemon) | Camera capture @ `webcam_fps` (15 FPS), FaceMesh/Hands/YOLO pipeline, gesture + telemetry emission |
| `lgi-radar` | `SpatialReporter` (daemon) | socketio client → Burst Receiver `:8083` (`vision_connect`, `vision_frame` @ 2 Hz, `radar_motion` @ 1 Hz; auto-reconnect) |
| `lgi-heartbeat` | `LGIApp._heartbeat` (daemon) | `GET /api/supervisor/status` every 15 s → POLARIS LED |
| `lgi-chat-N` | `ThreadPoolExecutor(max_workers=2)` | Blocking `POST /api/chat` calls |
| hotkey hook | `keyboard` lib | Global hotkeys → pushes strings into `command_q` |

### Queue inventory (worker → Qt main thread)

| Queue | Producer → Consumer | Payload |
|---|---|---|
| `status_q` | all workers → `_drain` | `(target, state)` LED updates (`mic`/`cam`/`supervisor`/`polaris`); `("tts", state)` → optional ducking |
| `command_q` | hotkeys / HUD buttons / gestures → `_dispatch_command` | `"mic" \| "audit" \| "stop_voice" \| "media"` |
| `stt_q` | `MicListener` → `_dispatch_utterance` | final utterance text |
| `chat_q` | chat executor → `_drain` | `("chat" \| "webcam", user_text, result)` |
| `supervisor_q` | `VisionSupervisor` → `_handle_audit` | audit result dict |

**The drain timer** (`QTimer`, 100 ms) is the *only* place worker output reaches
the HUD. Each queue is drained with `get_nowait()` until `Empty` — bounded ~100 ms
UI latency, zero cross-thread Qt calls.

## 5. Event Flows

### 5.1 Voice loop (wake-word mode — default)

```text
 mic ──► energy VAD ──► utterance ──► faster-whisper ──► stt_q
                                                            │
                                              _dispatch_utterance
      ┌─────────────────────┬─────────────────────────────┼────────────────────┐
      ▼                     ▼                             ▼                    ▼
 "stop talking" /       "polaris …" /              bare "polaris"         anything else
 "be quiet" / …         "hey polaris …"                                  
 tts.stop()             strip wake word ──► chat executor   tts.speak(          hud.set_ambient()
 (local, no network)      client.chat(text)  │ chat_q       "Polaris standing    (display only,
                                │            ▼              by.")                 never sent)
                                ▼     _handle_chat_result
                              hud.set_exchange(you, polaris)
                              └─► tts.speak(response)   [if speak_responses]
```

`VOICE_STOP_COMMANDS` (matched on the normalized utterance):
`stop talking`, `be quiet`, `quiet please`, `silence`, `stop voice`, `shut up`.

### 5.2 Audit loop

```text
 every 30 s  (or Ctrl+Shift+S / HUD button ──► vision.trigger(), honored ≤250 ms)
      │
      ▼
 _gather()  — each sensor independently guarded:
   active window title + app     (pygetwindow, title ≤200 chars)
   clipboard snippet             (pyperclip, change-gated, ≤4000 chars)
   screen keywords               (≤12 tokens parsed from title)
   screen JPEG base64, 1280px q60 (mss primary monitor, optional)
   webcam JPEG base64,  640px q50 (shared perception frame when the stack runs;
                          legacy one-shot CAP_DSHOW when LGI_WEBCAM_TRACK=0)
      │
      ▼  POST /api/supervisor/audit   (JSON over the wire)
  gateway triage (heuristic + LLM) ──► verdict
      │ result_callback ──► supervisor_q
      ▼
 _handle_audit:
   SUPERVISOR_ALERT ─► red SUP LED + alert banner + "ALERT <category>" line +
                       spoken "Supervisor alert. Category: <category>. <reason>" +
                       epoch-guarded 60 s auto-clear (QTimer.singleShot)
   LOG_ONLY         ─► green LED, "logged (<category>)"
   NO_ACTION        ─► green LED, "clear"
   ok=False         ─► "audit failed — <error>", idle LED
```

### 5.3 Liveness heartbeat

```text
 every 15 s: client.supervisor_status()
   ok   ──► POLARIS LED online  (green)
   fail ──► POLARIS LED offline (red)
```
### 5.4 Desktop-vision turn (Polaris Desktop Vision)

```text
 utterance matches VISION_INTENT_RE (after wake-word strip)
      │
      ▼
 _gather_vision_context()  — guarded, runs on the chat executor thread:
   dashcam JPEG base64   ScreenRoller (dxcam rolling buffer @ vision_fps,
                         mss one-shot fallback; 1920px q70 default)
   OCR text              pytesseract, ≤4000 chars, auto-disabled if missing
   active window title   pygetwindow, ≤200 chars
      │
      ▼  POST /api/chat {"prompt", "message", "image_base64", "ocr_text", "active_window"}
 gateway vision LLM ladder: polaris-ai:latest → mistral-large-3:675b-cloud
      │ chat_q ← ("chat", text, result)      [vision_chat_timeout 120 s]
      ▼
 _handle_chat_result ──► HUD exchange + Kokoro speak (same as any chat turn)
```

No frame available degrades to a text-only turn (logged). `LGI_VISION_CHAT=0`
disables vision routing entirely; `LGI_OCR=0` strips the OCR context.

### 5.5 Webcam perception loop (v1.2 "Her Seeing Me")

```text
 lgi-perception thread @ webcam_fps (15 FPS) — single camera owner:
   read frame ──► rolling buffer (latest_jpeg_b64 feeds audits + VLM turns)
              ├─► FaceMesh (~5 Hz) ──► head-pose proxy ──► user_attentive
              │       └─► False ──► VisionSupervisor skips the audit (gate on,
              │                    fail-open when unknown; LGI_ATTENTION_GATE=0)
              ├─► Hands (~7.5 Hz) ──► palm hold ~1.5 s ──► command_q "mic"
              │                    └► fist hold ~0.6 s ─► command_q "media"
              └─► YOLOv8n + ByteTrack (15 Hz) ──► persistent person IDs

 lgi-radar thread (SpatialReporter, socketio → Burst Receiver :8083):
   vision_connect on every (re)connect ──► vision_frame @ 2 Hz (detections)
   radar_motion @ 1 Hz {x, y, z, velocity, people_count} — real coordinates
   pass through the receiver untouched (no simulated jitter)

 Webcam-analyze turn (WEBCAM_ANALYZE_RE, checked BEFORE screen vision):
   "Polaris, analyze this component"
      ──► latest perception frame (640px q50; legacy one-shot if stack off)
      ──► POST Ollama /api/generate {model: polaris-ai:latest, images: [b64]}
      ──► chat_q ("webcam", text, result) ──► HUD exchange + Kokoro speak
```

## 6. Module Reference

### 6.1 `gateway_client.py` — `PolarisClient`

Thread-safe JSON-over-HTTP client. One `requests.Session` shared behind a
`threading.Lock` for keep-alive reuse. **Never raises** — every failure becomes
`{"ok": False, "error": ...}`.

```python
PolarisClient(base_url="http://192.168.50.51:8082",
              chat_timeout=65.0,   # gateway forwards to Ollama (60 s upstream)
              vision_chat_timeout=120.0,  # desktop-vision turns (multimodal pass)
              audit_timeout=30.0,  # includes optional LLM triage
              status_timeout=5.0,  # lightweight liveness probe
              max_retries=2, backoff=1.5, log=print)
```

* `chat(text, image_b64=None, ocr_text=None, active_window=None)` →
  `{"ok": True, "response": ...}` — POSTs `{"prompt": text, "message": text}`
  (both keys; `message` is the legacy alias); desktop-vision turns add
  `image_base64` / `ocr_text` / `active_window` and use the 120 s
  `vision_chat_timeout`. Empty response bodies are converted into errors.
* `supervisor_audit(payload)` → `{"ok": True, "verdict": "SUPERVISOR_ALERT" |
  "LOG_ONLY" | "NO_ACTION", "data": {...}}`; a missing verdict becomes an error.
* `supervisor_status()` → liveness probe for the POLARIS LED.
* `online` property — best-effort connectivity flag, updated on every attempt.
* Retry policy: `ConnectionError` / `Timeout` retried with linear backoff
  (`backoff × (attempt + 1)`); protocol-level `RequestException` aborts.
* Gateway JSON error bodies (e.g. `404 {"error": "Unknown user"}`) map to
  `ok=False` carrying the gateway's error string.

### 6.2 `audio_stt.py` — `MicListener(thread)`

Continuous open-mic listener: **16 kHz mono int16** PyAudio capture in 30 ms
chunks, normalised to float32 for an **energy-based VAD** with an adaptive noise
floor (`threshold = max(3 × floor, 0.004)`; floor tracks idle energy as an EMA).
Utterance segmentation: 200 ms pre-roll, 800 ms silence tail, ≥300 ms voiced,
30 s hard cap. Transcription: faster-whisper **large-v3-turbo**
(`deepdml/faster-whisper-large-v3-turbo-ct2`), CUDA/float16 with automatic
**CPU/int8 fallback**, `beam_size=1`, final text only (no partials). A second
**Silero VAD stage** (`vad_filter=True`, bundled with faster-whisper via
onnxruntime, `min_silence_duration_ms=500`) strips residual non-speech before
inference, and `condition_on_previous_text=False` prevents hallucinated
continuations on isolated utterances.

* `pause()` **discards frames without closing the audio stream** — resume is
  instantaneous and the capture graph is never torn down.
* Callbacks (fire on the listener thread): `final_text_callback(text)`,
  `status_callback(state ∈ {listening, muted, processing, error})`.
* Missing faster-whisper or PyAudio disables the listener gracefully (MIC LED
  red); the rest of LGI keeps running.

### 6.3 `audio_tts.py` — `KokoroTTS`

Local Kokoro pipeline (`lang_code='a'`, default voice `af_heart`, speed 1.2×,
24 kHz) with chunked generation and `sounddevice` playback on the `lgi-tts`
daemon thread. **Non-blocking `speak(text)`**; a generation counter makes
`stop()` interrupt playback mid-chunk within a ~20 ms poll window.

* The pipeline initialises **lazily on first speak** — `start()` only launches
  the worker, so LGI boots instantly; CUDA init failure degrades to CPU.
* eSpeak NG env vars (`PHONEMIZER_ESPEAK_LIBRARY`, `PHONEMIZER_ESPEAK_PATH`,
  `ESPEAK_DATA_PATH` → `C:\Program Files\eSpeak NG\...`) are set **before** the
  kokoro import, on Windows only.
* Speech cleaning: fenced code blocks → *"code block omitted"*, inline code
  unwrapped, URLs → *"link"*, markdown `*_#>|` characters stripped, hard cap
  1,500 chars.
* Status callback states: `ready` / `speaking` / `idle` / `error`.
* API: `start()`, `speak(text)`, `stop()`, `shutdown()`, `is_speaking`.
### 6.4 `vision_supervisor.py` — `VisionSupervisor(thread)`

Fixed-interval audit loop (default 30 s, floor 5 s) that sleeps in 250 ms
slices so `trigger()` (Ctrl+Shift+S / HUD button) forces a cycle within 250 ms.
Every sensor is independently guarded — an `ImportError` permanently disables
that sensor; runtime errors simply yield an empty value.

| Sensor | Source | Notes |
|---|---|---|
| `active_window` | pygetwindow | title (≤200 chars) + `app` derived from `" - "` split |
| `clipboard_snippet` | pyperclip | change-gated (stale clips sent as ""), ≤4000 chars |
| `screen_keywords` | title regex `[a-z][a-z0-9_-]{2,}` | ≤12 unique tokens |
| `screen_image` | mss + OpenCV | primary monitor → JPEG 1280px q60 → base64 |
| `webcam_image` | shared `WebcamPerception` frame (fallback: OpenCV `CAP_DSHOW` one-shot) | perception stack holds the camera open (LED on); legacy per-cycle open/release only when `LGI_WEBCAM_TRACK=0` → JPEG 640px q50 |

Audit payload (`screen_image` / `webcam_image` are currently **ignored by the
gateway** and shipped for forward-compatible vault vision ingestion):

```json
{ "active_window": "...", "clipboard_snippet": "...",
  "screen_keywords": ["..."], "user": "rich", "app": "...",
  "llm_triage": true, "screen_image": "<b64 jpeg>", "webcam_image": "<b64 jpeg>" }
```

### 6.5 `hud_ui.py` — `HUDWindow(QDialog)`

Frameless (`FramelessWindowHint | WindowStaysOnTopHint | Tool`), dark QSS
ops-terminal theme (cyan-on-slate, `rgba(8,12,18,214)` panel), opacity 0.94,
auto-positioned top-right with 24 px margins, **drag anywhere** to reposition.

* **Four `StatusDot` LEDs** — MIC: `listening` green / `muted` red /
  `processing` amber · CAM: `tracking` green / `degraded` amber / `off` gray ·
  SUPERVISOR: `idle` gray / `scanning` blue / `ok` green /
  `alert` red · POLARIS: `online` green / `offline` red / `unknown` gray.
* **Lines** — `You:` / `Polaris:` exchange, ambient transcript, `SUP:` verdict
  line, red alert banner (`⚠ CATEGORY — reason`).
* **Buttons** — 🎤 Mic On/Muted · Force Audit · Stop Voice · Quit. Handlers
  (`on_mic_toggle`, `on_force_audit`, `on_stop_voice`, `on_quit`) are injected
  by `lgi.py` after construction — the HUD has zero business logic.
* Mutators (Qt main thread only): `set_led(target, state)`, `set_exchange(you,
  polaris)`, `set_ambient(text)`, `set_supervisor_line(text)`,
  `show_alert(category, reason)`, `hide_alert()`, `set_mic_button(listening)`.
* All text is whitespace-normalised and elided (exchange 200/300, SUP 90,
  banner 140 chars).

### 6.6 `lgi.py` — `LGIApp` (orchestrator) + `LGIConfig`

`LGIConfig.from_env()` builds the configuration from environment overrides
(§8). `LGIApp(QObject)` wires the engines together:

* `__init__` — PolarisClient → HUDWindow (handlers injected) → KokoroTTS →
  MicListener → WebcamPerception + LocalVLM → VisionSupervisor (receives the
  shared `webcam_provider` + `attention_gate`) → chat
  `ThreadPoolExecutor(max_workers=2)` → 100 ms drain `QTimer` → alert epoch
  counter + user-mute flag.
* `start()` — POLARIS LED `unknown`; TTS worker starts first (pipeline warms
  off-thread); then listener, dashcam, perception stack (camera LED ON),
  vision, heartbeat thread, hotkeys, drain timer; `hud.show()`.
* `shutdown()` — stop event, drain timer off, all engines stopped (perception
  shutdown releases the camera → LED off), TTS worker terminated, executor
  abandoned (`wait=False`).
* `_dispatch_utterance` — routing per §5.1 with camera-first intent checks
  (§5.5): `WEBCAM_ANALYZE_RE` beats `VISION_INTENT_RE` ("what do you see"
  alone remains a desktop-vision phrase); with `wake_word_required=False`
  (open mode) **every** utterance is sent to chat or analysis.
* `_handle_audit` — verdict routing per §5.2; reason extracted from
  `data.triage.llm.reason` with fallback to `data.triage.heuristic.reason`;
  the 60 s alert auto-clear is **epoch-guarded** so an older timer can never
  clear a newer alert.
* `_on_tts_state` — optional ducking: `speaking` pauses STT; `idle` / `ready` /
  `error` resume it unless the user muted the mic (`_mic_muted_by_user`).
* `main()` — `QApplication`, `lgi.start()`, `app.exec()`, `finally` shutdown.
### 6.7 `webcam_perception.py` — `WebcamPerception(thread)` + `SpatialReporter`

The v1.2 "Her Seeing Me" stack — the X17 replaces MissPi as the sole spatial
sensor. One daemon thread owns `cv2.VideoCapture(0, CAP_DSHOW)` for the whole
LG session (hardware camera LED ON while tracking; released on shutdown).

* **Pipelines** — each lazy-imported and independently guarded
  (degrade-never-crash): MediaPipe FaceMesh (~5 Hz head-pose proxy →
  `user_attentive`), MediaPipe Hands (~7.5 Hz palm/fist dwell gestures),
  YOLOv8n + ByteTrack (`.track(persist=True)` → persistent person IDs,
  normalized x/y, delta velocity, bbox-height depth proxy). A missing library
  disables its stage only.
* **Gestures** — palm held ~1.5 s pushes `"mic"` (mute toggle), a held fist
  pushes `"media"` (OS media key via the `keyboard` lib). One fire per dwell +
  3 s cooldown; `LGI_GESTURES=0` disables.
* **`SpatialReporter`** — python-socketio client to the Burst Receiver
  (`LGI_RADAR_URL`, default `:8083`): `vision_connect` on every (re)connect,
  `vision_frame` @ 2 Hz (detections + narration), `radar_motion` @ 1 Hz with
  explicit `x/y/z/velocity` + `people_count` — real coordinates pass through
  the receiver untouched (coordinate-less legacy payloads keep the synthetic
  path). Link down → telemetry dropped, 10 s reconnect loop, local sensing
  unaffected.
* **Shared state** (lock-guarded, any thread): `latest_jpeg_b64()` (feeds
  supervisor audits + VLM turns), `user_attentive` (`Optional[bool]`, None =
  fail-open), `people_count`, `mode` (`off`/`tracking`/`degraded` → CAM LED).
* `one_shot_jpeg_b64()` module helper = the legacy open→grab→release path used
  by webcam-analyze turns when the stack is off (`LGI_WEBCAM_TRACK=0`).

### 6.8 `local_vlm.py` — `LocalVLM` (Ollama webcam-analyze client)

Thin, defensive Ollama `/api/generate` client answering camera-directed utterances
with `polaris-ai:latest` — the same vision model (and request shape) the
gateway already uses for its vision ladder (Modelfile wrap of glm-5.3-flash:cloud).

* One generate call carries the short spoken-style prompt plus the base64
  JPEG in the top-level `images` field; `num_predict` 220, temperature 0.3.
* Single-flight (one analysis at a time — concurrent requests fail fast),
  never raises; failures return `{"ok": False, "error": ...}` (no frame /
  busy / model-missing / timeout / Ollama offline).
* Knobs: `LGI_VLM_URL` (default the Mainframe's Ollama `:11434`),
  `LGI_VLM_MODEL` (default `polaris-ai:latest`), `LGI_VLM_TIMEOUT` (90 s).
  A fully-local VLM later = point `LGI_VLM_URL` at the X17's own Ollama —
  one env change, no code edit.

### 6.9 `screen_capture.py` — `ScreenRoller`

Rolling desktop "dashcam" behind voice-triggered desktop-vision turns (§5.4).
dxcam (Windows Desktop Duplication / DXGI) captures the primary monitor at
`vision_fps` (default 1 FPS) in `video_mode` into a small bounded ring buffer
(8 frames) — a ready frame always exists for on-demand vision queries, with
zero CUDA contention (capture rides the GPU copy engine, never the compute
queues whisper-turbo / Kokoro use).

* **No LGI-owned thread** — dxcam runs its own internal capture thread;
  `ScreenRoller` only owns the handle and a lock-guarded frame accessor
  (`latest_frame()` callable from any worker thread).
* `latest_jpeg_b64(max_width=1920, quality=70)` → base64 JPEG of the newest
  frame (the §5.4 vision payload); `ocr_text()` → local Tesseract OCR of the
  same frame (≤4,000 chars, auto-disabled permanently on first `ImportError`).
* Degrade, never crash: dxcam unavailable (non-Windows dev box, DXGI blocked,
  driver hiccup) → mss one-shot grabs per request — vision turns keep working
  at slightly higher latency; total capture failure → text-only turns.
* `mode` property: `idle` / `dxcam` / `mss` / `disabled`.

## 7. Gateway Contract

Confirmed against `polaris_continuous_learning_gateway.py` /
`supervisor_routes.py`. All calls are JSON over HTTP on **port 8082
(text/JSON only — never raw audio)**.

| Endpoint | Method | Request | Response |
|---|---|---|---|
| `/api/chat` | POST | `{"prompt": "..."}` (`"message"` legacy alias) + optional `image_base64` / `ocr_text` / `active_window` (vision turns) | Ollama JSON `{"response": "<text>"}` (+ `vision: true`, `vision_model` on vision turns) |
| `/api/supervisor/audit` | POST | snapshot payload (§6.4) | `{"status":"success", "verdict":"SUPERVISOR_ALERT"\|"LOG_ONLY"\|"NO_ACTION", "category":"...", "vault_persisted":bool, "triage":{"heuristic":{...},"llm":{...}}, "timestamp":...}` |
| `/api/supervisor/status` | GET | — | lightweight liveness (POLARIS LED heartbeat) |

**Gateway error semantics** — `404 {"error": "Unknown user"}` (calling IP not
allowlisted), `400 {"error": "Missing prompt or message field"}`, `503` Ollama
offline. All surface through `PolarisClient` as `{"ok": False, "error": ...}`.

**Access control** — the gateway resolves the caller by IP; the X17
(`192.168.50.227`) is allowlisted (already configured). Audit payloads
additionally carry the explicit operator identity via the `user` field.

## 8. Configuration Reference

All settings have safe defaults; every env var is read once at launch by
`LGIConfig.from_env()`.

| Env var | Default | Meaning |
|---|---|---|
| `LGI_POLARIS_URL` | `http://192.168.50.51:8082` | Polaris gateway base URL |
| `LGI_OPERATOR_USER` | `rich` | operator identity in audit payloads |
| `LGI_STT_MODEL` | `deepdml/faster-whisper-large-v3-turbo-ct2` | faster-whisper model (HF repo id or size name) |
| `LGI_STT_DEVICE` | `cuda` | `cuda` with auto-fallback to `cpu` (int8) |
| `LGI_STT_COMPUTE` | `float16` | CTranslate2 compute type on the primary device (`int8` halves VRAM) |
| `LGI_TTS_DEVICE` | `cuda` | Kokoro pipeline device, auto CPU fallback |
| `LGI_WAKE_WORD` | `polaris` | wake word (`hey <word>` also accepted) |
| `LGI_WAKE_REQUIRED` | `true` | `false` → **open mode**: every utterance is sent to chat |
| `LGI_AUDIT_INTERVAL` | `30` | seconds between supervisor audits (floor 5) |
| `LGI_HEARTBEAT` | `15` | seconds between POLARIS liveness polls |
| `LGI_WEBCAM` | `true` | include webcam JPEG in audit payloads |
| `LGI_SCREENSHOT` | `true` | include screen JPEG in audit payloads |
| `LGI_VISION_CHAT` | `true` | desktop-vision turns (dashcam frame attached to visual utterances) |
| `LGI_VISION_FPS` | `1` | ScreenRoller dashcam capture rate (frames/s) |
| `LGI_VISION_WIDTH` | `1920` | max JPEG width sent on vision turns |
| `LGI_VISION_QUALITY` | `70` | JPEG quality sent on vision turns |
| `LGI_OCR` | `true` | attach local Tesseract OCR text to vision turns |
| `LGI_DUCK_WHILE_SPEAKING` | `false` | pause mic while Polaris speaks (off = playback never stalls the mic) |
| `LGI_WEBCAM_TRACK` | `true` | continuous webcam perception stack — camera LED ON while LGI runs; `false` = legacy per-audit open/release capture |
| `LGI_GESTURES` | `true` | palm hold ~1.5 s = mic toggle · fist hold = media play/pause |
| `LGI_ATTENTION_GATE` | `true` | supervisor audits skipped while the operator is away (fail-open; `false` = gate off) |
| `LGI_RADAR` | `true` | spatial telemetry (`radar_motion` / `vision_frame`) to the Burst Receiver |
| `LGI_RADAR_URL` | `http://192.168.50.51:8083` | Burst Receiver socketio endpoint |
| `LGI_WEBCAM_FPS` | `15` | perception capture rate (frames/s) |
| `LGI_VLM` | `true` | webcam-analyze turns enabled |
| `LGI_VLM_URL` | `http://192.168.50.51:11434` | Ollama base URL for webcam-analyze passes |
| `LGI_VLM_MODEL` | `polaris-ai:latest` | vision model for webcam-analyze turns |
| `LGI_VLM_TIMEOUT` | `90` | seconds per webcam-analyze vision pass |

Non-env config (edit the `LGIConfig` defaults): `tts_voice` (`af_heart`),
`tts_speed` (1.2), `hud_opacity` (0.94), `speak_responses` (True),
`hotkey_audit` (`ctrl+shift+s`), `hotkey_mic` (`ctrl+shift+m`),
`alert_led_hold_s` (60.0).

First launch after the turbo-model upgrade downloads ~1.6 GB of CT2 weights
from HuggingFace into the user cache (`%USERPROFILE%\.cache\huggingface\hub`);
subsequent launches load from disk with no network access.

## 9. Operator Controls

| Control | Action |
|---|---|
| Say `"Polaris <question>"` | Send to Polaris chat; reply displayed + spoken |
| Say bare `"Polaris"` | Spoken *"Polaris standing by."* (attention ping) |
| Say `"Polaris look at this layout"` / *"what's wrong with this CSS"* | **Vision turn** — latest dashcam JPEG + OCR text + active window attached; reply spoken through Kokoro |
| Say `"Polaris, analyze this component"` / *"what am I holding"* / *"who's there"* | **Webcam-analyze turn** — live camera frame → Ollama `polaris-ai:latest` → short spoken description |
| Show an open palm to the camera ~1.5 s | Mic mute toggle (same as `Ctrl+Shift+M`) |
| Show a held fist to the camera | Media play/pause |
| Say `stop talking` / `be quiet` / `quiet please` / `silence` / `stop voice` / `shut up` | Silence TTS instantly — evaluated locally, no network |
| `Ctrl+Shift+S` | Force an immediate supervisor audit |
| `Ctrl+Shift+M` | Toggle mic (STT pause/resume; capture stream stays open) |
| HUD 🎤 Mic On/Muted | Same as `Ctrl+Shift+M` |
| HUD Force Audit / Stop Voice / Quit | Force audit · silence TTS · close LGI |
| Drag the HUD | Reposition the overlay anywhere on screen |
## 10. Deployment Runbook (X17)

LGI runs **natively on Windows** — it is not a compose service and needs no
container rebuild: mic, webcam, screen capture, GPU, and Qt GUI all require
native desktop access.

1. **Sync** the `LGI/` folder to the X17 (e.g. `scp -r LGI rich@192.168.50.227:C:/LGI`).
2. **eSpeak NG** — install from
   https://github.com/espeak-ng/espeak-ng/releases to the default location
   `C:\Program Files\eSpeak NG\` (required by Kokoro's phonemizer; LGI sets the
   env vars automatically at import).
3. **CUDA torch** — `pip install torch --index-url https://download.pytorch.org/whl/cu121`
4. **PyAudio** — `pip install pyaudio`; if the wheel fails to build:
   `pip install pipwin` then `pipwin install pyaudio`.
5. **Dependencies** — `pip install -r requirements.txt` (now includes `dxcam`
   for the vision dashcam; for OCR context also install the Tesseract OCR
   engine — UB-Mannheim build, added to PATH, see requirements.txt). v1.2 adds
   `mediapipe`, `ultralytics` (YOLOv8n weights auto-download on first run),
   and `python-socketio[client]` — the webcam-analyze VLM rides the
   Mainframe's Ollama over plain HTTP, no extra deps.
6. **Run** — `python lgi.py` (optionally set `LGI_*` env vars first).
7. **Verify** — HUD appears top-right; MIC LED green (`listening`); POLARIS LED
   green within ~15 s (heartbeat); `Ctrl+Shift+S` forces an audit; say
   *"Polaris, what's my AMM status"* for an end-to-end voice round trip;
   for desktop vision open the trading dashboard and say *"Polaris, tell me
   what data is missing from the top navigation bar"*.
   For webcam perception (v1.2) confirm the CAM LED is green, the radar room
   shows the `x17-webcam` device, and say *"Polaris, analyze this component"*
   for a spoken description of the camera view.

## 11. Failure Modes & Degradation

| Failure | Behavior |
|---|---|
| Gateway unreachable | chat/audit error lines (`⚠ gateway: …` / `audit failed`); POLARIS LED red; 2 retries with backoff |
| Ollama offline (503) | surfaced as an error line; LGI keeps running |
| IP not allowlisted (404 `Unknown user`) | error line — fix on the gateway side |
| No CUDA driver | Whisper → CPU/int8, Kokoro → CPU (logged); both still function |
| PyAudio missing | listener disabled (MIC LED red); chat/audits unaffected |
| faster-whisper missing | listener disabled; rest unaffected |
| mss / opencv missing | that image sensor auto-disables permanently; audits continue |
| Webcam busy / capture fails | empty `webcam_image` for that cycle only |
| Camera busy / unavailable (perception stack) | CAM LED "off", stack disabled; audits fall back to the legacy one-shot; LGI unaffected |
| mediapipe missing / init fails | attention + gestures disabled (logged); tracking + frame sharing continue — CAM "degraded" |
| ultralytics missing / YOLO init fails | person tracking + radar emission disabled (logged); camera + gestures + frame sharing continue — CAM "degraded" |
| Burst Receiver down | telemetry dropped silently, 10 s reconnect retry loop; local perception unaffected |
| VLM failure (Ollama offline / model missing / timeout) | HUD `⚠ webcam:` line + spoken "I could not analyze the camera view"; chat turns unaffected |
| Gesture misfire | one fire per dwell + 3 s cooldown bound false positives; `LGI_GESTURES=0` kills the feature |
| Operator away, attention gate on | audits skipped (SUP LED idle); explicit voice commands unaffected (fail-open) |
| Clipboard unavailable | empty snippet; pyperclip errors swallowed |
| `keyboard` lib unavailable | global hotkeys disabled (logged); HUD buttons still work |
| dxcam missing / DXGI blocked | ScreenRoller falls back to mss one-shot grabs (logged); vision turns still work |
| pytesseract / Tesseract missing | OCR context auto-disabled (logged); vision turns send the image without OCR text |
| Vision LLM offline / image rejected | gateway ladder falls back to `mistral-large-3:675b-cloud`; total failure → HUD error line, no crash |
| Kokoro pipeline init failure | TTS disabled (`error` state); HUD still shows chat replies |
| TTS playback error | chunk aborted; worker continues with the next request |

## 12. Security & Privacy

* **Wake-word gate** — with `LGI_WAKE_REQUIRED=1` (default) only wake-word
  utterances are transmitted; ambient speech is transcribed locally for display
  and **never leaves the machine**.
* **Voice stop commands are evaluated locally** — silencing the voice never
  requires (or uses) the network.
* **Port 8082 carries text/JSON only** — no audio streams, no video streams
  (northstar.md). Images ride inside JSON payloads as compressed base64 JPEGs
  (screen 1280px q60, webcam 640px q50).
* **Sensor minimisation switches** — `LGI_SCREENSHOT=0` / `LGI_WEBCAM=0` strip
  images from audit payloads entirely.
* **Vision turns are wake-word gated** — screenshots are captured only on
  voice demand (`VISION_INTENT_RE`) and leave the machine only on routed chat
  turns; the rolling dashcam buffer itself never leaves the X17.
* **Webcam light honesty (v1.2)** — the perception stack holds the camera
  open while LGI runs, so the hardware LED is ON whenever tracking is
  enabled. `LGI_WEBCAM_TRACK=0` reverts to the legacy behavior: the camera
  is opened per audit cycle and released immediately, LED off between audits.
* **Webcam-analyze frames** — camera-directed utterances send one 640px q50
  JPEG per turn to the configured Ollama endpoint (default the Mainframe's
  cloud-proxied `polaris-ai:latest`); the continuous perception stream never
  leaves the X17 as images — only coordinates/counts ride `radar_motion` /
  `vision_frame`. Point `LGI_VLM_URL` at a local Ollama to keep analyze
  frames fully on-device.
* **Clipboard change-gating** — stale clipboard contents are never re-sent.
* **Access control** — gateway allowlists the X17 IP; audit payloads carry the
  operator identity (`rich`); vault persistence decisions belong to the gateway.

## 13. Extension Roadmap

1. **Vault vision ingestion** — gateway-side consumption of `screen_image` /
   `webcam_image` (payload fields already shipped; currently ignored).
2. **Streaming TTS** — speak long Polaris replies sentence-by-sentence as they
   generate (today the executor delivers one complete reply).
3. **Alert actions** — wire `SUPERVISOR_ALERT` to optional operator prompts
   (Xaman push / trading interlock) via the Radar hand-off flow.
4. **Wake-word spotter upgrade** — replace text-prefix matching with a local
   keyword model (e.g. openWakeWord) to eliminate STT-only wake misses.
5. **Configurable voice commands** — externalise `VOICE_STOP_COMMANDS`; add
   spoken *"mute the mic"* / *"run an audit"* commands.
6. **HUD caption history** — collapsible transcript of recent exchanges.
7. **Fully-local VLM** — move webcam-analyze inference on-device (llava or
   Llama-3.2-Vision on the X17's own Ollama): one env change,
   `LGI_VLM_URL=http://localhost:11434`.

---

*Blueprint v1.0 written 2026-09-21 from the completed source in `LGI/`;
extended for v1.2 "Her Seeing Me" on 2026-09-27 — line counts, the 28-knob
config table, thread/queue inventory, module APIs, and payload contracts
re-verified against the code (webcam-analyze round trip live-tested against
`polaris-ai:latest` over Ollama `/api/generate`).*