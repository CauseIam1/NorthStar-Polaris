# Polaris — Current System Reference

Single source of truth for the Polaris AI stack as it currently sits: containers, services, APIs, subsystems, and deployment facts. Scope: `freeroam/polaris-gateway` (gateway) and `freeroam/polaris-dashboard` (dashboard). No history, no debug notes — current architecture only.

---

## 1. Overview

Polaris is the private AI assistant and system hub running on the Dell Mainframe host (`192.168.50.51`, LAN-only). Two primary Docker containers plus supporting infrastructure:

| Component | Container | Role |
|---|---|---|
| **Polaris Gateway** | `polaris-gateway` | One container, three Flask services: Chat Gateway (AI chat, tool calling, memory, voice, desktop-vision turns), Burst Receiver (telemetry, CSI radar, vision stream), Sandbox Gateway (Mini Pi SSH console) |
| **Polaris Dashboard** | `polaris-dashboard` | Static nginx page (`Polaris_page.html`) serving the Holodeck split-view chat UI |
| Ollama | `ollama` | Local LLM inference — `polaris-ai:latest` (chat), `nomic-embed-text` (embeddings) |
| Gotify | `gotify` | Push notification delivery (consumed by the AI as a tool) |
| Polaris Playground | `polaris-playground` | nginx serving AI-generated web apps from `playground/www` on :8090 |
| LiveCharts | `livecharts` | XRPL asset monitoring page (nginx) on :7001 |
| WireGuard | `freeroam-wireguard` | Secure tunnel for the FreeRoam Android client (UDP 51820) |

**Operator identity** is resolved in order: explicit `user_id` in the request body (the dashboard sends `rich`/`matt` on the user's behalf) → client IP (`X-Forwarded-For` first hop, else `remote_addr`). Known IPs: `RICH_IP = 192.168.50.42` (Rich), `MATT_IP = 192.168.50.98` (Matt), `OPERATOR_IP = 192.168.50.51` (the Mainframe itself — persona "Operator", operator data files), `DASHBOARD_TEST_IP = 192.168.50.227`, `MISS_PI_IP = 192.168.50.179` (mapped to Rich). Traffic from Docker bridge ranges (`172.17.*`–`172.20.*` — NAT-masked) falls back to `DEFAULT_USER_CONFIG` (persona "Dashboard", operator data files). Identity selects the user name, profile/behavior/lists/transcript paths, session partitioning, and chat history.

**Edge device:** MissPi / Mini Pi (Raspberry Pi 5) at `192.168.50.179`, user `minipi`, workdir `/home/minipi/miss-pi`. **Retired as the spatial sensor (Sep 27, 2026):** the X17's LGI webcam perception stack (LGI v1.2 "Her Seeing Me", device `x17-webcam`) now provides the `radar_motion` + `vision_frame` feeds into the Burst Receiver — the WiFi CSI radar and YOLO camera stream duties moved to the X17; the X17's desktop-vision chat turns (screenshot + OCR on `POST /api/chat`) are answered by the Chat Gateway's vision LLM ladder (§3.1). MissPi remains online for SSH command execution (Sandbox Gateway target) only.

---

## 2. Port Map

| Host Port | Container Port | Service | Purpose |
|---|---|---|---|
| 8082 | 5000 | Chat Gateway | AI chat, voice (TTS/STT), memory, tool calling, task tracking, Socket.IO |
| 8083 | 5001 | Burst Receiver | Burst/telemetry ingestion, CSI radar, vision stream, Socket.IO |
| 7007 | 7007 | Sandbox Gateway | Mini Pi SSH web console |
| 7000 | 80 | Polaris Dashboard | Holodeck UI (nginx) |
| 8090 | 80 | Polaris Playground | AI-generated web app server |
| 7001 | 80 | LiveCharts | XRPL asset monitor |
| 8081 | 80 | Gotify | Push notification server |
| 11434 | 11434 | Ollama | LLM inference + embeddings |

All services live on the external Docker network `docker-containers_default`.

---

## 3. Chat Gateway (host 8082 → container 5000)

**File:** `polaris_continuous_learning_gateway.py` — Flask + Flask-SocketIO app powered by Ollama, with tool calling, dual-tier memory, a voice pipeline, session management, and heuristics. Blueprints registered: `memory_bp` (Memory Vault), `supervisor_bp` (Supervisor Audit).

### 3.1 REST API

**Chat & History** — one server-side store per operator identity, shared across devices (see §3.4)
| Method | Route | Purpose |
|---|---|---|
| POST | `/api/chat` | Main chat endpoint (LLM + memory context + tool calling); persists both turns to chat history, echoes `chat_message` to the sender's IP room, and emits `tool_execution` when tools run (§3.2 tool pipeline). With `image_base64` it becomes a desktop-vision turn (§3.1 note below) |
| GET | `/api/history/<user_id>` | Paired user↔Polaris history for a user, dashboard format (identity by path, not IP — the dashboard's 3 s cross-device sync poll) |
| GET | `/api/chat/history` | Full history for the IP-authenticated user → `{status, user, messages}` (the phone's sync read) |
| POST | `/api/chat/history` | Replace the IP-authenticated user's full history (`{"messages": [...]}`) |
| DELETE | `/api/chat/history` | Clear the IP-authenticated user's history |
| POST | `/api/chat/history/sync` | Append one client-supplied message object (`{"message": {...}}`) to the IP-authenticated user's history |

**`POST /api/chat` HTTP return — raw envelope, by design.** The handler ends with `return jsonify(response_data)`, passing the raw Ollama envelope verbatim (unstripped `response` + `thinking`). It is a debug/API surface only — **no rendering path consumes it**: the dashboard's fetch handler discards the HTTP body (§11), the Android app renders only from `GET /api/chat/history`, and every persisted/broadcast surface (transcript, session, chat history, socketio `chat_message`) carries only the stripped prose (§3.2 pipeline).

**Desktop-vision turns (Polaris Desktop Vision, LGI v1.2).** A `POST /api/chat` request carrying `image_base64` (desktop screenshot JPEG) — optionally `ocr_text` (≤4000 chars) and `active_window` — bypasses the tool pipeline and is answered by the vision LLM ladder: `VISION_LLM_MODEL` (`polaris-ai:latest`) first, then `VISION_LLM_FALLBACK_MODEL` (`mistral-large-3:675b-cloud`) on any failure — degrade, never fabricate. The image rides Ollama's multimodal `images` field (`OLLAMA_URL`, default `http://ollama:11434/api/generate` via Docker DNS) with a desktop-context prompt built from the active-window title and OCR text; timeout is 120 s (`VISION_LLM_TIMEOUT`, hardcoded). Total ladder failure raises into the standard 503 error path. The reply envelope adds `vision: true` + `vision_model` (raw-HTTP surface only — clients render via `chat_message` as usual).

**Voice**
| Method | Route | Purpose |
|---|---|---|
| POST | `/api/tts` | Text → speech (Kokoro, WAV blob) |
| POST | `/api/tts-stream` | Chunked streaming synthesis variant |
| POST | `/api/tts/cancel` | Abort in-flight synthesis |
| POST | `/api/voice-command` | Full chain: audio → STT → LLM → TTS → WAV |
| GET | `/api/voice-health` | Voice subsystem status |

**Status & Tasks**
| Method | Route | Purpose |
|---|---|---|
| GET | `/api/status` | Overall system status |
| GET | `/api/status/tasks` | All tracked tasks |
| GET | `/api/status/tasks/active` | Active tasks |
| GET | `/api/status/tasks/completed` | Completed tasks |
| GET | `/status` | Status alias |

**Session & Heuristics**
| Method | Route | Purpose |
|---|---|---|
| GET | `/api/session` / `/api/session/status` / `/api/session/topics` | Session window, status, detected topics |
| POST | `/api/session/clear` / `/api/session/reset` | Reset session state |
| GET/POST | `/api/heuristics` | List / create behavior heuristics |
| GET | `/api/heuristics/staged` | Staged (unapproved) rules |
| POST | `/api/heuristics/approve/<rule_id>` | Approve a rule |
| DELETE | `/api/heuristics/reject/<rule_id>` | Reject a rule |
| PUT | `/api/heuristics/edit/<rule_id>` | Edit a rule |
| GET | `/api/behavior` | Behavior adaptation state |

**Memory, Users & Lists**
| Method | Route | Purpose |
|---|---|---|
| GET | `/api/memory/stats` | SQLite + ChromaDB tier stats |
| POST | `/api/vector/search` | Semantic vector search |
| GET | `/api/whoami` / `/api/users` | IP identity / registered users |
| GET/POST | `/api/profile` | User profile read/write |
| GET | `/api/transcripts` | Daily transcripts |
| GET/POST | `/api/lists`, `/api/lists/<name>` (+`/view`, `/add`, `/remove`) | Persistent list store |

**Distillation, Nodes, Supervisor**
| Method | Route | Purpose |
|---|---|---|
| POST | `/api/distill` / `/api/distill/auto` | Run / schedule memory distillation |
| POST | `/api/sync/daily-transcript` | Ingest daily transcript |
| POST | `/api/nodes/register`, `/api/nodes/heartbeat`; GET `/api/nodes` | Edge-node registry + heartbeats |
| GET | `/api/supervisor/status`, `/api/supervisor/audit/recent`; POST `/api/supervisor/audit` | Supervisor audit (screen/clipboard triage) |

### 3.2 Socket.IO Protocol (port 8082)

**Client → Server events**
| Event | Behavior |
|---|---|
| `connect` | Server resolves operator identity from client IP, joins the client into a per-IP room, emits a welcome `message` event |
| `message` | Chat / query messages from the dashboard **and the phone** (type-based routing); reply streams back as `chat_stream` chunks with a 48-char hold-back buffer, completed turns echo via `chat_message`, and **both turns are persisted** to chat history. Tool calls embedded in the reply are handled **server-side** (tool pipeline below) — raw tool JSON never reaches the client |
| `join_status_room` | Joins room `status`; server immediately emits current `active_tasks` |
| `join_tasks` | Dashboard TASK FEED's join — puts the feed socket into room `status` (the `tasks_dashboard` room name in the payload is ignored) and emits an `active_tasks` snapshot; joins regardless of operator config |
| `leave_status_room` | Leaves room `status` |

There is **no** `request_history` handler and no `chat_history` emit — history sync is REST-poll based (§3.4).

**Server → Client broadcasts** (to room `status`)
- `task_registered`, `task_started`, `task_progress`, `task_completed`, `task_failed` — real-time task lifecycle events (uploads, SSH commands, distillation, container scans, etc.)

**Server → Client broadcasts** (to the sender's per-IP room — live chat delivery)
- `chat_message` — instant echo of the user's own prompt (the phone deliberately skips adding sent messages locally to avoid duplicates) and of the completed Polaris reply the moment the stream ends, so the phone renders it without waiting for the 3 s history poll; payloads carry both `content` and `message` keys — Android's `handleChatMessage()` reads `message`, REST payloads use `content`
- `message` (type `chat_stream`) — streaming reply chunks (`stream_id` / `is_complete`); the same event also carries connection/welcome, `error`, and `active_tasks` payloads
- `tool_execution` — emitted the moment server-side tool execution finishes (payload: `tool_calls`, `results`, `timestamp`); fired by **both** REST `/api/chat` and the WS `message` handler

**Tool pipeline (server-side — identical on REST `/api/chat` and the WS `message` handler)**
1. The Ollama stream is scanned for tool-call markers; the WS stream keeps the last **48 chars** un-emitted (`STREAM_HOLD_BACK`) so a marker split across chunks can never leak mid-stream
2. On a tool marker (`{"tool":` …), the stream **freezes** — only the prose preceding it is emitted (dangling code fences trimmed)
3. After the stream ends: parse (`parse_tool_calls_from_response`) → strip (`strip_tool_calls_from_response`) → execute each call via `execute_tool_call()`; every call logs a `[TOOL]` transcript line
4. The stripper closes three leak vectors — **text cleanup only**, parser/execution semantics untouched:
   - **Malformed tool JSON:** a balanced `{"tool": …}` span `_safe_parse_tool_json` cannot repair is still stripped *if it names a known tool* (union of every schema registry + the legacy inline names, via `_known_tool_names()`); a `[DEBUG] Stripped malformed tool JSON…` line preserves the evidence in `docker logs`. JSON naming no known tool (e.g. the user's own pasted JSON echoed back) stays visible
   - **Legacy inline calls** (`read_doc(filename="…")`, `run_sandbox_code(code="…")`, …): executed by the parser but previously left in the bubble — stripped with the same name/argument shapes the parser matches
   - **Empty `json` fences** left behind after inner-JSON removal, plus triple-newline collapse
5. `tool_execution` carries the actual results; only the **cleaned prose** is flushed to the stream, persisted, echoed via `chat_message`, and fed to the session window — raw tool JSON never pollutes history, memory, or future-turn context
6. When tools ran, a **truthful tool-report follow-up turn** is generated: a second LLM pass over the real results (large fields truncated to 1 500 chars by `_truncate_tool_results()`), reporting-only instructions (never claim success on a failure result, plain text, no further tool calls); the report turn itself is stripped so it can never recurse. Fallback `generate_follow_up_message()` is result-aware — outcomes are reported exactly as they happened, never fabricated

### 3.3 Configuration (environment)

| Variable | Value | Purpose |
|---|---|---|
| `GATEWAY_PORT` | `5000` | Flask listen port |
| `OLLAMAHOST` | `172.17.0.1:11434` | Ollama endpoint |
| `OLLAMA_MODEL` | `polaris-ai:latest` | Chat model |
| `DATA_DIR` | `/app/data` | Distillation DB + user data |
| `USER_DATA_DIR` | `/data/freeroam` | Bursts, telemetry, profiles, chat history |
| `LIVECHARTS_DIR` | `/livecharts` | LiveCharts mount |
| `WWW_DIR` | `/playground/www` | Playground web root |
| `KOKORO_*` | see §6 | TTS configuration |
| `GOTIFY_*` | see §10 | Notification configuration |

### 3.4 Cross-Device Chat History

One server-side history store per operator identity (`rich` / `matt`), shared by every client — phone, dashboard, any browser — so chat flows across devices.

**Storage**
- In-memory dict `CHAT_HISTORY` keyed by identity, guarded by `CHAT_HISTORY_LOCK`, capped at **200 messages per identity** (oldest dropped on append).
- Persisted as JSON to `/data/freeroam/chat_history.json` (host bind `/mnt/containers/freeroam/data/chat_history.json`); loaded on startup; saves run on a daemon thread outside the lock.
- Record shape: `{ "role": "user"|"assistant", "content", "timestamp", "sender" }`.

**Timestamps — Z-suffixed UTC, non-negotiable.** `utc_now_iso()` emits ISO-8601 UTC with a trailing `Z` (e.g. `2026-09-25T01:23:45.678Z`). Android's `Instant.parse()` rejects the legacy formats (`%Y-%m-%d %H:%M:%S`, naive `.isoformat()`); parse failures collapse to `now()`, which corrupted ordering and made the phone's sync loop wholesale-replace its window with stale history. All records must use `utc_now_iso()`.

**Both turns always persisted** — every exchange appends the user prompt *and* the Polaris reply, on both transports:
- REST `POST /api/chat`
- WS `message` handler (previously appended only Polaris's reply, so the user side of WS chats never reached other devices)
- When the turn executed tools, a **third record** is appended on both transports: the truthful tool-report follow-up turn (transcript line `Polaris (tool report)` — see §3.2 tool pipeline)

**Delivery**
- **Live, on the sending device:** `chat_message` to the sender's per-IP room (§3.2) — the user's own prompt echo and the completed Polaris reply the instant the stream ends; the reply body streams as `chat_stream` chunks.
- **Cross-device, ≤3 s:** REST polling. The phone polls `GET /api/chat/history` (IP-authenticated) on connect and every 3000 ms (`CHAT_SYNC_INTERVAL_MS`), rebuilding its rolling **10-message** window (`MAX_CHAT_MESSAGES`) whenever the server's newest timestamp beats the local newest, and auto-speaking new Polaris replies (1 s dedupe gate). The dashboard polls `GET /api/history/<user_id>` every 3 s.

**Vestigial socket events:** the phone emits `request_history` and listens for `chat_history` over Socket.IO; the gateway implements neither. The REST poll loop is the sync path — the WS pair is dead code on the wire.

### 3.5 System Prompt & Persona (North Star primacy)

`build_system_prompt()` assembles the per-request system prompt. Subject-matter precedence is fixed: **Project North Star is Polaris's primary domain and the default subject of any ambiguous request** — Miss Pi, Gotify, and the Memory Vault are supporting infrastructure, routed to only when explicitly asked or clearly relevant.

Prompt blocks, in order:
- Persona header — witty, bubbly AI companion for `{user_name}`
- **[PROJECT NORTH STAR - YOUR PRIMARY ECOSYSTEM]** — Trading Matrix (operator-owned AMM mesh, 0.05% fee, multi-hop arbitrage loops, LP-fee recycling), Zero-Fiat Rule (stablecoins are transient pass-through settlement nodes only; capital retained in XRP + whitelisted meme coins), Wallet Topology (COLD_WALLET / BOT MPT_RPN hot wallet / TRADING_WALLET), Stack (Java engines, QuestDB :8812 with the epoch / `java.sql.Timestamp` binding rule, local rippled `http://rippled:5005`, North Star Holodeck dashboard), SCOPE (the primacy statement above)
- [REMOTE INFRASTRUCTURE - MISS PI] — SSH facts (user `minipi`, workdir `/home/minipi/miss-pi/`, key scripts, `behavior_lib.json`)
- [SSH REACHABILITY DOCTRINE] — ON/OFF answers, not failures; connection failures map to exact phrasings (OFF / ON but SSH down / auth failure / ON and responding)
- [REMOTE PROCESS MANAGEMENT RULES] — the `[b]racket` pgrep/pkill trick, nohup + redirect daemon starts, PID-stability proof, JSON-safe command strings, and HONEST REPORTING (never claim an outcome in the same response that performs the action)
- Tool schemas + **[TOOL USAGE EXAMPLES]** — 10 few-shot pairs covering the full tool surface, all North Star / trading-infrastructure flavored (XRP dashboard publishing, playground listing, sandbox code, trading notes, Miss Pi uptime/connectivity, arbitrage journaling, trading-bot + XRP-price Gotify alerts, QuestDB-lessons vault query) + [TOOL EXECUTION RULE] (the JSON tool-call contract)
- [PUSH NOTIFICATIONS - GOTIFY] · [MEMORY VAULT - YOUR LONG-TERM MEMORY] (read-only `memory_vault_query`; the vault is written only by the learning process)
- [KNOWN FACTS ABOUT {USER}] — memory context + behavior lib + heuristics + session window + tools block · CORE DIRECTIVES (vibe; TTS-ready, no emojis; ≤3 sentences unless asked; personalize; follow heuristics; use recent context)

**Context guardrail:** the prompt renders once; if it exceeds `SYSTEM_PROMPT_TOKEN_GUARD` (**6500** est. tokens, `chars ÷ 4`), the conversation window shrinks stepwise (`WINDOW_SHRINK_STEPS = [6, 3, 0]` exchanges) and re-renders so the persona, KNOWN FACTS, and directives survive — only the oldest exchanges are sacrificed (`[GUARDRAIL]` log line). ~1.6k headroom is kept under the model's 8192 `num_ctx` for the user turn, reply, and thinking.

**Modelfile mirror:** the Ollama-side base model carries the same persona — `freeroam/polaris-gateway/Modelfile` defines `polaris-ai:latest` as `FROM glm-5.3-flash:cloud` (temperature 0.7, `num_ctx 8192`) with the North Star primacy block in its SYSTEM text. Modelfile edits are applied with `ollama create polaris-ai:latest -f <Modelfile>` — an instant manifest swap on the running ollama container, no restart.

---

## 4. Burst Receiver (host 8083 → container 5001)

**File:** `burst_receiver.py` — high-throughput telemetry/burst ingestion, CSI radar spatial tracking, the vision stream (X17 webcam YOLO detections + narration), and Ollama image-analysis endpoints.

### REST API
| Method | Route | Purpose |
|---|---|---|
| POST | `/api/burst` | Submit burst message |
| POST | `/api/telemetry` | Submit telemetry data |
| GET | `/api/burst/list`, `/api/burst/<burst_id>` | List / fetch stored bursts |
| GET | `/api/telemetry/recent` | Recent telemetry |
| POST | `/api/radar/ingest` | Submit CSI vector for spatial processing |
| POST | `/api/vision/analyze` | Analyze an image (`image_path`, or base64 `image_data` + `filename`) with the Ollama vision model — prompt types: general / tactical / ocr / sharpness_check |
| GET | `/api/vision/analyze-burst/<burst_id>`, `/api/vision/models`, `/api/vision/test`, `/api/vision/stream/status` | Burst image analysis (cached `vision_analysis.json` or async trigger), vision-capable model list, pipeline test, live stream status |
| GET | `/api/jarvis/announcements` | Announcement feed |
| GET | `/api/status` | Receiver status |

### CSI Radar Processing (`POST /api/radar/ingest`)
Payload: `{ "csi_vector": [...], "timestamp"?, "device_id"?, "metadata"? }`

1. First vector initializes the weighted subcarrier array
2. X/Y coordinates from split subcarrier weights (left/right halves)
3. Z coordinate from amplitude decay/growth against baseline
4. Motion detection — velocity + confidence from position deltas
5. Broadcast `radar_update` to room `radar_dashboard`

### Socket.IO
| Event | Direction | Behavior |
|---|---|---|
| `join_radar` / `leave_radar` | client → server | Join/leave room `radar_dashboard` (default room) |
| `request_radar_snapshot` | client → server | Server responds with trajectory snapshot |
| `radar_update` | broadcast | Emitted on every CSI ingest (room `radar_dashboard`) |
| `radar_status`, `radar_snapshot`, `radar_history` | per-requester | Status / snapshot / history replies |
| `vision_connect` | client → server | Server emits `vision_ack` and joins client into room `vision_dashboard` |
| `vision_frame` | client → server | Live camera frame → processed → broadcast as `vision_update` (room `vision_dashboard`) |
| `vision_disconnect` | client → server | End vision stream |
| `radar_motion` | client → server | Pre-processed motion event (timestamp / confidence / intensity / duration). Legacy source: Miss Pi CSI radar daemon. Current source: X17 webcam perception (LGI SpatialReporter @ 1 Hz, `device_id: x17-webcam`) sending explicit `x` / `y` / `z` / `velocity` — real coordinates pass through untouched into the `radar_update` broadcast (`people_count` also rides the payload but is not currently forwarded); coordinate-less legacy payloads keep the intensity-driven simulation |
| `join_vision` | client → server | Join room `vision_dashboard`; server replies with `vision_history` (last 10 detections) |

### Configuration

All values are hardcoded constants in `burst_receiver.py` — the service reads no environment variables.

| Constant | Value | Purpose |
|---|---|---|
| `ALLOWED_IPS` | `192.168.50.42` / `.98` / `.51` / `.179` | LAN-only REST guard (Rich / Matt / Operator / MissPi). Loopback and Docker-bridge source IPs are denied — Mainframe-shell and `docker exec` curls get 403 by design (§13) |
| `USER_CONFIG` | per allowed IP | Per-user storage: `burst_dir` = `/data/freeroam/<user>/bursts`, `telemetry_log` = `/data/freeroam/<user>/telemetry.log`; MissPi flagged `radar_source: True` |
| `MAX_BURST_AGE_HOURS` | `24` | Burst retention before rotation |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama endpoint for `/api/vision/*` — **does not resolve inside the container** (verified: `ollama:11434` answers 200, `localhost:11434` refuses), so the image-analysis endpoints are dead in-container; the live vision paths are the Socket.IO feeds and the Chat Gateway's Desktop Vision ladder (§3.1) |
| `VISION_MODEL` | `mistral-large-3:675b-cloud` | Ollama model for `/api/vision/*` analysis |
| `VISION_TIMEOUT` | `60` | Ollama vision request timeout (s) |
| `VISION_PROMPTS` | general / tactical / ocr / sharpness_check | Analysis prompt presets |
| `CSIRadarEngine` | `window_size=30`, `movement_threshold=0.15` | Rolling CSI buffer for trajectory + motion detection threshold |
---

## 5. Sandbox Gateway (host 7007)

**File:** `sandbox_gateway.py` — web terminal for SSH operations against the Mini Pi.

### REST API
| Method | Route | Purpose |
|---|---|---|
| GET | `/` | Web dashboard (served from `sandbox_static/`) |
| POST | `/api/ssh/execute` | Execute command: `{ "target_host", "command", "timeout" }` → `{ success, returncode, stdout, stderr, timestamp }` |
| POST | `/api/ssh/check` | SSH connectivity check |
| GET | `/api/ssh/history` | Command history (last 50) |
| POST | `/api/ssh/history/clear` | Clear history |
| GET | `/api/status` | Gateway status |

### SSH Target
- Host: `192.168.50.179` (`MINI_PI_HOST`), user `minipi`, workdir `/home/minipi/miss-pi` (`MINI_PI_WORKDIR`)
- Key: `/root/.ssh/id_ed25519` (mounted read-only from `/home/causeiam/.ssh`)
- Options: `StrictHostKeyChecking=no`, `BatchMode=yes`, `IdentitiesOnly=yes`

---

## 6. Voice Pipeline (Kokoro TTS + Faster-Whisper STT)

All voice processing is server-side in the Chat Gateway. A global `VOICE_PROCESSING_LOCK` (threading.Lock) serializes synthesis and transcription; voice models lazy-load on first use.

**Startup warmup:** a daemon thread starts at gateway boot, waits 20 seconds, then calls `synthesize_speech("Voice warmup complete.")` to prime the Kokoro pipeline so the first real message is not a cold start. Warmup failure is logged and falls back to lazy loading.

### POST /api/tts
Request:
```json
{ "text": "The system is operating normally.", "voice": "af_heart", "speed": 1.0 }
```
(`voice` and `speed` optional — default to `KOKORO_VOICE_NAME` / `KOKORO_SPEED`)

Response `200`: `audio/wav`, 24 kHz, with headers `X-Text-Length`, `X-Voice`, `X-Sample-Rate`, `X-Format`.

Errors: `400` no text · `503` voice processing unavailable (Kokoro not initialized) · `500` synthesis failed.

| Variable | Default | Purpose |
|---|---|---|
| `KOKORO_VOICE_NAME` | `af_heart` | Default voice model |
| `KOKORO_SPEED` | `1.0` | Playback speed multiplier |
| `KOKORO_CACHE_DIR` | `/app/kokoro-cache` | Model weight cache (bind: `/mnt/rippled-data/kokoro-cache`) |

### Other voice endpoints
| Route | Purpose |
|---|---|
| `POST /api/tts-stream` | Chunked streaming synthesis variant |
| `POST /api/tts/cancel` | Abort in-flight synthesis |
| `POST /api/voice-command` | Full voice loop: audio bytes → `transcribe_audio()` (Faster-Whisper) → Ollama LLM → `synthesize_speech()` (Kokoro) → WAV response |
| `GET /api/voice-health` | Voice subsystem status |

### Dashboard audio playback (client-side queue)
- The **AUDIO RESPONSE** checkbox gates auto-play of Polaris responses.
- Auto-played and manually requested audio enter a sequential queue (`ttsQueue`) driven by a single-flight pump (`pumpTTSQueue`): fetch `POST /api/tts` blob → play through one shared `<audio id="ttsPlayer">` element → wait for playback end → next item.
- Every message bubble has a ▶ button (`toggleMessageAudio(messageId)`): while playing it stops the current item (queue order preserved); while idle it stops anything playing and jumps the clicked message to the front. Manual clicks work regardless of the checkbox.
- History loads never auto-play; closing a message removes it from the queue; blob object URLs are revoked after playback.

---

## 7. AI Tool Calling

Dispatcher: `execute_tool_call(tool_name, arguments)`. All tool schemas are injected into the LLM system prompt at request time.

| Tool group | Source module | Tools |
|---|---|---|
| File operations | `file_tools.py` | `read_doc`, `write_doc`, `append_doc`, `list_docs`, `delete_doc` |
| Playground | `playground_tools.py` | `publish_web_asset`, `list_playground_files`, `run_sandbox_code`, `spawn_service`, `stop_service`, `list_services` |
| Remote execution | `ssh_tools.py` | `ssh_execute`, `ssh_check_connectivity` (targets Mini Pi / Miss Pi) |
| Notifications | `gotify_service.py` | `send_gotify_notification` |
| Memory | `memory_tools.py` | `memory_vault_query` |

Exactly **15 tools** are routed by the `execute_tool_call` dispatcher — the table above is the complete registry; unknown names return `{"success": false, "error": "Unknown tool: …"}`. (`ssh_upload_file` appears in older docs but has no schema and no dispatcher branch — dead name.) The same 15-name union — every schema registry plus the 11 legacy inline-call names, via `_known_tool_names()` — drives the stripper's known-tool check (§3.2, point 4).

**Routing policy (in system prompt):** notification requests → Gotify tool (never SSH); remote command execution → SSH tools; file operations → file tools.

**Standalone toolkit modules not wired into the running gateway:** `system_tools.py`, `livecharts_tools.py`, `www_tools.py` (imported only by the non-launched legacy entrypoint `freeroam_gateway.py`).
---

## 8. Memory Architecture

### Dual-tier live memory
- **SQLite** (`memory_db.py`): users, messages, heuristics, profiles, lists, transcripts.
- **ChromaDB** (`vector_memory.py`): collections for global knowledge, user memories, and conversation context. Embeddings generated by Ollama `nomic-embed-text` (hardcoded in `vector_memory.py`).
- `GET /api/memory/stats` returns both tiers; `POST /api/vector/search` for semantic retrieval.

### Memory Vault (`memory_vault.py` + `memory_routes.py`)
Obsidian-style persistent markdown memory with a ChromaDB semantic index:
- Storage: `/polaris_memory_vault` in-container → host bind `/mnt/containers/freeroam/polaris-gateway/memory-vault` (host-editable).
- Index: ChromaDB collection `vault_documents` at `/data/polaris_vector_db` → host bind `/mnt/containers/freeroam/polaris-gateway/vector-db`.
- API: `GET /api/memory/vault/{stats,list,doc}` · `POST /api/memory/vault/{write,update,query,delete,reindex}`.
- `POST /api/memory/vault/update {"path", "content", optional "title"/"tags"}` edits an existing document in place (blocked for `core` and append-only `journal`) and re-embeds it into `vault_documents`.
- Query modes: `hybrid` (default — merges normalized keyword + semantic scores), `keyword`, `semantic`.
- **Watchdog:** daemon thread re-embeds created/changed markdown every 30 s; failed embeds (e.g., Ollama down) auto-retry on the next pass. Requires `nomic-embed-text` in Ollama.
- `POST /api/memory/vault/reindex {"force": false}` rebuilds the index; `force: true` wipes the collection and re-embeds everything.

### Distillation
`distill_memory.py` — nightly fact extraction for the FreeRoam memory loop. Triggered via `POST /api/distill` or scheduled via `POST /api/distill/auto`; daily transcripts ingested via `POST /api/sync/daily-transcript`.

### Sessions & heuristics
- Sliding-window session context with topic detection (`GET /api/session/topics`).
- Heuristics lifecycle: staged → approved / rejected / edited via `/api/heuristics/*`; persisted to `polaris_heuristics.json`.

---

## 9. Supervisor Audit

Blueprint (`supervisor_routes.py`), prefix `/api/supervisor` — screen/clipboard triage records:
`GET /api/supervisor/status` · `GET /api/supervisor/audit/recent` · `POST /api/supervisor/audit`

---

## 10. Gotify Notifications

**Module:** `gotify_service.py` (gateway) · **Server:** `gotify` container (`8081:80`).

Configuration (compose): `GOTIFY_URL=http://gotify:80`, `GOTIFY_TOKEN=<app token>`, `GOTIFY_ENABLED=true`.

Capabilities: `send_notification(title, message, priority, extras)` · scheduled status digests (`start_digest_scheduler`, `queue_digest`) · `health_check()` · `shutdown()`. Exposed to the AI as the `send_gotify_notification` tool; also used by the trading bot for arbitrage alerts.

---

## 11. Polaris Dashboard (host 7000)

Static page `Polaris_page.html` baked into an `nginx:alpine` image as `index.html`. **Depends on QuestDB.**

### Layout
- **Nav bar:** POLARIS logo, user selector (RICH / MATT), SESSION / MEMORY / RADAR dropdown HUD panels, WebSocket status indicator, radar status dot.
- **Chat (top section):** pinned message input + AUDIO RESPONSE checkbox; reverse-chronological exchange feed (newest at top), auto-pruned to 5 messages per sender; optimistic UI; 3-second cross-device auto-sync (polls `GET /api/history/<user_id>`, see §3.4).
- **Holodeck panel (bottom section):** resizable 150–800 px, collapsible; two tabs:
  - **📋 TASK FEED** — Socket.IO client on `:8082` (emits `join_tasks` on every connect/reconnect; the `tasks_dashboard` room name in the payload is ignored); 50-item rolling window; ▶ cyan started · ✓ green completed · ✗ red failed. The gateway's `join_tasks` handler joins the feed socket into room `status`, where all task lifecycle events broadcast, and emits an `active_tasks` snapshot on join — live push is functional.
  - **🧠 MEMORY VAULT** — REST to `:8082` (`/api/memory/vault/{stats,list,doc,write,update,query,delete,reindex}`): category-filtered browse list, hybrid/keyword/semantic search, in-browser read/edit (SAVE re-embeds the document into the ChromaDB index), new-doc compose (user follows the dashboard user switcher, source `dashboard`), confirm-guarded delete, one-click reindex, 30 s status polling with live doc/index counts. Replaces the former SSH SANDBOX tab — the `:7007` sandbox REST API remains for Polaris's own tools.

### HUD panels
- **SESSION:** session duration, sliding window.
- **MEMORY:** SQLite counts (users/messages/heuristics) + ChromaDB tabs GLOBAL / USER / CONTEXT with vector counts; delta-based ops/sec; polls `GET /api/memory/stats` every 5 s.
- **RADAR:** 350×350 canvas — concentric rings, position trail, X/Y coordinates, velocity, motion state, confidence; Socket.IO client on `:8083`.
- **Vision stream:** Socket.IO client on `:8083` receiving `vision_update` broadcasts (room `vision_dashboard`).

### Client connections (from `Polaris_page.html`)
| Target | Use |
|---|---|
| `http://<host>:8082` | Chat REST (`/api/chat`, `/api/history/<user>`, `/api/memory/stats`, `/api/tts*`, `/api/chat/history`) + Memory Vault REST (`/api/memory/vault/*`) |
| `ws://<host>:8082` (Socket.IO) | Chat `message` events + task feed |
| `<host>:8083` (Socket.IO) | CSI radar + vision stream |

The dashboard's `POST /api/chat` fetch **discards the HTTP response body** (source comment: "Do NOT add messages here — the WebSocket broadcast will add both user and Polaris messages") — chat bubbles render exclusively from socketio `chat_message` broadcasts, which carry the stripped reply (§3.2). The raw Ollama envelope in the HTTP return (§3.1 note) therefore has no rendering path on any client.

---

## 12. Deployment

### Compose service definitions (current, `docker-compose.yml`)

**polaris-gateway**
```yaml
polaris-gateway:
  build: { context: ./freeroam/polaris-gateway, dockerfile: Dockerfile }
  container_name: polaris-gateway
  ports: ["8082:5000", "8083:5001", "7007:7007"]
  volumes:
    - polaris_data:/app/data                                          # distillation DB
    - /mnt/containers/freeroam/data:/data/freeroam                    # bursts, telemetry, profiles, chat_history.json
    - /mnt/containers/freeroam/polaris-gateway/ingest:/data/freeroam/polaris-gateway/ingest
    - /mnt/containers/freeroam/polaris-gateway/playground:/data/freeroam/polaris-gateway/playground
    - /mnt/containers/freeroam/polaris-gateway/memory-vault:/polaris_memory_vault:rw
    - /mnt/containers/freeroam/polaris-gateway/vector-db:/data/polaris_vector_db:rw
    - /mnt/containers/freeroam/livecharts:/livecharts:rw
    - /mnt/containers/freeroam/polaris-gateway/playground/www:/playground/www:rw
    - /mnt/rippled-data/kokoro-cache:/app/kokoro-cache                # Kokoro model cache (sdc)
    - /home/causeiam/.ssh:/root/.ssh:ro                               # SSH keys (Mini Pi / Miss Pi)
    - /var/run/docker.sock:/var/run/docker.sock                       # container management
  environment:
    - GATEWAY_PORT=5000
    - OLLAMAHOST=172.17.0.1:11434
    - OLLAMA_MODEL=polaris-ai:latest
    - VISION_LLM_MODEL=polaris-ai:latest                        # Desktop Vision (LGI vision turns): primary image model
    - VISION_LLM_FALLBACK_MODEL=mistral-large-3:675b-cloud     # proven vision-capable fallback if the primary fails
    - DATA_DIR=/app/data
    - KOKORO_CACHE_DIR=/app/kokoro-cache
    - USER_DATA_DIR=/data/freeroam
    - LIVECHARTS_DIR=/livecharts
    - WWW_DIR=/playground/www
    - GOTIFY_URL=http://gotify:80
    - GOTIFY_TOKEN=<app token>
    - GOTIFY_ENABLED=true
  depends_on: [ollama]
  restart: unless-stopped
```

The Chat Gateway resolves Ollama via `OLLAMA_URL` (default `http://ollama:11434/api/generate` — Docker DNS on `docker-containers_default`); the compose `OLLAMAHOST` value is read by neither service, and `burst_receiver.py` hardcodes `http://localhost:11434` (dead in-container — §4 Configuration).

**polaris-dashboard**
```yaml
polaris-dashboard:
  build: { context: ./freeroam/polaris-dashboard, dockerfile: Dockerfile }
  container_name: polaris-dashboard
  ports: ["7000:80"]
  depends_on: [questdb]
  restart: unless-stopped
```

**freeroam** (one-shot Android APK builder)
```yaml
freeroam:
  build: { context: ./freeroam, dockerfile: Dockerfile }
  container_name: freeroam-apk-builder
  volumes: [ ./freeroam/build-output:/output ]
  restart: "no"   # one-shot: docker compose build freeroam && docker compose run --rm freeroam
```
Compiles the FreeRoam Android APK during image build (`RUN ./gradlew assembleDebug`), then copies `freeroam-debug.apk` to `./freeroam/build-output` on run and exits — build tooling, not running infrastructure.

Network: external `docker-containers_default`. Named volumes: `polaris_data`, `ollama_data`, `gotify_data`, `questdb_data` (the `ollama` service bind-mounts `/home/causeiam/.ollama` — `ollama_data` is declared but unused by it).

### Image contents

**Gateway Dockerfile** — `python:3.11-slim`; apt: `openssh-client`, `docker.io`, `curl`, `procps`; pip: flask / flask-cors / flask-socketio / requests / python-socketio / watchdog, `chromadb==0.5.23`, `faster-whisper`, `kokoro`, `soundfile`; `COPY *.py *.json *.html run_gateway.sh sandbox_static/`; `EXPOSE 5000 5001 7007`; `CMD ["./run_gateway.sh"]`.

**Entrypoint `run_gateway.sh`** — sets `FLASK_DEBUG=1` + `PYTHONUNBUFFERED=1`, then launches `burst_receiver.py` (:5001), `polaris_continuous_learning_gateway.py` (:5000), `sandbox_gateway.py` (:7007) as background processes and waits on all three PIDs.

**Dashboard Dockerfile** — `nginx:alpine`; `COPY nginx.conf` + `COPY Polaris_page.html /usr/share/nginx/html/index.html`; `EXPOSE 80`.

### Baked-source policy

Neither container bind-mounts its source code — all Python and HTML is baked into the image at build time. Any code change requires an image rebuild + container restart; the dashboard additionally requires a browser hard-refresh (Ctrl+Shift+R).

```bash
docker compose build polaris-gateway && docker compose up -d --no-deps polaris-gateway
docker compose build polaris-dashboard && docker compose up -d --no-deps polaris-dashboard
```

`--no-deps` guarantees linked services (`ollama`, `questdb`) are never recreated or restarted — only the named container is swapped.

### Required Ollama models

- `polaris-ai:latest` — chat model (compose `OLLAMA_MODEL`). Defined by `freeroam/polaris-gateway/Modelfile`: `FROM glm-5.3-flash:cloud`, temperature 0.7, `num_ctx 8192`, base SYSTEM persona with the North Star primacy block (§3.5). Modelfile edits are applied with `ollama create polaris-ai:latest -f <Modelfile>` on the running ollama container — instant manifest swap, no restart
- `nomic-embed-text` — embeddings (vector memory + Memory Vault index)

```bash
curl -s http://localhost:11434/api/tags          # verify both are present
curl -X POST http://localhost:11434/api/pull -d '{"name": "nomic-embed-text"}'
```

---

## 13. Health & Operational Commands

```bash
curl -s http://localhost:8082/api/status             # Chat Gateway
curl -s http://localhost:8082/api/status/tasks       # Task tracking
curl -s http://localhost:8082/api/memory/vault/stats # Memory Vault (documents + indexed entries)
curl -s http://localhost:8083/api/status             # Burst Receiver — 403 from the Mainframe shell by design (LAN-only guard sees Docker-bridge IP); call from an allowlisted device (Rich/Matt/MissPi) or use docker logs polaris-gateway
docker exec polaris-gateway curl -s --max-time 4 http://ollama:11434/api/tags   # in-container Ollama path used by chat + Desktop Vision ladder (http://ollama:11434)
curl -s http://localhost:7007/api/status             # Sandbox Gateway
curl -s http://localhost:7000/ | head -5             # Dashboard page
curl -s http://localhost:11434/api/tags              # Ollama models
docker exec polaris-gateway tail -50 /data/freeroam/operator/operator_transcripts.log   # transcripts (root-owned bind mount — read inside the container)
docker logs polaris-gateway 2>&1 | grep -E 'DEBUG|Tool Execution|GUARDRAIL'              # stripper / parser / guardrail evidence
```

---

## 14. Gateway File Inventory

| File | Role |
|---|---|
| `polaris_continuous_learning_gateway.py` | Chat Gateway app — HTTP + Socket.IO, North Star-primacy system prompt (§3.5), tool calling + stripper pipeline (§3.2), voice pipeline, sessions, heuristics, task tracking, Desktop Vision turns (§3.1) |
| `burst_receiver.py` | Burst Receiver app — telemetry ingestion, CSI radar, vision stream (X17 webcam `vision_frame` / `radar_motion` feeds), Ollama image-analysis endpoints (in-container Ollama URL dead, §4 Configuration) |
| `sandbox_gateway.py` | Sandbox Gateway app — Mini Pi SSH console |
| `run_gateway.sh` | Container entrypoint — launches all three services |
| `memory_db.py` | SQLite memory layer (users, messages, heuristics, profiles, lists) |
| `vector_memory.py` | ChromaDB vector layer — embeddings via `nomic-embed-text` |
| `memory_vault.py` | Memory Vault markdown store + 30 s re-index watchdog |
| `memory_routes.py` | Blueprint `/api/memory/vault/*` |
| `memory_tools.py` | Memory Vault query tool (LLM tool schema) |
| `supervisor_routes.py` | Blueprint `/api/supervisor/*` (screen/clipboard triage) |
| `file_tools.py` | LLM file operation tools |
| `ssh_tools.py` | LLM SSH execution tools (`ssh_execute`, `ssh_check_connectivity`) |
| `gotify_service.py` | Gotify notifications, digest scheduler, LLM tool schema |
| `playground_tools.py` | Web asset publishing, sandbox code execution, service spawning |
| `distill_memory.py` | Nightly fact extraction for the memory loop |
| `system_tools.py` / `livecharts_tools.py` / `www_tools.py` | Standalone toolkit modules (not wired into the running Chat Gateway) |
| `sandbox_static/` | Sandbox web dashboard assets |
| `polaris_heuristics.json` | Persisted heuristics store |
| `Dockerfile`, `requirements.txt`, `Modelfile`, `polaris-gateway.service` | Build config, dependency manifest, Ollama model definition, host systemd unit |

Legacy entrypoints on disk (copied into the image, **not launched**): `freeroam_gateway.py`, `polaris_gateway.py`, `polaris_websocket_gateway.py`, `websocket_server.py`, `status_dashboard.html`.

**Dashboard files** (`freeroam/polaris-dashboard/`): `Polaris_page.html` (the entire UI — single-file HTML/CSS/JS), `nginx.conf`, `Dockerfile`.