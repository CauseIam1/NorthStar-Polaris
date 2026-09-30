# FreeRoam Tactical Edge-Compute System

## Overview

A covert, edge-compute architecture for real-time AI-assisted situational awareness on the **Red Magic 9 Pro**. FreeRoam features automatic multi-user identity detection, a walkie-talkie loop with Polaris (AI) via the Polaris Gateway — **voice in** (watch PLAY/PAUSE push-to-talk or the in-app mic button, both working with the screen locked) and **voice out** (fully on-device TTS synthesis) — every AI reply is spoken by a local Kokoro engine (sherpa-onnx), streamed sentence-by-sentence while the reply generates (speak-while-she-writes, v3.5) — and since v3.6, playback is fully text-only: no WAV artifacts, every bubble replays by re-synthesizing from text. No cloud TTS. No server-side audio round-trips.

- **Current app version:** versionName **1.0.4** (versionCode 5 — Sep 28, 2026) — deployed and operator-verified; voice engine self-heal field-confirmed
- **Voice push-to-talk (v3.3):** watch PLAY/PAUSE / mic button → `VoiceCaptureController` (`SpeechRecognizer`, on-device engine preferred) → live partials in the input field → stop → auto-send — captures survive screen-lock via the mic-typed foreground service
- **Voice engine self-heal (v1.0.4):** the RedMagic's on-device speech engine can wedge mid-capture — no callbacks at all — and every later press hears nothing until a reboot (paired with a device-level logd freeze on this ROM). `finalizeAndDeliver()` now counts silent captures (`SILENT_CAPTURES_BEFORE_ENGINE_HOP`) and arms an engine hop: the next PTT press is served by the **system (cloud) recognizer** instead of the wedged on-device engine; a successful capture clears the hop and returns to the private engine. A wedge costs one failed press, no reboot.
- **Android build signing (v1.0.4):** the debug keystore is committed (`android/app/freeroam-debug.keystore`) and pinned into the build image (`freeroam/Dockerfile` → `/root/.android/debug.keystore`) so every build signs identically and `adb install -r` upgrades in place — fresh images previously generated a new key, forcing uninstall + reinstall. Note: ADB-over-TCP (5555) does not survive phone reboots on this ROM — re-arm via USB once; the gateway's watchdog (`run_gateway.sh`) reconnects automatically.
- **Reboot-proof wireless ADB (v1.0.4 era):** Wireless debugging is ON (`Settings.Global adb_wifi_enabled=1`) and the Mainframe is paired (pair GUID `adb-FY24031100E6-71NED1`). The Mainframe runs Google platform-tools v37 at `~/platform-tools/adb` (v37 has mDNS + TLS pairing; the distro adb v28 on `/usr/bin` does not). After any phone reboot the device re-appears automatically on the host as `adb-FY24031100E6-71NED1._adb-tls-connect._tcp` in `adb devices` — **no USB, no `adb tcpip`, no manual step**. From Polaris's sandbox (whose ADB is Debian 34, no mDNS): reach the phone through `ssh_execute` on the Mainframe → `~/platform-tools/adb`; the container's `:5555` bridge remains the quick path whenever TCP mode is armed.
- **Matt's phone wireless ADB (field-verified Sep 30, 2026):** Matt's RedMagic 9 (`FY24031100BE`, Android 16) is Wi-Fi-connected the same way: wireless debugging is ON and the Mainframe is TLS-paired (pair GUID **`adb-FY24031100BE-usU8h8`**, pairing is persistent across reboots). The phone serves two endpoints: legacy tcpip adbd on **5555** (static, currently armed — resets on reboot like Rich's) and the paired TLS wireless-debugging endpoint on a **dynamic** port (33679 at pairing time — re-discover with `adb mdns services`, connect to the `_adb-tls-connect` entry). One-touch recovery with no USB: `freeroam/polaris-gateway/rearm_phones.sh` (`--tcpip` also tries re-arming 5555 over the wireless transport; it re-asserts Matt's ACCESS_FINE_LOCATION grant and relaunches the app if it died). The gateway container's `run_gateway.sh` watchdog now arms **both** phones on 5555 (Rich 192.168.50.42 | Matt 192.168.50.98); from the sandbox only that 5555 path exists (Debian adb v28, no mDNS/TLS).
- **Watch tether (v3.3; play-state model v3.5.2):** the AVRCP media session stays listed with a silent 60 s seed track. While she speaks the seed loops silently (session PLAYING — the watch's pause button stops her speech and starts the mic); the moment her voice drains, the seed parks PAUSED and her full reply takes the track title with a fresh mediaId (a "new track" the Garmin reliably refreshes and marquee-scrolls) — so the watch ends every exchange showing her reply and a PLAY button, ready for push-to-talk
- **Strict text-chat loop:** speak or type → send → gateway → LLM → response → spoken aloud (v2.9 hardening: text in / text out only — voice capture just feeds the same text pipeline)
- **On-device speech pipeline (v3.1; text-only since v3.6):** `KokoroTTS` → `StreamingTtsPlayer` direct PCM playback (no audio artifacts); `PolarisAudioStore` remains only for the legacy WAV sweep

> **Gateway internals** — service architecture, full port map, baked-source policy, and `run_gateway.sh` — are documented authoritatively in [`Polaris-gateway.md`](Polaris-gateway.md). This file covers the end-to-end system with emphasis on the Android edge application.

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                 RED MAGIC 9 PRO (edge device)                  │
│                                                                │
│  MainActivity (Polaris UI)                                     │
│  ├─ Top nav: status • identity • voice • speed • media          │
│  ├─ Rolling 10-message chat window (Room-backed)                │
│  └─ Mic button • auto-speak • bubble playback • mini player    │
│                                                                │
│  On-device TTS pipeline                                        │
│  ├─ KokoroTTS            sherpa-onnx, 24 kHz mono PCM16         │
│  ├─ PolarisAudioStore    polaris_<id>.wav cache (MediaStore)    │
│  └─ MediaSessionManager  ExoPlayer playback (speed-aware)       │
│                                                                │
│  Network layer                                                │
│  ├─ PolarisWebSocketClient  ws://192.168.50.51:8082             │
│  │    (chat events, stateful auto-reconnect)                    │
│  ├─ ChatRepository          "dumb terminal" over gateway API    │
│  └─ NetworkRepository / PolarisGateway (REST endpoints)         │
│                                                                │
│  Voice capture (walkie-talkie input)                           │
│  ├─ VoiceCaptureController  SpeechRecognizer STT + partials    │
│  │    PARTIAL wake lock • 30 s cap • 2.5 s end-of-speech        │
│  └─ MediaSessionManager     WatchSessionPlayer: watch PLAY/    │
│       PAUSE → toggleVoiceCapture() (AVRCP media buttons)       │
│                                                                │
│  Tactical services                                             │
│  ├─ TacticalBackgroundService  mic FGS (dataSync|microphone)   │
│  │    started by MainActivity.onStart() (START_STICKY)         │
│  └─ ClipboardMonitorService    clipboard → TTS readout         │
│                                                                │
│  Local storage                                                │
│  ├─ ConversationDatabase (Room)   chat_db                      │
│  └─ ChatArchiverWorker             periodic transcript archive  │
│                                                                │
│  Overlays: PolarisAvatarView (AI face) • MiniPlayerView         │
└───────────────────────────────────────────────────────────────┘
                          │ LAN (192.168.50.x)
                          ▼
┌───────────────────────────────────────────────────────────────┐
│              POLARIS GATEWAY (192.168.50.51)                    │
│  ├─ Chat Gateway (Flask-SocketIO)   host port 8082             │
│  ├─ Burst Service                   host port 8083             │
│  ├─ Sandbox Service                 host port 7007             │
│  ├─ Ollama LLM backend              port 11434                 │
│  └─ Per-user context under /data/freeroam/<user>/               │
│      (profiles • transcripts • behavior libs)                  │
└───────────────────────────────────────────────────────────────┘
                          ▼
┌───────────────────────────────────────────────────────────────┐
│         POLARIS DASHBOARD (Next.js 14, port 7000)              │
│  ├─ Real-time chat monitoring across all users                  │
│  ├─ Multi-stream visualization (Rich / Matt / Operator)         │
│  └─ Operator transcript + history API access                    │
└───────────────────────────────────────────────────────────────┘
```

## User Identity Mapping

### Automatic IP-Based Detection (Android App)

The FreeRoam Android app **automatically detects user identity** from the device's local WiFi IP address — no login, no configuration:

| Device IP Last Octet | User Identity | Notes |
|---------------------|---------------|-------|
| `.42` | Rich | Rich's Red Magic 9 Pro |
| `.98` | Matt | Matt's device |
| Other | Rich (fallback) | Unknown devices default to Rich (logged) |

- Before an IP is resolved, the identity chip shows the placeholder `"Dashboard"`.
- Detected `userIdentity` + `deviceIpAddress` are stored in the WebSocket client and reused verbatim on reconnect — identity never drifts between sessions.
- Tap the identity button (top-right) to confirm: `Connected as: <identity> (<ip>)`

### Platform Summary

| Platform | User Identity | IP Address | Gateway Profile |
|----------|--------------|------------|------------------|
| FreeRoam (Rich's Red Magic 9 Pro) | Rich | 192.168.50.42 | `/data/freeroam/rich/` |
| FreeRoam (Matt's device) | Matt | 192.168.50.98 | `/data/freeroam/matt/` |
| Dashboard (Operator) | Operator | 192.168.50.51 | `/data/freeroam/operator/` |

## Key Features

### 1. Zero-Config Multi-User Support

- **Same APK for all users** — identity is derived automatically from the LAN IP
- Each user maintains a separate server-side chat history and gateway profile
- Unknown devices fall back to the Rich identity client-side (logged, not blocked)
- **Reinstall fragility (field-verified Sep 30, 2026):** runtime grants survive `adb install -r` but NOT uninstall/reinstall or "clear data" — a reset APK boots with `Dashboard` → identity falls back to `Rich` until ACCESS_FINE_LOCATION is granted and the app restarts. Repair:
  `adb -s 192.168.50.98:5555 shell pm grant com.freeroam.tactical android.permission.ACCESS_FINE_LOCATION && adb -s 192.168.50.98:5555 shell am force-stop com.freeroam.tactical ; adb -s 192.168.50.98:5555 shell monkey -p com.freeroam.tactical -c android.intent.category.LAUNCHER 1` — or just run `rearm_phones.sh`, which self-heals this (grant re-assert + relaunch) whenever Matt's phone is reachable.

### 2. Walkie-Talkie Flow (v3.3)

```
Voice input (v3.3): watch PLAY/PAUSE press or mic button tap
  → toggleVoiceCapture() → duck her TTS → VoiceCaptureController.startListening()
      → live partials stream into the input field (mic icon tinted red)
  → second press/tap → stopListening() → final transcript → auto-send

Text input
User types text → [Send]
  → PolarisWebSocketClient.sendChatMessage()          ("✓ Sent" toast)
  → Polaris Gateway :8082 → Ollama LLM (user profile + behavior lib)
  → response broadcast → onChatMessage()
  → auto-speak gate: only AI messages < 5 s old are spoken
  → synthesizeMessageToFile(autoPlay = true)
      ├─ KokoroTTS.setVoice / setSpeed
      ├─ KokoroTTS.synthesize(text)     → 24 kHz mono PCM16
      ├─ PolarisAudioStore.savePcmAsWav → polaris_<id>.wav (MediaStore)
      └─ playPolarisMessage()
          └─ MediaSessionManager.playUri(uri, currentSpeed)  [ExoPlayer]
  → PolarisAvatarView (AI face) + PolarisMiniPlayerView (controls)
  → message bubble becomes replayable audio
  → natural end: STATE_ENDED → 600 ms → watch tether (pause + rewind to 0:00)
      → Garmin keeps marquee-scrolling her reply as the track title
```

### 3. Audio Bubble Playback

- Tap any Polaris message bubble to toggle playback (`toggleBubblePlayback`)
- Cached WAV present: play → pause → resume → restart on subsequent taps
- Cached WAV missing: **synthesize-on-demand** (generate → save → play)
- Audio is synthesized once per message and reused from the cache thereafter

### 4. Mini Player

- `PolarisMiniPlayerView` overlay hosts play/pause, stop, close, and seek controls
- Drives the shared ExoPlayer instance held by `MediaSessionManager`

### 5. Auto-Speak Behavior

- Fresh AI responses (age < 5 seconds) are spoken automatically
- The 5-second gate prevents re-speaking old messages when the rolling window or chat history rebuilds
- During periodic history sync, **genuinely new** Polaris messages received from other devices are spoken, with a 1-second `lastSpokenTimestamp` dedupe gate preventing double-speak

### 6. Voice Options

- **Default:** `af_emma` • **Alternatives:** `af_heart`, `af_sarah`, `af_bella`
- Selector in the top navigation bar; applies to all subsequent synthesis

### 7. Speed Control

- Slider range 0–100 maps to **1.0x – 2.0x** via `speed = 1.0 + (progress / 100.0)`
- Default: **1.0x** (slider at 0)
- Applied to both the ExoPlayer playback rate (`setPlaybackSpeed`) and the TTS engine (`setSpeed`)

### 8. Rolling 10-Message Window

- The chat displays only the **last 10 messages** (`MAX_CHAT_MESSAGES = 10`)
- Applies identically to new incoming messages and loaded history (chronological, trimmed to the most recent 10)
- Pruning is UI-only — server-side storage retains the full history

### 9. Clipboard Readout (Control+S Equivalent)

- `ClipboardMonitorService` mirrors the X17 laptop's Control+S hotkey on Android
- Reads the primary clipboard, synthesizes via KokoroTTS, and plays through Kokoro's built-in AudioTrack player with an amplitude-tracked completion callback
- Skips empty or unchanged clipboard content; auto-stops after playback; absorbs native ONNX init failures so the process survives model problems

### 10. Garmin Watch Media Controls & Push-to-Talk

- **PLAY/PAUSE → push-to-talk toggle** (v3.3): 1st press starts a capture, 2nd press stops + sends — works with the screen locked
- **STOP → cancel capture** (mid-PTT, nothing sent) or **halt playback tethered** — the session stays listed and her reply keeps scrolling
- **BACK / NEXT → disabled** (v3.3): stripped from the advertised AVRCP commands; presses are logged and dropped
- **Watch tether:** the media session never dies — silent seed at launch, hold-paused at 0:00 after every response (details: [Garmin Watch PTT Integration](#garmin-watch-push-to-talk-ptt-integration))

### 11. Smart WebSocket Reconnection

- Stateful auto-reconnect with no identity drift: on reconnect, the client re-joins with the same stored `userIdentity` / `userIp`
- Connection is kept alive across app pause; on resume, reconnect occurs only if actually disconnected

## Android App Components

All paths relative to `android/app/src/main/java/com/freeroam/tactical/`:

| Component | Path | Role |
|-----------|------|------|
| `MainActivity` | `MainActivity.kt` | UI, chat orchestration, voice push-to-talk toggle (`toggleVoiceCapture()`), auto-speak gating, bubble playback, speed control |
| `KokoroTTS` | `voice/KokoroTTS.kt` | On-device TTS engine (sherpa-onnx, 24 kHz mono PCM16) |
| `PolarisAudioStore` | `media/PolarisAudioStore.kt` | WAV cache (`polaris_<id>.wav`) via MediaStore, app-dir fallback |
| `VoiceCaptureController` | `voice/VoiceCaptureController.kt` | Push-to-talk STT (`SpeechRecognizer` wrapper, PARTIAL wake lock, safety timeouts) |
| `MediaSessionManager` | `media/MediaSessionManager.kt` | ExoPlayer/Media3 session playback + watch AVRCP media-button routing (v3.3: PLAY/PAUSE = push-to-talk toggle, STOP = cancel/halt, BACK/NEXT disabled) + the always-on paused-session tether |
| `PolarisWebSocketClient` | `network/PolarisWebSocketClient.kt` | Socket.IO-style chat transport, stateful reconnection |
| `PolarisWebSocketListener` | `network/PolarisWebSocketListener.kt` | Callback interface (onChatMessage, onChatHistory, onMessageSent, …) |
| `NetworkRepository` | `network/NetworkRepository.kt` | Gateway REST endpoint layer |
| `PolarisGateway` | `network/PolarisGateway.kt` | Gateway API definitions |
| `ChatRepository` | `data/ChatRepository.kt` | "Dumb terminal" wrapper over the gateway chat API |
| `ChatMessage` | `data/ChatMessage.kt` | Message model (user, text, timestamp, isUser) |
| `ConversationDatabase` | `database/ConversationDatabase.kt` | Room database (`chat_db`) |
| `ChatArchiverWorker` | `worker/ChatArchiverWorker.kt` | WorkManager periodic transcript archival |
| `ClipboardMonitorService` | `service/ClipboardMonitorService.kt` | Clipboard → TTS readout |
| `TacticalBackgroundService` | `service/TacticalBackgroundService.kt` | Mic-capable foreground service (`dataSync\|microphone`, notification ID 1001), started by `MainActivity.onStart()` |
| `PolarisAvatarView` | `PolarisAvatarView.kt` | Eve-style AI face overlay (hidden/active/paused states) |
| `PolarisMiniPlayerView` | `ui/PolarisMiniPlayerView.kt` | Mini player overlay (play/pause/stop/close/seek) |
| `AppConfig` | `config/AppConfig.kt` | Central configuration constants |

---

## Component Details

### MainActivity UI Layout

- **Top navigation bar:** connection status indicator → identity button (`Connected as: …`) → voice selector (`af_emma` default) → speed slider (1.0x–2.0x) → media controls / mini player toggle
- **Chat area:** rolling 10-message window; user messages right-aligned, Polaris messages left-aligned with a per-bubble replay button
- **Input row:** text field + send button; sent messages toast `✓ Sent` and clear the input

### KokoroTTS (`voice/KokoroTTS.kt`)

On-device neural TTS powered by **sherpa-onnx** (Kokoro model, espeak-based phonemization, int8 quantized inference). Output is **24 kHz mono 16-bit PCM**.

**Public API**

| Function | Purpose |
|----------|---------|
| `setVoice(voiceId)` | Select speaker (`af_emma` default; `af_heart` / `af_sarah` / `af_bella`) |
| `setSpeed(speed)` | Set synthesis speed (1.0–2.0) |
| `synthesize(text): ByteArray` | Generate PCM16 audio for the current voice + speed |
| `playAudio(pcmData, onComplete)` | AudioTrack playback (USAGE_MEDIA routing) with amplitude tracking and a completion callback |
| `pause()` / `resume()` / `stop()` | Playback control; `stop()` halts audio, releases the AudioTrack, and clears state |
| `setAmplitudeCallback` / `isCurrentlyPlaying()` | Visualizer support / state query |

**v3.1 fix — the static-noise bug:** PCM16 conversion (`floatToPcm`) now uses `.order(ByteOrder.nativeOrder())` on its ByteBuffer. Previously the buffer defaulted to BIG_ENDIAN, byte-swapping every 16-bit sample on little-endian ARM — speech played as static that followed the speech envelope. The fix restores true audio output.

**Robustness:** `isValidModelFile()` validates model blobs (size floor + ONNX magic byte) before native init, preventing SIGSEGV crashes from corrupt model files.

### PolarisAudioStore (`media/PolarisAudioStore.kt`)

Persistent WAV cache keyed by message ID — audio is synthesized once per message, then replayed from cache:

- `savePcmAsWav(context, pcmData, messageId, sampleRate): Uri?` — writes `polaris_<messageId>.wav` into the MediaStore `Polaris` directory (app-dir fallback)
- `findExistingWav(context, messageId): Uri?` — cache lookup (fallback path validates file length > 44-byte header)
- `buildWav()` constructs the 44-byte RIFF header (LITTLE_ENDIAN) around the PCM payload

### MediaSessionManager (`media/MediaSessionManager.kt`)

Dual-role component:

1. **Playback** — owns the shared ExoPlayer + MediaSession; `playUri(uri, speed)` respects the current speed setting; pause/resume/stop/seek exposed to the UI and mini player
2. **Wearable media buttons + watch tether (v3.3)** — `WatchSessionPlayer` (a Media3 `ForwardingPlayer`) intercepts AVRCP commands from the paired Garmin before they reach ExoPlayer, so watch buttons drive the app even when no audio is loaded. `handleWatchCommand()` maps them (500 ms debounce): PLAY/PAUSE → `onWatchPlay()` (push-to-talk toggle), STOP → `onWatchStop()` (cancel capture / tethered halt); BACK/NEXT are stripped from `getAvailableCommands()` and no-op'd. The tether keeps the session permanently AVRCP-visible: `activateSession()` seeds a silent 1 s WAV paused at 0:00, `onPlaybackStateChanged(ENDED)` rewinds to 0:00 and holds paused after 600 ms, and `stop()` pauses + rewinds instead of idling the player — her latest reply (≤ 180 chars) stays loaded as the scrolling track title

### VoiceCaptureController (`voice/VoiceCaptureController.kt`)

Push-to-talk mic capture wrapping Android's `SpeechRecognizer` behind a small PTT-shaped API:

- **API:** `startListening()`, `stopListening()` (graceful flush of the final result), `cancel()` (hard abort, no result), `isListening()`, `destroy()`
- **Callbacks** (all delivered on the main thread): `onCaptureStarted()` → mic button tinted red, `onPartialResult(text)` → partials stream into the input field, `onCaptureComplete(text)` → transcript routed through `sendMessage()`, `onCaptureError(reason)` → capture UI dismissed
- **Engine:** on-device recognition preferred (API 31+, fully offline — no audio ever leaves the phone); system recognition service as fallback; no service at all → error callback, UI degrades to text input
- **Power:** holds a `PARTIAL_WAKE_LOCK` (`freeroam:voice_capture`) for the duration of a capture, released in `finishCapture()`; the lock auto-expires after `MAX_CAPTURE_DURATION + 5 s` so a missed release can never pin the CPU
- **Safety nets:** 30 s hard capture ceiling (`MAX_CAPTURE_DURATION_MS`) and a 1.5 s stop-flush timeout (`STOP_FLUSH_TIMEOUT_MS`) for OEM recognizers that never deliver a final result after `stopListening()`

### TacticalBackgroundService (`service/TacticalBackgroundService.kt`)

Microphone-typed foreground service that promotes the Free Roam session so speech captures survive **screen-lock**:

- Manifest: `android:foregroundServiceType="dataSync|microphone"` plus `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_DATA_SYNC`, `FOREGROUND_SERVICE_MICROPHONE`, `WAKE_LOCK`, `POST_NOTIFICATIONS` permissions — the **microphone FGS type** is what lets `SpeechRecognizer` hold the mic while the display is off (runtime proof on the Redmagic: `dumpsys` reports `types=0x00000081` = dataSync + microphone)
- Started by `MainActivity.onStart()` via `startForegroundService()`; `onStartCommand()` re-asserts `startForeground()` idempotently on every start (API 31+ demands a matching `startForeground()` per start call)
- Silent, ongoing, low-importance notification on `tactical_service_channel`, ID 1001: "Free Roam Active — Voice capture + tactical monitoring active"
- Returns `START_STICKY`; a denied FGS start (sticky restart while fully backgrounded) is caught and logged instead of crashing
- Watch media buttons do **not** route through this service — they arrive through the Media3 `MediaSession` (`WatchSessionPlayer` in `MediaSessionManager`)

### PolarisAvatarView (`PolarisAvatarView.kt`)

Eve-style AI face overlay in three states: `HIDDEN` (idle), `ACTIVE` (speaking, speech-synced), `PAUSED` (breathing pulse). Fades out when playback completes.

### Local Storage

- **Room database** `chat_db` (`ConversationDatabase`) — local message persistence
- **`ChatArchiverWorker`** — WorkManager periodic job that archives transcripts server-side

### Backend User Context (Gateway)

The gateway maps each identity to a per-user context tree under `/data/freeroam/<user>/` (profile, transcripts, behavior libraries). See [`Polaris-gateway.md`](Polaris-gateway.md) for the authoritative service architecture.

---

## Garmin Watch Push-to-Talk (PTT) Integration

Garmin watch buttons drive the app over standard AVRCP media-button routing — no screen interaction required. **PLAY/PAUSE is the push-to-talk mic toggle and works with the screen locked** (the mic-typed `TacticalBackgroundService` keeps captures alive through screen-off). The **watch tether (v3.3)** keeps the AVRCP session alive indefinitely: the player is never parked in ENDED or IDLE, so the watch media screen permanently lists "Polaris AI" and marquee-scrolls her latest response as the track title until the next reply replaces it. **Field-verified Sep 26, 2026** — the operator's physical Garmin retest passed end-to-end: locked-screen PTT via PLAY, the tether hold minutes after a reply, and both STOP behaviors; the `REPEAT_MODE_ONE` reserve lever was not needed (the Garmin retains paused sessions indefinitely).

### Button Mapping

| Watch Button | Command | Effect |
|--------------|---------|--------|
| PLAY / PAUSE | `PLAY` / `PAUSE` | **Push-to-talk toggle** — 1st press starts capture, 2nd press stops + sends (AVRCP sends whichever is opposite the reported state; both land in `onWatchPlay()`) |
| STOP | `STOP` | **Cancel capture** if mid-PTT (hard abort, nothing sent; `cancel()` fires no listener callback so the stop handler resets the mic tint) — otherwise **halt playback tethered** (pause + rewind to 0:00; the session stays listed) |
| BACK | `BACK` | Disabled (v3.3) — stripped from advertised commands; logged and dropped |
| NEXT | `NEXT` | Disabled (v3.3) — stripped from advertised commands; logged and dropped |

Button events are debounced at **500 ms** to prevent duplicate triggers. A silent 1 s seed WAV is loaded paused at session activation, so the watch sees a live (paused) player from launch; finished responses rewind to 0:00 and hold paused — the tether. Title metadata carries up to 180 chars of her reply for the Garmin marquee.

### Architecture

```
Garmin watch button press
  → AVRCP media-button event → Media3 MediaSession
  → WatchSessionPlayer (ForwardingPlayer) intercepts in MediaSessionManager
  → handleWatchCommand() (500 ms debounce)
  → MainActivity.watchMediaListener → onWatchPlay() → toggleVoiceCapture()
  → duck her TTS → VoiceCaptureController start / stop → transcript → sendMessage()

Watch tether (always-on AVRCP presence)
  → launch: silent 1 s seed WAV loaded paused at 0:00 ("Polaris standing by")
  → response plays: playUri() + updatePolarisMetadata(text ≤ 180 chars)
  → natural end: STATE_ENDED → 600 ms → pause() + seekTo(0) → READY-paused
  → watch keeps listing the player; marquee-scrolls her reply as the title
  → STOP: pause() + seekTo(0) — tethered halt, never IDLE, never ENDED
```

Because the Media3 `MediaSession` is owned by `MediaSessionManager`, watch events are received while the app is backgrounded — the watch does not need the app open to trigger a capture. With no listener attached, commands fall back to plain ExoPlayer control.

### Testing

All three watch-side checks below passed the operator's physical Garmin retest on Sep 26, 2026 — v3.3 is verified end-to-end.

1. ✅ Pair the watch; then lock the screen, press PLAY, speak, press PLAY again — the transcript is sent to Polaris and answered aloud
2. ✅ After the reply finishes, raise the wrist minutes later — the watch still lists "Polaris AI" with her reply marquee-scrolling as the title (the tether hold)
3. ✅ Press STOP mid-capture — the mic tint resets on the phone and nothing is sent; press STOP during playback — audio halts but the player stays listed on the watch
4. In-app, confirm the mic tint + live partials in the input field, and inspect logcat:

```bash
adb logcat | grep -E "MediaSessionManager|VoiceCaptureController|TacticalBackgroundService"
```

Expected: `MediaSession activated - wearable AVRCP commands live` on launch, `Watch command … routed to listener` per press, and `VoiceCaptureController` listening/stop logs for PLAY/PAUSE toggles.

**Redmagic logcat caveat:** vendor sensor spam rotates the logcat buffer within seconds, flushing app logs out almost immediately — an empty grep does **not** mean the session is dead. Grow the buffer before a capture window (`adb logcat -G 16M`), or skip logs entirely and use the authoritative log-free check:

```bash
adb shell dumpsys media_session
```

A healthy tether shows our Media3 session as the active media-button session for `com.freeroam.tactical`: `state=PAUSED(2)`, `position=0`, metadata **"Polaris standing by - press PLAY to talk"**. Ignore any stale `ERROR(7) "Bluetooth audio disconnected"` record with null metadata — it belongs to another (inactive, pre-install) session, not ours.

---

## Build System

### Native Gradle build (canonical — built, deployed & operator-verified v3.3)

The Gradle project lives under `android/`; the Android SDK is at `/home/causeiam/android-sdk` (wired via `android/local.properties`).

```bash
cd /home/causeiam/docker-containers/freeroam/android

./build-apk.sh      # wraps: ./gradlew :app:assembleDebug --no-daemon

# Output
ls -lh app/build/outputs/apk/debug/app-debug.apk   # ~234 MB (unminified debug)

# Install to the attached Redmagic, over the existing install
adb install -r app/build/outputs/apk/debug/app-debug.apk

# Matt's phone over WiFi (no cable): 5555 while tcpip is armed, or the mDNS
# TLS serial after a phone reboot (run rearm_phones.sh to re-establish)
#   ~/platform-tools/adb -s 192.168.50.98:5555 install -r app/build/outputs/apk/debug/app-debug.apk
# -r preserves the ACCESS_FINE_LOCATION grant; verify the identity chip shows Matt
```

- **Toolchain:** Gradle 9.2 wrapper, `compileSdk 35`, JDK 17 toolchain, `minSdk 26`, `targetSdk 34`
- **Package:** `com.freeroam.tactical`, `versionName 1.0.3` (`versionCode 4`) — internal doc version v3.3
- **Size note:** the debug APK is unminified and bundles sherpa-onnx JNI libraries for every ABI → ~234 MB. The release build type already has ProGuard enabled (`isMinifyEnabled = true`); a minified release build will shrink it considerably when a daily-driver APK is wanted.
- **Crash logs:** `./get_crash_log.sh` (repo root)
- Build logs land in `android/build/` intermediates (`build.log` / `build-output/` at the repo root belong to the legacy container build)

### Containerized build (legacy)

The older Docker path is fully containerized (Dockerfile-baked Gradle + SDK) and produced the v3.0-era ~74 MB APK (`freeroam-debug.apk` at the repo root):

```bash
cd /home/causeiam/docker-containers/freeroam
./docker-build.sh
docker cp freeroam-tmp:/app/android/app/build/outputs/apk/debug/app-debug.apk \
  /home/causeiam/docker-containers/freeroam/app-debug.apk
adb install app-debug.apk
```

## Version History

| Version | Date | Changes |
|---------|------|---------|
| v3.5.2 | Sep 28, 2026 | **Watch play-state model (operator field feedback)** — the v3.5.1 always-PLAYING keep-alive loop held the connection but left the watch showing a PAUSE button at idle ("caught out of sync"). New model: the silent seed (now 60 s, was 1 s — the 1 s loop's per-second track-end churn destabilized AVRCP title display) rests PAUSED at launch and after every exchange; `ensureTetherLooping()` engages it only while she speaks (pause affordance — pressing it stops her speech and starts the mic; both PLAY and PAUSE route to the PTT toggle), and `pauseTetherLoop()` parks it the moment her voice drains — the watch flips to PLAY exactly as she finishes. Every retitle now stamps a fresh mediaId (`updateMetadata`) so legacy AVRCP stacks treat it as a new track and reliably refresh the title — her full reply sticks on the display instead of flashing back to standby. Mid-speech text updates removed: the Garmin ignores in-place retitles anyway, and the metadata churn destabilized the display |
| v3.6 | Sep 28, 2026 | **Text-only playback (operator decision)** — live replies already streamed text→Kokoro→AudioTrack with no artifacts (v3.5); now the artifact layer is gone entirely: no WAV is ever written, bubble replay re-synthesizes from text through the streaming pipeline via `speakText()` (first word ~1-2 s; tap speaks, tap again stops), and a startup sweep (`PolarisAudioStore.pruneAllWavs`) deletes the legacy cache (56 MB / 79 files at cutover) — with nothing new written, the sweep IS the long-term pruning. `PolarisTextStore` persists reply text for cross-restart replays; watch HUD and tether behavior unchanged |
| v3.5.1 | Sep 28, 2026 | **Streamed-tail clip fix** — `AudioTrack.write` only QUEUES PCM (up to ~1 s of speech still buffered at the last chunk); the playback thread now waits for the playback head to reach the final written frame before teardown, mirroring KokoroTTS's completion wait — no more clipped last words. **Live text on the watch HUD** — her reply streams onto the Garmin title the moment it generates (the LLM writes ~3x faster than she speaks, so the watch runs well ahead of the voice); AVRCP metadata updates throttled to a 1.5 s cadence so watch firmware is never flooded; the marquee's pixel scroll rate itself is Garmin-side firmware and not commandable. **Keep-alive loop (the v3.3 reserve lever, engaged)** — the silent seed now plays forever on `REPEAT_MODE_ONE` from launch, so the session reads PLAYING instead of paused-idle and the AVRCP connection never ages out; real playback (bubble replay) clears the loop, and tethered STOP + natural completion both re-seed it, re-titled with her latest reply. Mic hygiene still pauses the loop during captures (`ensureTetherLooping()` resumes it at stream start/finish) |
| v3.5 | Sep 28, 2026 | **Speak-while-she-writes streaming playback** — the gateway has streamed her replies as sentence-safe `chat_stream` deltas since v3.3, but the phone never registered the "message" socket event, so chunks were dropped and the app waited for the whole reply then synthesized it end-to-end before a single word. v3.5 opens the ear: `PolarisWebSocketClient` dispatches `chat_stream` deltas (plus `user_chat` for cross-device messages), new `StreamingTtsPlayer` turns arriving text into continuous speech — sentences synthesize on the shared Kokoro executor while earlier ones are already audible through one persistent 24 kHz mono AudioTrack (blocking write = natural backpressure), monster sentences split at clause boundaries, first words land seconds after the reply starts generating. On completion the concatenated PCM persists as the standard bubble WAV; mic capture / watch STOP abort mid-stream (aborted streams re-synthesize whole on the final broadcast); TTS-unavailable devices keep the whole-text path. **Watch HUD:** "Polaris is speaking..." marquee-scrolls during the stream; her full reply replaces it at completion |
| v3.4 | Sep 28, 2026 | **bf_emma default** — voice spinner reordered so af_emma leads (the position-0 `onItemSelected` at layout time had been overwriting the af_emma default with af_heart before Kokoro init); `VOICE_DEFAULT_SID` 0 → 7 so unknown names also fall back to bf_emma. **End-of-speech patience (2.5 s)** — the recognizer's own ~1 s VAD no longer auto-sends mid-pause messages: service-finalized segments fold into a running transcript and the recognizer restarts; the capture finalizes only on a manual stop, 2.5 s of true silence (silence watchdog), or the 30 s cap; quiet-segment errors deliver the accumulated transcript instead of discarding it; live partials show the full running transcript. **Lock-screen audio** — `onPause()` no longer pauses Polaris playback (it was cutting replies mid-sentence at every screen-lock; the ExoPlayer wake lock + TacticalBackgroundService FGS were already in place). **Capture visibility** — a live capture swaps the input field to a red-bordered "Listening..." state (`input_field_bg_capture`) with a focused caret and the IME suppressed, so a watch PLAY press is obvious without tapping the field first. **User bubble restyle** — `bubble_user.xml` now uses the reserved deep-navy `@color/bubble_user` (#1e3a5f) instead of `electric_blue`; bubble text 14sp → 16sp |
| v3.3 | Sep 26, 2026 | **Watch tether** — the AVRCP media session never drops: silent 1 s seed WAV loaded paused at launch (`seedSilentTetherIfEmpty`), natural completion rewinds to 0:00 and holds paused 600 ms after `STATE_ENDED`, `stop()` pauses + rewinds instead of idling the player; her latest reply (≤ 180 chars, up from 100) marquee-scrolls on the Garmin as the track title until the next response. **Button remap:** PLAY/PAUSE → push-to-talk toggle; STOP → cancel capture mid-PTT (hard abort — `cancel()` fires no listener callback, so the stop handler resets the mic tint itself) / tethered halt; BACK/NEXT disabled (stripped from `getAvailableCommands()` + no-op overrides). **Mic hygiene:** starting a capture pauses any playing TTS. **Honest state reporting:** READY/BUFFERING map to PAUSED when `playWhenReady == false`. Reserve lever if a stack drops paused players: play the seed on `REPEAT_MODE_ONE`. **Field-verified Sep 26, 2026:** built in 1m 21s (233 MB debug APK), installed over the air on the Redmagic (FY24031100E6), `dumpsys media_session` confirmed the paused seed live at launch (`state=PAUSED(2)`, position 0, "Polaris standing by"), and the operator's physical Garmin retest passed end-to-end — PTT works screen-locked, the tether holds, both STOP behaviors correct; REPEAT_MODE_ONE reserve not needed |
| v3.2 | Sep 26, 2026 | Voice push-to-talk: watch NEXT rewired as the mic toggle (press = start capture, press again = stop + send); in-app mic button tap-to-toggle with live partials in the input field (`alert_red` tint); `VoiceCaptureController` (PARTIAL wake lock, 30 s cap, 1.5 s stop-flush timeout); `TacticalBackgroundService` converted to a mic-typed FGS (`dataSync\|microphone`, ID 1001) started from `MainActivity.onStart()` so captures survive screen-lock; `WatchTriggerReceiver` removed — watch buttons now route through the Media3 `MediaSession` (`WatchSessionPlayer`) |
| v3.1 | Sep 25, 2026 | KokoroTTS static-noise fix — `nativeOrder()` PCM16 conversion (the default BIG_ENDIAN ByteBuffer byte-swapped every sample, producing static that followed the speech envelope); completion-tracked `playAudio()` + explicit `stop()`; WAV caching (`polaris_<id>.wav`) with replayable audio bubbles; mini player; `freeroam.md` rewritten to match the current architecture |
| v3.0 | Sep 14, 2026 | ChatArchiverWorker Room timing fix; APK size optimization 193 MB → 74 MB (61% reduction) |
| v2.9 | Sep 7, 2026 | Security hardening: strict text-chat operation (text in / text out only) |
| v2.8 | Sep 5, 2026 | Rolling 10-message window + auto-speak freshness gating |
| v2.7 | Sep 5, 2026 | Multi-user chat sync (per-identity server-side histories) |
| v2.6 | Sep 5, 2026 | Garmin watch PTT (PLAY/NEXT media-button integration) |
| v2.5 | Sep 4, 2026 | Automatic IP-based identity detection |

## License

Private use only. Not for commercial distribution.
