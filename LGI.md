# LGI — Looking Glass Interface
## Executive Supervisor Client — Architectural Blueprint

**The North Star team — one intelligence, three stations.** Polaris is one intelligence embodied where she is needed: the Mainframe gateway is her brain and hands (`polaris-ai` persona, dispatcher, memory — strictly text-only), this X17 station running LGI is her senses and voice (webcam + desktop vision IN; Kokoro TTS + Faster-Whisper STT OUT), and FreeRoam on the phone is her field presence (on-device STT/TTS over WireGuard). The operator is the final authority; only the operator and **Cline** ever edit the workspace repo — LGI sees, hears, and speaks, but never writes. Charter: the top team section of `northstar.md`; the gateway contract LGI must live up to is §7.

| | |
|---|---|
| **Version** | 1.5.1 — Guarded Session (v1.5.1: single-instance veto — a second LGI launch takes a `Global\` named mutex before any GPU/audio init and refuses itself if a session is live; v1.5.0: session-marker guarded-watchdog protocol; v1.4.2: device-gated voice — the `/api/chat` device tag routes `chat_tool_report` spoken tool reports to `lgi-hud-x17` only, other devices stay silent, + ⚙ CHAT_TOOL one-row-per-call in ▼ TACTICAL OPERATIONS; v1.4.1: ⚙ TRANSPARENCY slider fades the ENTIRE HUD via whole-window opacity; v1.4 Phase 3: gateway feed mirror — latest-5-per-side rolling window, TASK FEED + read-only MEMORY VAULT ops mirror, ⋯ HUD settings kebab; builds on v1.2 "Her Seeing Me" + v1.3 talk window) |
| **Target host** | Alienware X17 — Windows 11, NVIDIA GPU (CUDA) |
| **Runtime model** | Native Python process (**not** containerized — requires mic, webcam, screen capture, GPU, and Qt GUI access) |
| **Gateway** | Polaris Mainframe — `http://192.168.50.51:8082` (LAN, IP-allowlisted) |
| **Codebase** | 13 Python files + requirements.txt — 4,591 Python lines (incl. Phase D probes; verified 2026-10-04) + `crash_query.ps1` (18 lines), `watchdog.ps1` (102 lines, v2.1 guarded) and `crash_analyze.ps1` (317 lines, v1.0 postmortem driver) diagnostics + this blueprint |

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

* **Open-mic voice chat with Polaris** — say `"Polaris <question>"` (punctuation
  right after the name is tolerated); the reply is spoken back through local
  Kokoro TTS and mirrored on the HUD. Each successful exchange opens a wake-free
  conversation talk window (`LGI_TALK_WINDOW`, default 180 s) — see §5.1.
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
  verdict turns the SUPERVISOR LED red, raises an alert banner, and (🔊
  Advisories toggle on) speaks the category + reason (🔇 silent default).
* **Always-listening feed mirror (v1.4)** — LGI is a full Looking-Glass peer:
  ONE persistent Socket.IO connection joins the operator's identity room
  (`chat_history` 40-message seed, `chat_message` finals, `chat_stream`
  live deltas — push, never poll, unlimited reconnect + seed-on-return) and
  the `status` room (task feed). The HUD renders a rolling **latest 5 per
  side** window under the current exchange, mirrors the dashboard's bottom
  half (TASK FEED + read-only MEMORY VAULT), and a ⋯ kebab in the header
  opens a settings pull-down (HUD font size + panel transparency, persisted
  across restarts). Other-device traffic renders visually only — never TTS.
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
| `lgi.py` | 1105 | Entry point + orchestrator: `LGIConfig`, `LGIApp`, queue drain, voice routing (punctuation-tolerant wake match + v1.3 conversation talk window + v1.3.1 `[voice]` routing trace), vision/webcam turns, audit routing (+ 🔇/🔇 advisory voice toggle), gesture dispatch (v1.3.2 source-tagged mic commands), Qt liveness tick + pythonw console guard & `sys.excepthook` (§6.6); v1.4: feed/vault queue drains, REST seed fallback poll, vault executor fan-out; v1.5.0: session marker write/clear; v1.5.1: `_acquire_instance_lock()` `Global\LGI_X17_SingleInstance` second-instance veto in `main()` before GPU/audio init |
| `hud_ui.py` | 791 | Frameless always-on-top PyQt6 HUD (MIC/CAM/SUP/POL/FEED LEDs, exchange, alert banner, buttons incl. 🔊/🔇 Advisories); v1.4: parameterized `build_hud_qss()` (font scale + fixed glass alphas), ⚙ settings pull-down (⋯ kebab, sliders + QSettings persistence), ⌁ last-exchanges rolling window, ▼ TACTICAL OPERATIONS mirror (task feed browser + read-only vault page), persisted geometry; v1.4.1: TRANSPARENCY = whole-window `setWindowOpacity` — the slider fades the entire HUD uniformly |
| `audio_stt.py` | 275 | Continuous open-mic listener: energy VAD + Silero VAD + faster-whisper transcription + v1.3.1 stage traces (device open / onset / transcribe duration / empty) + v1.3.2 degenerate-transcript junk filter |
| `audio_tts.py` | 303 | Local Kokoro TTS worker (CUDA, 24 kHz, chunked, interruptible playback) + `_trace` diagnostic tee w/ 1 MB cap (§6.3) |
| `screen_capture.py` | 177 | Rolling DXcam "dashcam" (1 FPS ring buffer, mss fallback) → on-demand JPEG + OCR |
| `webcam_perception.py` | 588 | Continuous webcam perception (v1.2): shared camera owner, FaceMesh attention, Hands gestures (v1.3.2 hardened dwell: continuous hold + cooldown that clears dwell), YOLOv8n+ByteTrack tracking, socketio spatial reporter |
| `local_vlm.py` | 99 | Webcam-analyze VLM client (Ollama `polaris-ai:latest`, single-flight, never raises) |
| `vision_supervisor.py` | 244 | Periodic desktop-context audit loop (sensors → gateway verdict; attention-gated) |
| `gateway_client.py` | 255 | Thread-safe JSON HTTP client for the Polaris gateway (retry/backoff, never raises); v1.4: read-only Memory Vault REST (stats/list/doc/query) + paired-history fallback seed |
| `gateway_feed.py` | 428 | v1.4 always-listening feed (§6.10): ONE Socket.IO client — gateway identity room (chat seed/finals/stream deltas) + `status` task-feed room, unlimited reconnect + seed-on-return — and the pure-logic `ConversationStore` (latest-5-per-side window, v1.0.4 dedupe, stream exactly-once promotion) |
| `dep_probe.py` | 58 | Dependency self-check (runbook step 10) — imports the full declared stack, prints versions, exit 1 on hard failure |
| `tts_probe.py` | 84 | Phase D diagnostic: kokoro import + synthesis probe → wav/log in `%TEMP%` (kept after the Sep 30 cleanup) |
| `sd_play_probe.py` | 41 | Phase D diagnostic: sounddevice device enumeration + one-shot tone play |
| `crash_query.ps1` | 18 | Phase D diagnostic: last pythonw Application-Error + WER crash events with faulting module (§11 watchlist) |
| `watchdog.ps1` | 102 | **v2.1 GUARDED relauncher** (fired via `watchdog_hidden.vbs`; the `LGI Watchdog` task was RE-ENABLED Oct 4, 2026 evening): relaunches via the no-trigger `LGI Manual` task ONLY when the pythonw pair is gone AND lgi.py's session marker `D:\LGI\.session_active` exists (crash of an ignited session); no marker (graceful quit / boot) → stays down — manual-ignition semantics intact; v2.1: clears the stale marker right before the relaunch fires (boot-crash chains self-arrest — a relaunch dying before writing its own marker leaves no marker) + crash-loop brake (≤3 guarded relaunches per 10 min, then cooldown slow-retry while the marker persists; a healthy 10-minute session expires the streak) (§10 items 9+14, §11 WATCH). `-TestGuard` dry-runs the branches without touching state |
| `crash_analyze.ps1` | 317 | **v1.0 crash postmortem driver** (§10 item 17, §11 WATCH): `-Posture` readiness audit (live pair/marker, LGI Watchdog task, WER LocalDumps capture config, pythonw dump inventory, debugger presence, READY/NEEDS-DEBUGGER verdict); `-EnsureDumps` pins HKCU `LocalDumps\pythonw.exe` full-dump capture (DumpType=2, keep 15); `-Install` stages winget `Microsoft.WinDbg` then the Windows-SDK Debuggers-only fallback (bootstrapper to `D:\LGI\winsdksetup.exe` + printed elevated one-liner — never fires UAC itself); `-Latest`/`-Dump`/`-All` run `cdb -z <dump>; !analyze -v` over `%LOCALAPPDATA%\CrashDumps` into `D:\LGI\crash_reports\*.analysis.log` + verdict extract (async default, `-Wait` blocks; idempotent skip when a verdict is newer than its dump; symbols cached to `D:\LGI\symcache`). Writes only its own dirs (+HKCU keys with `-EnsureDumps`) — read-only toward the live session/watchdog/marker |
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
UI latency, zero cross-thread Qt calls. A v1.3.1 liveness tick (`_tick_timer`,
60 s) prints one `[qt] main-thread tick <n>` line per minute — a tick gap with
flowing STT/audit traces proves a frozen Qt drain (blackout forensics).

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
 tts.stop()             strip wake word ▬▬▬► chat executor   silent 180 s wake-   hud.set_ambient()
 (local, no network)      client.chat(text)  │ chat_q       free window opens   "(listening — no
                                │            ▼              (v1.4.2, quiet)     wake word needed)"
                                ▼     _handle_chat_result
                              hud.set_exchange(you, polaris)
                              └─► tts.speak(response)   [if speak_responses]
```

`VOICE_STOP_COMMANDS` (matched on the normalized utterance):
`stop talking`, `be quiet`, `quiet please`, `silence`, `stop voice`, `shut up`.

**Conversation talk window (v1.3)** — every successful Polaris reply (chat or
webcam-analyze) opens or **renews** a `LGI_TALK_WINDOW` window (default **180 s**):
while it is open, follow-up utterances are transmitted **without the wake word**
(vision/webcam intent checks still apply; a spoken wake word is still stripped),
and the HUD ambient line shows a live countdown (`(talk window 2:31)`). Since
v1.4.2 every open/renew is SILENT — no spoken cue at all — and a BARE
`polaris` opens the window by itself: previously it only spoke *"Polaris
standing by."* while follow-ups without the wake word were ambient-dropped,
never sent (the "have to say it twice" symptom, second variant). The window
still expires silently back to strict wake-word mode. Close early with `polaris, sleep` /
`go to sleep` / `end the conversation` / `stop the conversation` /
`close the talk window` (HUD note, no TTS).

**Wake-word matching (v1.3 fix)** — prefix matching tolerates punctuation right
after the name (`Polaris,`, `Polaris.`, `Polaris?`, `hey polaris …`). Previously
only a trailing space matched, so whisper's near-constant `Polaris, <question>`
transcription was silently demoted to ambient-only — the operator's
"have to say it twice" symptom (fixed Sep 30, 2026).

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
                       spoken "Supervisor alert. Category: <category>. <reason>" (🔊-gated) +
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
              ├─► Hands (~7.5 Hz) ──► palm hold 2 s (v1.3.2) ──► command_q "mic:gesture"
              │                    └► fist hold 1 s ─► command_q "media:gesture"
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
              chat_timeout=None,   # None → LGI_CHAT_TIMEOUT env (default 180 s)
              vision_chat_timeout=120.0,  # desktop-vision turns (multimodal pass)
              audit_timeout=30.0,  # includes optional LLM triage
              status_timeout=5.0,  # lightweight liveness probe
              max_retries=2, backoff=1.5, log=print)
```

* `chat(text, image_b64=None, ocr_text=None, active_window=None)` →
  `{"ok": True, "response": ...}` — POSTs `{"prompt": text, "message": text,
  "device": "lgi-hud-x17"}` (both prompt keys, `message` is the legacy alias;
  `device` is the v1.4.2 origin tag letting the gateway target
  device-specific broadcasts — `chat_tool_report`); desktop-vision turns add
  `image_base64` / `ocr_text` / `active_window` and use the 120 s
  `vision_chat_timeout`. Empty response bodies are converted into errors.
  Tool turns run TWO gateway LLM passes (ack + truthful report) plus the
  tools — `LGI_CHAT_TIMEOUT` (default 180 s) covers the full pipeline.
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

> **Sole voice host (Oct 2026):** under the text-only mainframe directive, X17 is the ecosystem's only Kokoro TTS / Faster-Whisper STT host — the Polaris Gateway performs zero audio processing (voice routes deleted, audio libraries stripped from the gateway image).

Local Kokoro pipeline (`lang_code='a'`, default voice `bf_emma`, speed 1.2×,
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
* **`_trace` diagnostic tee (added Sep 30, 2026 — made permanent, decision
  closed Sep 30, 2026)** — every lifecycle point (construct, enqueue,
  worker-ready, device introspection, `sd.play` dispatch / success /
  failure) is appended to `%TEMP%\lgi_tts_log.txt`. This tee is what cracked
  the silent-voice case (§10 item 11) and is kept permanently as the
  first-read triage surface. **Size cap:** the file never exceeds ~1 MB
  (`TRACE_MAX_BYTES = 1_000_000` in `audio_tts.py`) — when the cap is
  exceeded the next write reopens the file in mode `"w"`, restarting the log
  in place (truncate-on-open: no rename, no pruning; the tee still never
  raises).
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
* **Buttons** — 🎤 Mic On/Muted · Force Audit · 🔊/🔇 Advisories · Stop Voice ·
  Quit. Handlers (`on_mic_toggle`, `on_force_audit`, `on_stop_voice`,
  `on_quit`, `on_toggle_alert_voice`) are injected by `lgi.py` after
  construction — the HUD has zero business logic.
* Mutators (Qt main thread only): `set_led(target, state)`, `set_exchange(you,
  polaris)`, `set_ambient(text)`, `set_supervisor_line(text)`,
  `show_alert(category, reason)`, `hide_alert()`, `set_mic_button(listening)`,
  `set_alert_voice_button(enabled)`.
* All text is whitespace-normalised and elided (exchange 200/300, SUP 90,
  banner 140 chars).

### 6.6 `lgi.py` — `LGIApp` (orchestrator) + `LGIConfig`

`LGIConfig.from_env()` builds the configuration from environment overrides
(§8). `LGIApp(QObject)` wires the engines together:

* `__init__` — PolarisClient → HUDWindow (handlers injected) → KokoroTTS →
  MicListener → WebcamPerception + LocalVLM → VisionSupervisor (receives the
  shared `webcam_provider` + `attention_gate`) → chat
  `ThreadPoolExecutor(max_workers=2)` → 100 ms drain `QTimer` → alert epoch
  counter, user-mute flag, advisory voice toggle.
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
  clear a newer alert; verbal speech is gated by `speak_alerts` (🔇 silent
  default — HUD 🔊/🔇 Advisories button, `LGI_SPEAK_ALERTS` boot override).
* `_on_tts_state` — optional ducking: `speaking` pauses STT; `idle` / `ready` /
  `error` resume it unless the user muted the mic (`_mic_muted_by_user`).
* `main()` — `QApplication`, `lgi.start()`, `app.exec()`, `finally` shutdown.
* **Console guard (runs first, before heavy imports)** — under `pythonw.exe`
  (any Scheduled-Task launch — `LGI AutoStart` then, `LGI Manual` now)
  `sys.stdout`/`sys.stderr` are `None`;
  any library logging to stdout during import (huggingface_hub's
  unauthenticated-request warning, inside the kokoro import chain) raises
  `Cannot log to objects of type 'NoneType'` and the import dies — the TTS
  worker exited silently on **every console-less boot**, regardless of output
  device (the device switch was a red herring). `lgi.py` redirects both
  streams to `%TEMP%\lgi_console.log` (append) when they are `None` —
  validated live: `worker ready pipeline=cuda`.
* **Custom `sys.excepthook`** — PyQt6 kills the whole process via qFatal
  (`Qt6Core.dll` 0xc0000409) when an uncaught Python exception reaches a Qt
  slot (observed on the supervisor-alert path). The hook logs the traceback
  and returns, so LGI survives the abort; tracebacks land in
  `%TEMP%\lgi_console.log`.
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
* **Gestures** — palm held 2 s CONTINUOUSLY pushes `"mic"` (mute toggle), a
  held fist (1 s) pushes `"media"` (OS media key via the `keyboard` lib).
  v1.3.2 hardening: dwell resets on any frame without a confirmed large hand
  pose (no cumulative dwell across flicker gaps), a cooldown-blocked fire
  still clears the dwell (a held palm can never re-fire and flap the mic
  state), 6 s cooldown, and fires log their held seconds;
  `LGI_GESTURES=0` disables. Gesture commands arrive source-tagged
  (`mic:gesture`).
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

### 6.10 `gateway_feed.py` — `GatewayFeedClient` + `ConversationStore` (v1.4)

NEW Phase 3 module — the always-listening pair:

* **`GatewayFeedClient`** — ONE daemon-thread Socket.IO client to
  `LGI_POLARIS_URL` (modeled on the radar `SpatialReporter`): on every
  `connect` it announces `{'type': 'identity', 'identity': 'Rich'}` (the
  gateway joins it to the identity room), emits `request_history` (the
  gateway answers `'chat_history'` with a 40-message seed), and emits
  `join_tasks` (joins the `status` room: `'active_tasks'` snapshot +
  `'task_registered'`/`'task_started'`/`'task_progress'`/`'task_completed'`/
  `'task_failed'` lifecycle events). Chat arrives via `'chat_message'`
  finals and `'message' {'type': 'chat_stream'}` typed-out deltas;
  `tool_execution` renders as ⚙ CHAT_TOOL rows in ▼ TACTICAL OPERATIONS
  (v1.4.2: one row per call — `task_id` = tool name, description = `args… → ok/error`)
  and the device-tagged `chat_tool_report` (gateway `/api/chat` report pass) swaps
  the truthful tool answer into the main exchange + speaks it — ONLY when
  `origin_device == DEVICE_ID` (lgi-hud-x17); other devices stay silent.
  All events funnel into `feed_q`;
  link state pushes `("feed", online|connecting|offline)` to `status_q`.
  **Reconnection is UNLIMITED** (`reconnection_attempts=0` — the dashboard
  v1.0.4 lesson) and every reconnect re-announces + re-seeds, so messages
  missed during a worker recycle render on return. The thread never
  touches Qt.
* **`ConversationStore`** — pure-stdlib rolling window (unit-testable on
  any host): `seed()` replaces content from a replay (WS or REST-paired
  fallback) and clears in-flight partials; `apply_chat_message()` dedupes
  exactly like the dashboard v1.0.4 repair (user echoes collapse by
  sender+content within 150 s, Polaris keeps the strict sender+content+
  timestamp triple); `apply_stream()` types partials out live and promotes
  each stream exactly once — a content digest suppresses the trailing
  final `chat_message`; `visible()` returns the latest N per side
  (default 5, `LGI_FEED_PER_SIDE`) interleaved chronologically with any
  partial pinned last.
* **`_feed_fallback_loop` (lgi.py)** — REST paired-history poll
  (`GET /api/history/<user>`, every `LGI_FEED_FALLBACK` s, default 15)
  only while the socket link is down, so the window still advances when
  the gateway is sick (POL LED already reports gateway health).


Confirmed against `polaris_continuous_learning_gateway.py` /
`supervisor_routes.py`. All calls are JSON over HTTP on **port 8082
(text/JSON only — never raw audio)**.

| Endpoint | Method | Request | Response |
|---|---|---|---|
| `/api/chat` | POST | `{"prompt": "..."}` (`"message"` legacy alias) + `"device": "lgi-hud-x17"` (v1.4.2 origin tag) + optional `image_base64` / `ocr_text` / `active_window` (vision turns) | Ollama JSON `{"response": "<text>"}` (+ `vision: true`, `vision_model` on vision turns); on tool turns a device-tagged `chat_tool_report` Socket.IO broadcast carries the truthful report |
| `/api/supervisor/audit` | POST | snapshot payload (§6.4) | `{"status":"success", "verdict":"SUPERVISOR_ALERT"\|"LOG_ONLY"\|"NO_ACTION", "category":"...", "vault_persisted":bool, "triage":{"heuristic":{...},"llm":{...}}, "timestamp":...}` |
| `/api/supervisor/status` | GET | — | lightweight liveness (POLARIS LED heartbeat) |
| `/api/history/<user_id>` | GET | — | paired unification envelope `{status, user, count, history:[{user_message, ai_response, timestamp, sender}]}` — v1.4 fallback seed poll while the always-listening feed link is down |
| `/api/memory/vault/stats` | GET | — | vault + index health (v1.4 HUD vault status line) |
| `/api/memory/vault/list` | GET | `?limit=N[&category=core\|architecture\|user\|lesson\|journal\|attachment]` | `{documents:[{path, title, category, updated, snippet}]}` (v1.4 HUD vault browser, read-only) |
| `/api/memory/vault/doc` | GET | `?path=<enc>` | `{document:{path, content, meta:{...}, modified}}` (v1.4 HUD doc viewer, read-only) |
| `/api/memory/vault/query` | POST | `{"query", "mode": "hybrid"\|"keyword"\|"semantic", "limit", ["categories"]}` | `{results:[{path, title, category, updated, snippet, score, match_type}]}` (v1.4 HUD vault SEARCH, read-only) |

**Always-listening Socket.IO contract (v1.4, gateway host `:8082`)** — the
`GatewayFeedClient` joins over ONE connection: on every `connect` it emits
`'message' {'type': 'identity', 'identity': 'Rich', 'device': 'lgi-hud-x17'}`
(gateway joins it to the identity room), `'request_history' {'device_id':
'Rich'}` (replies with `'chat_history' {status, user, user_id, messages:[40]
{id, message, content, role, sender, timestamp}}`), and `'join_tasks'`
(joins the `status` room → `'active_tasks'` snapshot + `'task_registered'` /
`'task_started'` / `'task_progress'` / `'task_completed'` / `'task_failed'`
lifecycle events). Chat traffic arrives as `'chat_message'`
{sender, content|message, user_id, timestamp} and `'message' {'type':
'chat_stream', 'stream_id', 'content', 'is_complete', 'sender':'polaris'}`
deltas — latest 5 per side render in the ⌁ window; reconnection is UNLIMITED
and every reconnect re-announces + re-seeds (dashboard v1.0.4 lesson).

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
| `LGI_OPERATOR_USER` | `rich` | operator identity in audit payloads + feed announce (`'Rich'` at the identity room) |
| `LGI_FEED` | `true` | v1.4 always-listening feed: ONE Socket.IO link to `LGI_POLARIS_URL` (chat identity room + `status` task feed) |
| `LGI_FEED_PER_SIDE` | `5` | rolling window depth — latest 5 rows per side (Looking-Glass seed v2 rule) |
| `LGI_FEED_FALLBACK` | `15` | seconds between REST paired-history polls while the feed link is down (`0` disables) |
| `LGI_HUD_FONT` | `14` | HUD base font px (first-run seed — the ⚙ panel persists its own value) |
| `LGI_HUD_OPACITY` | `94` | HUD solidness % — v1.4.1: window opacity fading the ENTIRE HUD incl. text (accepts legacy 0–1 floats; first-run seed — the ⚙ panel persists) |
| `LGI_STT_MODEL` | `deepdml/faster-whisper-large-v3-turbo-ct2` | faster-whisper model (HF repo id or size name) |
| `LGI_STT_DEVICE` | `cuda` | `cuda` with auto-fallback to `cpu` (int8) |
| `LGI_STT_COMPUTE` | `float16` | CTranslate2 compute type on the primary device (`int8` halves VRAM) |
| `LGI_TTS_DEVICE` | `cuda` | Kokoro pipeline device, auto CPU fallback |
| `LGI_WAKE_WORD` | `polaris` | wake word (`hey <word>` also accepted) |
| `LGI_WAKE_REQUIRED` | `true` | `false` → **open mode**: every utterance is sent to chat |
| `LGI_TALK_WINDOW` | `180` | seconds of wake-free conversation opened/renewed by each successful reply (v1.4.2: also opened silently by a bare `polaris`); `0` = always require the wake word |
| `LGI_CHAT_TIMEOUT` | `180` | seconds LGI waits for `/api/chat` before `⚠ gateway: timed out` — tool turns run ~2 LLM passes + tool time; raise if heavy chains grow longer |
| `LGI_AUDIT_INTERVAL` | `30` | seconds between supervisor audits (floor 5) |
| `LGI_HEARTBEAT` | `15` | seconds between POLARIS liveness polls |
| `LGI_WEBCAM` | `true` | include webcam JPEG in audit payloads |
| `LGI_SCREENSHOT` | `true` | include screen JPEG in audit payloads |
| `LGI_SPEAK_ALERTS` | `false` | verbal SUPERVISOR_ALERT speech (visual LED/banner always fire); HUD 🔊/🔇 Advisories button toggles per session |
| `LGI_VISION_CHAT` | `true` | desktop-vision turns (dashcam frame attached to visual utterances) |
| `LGI_VISION_FPS` | `1` | ScreenRoller dashcam capture rate (frames/s) |
| `LGI_VISION_WIDTH` | `1920` | max JPEG width sent on vision turns |
| `LGI_VISION_QUALITY` | `70` | JPEG quality sent on vision turns |
| `LGI_OCR` | `true` | attach local Tesseract OCR text to vision turns |
| `LGI_DUCK_WHILE_SPEAKING` | `false` | pause mic while Polaris speaks (off = playback never stalls the mic) |
| `LGI_WEBCAM_TRACK` | `true` | continuous webcam perception stack — camera LED ON while LGI runs; `false` = legacy per-audit open/release capture |
| `LGI_GESTURES` | `true` | palm hold 2 s = mic toggle · fist hold 1 s = media play/pause (v1.3.2 dwell hardening, 6 s cooldown) |
| `LGI_ATTENTION_GATE` | `true` | supervisor audits skipped while the operator is away (fail-open; `false` = gate off) |
| `LGI_RADAR` | `true` | spatial telemetry (`radar_motion` / `vision_frame`) to the Burst Receiver |
| `LGI_RADAR_URL` | `http://192.168.50.51:8083` | Burst Receiver socketio endpoint |
| `LGI_WEBCAM_FPS` | `15` | perception capture rate (frames/s) |
| `LGI_VLM` | `true` | webcam-analyze turns enabled |
| `LGI_VLM_URL` | `http://192.168.50.51:11434` | Ollama base URL for webcam-analyze passes |
| `LGI_VLM_MODEL` | `polaris-ai:latest` | vision model for webcam-analyze turns |
| `LGI_VLM_TIMEOUT` | `90` | seconds per webcam-analyze vision pass |

Non-env config (edit the `LGIConfig` defaults): `tts_voice` (`bf_emma`),
`tts_speed` (1.2), `speak_responses` (True), `hotkey_audit`
(`ctrl+shift+s`), `hotkey_mic` (`ctrl+shift+m`), `alert_led_hold_s` (60.0).
HUD chrome (font px, panel transparency, tactical collapse, window
geometry) lives in **QSettings** (`Polaris` / `LGI` — registry-backed on
Windows) and survives restarts; the env vars above only seed first run.

First launch after the turbo-model upgrade downloads ~1.6 GB of CT2 weights
from HuggingFace into the user cache (`%USERPROFILE%\.cache\huggingface\hub`);
subsequent launches load from disk with no network access.

## 9. Operator Controls

| Control | Action |
|---|---|
| Say `"Polaris <question>"` | Send to Polaris chat; reply displayed + spoken |
| Say bare `"Polaris"` | **Silent wake-free window** — opens the 180 s talk window without speaking; ambient shows `(listening — no wake word needed)` (v1.4.2) |
| Say `"Polaris look at this layout"` / *"what's wrong with this CSS"* | **Vision turn** — latest dashcam JPEG + OCR text + active window attached; reply spoken through Kokoro |
| Say `"Polaris, analyze this component"` / *"what am I holding"* / *"who's there"* | **Webcam-analyze turn** — live camera frame → Ollama `polaris-ai:latest` → short spoken description |
| Hold an open palm to the camera ~2 s (continuous) | Mic mute toggle (same as `Ctrl+Shift+M`) — trace records the source |
| Show a held fist to the camera | Media play/pause |
| Say `stop talking` / `be quiet` / `quiet please` / `silence` / `stop voice` / `shut up` | Silence TTS instantly — evaluated locally, no network |
| `Ctrl+Shift+S` | Force an immediate supervisor audit |
| `Ctrl+Shift+M` | Toggle mic (STT pause/resume; capture stream stays open) |
| HUD 🎤 Mic On/Muted | Same as `Ctrl+Shift+M` |
| HUD 🔊/🔇 Advisories | Toggle verbal supervisor alerts — red LED, banner, and SUP line fire visually regardless |
| HUD Force Audit / Stop Voice / Quit | Force audit · silence TTS · close LGI |
| Drag the HUD | Reposition the overlay anywhere on screen |
| HUD ⋯ (top-bar kebab) | Slide down the ⚙ HUD settings panel |
| ⚙ FONT SIZE slider (10–24 px) | Rescale every HUD font live — persisted via QSettings |
| ⚙ TRANSPARENCY slider (40–100% solid) | Fade the ENTIRE HUD live — window opacity fades every surface, border, LED and text as one uniform layer — persisted via QSettings |
| ⌁ LAST EXCHANGES window | Always-listening mirror: latest 5 messages per side update live from any device (phone / dashboard / LGI voice) — FEED LED shows pipe health |
| ▼ / ▶ TACTICAL OPERATIONS | Collapse / expand the dashboard bottom-half mirror (persisted) |
| TASK FEED / MEMORY VAULT tabs | Live task stream (▶ / ✓ / ✗, 50-event rolling window, active snapshot count) · read-only vault browser (category filter, hybrid/keyword search, open doc) |
| Bottom-right grip | Resize the whole HUD — geometry persisted |
## 10. Deployment Runbook (X17)

LGI runs **natively on Windows** — it is not a compose service and needs no
container rebuild: mic, webcam, screen capture, GPU, and Qt GUI all require
native desktop access.

1. **Sync** the `LGI/` folder to the X17 — push with a **glob so the folder
   merges instead of nesting**:
   `scp -r LGI/* rbuit@192.168.50.227:D:/LGI`.
   A bare `scp -r LGI … D:/LGI` copies the folder *into* `D:\LGI` as
   `D:\LGI\LGI\` and leaves the stale root build live (Oct 3, 2026 incident —
   the fresh code sat unexecuted inside the nested folder while the root .py
   files stayed on the old build). Recover by flattening:
   `copy /Y D:\LGI\LGI\*.* D:\LGI\` + `rmdir /S /Q D:\LGI\LGI`.
   After every push, verify MD5 parity: local `md5sum LGI/lgi.py` vs remote
   `certutil -hashfile D:\LGI\lgi.py MD5` (compare the first 8 hex chars).
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
8. **Gateway SSH access** — authorize the gateway's SSH key on the X17 so the
   Polaris tools (`ssh_execute`, `ssh_deploy_file`, `ssh_fetch_file`) run
   hands-free: copy the gateway's public key into
   `C:\Users\rbuit\.ssh\authorized_keys`. Windows OpenSSH gotcha: if `rbuit`
   is an Administrator, sshd reads **only**
   `C:\ProgramData\ssh\administrators_authorized_keys` — paste the key there
   instead and fix its ACLs:
   `icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "SYSTEM:F" /grant "BUILTIN\Administrators:F"`.
9. **Ignition — MANUAL since Oct 4, 2026 (operator request)** — LGI does NOT
   auto-start on boot. The legacy `LGI AutoStart` (at logon) task remains
   **Disabled** (XML backup kept at `D:\LGI\backup_tasks\LGI_AutoStart.xml`); a
   **no-trigger** `LGI Manual` Scheduled Task carries the same windowless launch
   (survives disconnects, unlike a bare background start): action
   `D:\LGI\venv312\Scripts\pythonw.exe D:\LGI\lgi.py`, Interactive-only,
   RunLevel Highest, **no execution-time limit** — Ready in Task Scheduler but
   never self-firing (Next Run Time: N/A).
   — fire her up on demand, from any box: `schtasks /run /tn "LGI Manual"`
     (works from the X17, the `:7007` Sandbox dashboard, or Polaris's
     `ssh_execute` tool), or double-click the desktop shortcut
     `LGI (manual).lnk` (same pythonw line). A DISABLED task cannot be `/run` —
     that is why the separate enabled, trigger-less task exists.
   — verify: `schtasks /query /v /fo list /tn "LGI Manual"`; a running instance
     shows `Last Result 267009` / `0x41301`; the task runs the **`D:\LGI\venv312`**
     interpreter, not the AppData Python.
   — **Watchdog RE-ENABLED as GUARDED (Oct 4, 2026 evening, v2.1)** — the
     `LGI Watchdog` 60 s task is back ON, but `watchdog.ps1` v2.1 relaunches via
     `LGI Manual` ONLY when LGI died while `D:\LGI\.session_active` exists —
     and it now clears that stale marker right before firing, so a relaunched
     session that dies BEFORE writing its own marker leaves no marker at all
     (boot-crash chains self-arrest), while a crash-loop brake caps guarded
     relaunches at 3 per 10 minutes and then stands down to a cooldown
     slow-retry for as long as the marker persists (every refusal logged once).
     `lgi.py` v1.5.0+ writes that marker right after a successful start and
     removes it on graceful exit (`main()` finally): crash/force-kill of an
     ignited session → guarded resurrect; operator quit or any boot → stays
     down. A healthy session that has run 10+ minutes expires the streak file
     by itself. Boot-time failures never trip the watchdog (marker is written
     only after `start()` succeeds).
   — **single-instance veto (v1.5.1)** — `main()` acquires the `Global\`
     `LGI_X17_SingleInstance` mutex BEFORE any GPU/audio/GUI init; a second
     launch logs `second instance refused` to `%TEMP%\lgi_console.log` and
     exits cleanly. Reason: two concurrent sessions racing the CUDA/audio
     native stack heap-corrupt within seconds (live proof Oct 4 15:25:20→21,
     §11). Ignite ONLY after the previous session has fully exited.
   — transition note: RESOLVED Oct 4 late evening — the stale 08:38 pre-v1.5.0
     session was stopped and one clean `LGI Manual` ignition cutover to
     v1.5.1 ran with the marker protocol live end-to-end. Pre-v1.5.0 sessions
     (e.g. after a rollback) never write the marker — a crash of such a
     session stays down (manual rule wins).
   — full rollback to unguarded auto-start:
     `schtasks /change /tn "LGI AutoStart" /enable` (+ optionally restore
     `watchdog.ps1` v1.3.1 from `D:\LGI\watchdog.ps1.bak_20261004` so the
     watchdog relaunches `LGI AutoStart` again), and optionally
     `schtasks /delete /f /tn "LGI Manual"`.
10. **Dependency self-check** — probe the Python stack of the task venv
    (`D:/LGI/venv312`) and — critically — the legacy `mp.solutions` API that
    `webcam_perception.py` needs (mediapipe 1.0.x and the 0.10.30+ builds
    stripped it — Oct 3, 2026 gesture outage), without touching the UI:
    `ssh rbuit@192.168.50.227 "D:/LGI/venv312/Scripts/python.exe -c \"import socketio, cv2, mediapipe as mp, mediapipe.solutions.hands, ultralytics, faster_whisper, kokoro, PyQt6; print('LGI deps OK', mp.__version__, 'solutions', hasattr(mp, 'solutions'))\""`.

11. **Console-less boot & diagnostics (Sep 30, 2026)** — the auto-start task
    runs `pythonw.exe` (no console): `sys.stdout`/`sys.stderr` are `None`.
    Historically this killed the kokoro import chain silently
    (huggingface_hub stdout warning) — voice dead with zero symptoms, while
    every device-side probe passed. `lgi.py` now carries the console guard
    (§6.6). Live diagnostic surfaces:
    * `%TEMP%\lgi_console.log` — boot stdout/stderr + excepthook tracebacks
    * `%TEMP%\lgi_tts_log.txt` — TTS `_trace` tee (§6.3)
    * `D:\LGI\crash_query.ps1` — Windows Error Reporting dump query for the
      flaky native crash class (§11 watchlist)
    **Triage rule:** voice dead → read `lgi_tts_log.txt` first. A missing
    `worker ready` line means the boot killed the worker (check
    `lgi_console.log`), not the output device.
12. **Probe cleanup (Sep 30, 2026)** — the four probe Scheduled Tasks
    (`X17-TTS-PlayTest`, `X17-SD-PlayTest`, `X17-SD-PlayTestW`,
    `X17-KOKORO-W`) were deleted and probe artifacts purged from `%TEMP%`
    (`lgi_tts_probe.wav`, `tts_probe_w.log`, `sd_probe_out.txt`,
    `sd_probe_out_w.txt`). Kept as Phase D diagnostics (repo + `D:\LGI`):
    `tts_probe.py`, `sd_play_probe.py`, `crash_query.ps1`, `dep_probe.py`.
    The live instrumentation (`lgi_console.log`, `lgi_tts_log.txt`) and the
    `LGI AutoStart` task are untouched.
13. **Talk window + wake-word comma fix (Sep 30, 2026)** — operator symptom
    "have to say it twice": whisper transcribes direct address as
    `Polaris, <question>` far more often than `Polaris <question>`, and the old
    exact-space prefix match silently demoted those utterances to ambient-only.
    Fixed with punctuation-tolerant prefix matching, plus the v1.3 conversation
    talk window (`LGI_TALK_WINDOW`, default 180 s — successful replies open/
    renew a wake-free window, `polaris, sleep` closes it, spoken open cue once
    per window, HUD countdown while open). Same-boot diagnosis: LGI was alive
    all along — the day's silent gaps were swallowed ambient utterances and a
    mic left **muted** at 19:24:40 EDT (user-initiated; palm gestures were
    offline), confirmed via gateway chat history (`GET /api/chat/history`
    resolves per-IP — LGI's exchanges land under the X17's identity) +
    console timeline. Restart clears a stuck mic-mute state.
14. **Watchdog relaunch + voice-path black-box traces (Sep 30, 2026, v1.3.1)** —
    after the first steady-state native crash (§11 WATCH row) left LGI down for
    ~2 min and a **deaf boot** went undiagnosable (wake-word attempts produced
    no dispatches — HUD ambient stayed empty, console silent, restart-on-failure
    didn't fire): the `LGI Watchdog` Scheduled Task (every 60 s,
    `D:\LGI\watchdog.ps1`) relaunches `LGI AutoStart` whenever the pythonw pair
    vanishes, logging `%TEMP%\lgi_watchdog.log`. Keep `watchdog.ps1` pure-ASCII
    + CRLF — PS 5.1 parses a non-BOM UTF-8 script as system ANSI and the em-dash
    bytes (0x94 → U+201D inside a string) close the string early, killing the
    whole parse (caught in the deploy dry-run 19:54:09). Same run — first live
    pass — then caught a **real** LGI death (`pythonw x0`, LGI's second outage
    that day) and relaunched her cleanly on v1.3.1. `lgi.py` + `audio_stt.py` now
    trace the whole voice path so silence is diagnosable from one live session:
    `[stt] mic open:` (captured device name), `[stt] speech onset` (energy vs
    threshold), `[stt] transcribing <N> ms ...` / `[stt] transcribe done in <N>
    ms` (hang detector), `[stt] empty transcription`, `[voice] text=… wake=…
    solo=… in_window=…` routing trace, and one-per-minute `[qt] main-thread
    tick` (a tick gap with flowing STT/audit traces ⇒ frozen Qt drain). All
    traces are machine-local (§12 privacy). `watchdog.ps1` lives in the repo
    and on `D:\LGI` for source parity.
    **v1.3.1a windowless watchdog (Sep 30, 2026)** — operator saw a console
    window flash open/closed every 60 s: the task ran bare `powershell`
    (console-subsystem) directly in the interactive session, and
    `-WindowStyle Hidden` cannot prevent it (conhost spawns before flags are
    parsed). The action is now `wscript.exe D:\LGI\watchdog_hidden.vbs` —
    wscript is a GUI-subsystem host (no console ever appears) whose
    `Run(…, 0, False)` hides the child powershell too; launcher errors append
    to `%TEMP%\lgi_watchdog_launcher.log` instead of popping an error dialog
    each minute. Flags preserved byte-faithful via XML round-trip:
    `schtasks /query /tn "LGI Watchdog" /xml` → swap only `<Command>` +
    `<Arguments>` → `schtasks /create /f /tn "LGI Watchdog" /xml
    D:\LGI\watchdog_task.xml` (XML backup kept at `D:\LGI\watchdog_task.xml`).
    The live task carries `<RunLevel>HighestAvailable</RunLevel>` +
    InteractiveToken + PT1M repetition — a plain `schtasks /create` would have
    silently downgraded the run level. Verified: create OK, 20:07:01 minute
    tick + manual `/run` both `Last Result: 0`, pythonw pair untouched, md5
    parity `f0d18ae8`. `watchdog_hidden.vbs` lives in the repo and on `D:\LGI`
    for source parity.

    **v1.3.2 deafness hardening (Sep 30, 2026)** — operator went deaf at
    20:01:25 without acting; prime suspect was the palm gesture toggling the
    mic on natural hand movement. Fixed at the source: (a) every mic
    pause/resume trace carries its origin — `mic muted
    (source=hotkey|gesture|hud)` — via `mic:hotkey` / `mic:gesture` command
    tags (the HUD button calls the toggle directly), plus `[tts] ducking`
    pause/release traces; (b) gesture dwell hardened in
    `webcam_perception.py`: dwell resets on ANY frame without a confirmed
    large hand pose (the old hand-size early-return let dwell survive
    flicker gaps — cumulative false fires), `_fire_gesture` clears dwell even
    when cooldown-suppressed (a palm held through the cooldown can no longer
    double-toggle the mute), palm dwell 1.5→2.0 s, fist 0.6→1.0 s, cooldown
    3→6 s, fires log their held seconds; (c) whisper-on-silence degenerates
    ("A. . A. .") are dropped in `audio_stt._transcribe` — fewer than two
    distinct alphanumerics or a filler — logged as
    `[stt] degenerate transcription ignored: …` and never shown or routed.

    **v2.0 guarded relauncher (Oct 4, 2026, manual-ignition era)** — the old
    unguarded watchdog (relaunch whenever the pair vanished) fights manual
    ignition at boot, so it was briefly disabled together with `LGI AutoStart` —
    then re-armed the same day with a session-marker guard: `lgi.py` (v1.5.0)
    writes `D:\LGI\.session_active` after a successful start and clears it in
    `main()`'s finally on graceful exit; `watchdog.ps1` v2.0 relaunches via
    `schtasks /run /tn "LGI Manual"` only when the pair is gone AND the marker
    exists (crash semantics), logging once per state transition to
    `%TEMP%\lgi_watchdog.log` (no 60 s spam). `-TestGuard` dry-runs the branch
    logic without relaunching — live proof 15:18:01→15:18:03: no-marker →
    "staying down"; marker → "guarded relaunch [TESTGUARD dry-run]"; live pair →
    "healthy pythonw x2". Keep pure-ASCII + CRLF (§6.6 lesson); md5 parity
    `8c6ce328` (backups: `watchdog.ps1.bak_20261004` = v1.3.1 `e473ec9a`,
    `lgi.py.bak_20261004` = pre-marker `b556e13f`).

15. **Always-listening feed mirror (Oct 2, 2026, v1.4)** — Phase 3 of the
    Looking-Glass unification: LGI stopped showing only its own live voice
    exchange and became a full peer. NEW `gateway_feed.py`
    (`GatewayFeedClient` + `ConversationStore`, §6.10): ONE persistent
    Socket.IO connection to the gateway `:8082` joins the operator's
    identity room (announce → `chat_history` 40-message seed → live
    `chat_message` finals + `chat_stream` typed-out deltas — push, never
    poll) and the `status` room (`active_tasks` snapshot + `task_*`
    lifecycle events). Unlimited reconnection with re-announce + re-seed
    on every reconnect (dashboard v1.0.4 lesson — a capped attempts
    counter caused permanent silence), plus a REST paired-history fallback
    poll (`/api/history/<user>` every `LGI_FEED_FALLBACK` s, default 15)
    keeps the window advancing while the gateway is down. HUD v1.4: top
    band unchanged; below it a ⌁ LAST EXCHANGES rolling window renders the
    latest 5 per side (seed v2 rule) with a FEED LED (`link-up`/`link-wait`/
    `link-down`); a ▼ TACTICAL OPERATIONS mirror carries the TASK FEED
    (▶/✓/✗ 50-event rolling window + active-task snapshot line) and a
    READ-ONLY MEMORY VAULT browser (category filter, hybrid/keyword/
    semantic search, open-doc viewer — NEW/EDIT/REINDEX stay terminal-side
    by operator decision). A ⋯ settings kebab in the header slides down
    HUD font-size (10–24 px) + transparency (40–100% solid) sliders —
    live-applied via the parameterized `build_hud_qss()` (window opacity
    retired; painted alphas own the glass) and persisted in QSettings
    with the collapse state and window geometry. Other-device traffic
    renders visually only — never TTS. Dedupe = dashboard v1.0.4 policy:
    user echoes collapse inside 150 s, Polaris strict triple, stream
    promotions suppress their trailing final via a content digest
    (exactly-once typing). Pre-deploy verification: `py_compile` clean
    across all four touched files; pure-logic simulation green (seed
    slice, echo suppression/resend, strict triple, stream exactly-once,
    paired-envelope parse, pump wiring, keep-limit).

16. **Whole-HUD transparency (Oct 2, 2026, v1.4.1)** — operator request: the
    ⚙ TRANSPARENCY slider now fades the ENTIRE HUD, not just where the
    sliders are. v1.4 scaled only the painted QSS alphas (backdrops,
    bubbles, borders) and deliberately kept text full-opacity — against a
    dark desktop the big surfaces barely appeared to move, so the knob read
    as local to the settings pull-down. Fix: `build_hud_qss()` now paints
    FIXED v1.3 glass alphas (no double-dim with the compositor) and
    `_apply_style_now()` applies `setWindowOpacity(panel_pct / 100)` — the
    Windows compositor fades the whole window (every surface, border, LED
    and text) as one uniform layer. Same 40–100% solid range, same
    QSettings key (`hud/panel_pct`), same 150 ms style-debounce live-apply
    as the font slider. Trade-off: transparency includes text — readability
    dips at low solidness, slide up for crisper glass.

17. **Crash postmortem driver (Oct 4, 2026 late night, `crash_analyze.ps1` v1.0)** —
    "prepare in case there is an error": one command turns a §11 WATCH crash
    dump into a WinDbg verdict, and one command audits crash-readiness before
    the next event strikes. Default with no switches (and `-Posture`) is a
    read-only audit — live pair/marker combination check, `LGI Watchdog` task
    state/last/next run, WER LocalDumps capture config (HKCU + HKLM, pythonw
    key and default), dump inventory newest-first, debugger presence, report
    and symbol dirs — closing with a READY / NEEDS DEBUGGER verdict line.
    `-EnsureDumps` pins `HKCU\...\LocalDumps\pythonw.exe` to DumpFolder
    `%LOCALAPPDATA%\CrashDumps`, DumpType 2 (full), DumpCount 15,
    CustomDumpFlags 0: capture currently has NO explicit config (8 dumps
    exist on Windows default WER behavior — ten at recon time, Sep 30's six
    aged past the default 10-dump retention), so the pin makes retention
    deliberate instead of incidental. `-Install` installs a debugger with
    ZERO UAC: the winget `Microsoft.WinDbg` route was proven live Oct 4 ~16:00 —
    the MSIX (1.2606.22001.0) ships `amd64\cdb.exe` + `kd.exe` inside
    `C:\Program Files\WindowsApps\Microsoft.WinDbg_..._x64__8wekyb3d8bbwe\`,
    which `Find-Debugger` resolves through its Appx-package discovery branch
    (the earlier "UI-only" assumption was wrong; the Windows-SDK
    Debuggers-only fallback — bootstrapper staged to `winsdksetup.exe`,
    printed elevated `/features OptionId.WindowsDesktopDebuggers` one-liner —
    stays coded as plan B and was never needed). `-Latest` / `-Dump <name>` /
    `%LOCALAPPDATA%\CrashDumps\pythonw*.dmp` with
    `srv*D:\LGI\symcache*https://msdl.microsoft.com/download/symbols` symbols:
    reports land at `D:\LGI\crash_reports\<dump>.analysis.log` plus a stdout
    VERDICT EXTRACT (exception code, bucket ids, symbol/module names, top
    STACK_TEXT frames). Launches async by default (poll recipe printed;
    `-Wait` blocks — first runs may download MS symbols, allow minutes),
    skips dumps whose verdict log is already newer (idempotent `-All`),
    refuses dumps still locked by WER, exit codes 0/2/3/4. Read-only toward
    the live session, the watchdog and the marker (§10 items 9+14). Recon
    Oct 4 15:5x: no debugger on the X17 (all Windows Kits paths empty, no
    Store WinDbg, winget 1.29.380 present); 10 pythonw dumps 320-414 MB
    (newest `pythonw.exe.30564.dmp` 336 MB @ Oct 4 15:25:34); D: 865 GB free.
    After any FUTURE native crash: run `-Posture`, then `-Latest`. Async
    analysis from the Mainframe rides the one-shot Scheduled-Task trick
    (`schtasks /create /tn "LGI Dump Analyze" /tr "cmd.exe /c powershell
    -ExecutionPolicy Bypass -File D:\LGI\crash_analyze.ps1 -<flags> >
    D:\LGI\crash_reports\analyze.log 2>&1" /sc once /st 00:00` then
    `/run` — Task Scheduler runs survive the ssh job-object teardown that
    killed every direct Start-Process attempt; ssh-synchronous invocation
    is only safe for the quick modes `-Posture`/`-EnsureDumps`).
    **First results (Oct 4 ~16:00, one `!analyze -v` each):**
    `pythonw.exe.30564.dmp` (GPU-race victim) =
    `HEAP_CORRUPTION_ACTIONABLE_BlockNotBusy_DOUBLE_FREE_c0000374_ucrtbase.dll!free_base`;
    `pythonw.exe.12900.dmp` (Oct 4 08:31 solo kill) and
    `pythonw.exe.19204.dmp` (Oct 3 12:01 solo kill) = the SAME bucket family
    with `cv2.pyd!unknown_function` — the ~daily spontaneous class is a
    **native double-free inside OpenCV's cv2.pyd**, fail-fast surfaced at
    `ntdll!RtlReportFatalFailure`, NOT the STT/TTS voice stack: the
    `LGI_STT_DEVICE=cpu` A/B is demoted; the interesting experiments are
    camera-path observation windows (`LGI_WEBCAM=0` / `LGI_WEBCAM_TRACK=0`)
    or a cv-wheel pin A/B — operator decision (§11 WATCH).

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
| Gesture misfire | v1.3.2: dwell resets on any unconfirmed frame + 6 s cooldown that clears dwell on suppression bind false positives; every mic log line records its source; `LGI_GESTURES=0` kills the feature |
| Operator away, attention gate on | audits skipped (SUP LED idle); explicit voice commands unaffected (fail-open) |
| Clipboard unavailable | empty snippet; pyperclip errors swallowed |
| `keyboard` lib unavailable | global hotkeys disabled (logged); HUD buttons still work |
| dxcam missing / DXGI blocked | ScreenRoller falls back to mss one-shot grabs (logged); vision turns still work |
| pytesseract / Tesseract missing | OCR context auto-disabled (logged); vision turns send the image without OCR text |
| Vision LLM offline / image rejected | gateway ladder falls back to `mistral-large-3:675b-cloud`; total failure → HUD error line, no crash |
| Kokoro pipeline init failure | TTS disabled (`error` state); HUD still shows chat replies |
| TTS playback error | chunk aborted; worker continues with the next request |
| Kokoro import dies under pythonw (**defused**) | console guard maps stdout/stderr → `%TEMP%\lgi_console.log` (§6.6); unguarded, the import chain dies silently on every console-less boot — voice dead, no error anywhere |
| Uncaught exception in a Qt slot (**defused**) | PyQt6 qFatal aborts the process (`Qt6Core.dll` 0xc0000409); custom `sys.excepthook` (§6.6) logs to `lgi_console.log` and survives |
| Second-instance GPU race (**defused v1.5.1**) | igniting a second LGI session while another is live double-stacks every CUDA/dxcam/webcam/mic consumer — the newcomer heap-corrupts within ~1 s: live proof Oct 4 15:25:20→21 (v1.5.0 booted fully beside the live 08:38 session, wrote marker pid=30564, `ntdll` 0xc0000374 event at 15:25:21, dump `pythonw.exe.30564.dmp` 336 MB). `lgi.py` v1.5.1 takes the `Global\LGI_X17_SingleInstance` mutex before GPU/audio init: the second launch logs `second instance refused` and exits; the OS frees the mutex when the owning process dies, so guarded crash-restarts never wedge |
| Flaky native crash (**WATCH**) | an intermittent GPU-stack class — `cudnn64_9.dll` 0xc0000409 / `ntdll` heap corruption 0xc0000374 / `ucrtbase.dll` 0xc0000409 — striking from STT init **and** steady state, unrelated to TTS. Six dumps Sep 30, 2026 (WER `%LOCALAPPDATA%\CrashDumps`, 432–460 MB each): 18:00:36, 18:01:35 (event 18:01:20 `Qt6Core.dll` 0xc0000409), 18:04:04, 18:22:47, 19:06:27, and 19:37:54 — the 19:37:54 one is the first **steady-state** kill: pythonw PID 14192, ucrtbase fail-fast 0xc0000409 (event ≈19:37:36), 4½ min post-boot while listening with **zero chat dispatches in flight** (`pythonw.exe.14192.dmp`, 460 MB — analysis pending). The `LGI AutoStart` restart-on-failure (PT1M × 3) is present in the task XML but **did not fire** on that crash (dead zone 19:37:24 → manual relaunch 19:39:32; suspected on-demand-start non-application) — so the belt-and-braces `LGI Watchdog` task (§10 item 14) relaunches LGI every minute the moment the pythonw pair vanishes. Forensics: `crash_query.ps1`. **Oct 4, 2026: `LGI AutoStart` + `LGI Watchdog` tasks DISABLED (manual-ignition mode, §10 item 9) — crashes no longer auto-recover; relaunch via `schtasks /run /tn "LGI Manual"` or the desktop shortcut. Oct 4, 2026 evening: `LGI Watchdog` RE-ENABLED as GUARDED — resurrects only when lgi.py's `D:\LGI\.session_active` marker exists (§10 items 9+14); operator quits and power cycles always stay down. Oct 4 late evening (v2.1): the class recurred ~daily while unguarded — pythonw Application-Error events Oct 2 16:17:32, Oct 3 10:41:25 + 10:47:08 + 12:01:04, Oct 4 08:31:49 (dumps 17476/28060/18080/19204/12900 — all 0xc0000374 ntdll or 0xc0000409 ucrtbase, same fault offsets), silent to Python every time. Root cause still pending: freshest dump is tonight's pristine 336 MB `pythonw.exe.30564.dmp` under `%LOCALAPPDATA%\CrashDumps` — WinDbg `!analyze -v` requires Debugging Tools installed on the X17 (operator decision; also candidate: `LGI_STT_DEVICE=cpu` A/B). **Oct 4 late night: `crash_analyze.ps1` v1.0 prepped on `D:\LGI` (§10 item 17) — `-Posture` audit + cdb `!analyze -v` driver + `-Install` staging + `-EnsureDumps` capture pin all ready. Oct 4 ~16:00 the plan paid off immediately: `-Install` proved the winget `Microsoft.WinDbg` 1.2606.22001.0 MSIX ships `amd64\cdb.exe` — debugger in place with ZERO UAC (the SDK fallback was never needed), and three `!analyze -v` postmortems landed: **30564 = `DOUBLE_FREE ucrtbase!free_base` (GPU-race victim); 12900 (Oct 4 08:31 solo) + 19204 (Oct 3 12:01 solo) = `DOUBLE_FREE cv2.pyd!unknown_function` — the daily spontaneous class is a native double-free inside OpenCV's cv2.pyd, surfaced at `ntdll!RtlReportFatalFailure`, NOT the voice stack; `LGI_STT_DEVICE=cpu` demoted — the mitigations to weigh are camera-path observation windows (`LGI_WEBCAM=0` / `LGI_WEBCAM_TRACK=0`) or a cv-wheel pin A/B.** |
| Alert-slot exception (**WATCH**) | an exception preceded the Sep 30 18:03:21 abort whose full traceback was not captured (ruled stale/interleaved); if it recurs the traceback now lands in `%TEMP%\lgi_console.log` |

## 12. Security & Privacy

* **Wake-word gate** — with `LGI_WAKE_REQUIRED=1` (default) only wake-word
  utterances are transmitted; ambient speech is transcribed locally for display
  and **never leaves the machine**.
* **Voice-path traces stay on the X17 (v1.3.1)** — the `[stt] …`, `[voice] …`
  and `[qt] tick` diagnostic lines (§10 item 14) write only to
  `%TEMP%\lgi_console.log`; utterance text they carry is ambient-privacy
  equivalent (machine-local, never transmitted).
* **Talk window is the deliberate exception (v1.3)** — while a
  `LGI_TALK_WINDOW` (default 180 s) is open after an exchange, follow-up speech
  is transmitted without the wake word by design; it expires back to the strict
  gate, and `LGI_TALK_WINDOW=0` restores gate-only transmission.
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
* **Access control** — the chat gateway accepts the X17 IP for the socketio
  sensor feed (radar/vision handlers are IP-ungated), and the Burst Receiver
  REST allowlist carries it as `LGI_X17_IP` (192.168.50.227) with storage
  under `/data/freeroam/x17/`; audit payloads carry the operator identity
  (`rich`); vault persistence decisions belong to the gateway. The legacy
  MissPi (192.168.50.179) allowlist entries were removed Sep 30, 2026.

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
`polaris-ai:latest` over Ollama `/api/generate`).

*Sep 30, 2026 (voice-restoration hardening)* — pythonw console guard +
custom `sys.excepthook` shipped in `lgi.py`; `_trace` diagnostic tee in
`audio_tts.py` — made permanent, 1 MB truncate-on-open cap. Root cause of the silent-voice era: kokoro import death under
console-less boots (`sys.stdout = None`, huggingface_hub stdout warning) —
the output-device switch was a red herring. TTS chain live-verified end to
end (Kokoro CUDA → sounddevice → headphones) and operator-heard E2E. Probe
tasks deleted (§10 item 12); defused + watchlist entries in §11.*

*Sep 30, 2026 (v1.3.1 crash forensics & watchdog)* — first steady-state native
crash (ucrtbase 0xc0000409, pythonw PID 14192, dump kept) + a restart policy
that didn't fire + a fully deaf boot produced the `LGI Watchdog` task and the
voice-path trace set (`[stt] mic open / onset / transcribing / transcribe done
/ empty`, `[voice]`, `[qt] tick`) — see §10 item 14 and the updated §11 WATCH
row (six dumps today). Deployed with MD5 parity: `53a26717` lgi.py,
`d3b93fd0` audio_stt.py, `e473ec9a` watchdog.ps1 (re-fixed to ASCII/CRLF after a
PS 5.1 parse failure on the em dashes — see item 14; LGI.md parity verified live
at deploy time).*

*Sep 30, 2026 (v1.3.1a windowless watchdog)* — operator-reported console window
flashing open/closed every 60 s: the `LGI Watchdog` task ran bare `powershell`
(console-subsystem) in the interactive session each minute. Action is now
`wscript.exe D:\LGI\watchdog_hidden.vbs` — a GUI-subsystem hidden launcher
(`Run(…, 0, False)`; launcher errors → `%TEMP%\lgi_watchdog_launcher.log`,
never a dialog). Task flags preserved via XML round-trip (the
`<RunLevel>HighestAvailable</RunLevel>` would have been silently dropped by a
plain `schtasks /create`). Verified live: minute tick + `/run` both `Last Result: 0`,
pythonw pair untouched. MD5 parity: `f0d18ae8` watchdog_hidden.vbs; LGI.md
parity verified live at deploy time.*

*Sep 30, 2026 (v1.3.2 deafness hardening)* — operator deaf at 20:01:25 with
no deliberate action: root-caused to the palm-gesture mic toggle (dwell
survived the hand-too-small early-return, so flicker accumulated into false
fires; a cooldown-blocked fire left the dwell armed, so a held palm flapped
the mute; mic logs carried no source). Now: every mic pause/resume logs its
source (`hotkey|gesture|hud` + ducking traces), gesture dwell is a
continuous 2.0 s with a 6 s cooldown that always clears the dwell, and
degenerate whisper transcripts (`A. . A. .`) are junk-filtered at the STT
boundary. LGI restart required to take effect; MD5 parity recorded at deploy
*Oct 3, 2026 (v1.4 deploy reset + gesture resurrection)* — the v1.4 TTS pause/stop
push surfaced two latent failures, both fixed. **(1) Nested push:** `scp -r LGI
… D:/LGI` copies the folder *into* the target as `D:\LGI\LGI\`, leaving the
stale root build live while fresh code sits unexecuted inside the nested
folder; §10 step 1 now pushes with a glob (`scp -r LGI/* …`) and mandates
post-push MD5 parity (§10 step 1), the auto-start example documents the real
task interpreter (`D:\LGI\venv312\Scripts\pythonw.exe` — the old AppData
pythonw path was wrong for the live task; `Last Result 267009` = 0x41301
task-is-running, not an error), and the validated reset recipe is
`schtasks /end` + `taskkill /F /IM pythonw.exe` mop-up + `/run` (raw process
kills never stick — the watchdog resurrects the pythonw pair every 60 s).
**(2) Gesture outage** (`[perception] mediapipe init failed (module
'mediapipe' has no attribute 'solutions')`): pip satisfied the old
`mediapipe>=0.10.9` floor with 1.0.1, and the 1.0.x line *and* the 0.10.30+
builds have the legacy `mp.solutions` API stripped. venv312 rebuilt to
`mediapipe==0.10.21` (last legacy-API build; pins numpy<2 → numpy 1.26.4)
with `opencv-python` + `opencv-contrib-python ==4.10.0.84` (opencv 5.x needs
numpy≥2 and ABI-broke cv2 under that matrix; 4.10.0.84 is camera-proven on
the X17 DirectShow path). requirements.txt exact-pins all three; the §10
self-check runs on venv312 and asserts `mp.solutions` so the next env rebuild
*Oct 3, 2026 (v1.4.2 — silent wake entry, tool visibility, workspace-reading
copilot)* — three complaints closed with runtime-evidence root causes: (1)
"wakeward mode" chatter: bare `polaris` now opens the 180 s wake-free window
SILENTLY (ambient `(listening — no wake word needed)`) and every talk-window
open/renew cue is deleted (§5.1, §9) — she just speaks when she has something
to say. (2) "answer came with the ack, not the answer": root cause was Edit E
by construction — `/api/chat` puts the first-pass ack into the HTTP body while
the truthful tool report is a room broadcast LGI never speaks. The gateway now
emits device-tagged `chat_tool_report` (`origin_device: lgi-hud-x17`); LGI
speaks it and swaps it into the exchange only for its own asks, tool runs
render as ⚙ CHAT_TOOL task rows (TASK_ROW_STYLES + `_render_tool_execution`)
and chat-path tool turns register an aggregate TaskManager task on the gateway.
(3) `LGI_CHAT_TIMEOUT` env (default 180 s; was 65 s) covers the 2-LLM-pass
tool pipeline (§6.1/§8). Gateway also gained read-only workspace eyes:
`list_workspace_files` + `read_workspace_file` over the new
`/data/workspace:ro` compose mount (the whole /home/causeiam/docker-containers
*Oct 4, 2026 (manual ignition — operator request; changed from the Mainframe via
SSH, no LGI code touched)* — the operator now fires LGI up personally instead of
schedule magic at boot: `LGI AutoStart` (onlogon) and `LGI Watchdog` (60 s
relauncher via `wscript.exe D:\LGI\watchdog_hidden.vbs`) are **Disabled** — the
watchdog log showed the boot→resurrect dance on every boot (latest:
`2026-10-04 08:38:02 LGI down (pythonw x0) - relaunching via 'LGI AutoStart'`).
Replacement: **no-trigger `LGI Manual` Scheduled Task** (same venv pythonw
action, Interactive-only, RunLevel Highest, no execution-time limit) — Ready in
Task Scheduler but NEVER self-firing; ignition = the desktop shortcut
`LGI (manual).lnk` (`OneDrive\Desktop`, same pythonw line, created Oct 4) or
`schtasks /run /tn "LGI Manual"` from any box (X17 cmd, the `:7007` Sandbox
dashboard, Polaris `ssh_execute`, hand SSH). XML backups of the retired tasks:
`D:\LGI\backup_tasks\LGI_AutoStart.xml` + `LGI_Watchdog.xml`. Rollback:
`/enable` both tasks (§10 item 9). Verified live: retired tasks Disabled,
`LGI Manual` Ready/Enabled/Next-Run N/A, running pythonw pair untouched;
supervisor audits + `vision_frame` telemetry flowed through the whole change.*
tree) — Polaris can pull trading-bot.md / LGI code into the conversation with
no SSH hop, grounding the operator↔Polaris↔Cline three-way loop (§8 gateway
contracts: read-only by design; edits stay with operator + Cline). MD5
parity + validated reset recipe applied at deploy; gateway needs one
`polaris-gateway` rebuild for the emit/task/mount changes.*
cannot silently kill gestures again. `webcam_perception.py` camera bring-up
is now fully guarded with a 3×2 s retry ladder (the 12:00 boot race in which
the dying instance briefly held the camera crashed an unguarded `cap.set` and
killed the perception thread). Verified live 12:02:06 boot: `[perception]
running — camera LED stays ON while tracking (face yes, hands yes, yolo yes)`.
WATCH: that boot logged `dxcam unavailable (DXGI -2005270524)` and fell back
to mss one-shot screen capture (documented degradation ladder; recheck dxcam
after the next natural reboot if vision frames look coarse).*
time.*

*Oct 4, 2026 evening (v2.0 guarded watchdog + gateway voice journal — improvement
items 3 + 6)* — (1) **Guarded relauncher**: `watchdog.ps1` v2.0 + `lgi.py` v1.5.0
session-marker protocol (`.session_active` — written after a successful start,
cleared on graceful exit); the `LGI Watchdog` task is RE-ENABLED but only
resurrects crash-killed ignited sessions, while boots and operator quits stay
down (§10 items 9+14, §11 WATCH). Deployed repo→scp with MD5 parity
(`3222dc89` lgi.py, `8c6ce328` watchdog.ps1); dry-run proof logged 15:18:01→03;
live pythonw pair untouched. (2) **`memory_journal_write`** gateway tool —
"Polaris, log this" appends a timestamped, instantly embedded entry to
`04-Journal/YYYY-MM-DD.md` via the vault's append-only journal path: the ONLY
chat-side vault write (source `chat-journal`, 2,000-char cap, titles from the
entry's first words; prompt section [MEMORY VAULT] + few-shot updated). Needs
one polaris-gateway rebuild + restart.*

*Oct 4, 2026 late evening (crash forensics + v1.5.1/v2.1 hardening — operator
report "for some reason, LGI crashes")* — `crash_query.ps1` + console log told
two stories: (a) the §11 flaky-native class recurred ~daily while the watchdog
was unguarded (pythonw events Oct 2 16:17:32, Oct 3 10:41:25 / 10:47:08 /
12:01:04, Oct 4 08:31:49 — always ntdll 0xc0000374 or ucrtbase 0xc0000409,
silent to Python); (b) a fresh v1.5.0 ignition at 15:25:05 beside the still-live
08:38 session booted fully ("LGI online" 15:25:20, marker pid=30564) and was
heap-corruption-killed at 15:25:21 — two CUDA stacks + two dxcam/mic pipelines
racing one RTX 3080 Ti. Response: orphaned marker deleted; stale pre-v1.5.0
session stopped; one clean `LGI Manual` ignition cutover to v1.5.1; `lgi.py`
v1.5.1 `Global\` single-instance mutex (second launch refuses before GPU/audio
init — live-tested "second instance refused"); `watchdog.ps1` v2.1 clears the
stale marker at relaunch (boot-crash chains self-arrest) + crash-loop brake
(≤3 guarded relaunches / 10 min, cooldown slow-retry while the marker persists,
streak expires after a healthy 10-minute session). Spontaneous-class root
cause still WATCH: WinDbg `!analyze -v` on the pristine 336 MB
`pythonw.exe.30564.dmp` is the next step (needs Debugging Tools on the X17 —
operator decision).*

*Oct 4, 2026 late night contd. (crash-postmortem readiness — "prepare in case
there is an error")* — `crash_analyze.ps1` v1.0 (317 lines, md5 `5b408143`)
deployed repo → `D:\LGI\crash_analyze.ps1` with MD5 parity: `-Posture`
readiness audit (live pair/marker, watchdog task state, WER LocalDumps
config, dump inventory, debugger search), `-EnsureDumps` (pins HKCU
pythonw.exe full-dump capture, DumpType=2 keep 15 — until then capture rode
undocumented Windows default WER behavior), `-Install` (winget
`Microsoft.WinDbg` → Windows-SDK `OptionId.WindowsDesktopDebuggers`
fallback; stages the bootstrapper, never fires UAC itself), `-Latest`/
`-Dump`/`-All` cdb `!analyze -v` drivers → `D:\LGI\crash_reports\*.analysis.log`
+ verdict extracts (async default, `-Wait` blocks; symbol cache
`D:\LGI\symcache`). Recon: no debugger on the X17 (winget 1.29.380 present),
8 pythonw dumps 321-414 MB (newest `pythonw.exe.30564.dmp` 336 MB @ Oct 4
15:25:34). X17 `LGI.md` copy refreshed in the same deploy (had still been the
stale Oct 3 revision). §10 item 17, §11 WATCH row, §3 file map, Codebase row
updated.*

*Oct 4, 2026 ~16:00 (verdicts — the postmortem prep paid off same night)* —
`-Install` ran via the one-shot Scheduled-Task pattern (direct
Start-Process/ssh async children die instantly under Windows OpenSSH's
job-object teardown — both attempts left 0-byte logs; Task Scheduler tasks
survive, which is now the documented async recipe): winget `Microsoft.WinDbg`
1.2606.22001.0 installed per-user, ZERO UAC — the MSIX ships
`amd64\cdb.exe`/`kd.exe`, resolved live by `Find-Debugger`'s Appx branch
(the "UI-only" fear was wrong; SDK fallback unneeded). Three `!analyze -v`
runs (symbols cached to `D:\LGI\symcache` after the first): **30564 =
`HEAP_CORRUPTION_ACTIONABLE_BlockNotBusy_DOUBLE_FREE_c0000374_ucrtbase.dll!free_base`
(GPU-race victim), 12900 + 19204 = the same bucket with
`cv2.pyd!unknown_function` — the ~daily spontaneous class is a native
double-free inside OpenCV's cv2.pyd (`ntdll!RtlReportFatalFailure`
fail-fast), not the voice stack.** Consequence: `LGI_STT_DEVICE=cpu` A/B
demoted; camera-path observation windows (`LGI_WEBCAM=0`/`LGI_WEBCAM_TRACK=0`)
or a cv-wheel pin A/B are the operator's experiment menu (§11 WATCH).
`-EnsureDumps` pinned HKCU pythonw capture (full dumps, keep 15). §10 item
17 carries the full story.*
