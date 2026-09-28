# Polaris Dashboard — Looking Glass Data Interface
## Version 4.4 — Holodeck Tactical Operations Interface (September 2026)

---

## Overview

The Polaris Dashboard is the primary operator console for the North Star ecosystem — an immersive, JARVIS-inspired tactical command interface with a dark HUD aesthetic (cyan/teal glow, scanline overlay, bubbly rounded elements) optimized for desktop and mobile viewing.

The entire interface is a **single-file application**: `Polaris_page.html` (~2,250 lines of self-contained HTML + CSS + JS, no build step) served from a lightweight nginx container. All intelligence — chat, memory, voice synthesis, task execution, radar/vision telemetry — lives server-side in the **Polaris Gateway**.

| Component | Address |
|---|---|
| Dashboard UI (nginx static) | `http://localhost:7000/` or `http://192.168.50.51:7000/` |
| Polaris Gateway — REST + Socket.IO | `http://<host>:8082/` |
| Radar / Vision Socket.IO bridge | `http://<host>:8083/` (gateway container port 5001) |
| Active users | RICH, MATT |

**Version 4.4 headline features:**

- **Holodeck split-view** — resizable top chat pane + bottom **TACTICAL OPERATIONS** panel with two tabs: **TASK FEED** (live Gateway task events) and **MEMORY VAULT** (full CRUD + semantic search over Polaris's long-term memory).
- **Hardened sequential TTS voice pipeline** — one voice at a time, error-storm eliminated (see TTS section).
- **Per-message audio controls** — every Polaris bubble carries a ▶ replay button and a × close button.
- **Miss Pi Vision Stream** dropdown HUD — live camera-detection telemetry and narration.
- **CSI Radar Tracking** dropdown HUD — position/velocity/motion telemetry on an animated canvas scope.
- Dropdown HUD panels for Session Diagnostics and Memory Architecture.

---

## Changelog: v4.3 → v4.4

| Change | Detail |
|---|---|
| SSH Sandbox tab retired | The bottom panel's second tab is now **MEMORY VAULT** — a full document browser/editor for the Polaris memory vault (previously the SSH Sandbox command runner). |
| TTS error-storm fix | Every cleanup site now uses the spec-safe `resetAudioElement()` helper instead of `ttsPlayer.src = ''`. The old empty-string reset re-invoked the media load algorithm → instant `error` event → `onerror` re-assigned `src = ''` → infinite error loop (~60 console errors from ~3 requests), with stale queued errors cascading into repeated `/api/tts` gateway hits. |
| onerror re-entrancy guard | `setupPlayerHandlers` now ignores error events when the element has no `src` attribute — a genuine playback failure logs exactly once. |
| Sequential TTS queue | One shared hidden `<audio id="ttsPlayer">` element; messages synthesize and play strictly one at a time in arrival order. |
| Per-message replay | ▶ re-synthesizes a message on demand (jumps the queue head); × closes the bubble and stops/dequeues its audio. |
| Miss Pi Vision dropdown | New nav dropdown streaming live detection results (class, confidence, narration) from the vision pipeline. |
| Stale-doc corrections | Chat window is the **last 10 messages total** (not 5 per sender); fallback history sync runs every **10 s** with WebSocket as primary (not a 3 s poll); Radar/Vision connect via Socket.IO on **:8083** (not a raw WebSocket on :5001); bottom panel resizes to **20 %–70 %** of container height (not fixed pixel bounds). |

---

## System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│            OPERATOR BROWSER (desktop / RedMagic phone)          │
│        Polaris_page.html — single-file tactical HUD            │
└──────┬─────────────────────────────────────────┬──────────────┘
       │ static HTML/JS (HTTP :7000)               │ Socket.IO + REST
       ▼                                          ▼
┌──────────────────────┐      ┌─────────────────────────────────────┐
│ POLARIS DASHBOARD    │      │ POLARIS GATEWAY (Dell Mainframe)     │
│ nginx:alpine, :7000  │      │                                       │
│ COPY'd index.html    │      │ REST :8082                            │
│ wildcard CORS + SPA  │      │  /api/chat   /api/history/{user}      │
│ fallback             │      │  /api/tts    /api/tts/cancel           │
└──────────────────────┘      │  /api/memory/stats                    │
                              │  /api/memory/vault/* (8 endpoints)   │
                              │                                       │
                              │ Socket.IO :8082                       │
                              │  chat_message · tool_execution         │
                              │  task_started/_completed/_failed     │
                              │                                       │
                              │ Socket.IO :8083 (container :5001)     │
                              │  radar_update/_status/_snapshot      │
                              │  vision_update/_history              │
                              │  polaris_vision                      │
                              └────────────┬──────────────────────────┘
                                           │
                    ┌──────────────────────┼─────────────────────┐
                    ▼                      ▼                     ▼
             SQLite memory         ChromaDB vectors        Kokoro TTS engine
             (structured store,    (nomic-embed-text,      (24 kHz WAV synthesis
              chat + heuristics +   vault_documents          for /api/tts)
              memory vault)        collection)
```

---

## Holodeck Split-View Layout

```
┌──────────────────────────────────────────────────────────────┐
│ NAV: POLARIS │ USER selector │ Chat History │ Memory │        │
│      Radar │ Vision │ WS status                            │
├──────────────────────────────────────────────────────────────┤
│ TOP — CHAT                                                    │
│   [pinned message input] [AUDIO RESPONSE toggle]              │
│   reverse-chronological feed (10-message window)              │
├──── RESIZE HANDLE (drag: bottom pane 20 % – 70 %) ────────────┤
│ BOTTOM — TACTICAL OPERATIONS                                   │
│   [▼ HOLODECK collapse] [TASK FEED] [MEMORY VAULT]            │
│   active tab content area                                     │
└──────────────────────────────────────────────────────────────┘
```

- **Collapse toggle** — collapses the bottom pane to a slim `▼ HOLODECK` bar; expands to `▲ HOLODECK`.
- **Resize** — drag the grip handle to set the bottom pane height as a percentage of the holodeck container, clamped to **20 %–70 %**. Dragging also clears the collapsed/expanded class state.

## Navigation Bar & Dropdown HUD Panels

Only one dropdown is open at a time; `×` closes. The **Radar** and **Vision** buttons carry live status dots that mirror their socket connection state.

| Nav button | Dropdown panel | Contents |
|---|---|---|
| Chat History | **Session Diagnostics** | SESSION ID (`SESSION_{USER}_{epoch}`), ACTIVE USER, MESSAGES, DURATION (`mm:ss`), and a **Sliding Window** preview of the last 10 messages (40-char snippets). Updates every 1 s. |
| Memory | **Memory Architecture** | SQLite block (USERS / MESSAGES / HEURISTICS / OPS-SEC) and ChromaDB block with GLOBAL / USER / CONTEXT tabs (vector counts) + OPS/SEC. Polls `/api/memory/stats` every 5 s; ops/sec computed from counter deltas between polls. |
| Radar | **CSI Radar Tracking** | 350 × 350 canvas scope + telemetry rows (see CSI Radar section). |
| Vision | **Miss Pi Vision Stream** | Detection telemetry + narration (see Miss Pi Vision section). |

## Chat System

**Identity** — user selector offers **RICH** / **MATT** (default `rich`).

**Transport** — Socket.IO to `:8082` is the primary channel. Sending a message POSTs `/api/chat` with `{prompt, user_id, client_source: 'dashboard_browser', session_id: '<user>_default'}` and does **not** append anything locally: the gateway broadcasts every message (user + Polaris) back over Socket.IO (`chat_message`), which is the single render path for all devices — duplicate-proof by construction. Incoming `chat_message` events are filtered to the active user or Polaris; `tool_execution` events are logged to console only (the follow-up chat message reports completion).

**Fallback sync** — every 10 s, `syncChatHistory()` fetches `/api/history/{user}` and content-diffs against the local `messageHistory`; any message the WebSocket missed is inserted. Failures are silent (WebSocket remains primary).

**Rendering** — `addMessageToChat()` prepends newest at the top and the feed keeps only the **last 10 messages total**. Polaris bubbles get audio controls (▶ replay / × close). Content containing `{"tool": …}` JSON is reduced to the human-readable text preceding the tool call. On page load, `fetchChatHistory()` renders the last 10 history messages newest-first with **no auto-play**.

---

## TTS Voice Pipeline (v4.4 Hardened)

**Model** — one shared, hidden `<audio id="ttsPlayer">` element driven by a strictly sequential queue.

**Playback flow**

1. A Polaris message arrives → `playTTSResponse()` (gated by the **AUDIO RESPONSE** checkbox) → `enqueueTTS()` (skips if that message is already playing or queued; supports `atFront` priority).
2. `pumpTTSQueue()` plays **one message at a time**, in arrival order:
   - `resetAudioElement(ttsPlayer)` — clears `onended`/`onerror` handlers, revokes the previous blob object URL, then `removeAttribute('src')` + `load()`.
   - `POST /api/tts {text}` (abortable via `AbortController`) → Kokoro WAV blob → `URL.createObjectURL()` → assign `src` → `setupPlayerHandlers()` → `play()`.
   - Polls every 200 ms until the message finishes or is stopped, then advances the queue.
3. Stops — `stopAllAudioPlayers()` aborts every per-message controller and resets the element; `cancelCurrentStream()` additionally POSTs `/api/tts/cancel` and tears down any MediaSource.

**Why `resetAudioElement()` exists (the v4.4 fix)** — assigning `ttsPlayer.src = ''` re-invokes the media load algorithm; an empty source fails instantly and fires an `error` event; the `onerror` handler then assigned `src = ''` again → infinite error loop. Because error events queue asynchronously, stale errors could also fire **after** the next message's blob loaded — killing fresh playback and cascading into repeated `/api/tts` requests. A src-less element (`removeAttribute('src')` + `load()`) fires **no** error event, making cleanup silent. As belt-and-braces, `onerror` begins with a re-entrancy guard — `if (!ttsPlayer.getAttribute('src')) return;` — so a genuine decode/network failure still logs exactly once.

**Per-message controls** — ▶ re-synthesizes that message on demand (inserts at the queue front); × closes the bubble, dequeues it, and aborts playback if active. The **AUDIO RESPONSE** toggle gates auto-play only — manual ▶ always works.

**Known-good console signatures** — one `[TTS] Synthesizing message …` and one `[TTS] Finished message …` per message; **zero** `[TTS] Audio playback error` entries in normal operation.

**Legacy note** — `streamTTSAudio()` (`/api/tts-stream`, MediaSource chunked streaming) is retained as dead code; the live pipeline uses whole-blob `/api/tts`.

## Task Feed System

- Connects Socket.IO to `:8082`, emits `join_tasks {room: 'tasks_dashboard'}`.
- Events → feed rows: `task_started` → **▶** TASK STARTED · `task_completed` → **✓** TASK COMPLETED · `task_failed` → **✗** TASK FAILED. Each row shows `task_id • HH:MM:SS`.
- Newest at top; rolling window keeps the **last 50 items**.
- Auto-reconnect: up to 10 attempts, 1 s delay; websocket transport with polling fallback.

## Memory Vault System

Full operator access to Polaris's long-term distilled knowledge — the Obsidian-style markdown vault bind-mounted at `/polaris_memory_vault` (categories 00-Core → 05-Attachments), embedded into the ChromaDB `vault_documents` collection via `nomic-embed-text` with a 30 s watchdog re-embedding changed files.

**UI**

- **Status bar** — gateway online indicator, doc / indexed counts, vault path label.
- **Browser panel** — category filter (ALL / CORE / ARCHITECTURE / USER / LESSONS / JOURNAL / ATTACHMENTS), search input + SEARCH, search-mode selector (**HYBRID / KEYWORD / SEMANTIC**), BROWSE / NEW / REINDEX buttons, and the document list (title, meta, snippet — up to 200 docs).
- **Editor panel** — document title + meta, SAVE / DELETE actions, a compose bar (category + title) revealed when composing a NEW document, full-width textarea, inline status messages. The textarea stays disabled until a document is selected or NEW is pressed. CORE is deliberately absent from the new-document category list (protected doctrine).

---

## CSI Radar Integration

- Socket.IO to `:8083` (gateway container port 5001), emits `join_radar {room: 'radar_dashboard'}`.
- Events — `radar_update` (x, y, velocity, motion_detected, motion_confidence), `radar_status`, and `radar_snapshot` (initial state on connect: `current_position[]`, velocity, motion fields).
- **Canvas scope** (350 × 350) — concentric range rings, crosshairs, N/E/S/W compass labels, glowing position dot with a fading trail (**max 20 points**).
- **Telemetry rows** — POSITION X, POSITION Y, VELOCITY, MOTION (DETECTED / NONE), CONFIDENCE (%), STATUS (CONNECTED / DISCONNECTED / ERROR).

## Miss Pi Vision Stream

- Socket.IO to `:8083` (same bridge as radar), emits `join_vision {room: 'vision_dashboard'}`.
- Events — `vision_update` (detections array → count, primary object `CLASS (CONFIDENCE %)`, narration), `vision_history` (batch → displays most recent), `polaris_vision` (narration-only integration channel).
- **UI** — stream status line (**WAITING FOR STREAM** → **DETECTIONS ACTIVE** green / **STREAM INACTIVE** red), italic narration quote box (default "Polaris is watching…"), DETECTIONS, PRIMARY OBJECT, LAST UPDATE (`HH:MM:SS`), STREAM rows.

## API Reference

**REST** (base `http://<host>:8082`)

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/chat` | Send prompt `{prompt, user_id, client_source: 'dashboard_browser', session_id: '<user>_default'}` |
| GET | `/api/history/{user}` | Chat history (oldest-first; UI renders the last 10) |
| POST | `/api/tts` | Kokoro TTS synthesis `{text}` → WAV blob |
| POST | `/api/tts/cancel` | Cancel in-flight synthesis `{session_id}` |
| GET | `/api/memory/stats` | SQLite + ChromaDB counters (Memory Architecture panel) |
| GET | `/api/memory/vault/stats` | Vault totals (docs / indexed) |
| GET | `/api/memory/vault/list?limit=200&category=` | Browse vault documents |
| POST | `/api/memory/vault/query` | Search `{query, mode: hybrid\|keyword\|semantic}` |
| GET | `/api/memory/vault/doc?path=` | Fetch one document |
| POST | `/api/memory/vault/write` | Create a document |
| POST | `/api/memory/vault/update` | Save edits |
| POST | `/api/memory/vault/delete` | Delete a document |
| POST | `/api/memory/vault/reindex` | Re-embed the vault into ChromaDB |

**Socket.IO channels**

| Port | Room (join emit) | Events |
|---|---|---|
| 8082 | — | `chat_message`, `tool_execution` |
| 8082 | `tasks_dashboard` (`join_tasks`) | `task_started`, `task_completed`, `task_failed` |
| 8083 | `radar_dashboard` (`join_radar`) | `radar_update`, `radar_status`, `radar_snapshot` |
| 8083 | `vision_dashboard` (`join_vision`) | `vision_update`, `vision_history`, `polaris_vision` |

All Socket.IO clients use `transports: ['websocket', 'polling']` with reconnection (10 attempts, 1 s delay). The client library is Socket.IO **4.5.4** loaded from the CDN in `<head>`.

---

## Build & Deploy

The HTML is **baked into the image** (no bind mount) — every source change requires a rebuild:

```
# Rebuild + redeploy (only the dashboard service; never touch ollama)
docker compose build polaris-dashboard
docker compose up -d polaris-dashboard

# Verify
docker ps --filter name=polaris-dashboard
curl -s http://localhost:7000/ | head
```

- **Compose service** — `polaris-dashboard` in `/home/causeiam/docker-containers/docker-compose.yml`; build context `./freeroam/polaris-dashboard`; ports `7000:80`; `depends_on: questdb`.
- **Dockerfile** — `FROM nginx:alpine`; `COPY nginx.conf → /etc/nginx/conf.d/default.conf`; `COPY Polaris_page.html → /usr/share/nginx/html/index.html`.
- **nginx** — `try_files $uri $uri/ /index.html` SPA fallback; wildcard CORS headers; `OPTIONS` preflight → `204`.
- After a rebuild, **hard-refresh the browser (Ctrl+Shift+R)** to bypass cached HTML.

## Browser Verification Checklist (v4.4)

**Shell & panels**
- [ ] POLARIS logo renders; WS indicator shows ONLINE (cyan)
- [ ] User selector offers RICH / MATT
- [ ] Chat input pinned below nav; AUDIO RESPONSE checkbox visible
- [ ] Messages appear newest-at-top; feed prunes to the last 10
- [ ] Chat History / Memory / Radar / Vision dropdowns open, show live values, close via ×
- [ ] Memory panel: SQLite + ChromaDB (GLOBAL/USER/CONTEXT) counts refresh every 5 s
- [ ] Session Diagnostics: SESSION ID / ACTIVE USER / MESSAGES / DURATION tick every second

**Holodeck bottom panel**
- [ ] Expand/collapse toggle (▼/▲ HOLODECK) works
- [ ] Drag resize handle; bottom pane clamps between 20 % and 70 %
- [ ] TASK FEED / MEMORY VAULT tabs switch cleanly
- [ ] Task Feed rows arrive with ▶/✓/✗ icons + timestamps; window caps at 50
- [ ] Memory Vault: BROWSE lists docs; HYBRID/KEYWORD/SEMANTIC search return; open → edit → SAVE persists; NEW composes with category + title; DELETE removes; REINDEX succeeds

**Voice**
- [ ] Each Polaris reply auto-plays once: one `/api/tts` request, one finish log, zero `[TTS] Audio playback error`
- [ ] Back-to-back messages queue and play strictly one at a time (no overlap, no cascade)
- [ ] ▶ replays an old message immediately; × on a playing message stops its audio
- [ ] Unchecking AUDIO RESPONSE silences auto-play; re-checking restores it (manual ▶ still works)

**Sensors**
- [ ] Radar: canvas rings/crosshairs/compass render; POSITION/VELOCITY/MOTION/CONFIDENCE/STATUS update live
- [ ] Vision: WAITING FOR STREAM when idle; DETECTIONS ACTIVE + narration while the camera pipeline runs

**Resilience**
- [ ] Restarting the gateway → all four sockets auto-reconnect; indicators recover
- [ ] Messages missed by WebSocket are recovered by the 10 s fallback sync
- [ ] Mobile viewport sanity check (Red Magic 9 Pro over LAN)

## Technical Notes

- **Single-file architecture** — no bundler, no framework; vanilla JS + Socket.IO 4.5.4 (CDN). All state lives in module-scope variables (`currentUserId`, `messageHistory`, `ttsQueue`, `messageAudioPlayers`, per-system socket handles).
- **Timing constants** — memory stats 5 s · session diagnostics 1 s · TTS completion poll 200 ms · chat fallback sync 10 s · socket reconnect 10 × 1 s · task-feed window 50 · radar trail 20 · chat window 10 · vault list 200.
- **Duplicate safety** — the sending device never appends locally; rendering happens exclusively via the gateway's Socket.IO broadcast, plus content-based diffing in the fallback sync.
- **Reverse-chronological window** — the UI intentionally keeps only the last 10 bubbles; full history remains in the gateway's SQLite store.
- **Mobile** — `clamp()`-based typography and touch-friendly targets; validated on the Red Magic 9 Pro.
- **History** — this document supersedes v4.3 (SSH Sandbox era); the previous version remains retrievable from git history.
