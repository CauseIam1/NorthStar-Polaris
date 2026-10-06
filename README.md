# Project North Star

# Project North Star - Trading Bot & Holodeck Dashboard
**Document Status: PRODUCTION ACTIVE**
This section of the document defines the overarching vision, theoretical framework, and operational rules of the ecosystem. It serves as the architectural north star for the private AMM matrix and North Star Holodeck dashboard.

## 1. System Purpose & The Zero-Fiat Philosophy
This ecosystem is a closed-loop inventory management and mesh rebalancing engine operating on the XRP Ledger.

* **Primary Objective:** Maximize the raw quantity volume of the asset "pile" (specifically XRP and whitelisted meme coins).
* **Mechanism:** Capture network inefficiencies and execute multi-hop arbitrage loops to recycle 0.05% LP fees back into operator-owned pools.
* **The Zero-Fiat Rule:** The system DOES NOT hold, track, or calculate value in fiat currency or stablecoins. Stablecoins (RLUSD, USDC, USDT) are strictly temporary, pass-through atomic settlement routing nodes used only for transient execution paths.
* **The Dinghy Philosophy:** We have zero fear of holding any asset within our carefully curated AssetWhitelist. The primary objective is absolute asset accumulation (stacking more whitelisted tokens), never converting to fiat.

## 2. Three-Wallet Topography & Execution Partitioning
Capital and execution logic are cryptographically partitioned into three isolated r-addresses to ensure zero resource contention.

### A. COLD_WALLET_ADDRESS (Liquidity Foundation - AMM Pools)
* **Role:** The institutional vault and primary liquidity engine.
* **Function:** Exclusive creator and deployer of private AMM pools; holds foundational LP tokens.
* **Constraint:** Strictly isolated; never interacts with live trading engines and requires no hot-key access.

### B. MPT_RPN_WALLET_ADDRESS (Bot Page - Live Trading Engine)
* **Role:** Programmatic hot wallet for the automated 24/7 execution engine.
* **Function 1:** Executes cumulative pile "flip-flops" across our asset pools (Matrix Rotator).
* **Function 2:** Executes The "Splash and Strike" Sequence (StreamProcessor).
* **Function 3:** Executes rapid, multi-hop mesh network arbitrage to capture micro-inefficiencies (Four-Hop Atomic Engine).
* **Function 4:** Runs macro swing trades on external pools via stablecoin routers (Offshore Rig Engine).
* **Constraint:** Strictly rules-based execution governed by liquidity guardrails; operates with hard-capped capital allocation per strategy to prevent broader portfolio exposure.

### C. TRADING_WALLET_ADDRESS (Radar Page - Manual Swapping)
* **Role:** Operator's high-density command portal for active momentum trading.
* **Function:** Executes cumulative pile "flip-flops" across our asset pools via Human-In-The-Loop using Xaman.
* **Constraint:** Utilizes secure Xaman SDK push-to-sign payloads; private keys are never exposed to the server.
## 3. Mesh Architecture & Asset-to-Asset Matrix
The bot loads a predefined list of core assets and dynamic routing tokens. It maps every AMM pool where both assets are whitelisted, creating a highly interconnected, trusted trading graph.

* **The Closed Mesh:** The bot monitors all AMM pools where both assets are whitelisted, creating a decentralized, direct asset-to-asset mesh covering 34+ active trade pairs.
* **Canonical Ordering:** The bot strictly adheres to XRPL AMM standards (XRP takes precedence; issued tokens are sorted by hex code, then issuer) to accurately map the mesh and route transactions without failure.
* **QuestDB as Single Source of Truth:** The bot entirely bypasses internal state caching for active trades. It queries QuestDB directly to instantly read current ledger states, live wallet balances, and the exact status of all active positions.

## 4. Multi-Strategy Execution Engine (The Apex Predator)
The Trading Bot acts opportunistically within the mesh, executing trades based on **five distinct, simultaneous accumulation strategies** running in parallel:

> **Live mode (since Sept 25, 2026):** `FlipFlopEvaluator` is the sole autonomous-swap engine — §4.1 tranche execution, §4.3, §4.4 and §4.5 swaps are env-gated dormant via `.env` kill switches (MRE ticks analysis-only, FHAE discovery-only, MPT & StagnantPositionMonitor run tracking + alerts only). The operator-driven swap path is **§4.6 Manual Injection Portal** below.

### 4.1 Mean Reversion Engine (Matrix Rotator - Core Strategy)
* **Purpose:** Adaptive mean-reversion trading on internal AMM pools with tranche-based laddering.
* **Mechanism:** Monitors pool ratios against a rolling macro baseline (4-12 hour anchor). When a pool's ratio breaks equilibrium (2σ+ deviation), the engine executes SHORT-style trades that fade price pumps and sweep profits on mean reversion.
* **Key Features:**
  - **Adaptive Tranche Sizing:** Dynamic sizing driven by Velocity × Resistance × Confidence factors.
  - **Pool-Tier Classification:** Pools classified as DEEP_HUB (2.5× tolerance, 6h half-life), MODERATE (1.5×, 3h), or THIN_EDGE (1.0×, 1h) with tier-scaled thresholds.
  - **Dual-Window Evaluation:** Compares tactical short window against macro baseline to avoid false entries during slow digestion.
  - **Mesh Stress Gating:** Gates new entries when broader mesh is absorbing liquidity waves (aggregate score > 0.70).
  - **Variance-Decay Exit:** Exits early when rolling variance drops below 5% of peak (provided 25%+ profit locked).
* **Target:** ~30% accumulation per cycle on mean-reversion flips.
* **Capital Allocation:** 80% Core War Chest (primary pools), 20% Matrix Rotator (flip-flop trades).

### 4.2 StreamProcessor (Splash and Strike Sequence)
* **Purpose:** Real-time reaction to external actors creating slippage opportunities in our pools.
* **Mechanism:** Continuously polls the XRPL ledger for new transactions affecting our AMM pools. When an outside trader imbalances a pool with a large order causing heavy slippage, the bot immediately executes a counter-trade to absorb the mathematical advantage.
* **Key Features:**
  - **Mesh Stress Index Integration:** Records stress events on splash detection to gate concurrent entries.
  - **Alert Rule Scheduler:** Fires real-time notifications via Gotify when significant events occur.
* **Target:** Capture transient inefficiencies created by external traders.

### 4.3 Four-Hop Atomic Engine (Riskless Arbitrage Loops)
* **Purpose:** Execute ultra-fast, low-margin (0.01%) atomic arbitrage loops on the private AMM mesh.
* **Mechanism:** Scans for 4-hop paths (A → B → C → D → A) where stablecoins (RLUSD, USDC, USDT) can appear as intermediate nodes but never as start/end assets.
* **Key Features:**
  - **Dedicated Scheduler:** Runs on 1-5 second polling interval with zero latency impact on reactive StreamProcessor.
  - **Adaptive Position Sizing:** Caps at 50% of pool depth to prevent self-slippage.
  - **Zero Balance Enforcement:** All stablecoin balances must be 0 at end of each execution block.
  - **Telemetry Tracking:** Persists execution metrics to QuestDB (loops executed, profit XRP, latency).
* **Target:** 0.0025 XRP profit per loop (0.01% margin on 25 XRP capital).

### 4.4 Offshore Rig Engine (Macro Swing Trading)
* **Purpose:** Headless, macro-level statistical arbitrage on external public AMM pools.
* **Mechanism:** Monitors external pools (specifically those containing RLUSD or USDC) for macro price divergences on whitelisted native assets (SGB, XAH, ATM, XPM). Executes slow "flip-flop" trades between native assets and eventually sweeps profit back into core XRP pile.
* **Key Features:**
  - **Multi-Timeframe Analysis:** Tracks 7-day, 14-day, and 30-day moving averages with z-score signals.
  - **Adaptive Tranche Sizing:** Velocity × Resistance × Confidence multipliers.
  - **Hard Capital Ceiling:** 500 XRP equivalent per cycle maximum.
  - **Parallel Execution:** Runs independently alongside StreamProcessor and MeanReversionEngine.
* **Target:** Macro divergences (2σ+ z-score) on 7-30 day windows.

### 4.5 Stagnant Position Monitor (Pile Accumulation Swaps)
* **Purpose:** Automated pile-accumulation swap trigger for 19 whitelisted XRP pairs.
* **Mechanism:** Monitors tracked positions for peak detection and stagnation conditions. When a pair's ratio pumps +30% or more and then goes flat (variance < 2% for 15+ minutes), triggers a swap back to grow the pile.
* **Key Features:**
  - **Force Swap at 25%:** CRITICAL - if PnL > 25%, FORCE THE SWAP before it drops below 25%.
  - **Peak Tracking:** Tracks peak ratios and PnLs for all monitored positions.
  - **Stagnation Detection:** Identifies low-variance consolidation after pumps.
  - **Event Logging:** Persists STAGNATION_EXIT events to QuestDB via ManualPositionTrackerService.

### 4.6 Manual Injection Portal (Operator-Driven Revival Swaps — live Oct 5, 2026)
* **Purpose:** Human-in-the-loop manual inject-swaps through the **bot wallet (MPT_RPN_WALLET_ADDRESS)** to revive dust-frozen meme pairs — a genuine swap re-bases the FlipFlop pair/size and re-arms the engine on the new position.
* **Surface:** Bot-page right-HUD `InjectionModal` (issuer-aware input/output dropdowns, XRP/RAW spend modes, **pre-flight review panel** — From/To issuer-aware labels, mode-aware spend, converted raw units, XRP value, estimate-only receive, price source, **est. impact (approx)** (micro-probe marginal-rate row: emerald ≤3% / amber ≤5% / red >5% — meme→meme slippage boundary), and one-tap **sizing chips** (6/7.5/10/12/25/50 XRP, XRP mode) — behind a **3s hold-to-confirm**; the single-click deploy / Ctrl+Enter bypass is removed. SKIPPED → amber banner, success → tx hash) → `POST /api/trading-bot/inject-swap` on :8080 (TES execution with fee-preserving pre-flight).
* **Pricing (cold-start safe):** `OrderBookGraph` edges first; on a cold boot the graph is empty until edges hydrate, so both the XRP-budget conversion and the pool pre-flight fall back to the `AmmPriceOracle` (local `amm_info` marginal prices, 15 s cache). A 400 refusal only when neither source can price the route. `GET /api/radar/assets` feeds the dropdowns (XRP + whitelist memes; stablecoins listed but `injectable: false` per the Zero-Fiat Rule — pass-through routing nodes only).
* **Validator:** `POST /api/radar/validate-swap` returns `VALID / NO_DIRECT_AMM_POOL / INSUFFICIENT_DEPTH (graph-source only)` plus `priceSource: GRAPH | ORACLE | NONE`; display names auto-normalize to canonical hex via `AssetWhitelist.stringToHex` (idempotent), so plain names like `Hand` or `$TRUMP` paste straight in.
* **Revival recipe (Oct 5, 2026 — pre-validated, awaiting operator):** 8 injectable memes × 12 XRP = 96 XRP (bot wallet 184.99 XRP): **XAH, SGB, Hand, LedgerGirl, LEDGERGUY, BEAR, ATM, X** — all `VALID / ORACLE`. **DELANI dropped by operator decision (not an asset we care to hold)**; re-addable any time by pasting its issuer into `AssetWhitelist` + one repackage. Injection size is operator-chosen per pool at inject time (sizing chips + est.-impact row: ~6–8 XRP on the shallowest pools up to 25–50 XRP on the deepest); the autonomous flop engine mirrors the same boundary with the phase-4 retune (`FLIPFLOP_FLOP_SPEND_CAP_XRP=7.5`, `FLIPFLOP_SLIPPAGE_TOLERANCE=0.05`, `FLIPFLOP_MIN_POSITION_XRP=5`).
* **Full implementation details:** [trading-bot.md](trading-bot/trading-bot.md) — "Manual Injection Portal" section in Recent Additions.

## 5. Real-Time Infrastructure

### 5.1 QuestDB State Ingestion
* **Time-Series Data:** All trade executions, balance snapshots, alert firings, and system telemetry persisted via PostgreSQL wire protocol (port 8812).
* **Live Accumulation Tracking:** Queries provide millisecond-level visibility into Cold Wallet holdings, deployed liquidity, and total accumulated pile.
* **Idempotent Currency Handling:** Resilient hex-to-string and string-to-hex conversions for standard 3-character tickers and 40-character XRPL hex codes.
* **Critical Constraint:** Always map time objects via `java.sql.Timestamp.from(instant)` or epoch microseconds when binding parameters. Never bind `java.time.Instant` directly into JDBC prepared statements.

### 5.2 Local Rippled Node
* **Endpoint:** `http://rippled:5005` (private network)
* **Function:** Direct JSON-RPC access for transaction submission and ledger state queries.
* **Config (live, `rippled/config/rippled.cfg` as of 2026-10-02):** `[node_db]` NuDB + `online_delete=2048` / `[ledger_history] 2048` — a ~2.5–4 h mainnet window sized for live-position verification depth. The bot's 22-day latest-run basis walk **cannot** resolve inside this window — proven live (0 swaps/0 pairs after the 16:20 recreate) — so `SWAP_HISTORY_RPC_URL` (s2.ripple.com:51234) was un-retired 2026-10-02 16:2x as the walk endpoint, to be retired again once `SwapHistoryService` gains a persisted `shared/` JSON checkpoint (cold-start-from-file + local incremental walks, planned same day) — **superseded 2026-10-03 02:42 UTC, Option B shipped:** `SwapHistoryService` now boots the 22-day basis from a durable checkpoint — `/app/shared/swap_history.json` on the existing `./shared:/app/shared` bind (the same one `mute_state.json` rides), atomically rewritten (temp-file + `ATOMIC_MOVE`) after every refresh that ingests swaps, schema-version + wallet validated on load. Verified live: `02:42:57` cold start 30 swaps/22 pairs → file written; `02:43:40` warm restore of the identical basis from disk with **zero RPC walk**. s2 remains **armed as tier-3 last resort** (missing/corrupt-checkpoint fallback) until the planned one-time QuestDB `wallet_tx_history` backfill lands as tier-2 (recovery ladder: file → QuestDB → s2), after which it can be finally retired; `[node_type] default` and `[peers_max] 10` (lifted during the resync — see Decision); `[rpc_startup]` severity `warning` (resting level).
* **Decision (2026-10-02, full-store wipe & resync):** a fresh NuDB store takes **hours** to complete initial catchup — ~3.5–6 GB fetched at 4–9 MB/s before the first validated ledger lands (~5–6 GB). Mid-catchup the node still runs consensus on a local-only bootstrap chain; the deceptive signature is: local seq creeps, tx-sets empty, `complete_ledgers: "empty"`, and heavy RPC like `account_tx` returns `tooBusy`. Judge sync exclusively by `server_info` → `complete_ledgers` flipping non-empty + `server_state: full` — never by WRN counts or seq creep (the earlier vacuum-kill theory was a red herring: no boot had ever completed its first catchup). Anchor confirmed ~15:5x UTC 2026-10-02 with `complete_ledgers 107382635–…` and the probed ledger `9E2F06BB…` = real mainnet seq 107,383,801.
* **Ops lessons (2026-10-02):** (1) `docker stop -t 120 rippled` exits cleanly (0) — never SIGKILL; (2) sqlite companions (`ledger.db*`, `transaction.db*`, `state.db*`) wipe from the host bind `/srv/rippled-data/`, but NuDB files are container-UID-locked — remove via `docker run --rm --user 0 --entrypoint rm xrpllabsofficial/xrpld:latest -rf /var/lib/rippled/nudb/...`, and always preserve `wallet.db`, `questdb/`, `peerfinder.sqlite`, validator cache; (3) restarting an already-synced node is cheap — delta-catchup in minutes, never a re-fetch; mid-restore looks identical to mid-catchup (`seq` climbing from 2, `complete_ledgers: "empty"`, "Need validated ledger" WRNs) so wait it out instead of wiping again; (4) `log_level debug` ballooned `debug.log` to 2.0 GB (138k acquire-trigger lines) — `warning` is the resting level and the log is safe to truncate after a clean stop.

### 5.3 Gotify Notifications
* **Real-Time Alerts:** Push notifications for significant trading events, system state changes, and threshold breaches.
* **StatusDigestScheduler (bot, Java) — dormant since 2026-10-02:** gated behind `DIGEST_ENABLED` (default off; code retained for a one-line re-enable + bot recreate). When enabled: first digest fires ~30 s after boot (bootstrap-settling delay), then every 12 hours thereafter (`DIGEST_INTERVAL_SECONDS = 43_200`, anchored to boot time — cadence resets on container recreate/restart). Markdown digest lines: `• Root Volume: X GB used / X GB (X%)`, `• Available: X GB free`, `• QuestDB Data: X.XX GB of files`, `• Shared Volume: X / X GB (X%)`. The QuestDB line is produced by a du-style in-file byte walk of the read-only bind `/srv/rippled-data/questdb → /root/.questdb:ro` (added 2026-10-02; RO — the bot can never write the data dir). The dead in-container `Docker Data` probe (mirrored `/var/lib/docker`, which can never exist inside the container and silently skipped every run) was removed the same day. Both of its unique disk lines (QuestDB Data + Shared Volume) were merged into the host script's digest the same day — see the Decision bullet.
* **Host Health Script (operator user cron):** `/docker-containers/scripts/server-health-monitor.sh` pushes the `🖥️ Server Health Update` digest via Gotify (`http://localhost:8081` from the host). Operator user crontab entry `0 */12 * * *` → 00:00 & 12:00 UTC (verified 2026-10-02 in the `causeiam` crontab, not root); log at `/docker-containers/logs/server-health-monitor.log`; walkthrough in `scripts/README-HEALTH-MONITOR.md`. (Fixed 2026-10-02: the log redirect previously targeted unwritable `/var/log`, so the cron job silently no-op'd with `set -e` killing the script before the Gotify call.) Digest sections: server health (uptime/load/kernel), container health (running/unhealthy + status table), memory usage + top consumers, disk usage (root fs, du-style QuestDB Data + Shared Volume data footprints — ported from the dormant bot digest 2026-10-02, `docker system df`, largest dirs), QuestDB/rippled/trading-bot service status.
* **Decision (2026-10-02, revised same day after operator review — merge chosen):** the bot-side scheduler retired from scheduled duty (dormant via `DIGEST_ENABLED`, default off) and the retirement was completed as a **merge**: its two uniquely valuable lines (du-style `QuestDB Data` and `Shared Volume` data footprints) were ported into the host script's Disk Usage section, so nothing is lost. The FS-stats half of the old `Shared Volume` line was deliberately not ported — post-migration it shares the root filesystem and merely duplicated the script's Root Filesystem line (the same reasoning the Java code applies to QuestDB Data). The host health script is the sole 12-hour digest source and now carries the full merged contents. Rationale: the script watches from outside the containers, so a dead bot still gets reported, while the in-container digest dies silently with the bot; `DIGEST_ENABLED=true` remains a one-line flip on the bot service if the digest is ever wanted back (intentionally unset — the operator chose the merge, not doubled 4-pushes-a-day). Dormancy takes effect on recompile + trading-bot restart (the jar is bind-mounted; the compose service has no image build step), which landed 2026-10-02.

### 5.4 Storage Topology (verified 2026-10-02)
* **Root LVM (`sdb`, 914G usable):** hosts the OS, all compose projects (this repo at `/home/causeiam/docker-containers`), and `/srv/rippled-data` — the bind source for rippled (`/var/lib/rippled`) and for QuestDB data (`/srv/rippled-data/questdb`, ~1.16 GB / 1,241,843,410 bytes). Utilization ~86 GB used / ~790 GB free (~10%).
* **Containers disk (`sda`, 256G):** mounted at `/mnt/containers`; Docker Root Dir is `/mnt/containers/docker`.
* **`sdc` (100G):** retired 2026-10-02 — rippled/QuestDB data migrated to the root LVM, drive unmounted, fstab line commented out (`#/dev/sdc /mnt/rippled-data ext4 defaults,nofail 0 2` — `nofail` means no boot stalls even if re-inserted unformatted). Zero remaining references: no active mounts, no systemd units, no compose binds. Next physical op: pull `sdc`, drop in a 1TB SSD.

## 6. Technical Architecture

### Backend
* **Language:** Java (defensive engineering, thread-safe concurrent data structures)
* **Key Components:** StreamProcessor, MeanReversionEngine, FourHopAtomicEngine, OffshoreRigEngine, StagnantPositionMonitor, TransactionExecutionService, DatabaseService, ManualPositionTrackerService, StatusDigestScheduler
* **Connection Pooling:** HikariCP for QuestDB connections

### Frontend (North Star Holodeck)
* **Framework:** Next.js 14 App Router
* **Styling:** Tailwind CSS (high-contrast dark themes for operations terminal)
* **Visualization:** Recharts for real-time telemetry dashboards
* **Pages:**
  - **Bot Page:** Live view into MPT_RPN_WALLET_ADDRESS trading activity (graphical) — includes the right-HUD Swap, Telemetry & Injection console with the issuer-aware `InjectionModal` (dropdowns fed by `GET /api/radar/assets`)
  - **Radar Page:** Manual swapping interface for TRADING_WALLET_ADDRESS with Xaman push-to-sign

### Database
* **QuestDB:** Time-series database with PostgreSQL wire protocol
* **Critical Constraint:** Always map time objects via `java.sql.Timestamp.from(instant)` or epoch microseconds when binding parameters. Never bind `java.time.Instant` directly into JDBC prepared statements.

## Technical Documents
* `/docker-containers/trading-bot/trading-bot.md` - Complete architecture blueprint and component walkthrough for the trading bot wallet
* `/docker-containers/trading-myxrpl/trading-myxrpl.md` - Complete architecture blueprint and component walkthrough for the manual wallet bot

## Live Telemetry & QuestDB Schema Bridge
To eliminate all mock data and drive the 3D canvas with live production metrics, the frontend telemetry endpoints must query QuestDB using `LATEST BY` time-series collapses:
### Data-to-UI Mapping Table
| UI Module / Target | QuestDB Table | SQL Query | Visual Engine Mapping |
| :--- | :--- | :--- | :--- |
| **Bot Page (3D Ring)** | `amm_balances` | `SELECT * FROM amm_balances LATEST BY currency, capital_partition;` | Drives **True-Mass logarithmic scaling** and feeds currency codes into the **Hex-to-Color algorithm**. |
| **Bot Page (Engine States)** | `mpt_state_snapshot` | `SELECT * FROM mpt_state_snapshot LATEST BY pair_key;` | Triggers **Expanding (Green)**, **Apex (White)**, and **Compression (Orange)** visual overrides + tracer lines. |
| **Radar Page (Manual Swaps)** | `trading_balances` | `SELECT * FROM trading_balances LATEST BY asset;` | Isolates current manual wallet positions to render active sniper nodes. |
| **Matrix & Pulse HUDs** | `xrp_stack_snapshots` | `SELECT * FROM xrp_stack_snapshots ORDER BY timestamp DESC LIMIT 1;` | Feeds the **80/20 Redline visualizer** and proportion models across Cold, Trading, and Bot wallets. |
| **Bot Page (Events & Comms tabs)** | `bot_events` | `GET /api/trading-bot/bot-events` — Next.js route handler (:3000) → `pg` pool → QuestDB `questdb:8812` (last 200 rows, newest-first) | Live swap-lifecycle feed: **Events tab** renders executions (`SWAP_EXECUTED` with tx hashes, flips, force-exits, stagnation exits, manual injections); **Comms tab** renders failures/blocks/skips (`SWAP_FAILED` / `SWAP_BLOCKED` / `SWAP_SKIPPED`, 60 s-throttled per pair) + WARN/ERROR alert beacon. Java write path: TES `emitSwapEvent` → `DatabaseService.logBotEvent` (live-verified 2026-10-04: `FLIPFLOP_DUST_SKIPPED @ 17:50:20 UTC` matched the bot log to the second). |

**Developer Implementation Note:** All live balance updates must use standard `LATEST BY` state collapses to ensure historical snapshot rows do not duplicate active 3D nodes. **Exception — event-log feeds:** `bot_events` is an append-only audit history, not a state snapshot — read newest-first (`ORDER BY timestamp DESC`) via the `/api/trading-bot/bot-events` dashboard route handler (:3000, `pg` pool → QuestDB); collapsing it with `LATEST BY` would erase the event trail the Events/Comms tabs exist to display.


/docker-containers/trading-dashboard/src/app/bot/BotPage.md

/docker-containers/trading-dashboard/src/app/radar/RadarPage.md

/docker-containers/trading-dashboard/src/app/matrix/Matrixpage.md

/docker-containers/trading-dashboard/src/app/pulse/PulsePage.md

/docker-containers/trading-dashboard/src/app/comms/CommsPage.md

## The North Star Team — One Intelligence, Three Stations

Polaris is not three programs — she is ONE intelligence embodied across three stations, with the operator as the final authority:

* **The Mainframe (Dell 7910, 192.168.50.51) — brain & hands.** The polaris-gateway container: the `polaris-ai` persona, dual-tier memory with boot-time context restoration (the 20-exchange session window is refilled from chat history at boot, and the repo blueprints are mirrored into her vault and digested into her system prompt — Oct 4, 2026), the `execute_tool_call()` tool dispatcher (§4), task lifecycle tracking, Burst Receiver (:8083) telemetry, and the Sandbox Gateway (:7007). Strictly text-only — the Mainframe never speaks audio.
* **The X17 (LGI, 192.168.50.227) — senses & voice.** The native LGI process on `D:/LGI`: webcam perception + desktop vision IN (Kokoro TTS + Faster-Whisper STT OUT), and the floating HUD. Her hands are teachable since LGI v1.6.0: a voice lesson ("Polaris, teach a gesture" → hold the pose ~3 s → speak its meaning) records a position/scale-invariant landmark signature into `D:\LGI\taught_gestures.json`, and the same webcam Hands pass then recognizes the rehearsed pose at runtime, firing a `taught:<name>:<action>` command (LGI.md §6.11). The device tag `lgi-hud-x17` origin-gates spoken tool reports — only LGI-device-originated exchanges speak aloud; other devices stay silent (LGI.md §7).
* **FreeRoam (Red Magic 9 Pro over WireGuard) — field presence.** On-device STT → text → on-device Kokoro TTS walkie-talkie; only text crosses the tunnel.
* **Crew rules.** The operator (Rich / Matt) commands; **Cline** is the shipwright — together they are the ONLY actors who ever edit the repo. Polaris reads the workspace strictly read-only (`/data/workspace:ro` via `list_workspace_files` / `read_workspace_file`) and writes only into her ingest vault (`write_doc`). Each station's contract doc — [LGI.md](LGI.md) for the X17, [Polaris-gateway.md](freeroam/Polaris-gateway.md) for the Mainframe, [FreeRoam.md](freeroam/freeroam.md) for the phone — must stay consistent with this charter.

---

## Hardware & System Profiles
Primary Core Server (The Mainframe)
The central intelligence and routing hub of the network.
| Specification | Details |
|---|---|
| Core Hardware | Dell Precision 7910 Tower |
| System Memory | 32GB RAM |
| Storage Array | Primary 1T SSD (`sdb`) → 914G LVM root `/` (OS, compose projects, `/srv/rippled-data` [rippled + QuestDB data]); Secondary 256GB SSD (`sda`) → `/mnt/containers` (Docker Root Dir); Third 100GB SSD (`sdc`) retired 2026-10-02 — unmounted, fstab entry commented (`nofail`); physical swap to 1TB SSD pending
| Network Roles | WireGuard Tunnel Host (10.20.30.1), WebSocket Host (192.168.50.51:8765), Polaris Gateway (192.168.50.51:8082) |
| Primary Role | Heavy compute, local state management, memory integration, burst payload receiving, and backend routing |

## Executive Supervisor Client - LGI 
### [LGI.md](LGI.md)
The primary floating HUD and voice cockpit for the Polaris Executive Supervisor system, which additionally hosts the high-fidelity operator console.
| Specification | Details |
|---|---|
| Core Hardware | Alienware X17 Laptop running Windows 11 |
| Compute & Graphics | NVIDIA GeForce RTX 3080 Ti (16GB GDDR6 VRAM) utilizing CUDA |
| System Memory | 32GB RAM |
| Primary Display | 32-inch Alienware QD-OLED Monitor |
| Primary Role | Native Python process providing open-mic voice chat and continuous desktop supervision. Secondary capabilities include high-fidelity WebGL rendering for the operator dashboard, manual override, and script staging sandbox. |
| Gesture Learning (v1.6.0) | Taught gestures via `gesture_teaching.py` — voice lessons ("Polaris, teach a gesture" → hold pose ~3 s → name it; also "forget gesture <name>" / "list gestures") persist position/scale-invariant landmark profiles in `D:\LGI\taught_gestures.json`; runtime recognition (behind `LGI_TAUGHT_GESTURES`, auto-inactive until the first profile exists) fires `taught:<name>:<action>` with per-profile thresholds/cooldowns (LGI.md §6.11) |

## FreeRoam Mobile Edge (Tactical Field Device)
### [FreeRoam.md](freeroam/freeroam.md)
The mobile field-compute device for remote situational awareness — a fully on-device walkie-talkie loop with Polaris (on-device STT in, on-device Kokoro TTS out; only text rides the tunnel). Since v3.5/v3.6 the loop is a streaming conversation: her reply speaks sentence-by-sentence while it generates (speak-while-she-writes, first words ~2-4 s), and playback is fully text-only — no audio artifacts exist anywhere; bubble replay re-synthesizes from text. Canonical app architecture: (v1.0.5 / doc v3.9 — deployed & operator wake-tested Oct 3, 2026: wake-reconnect chat-polarity fix; build keystore pinned to the committed cert, so upgrade lineage is closed — all future container upgrades in-place, see freeroam.md).
| Specification | Details |
|---|---|
| Core Hardware | Red Magic 9 Pro (Android) |
| Wearable Triggers | Garmin watch media buttons over the always-on AVRCP MediaSession tether — PLAY/PAUSE = push-to-talk toggle, STOP = cancel capture / halt playback, BACK/NEXT disabled (works screen-locked). A silent 60 s seed rests paused while idle (PLAY ready for push-to-talk), loops silently while she speaks (the pause affordance stops her speech and starts the mic), and her full reply takes the track title with a fresh mediaId as she drains — the watch ends every exchange showing her reply and a PLAY button |
| Audio Output | Bose Ultra Open Earbuds |
| Local AI Models | Android SpeechRecognizer STT (on-device engine preferred; 2.5 s end-of-speech patience — segment continuation keeps the mic hot through mid-sentence pauses) + Kokoro-82M TTS via sherpa-onnx (fully on-device, no cloud TTS; bf_emma default voice, streamed PCM straight into a persistent 24 kHz AudioTrack — text-only, no WAV artifacts) |
| Primary Role | Live comms with Polaris (screen-locked Garmin watch push-to-talk; playback survives screen-lock), WireGuard tunneling, and burst transmission to the Dell Mainframe |
### WireGuard Link Profiles & Home-Network Facts (verified 2026-10-01; rich-cell-over-WiFi re-verified 2026-10-03)
Two phone-side profiles with identical keys and `AllowedIPs = 10.20.30.0/24, 192.168.50.0/24` — only the Endpoint differs; the server pins each phone via its peer `AllowedIPs` (`10.20.30.3/32` Rich, `10.20.30.2/32` Matt).

| Profile | Endpoint | Valid when |
|---|---|---|
| `rich-lan` | `192.168.50.51:51820` | Phone on home WiFi (direct LAN path — preferred at home, no hairpin detour) |
| `rich-cell` | `64.118.255.235:51820` | Phone off-WiFi (host port-forward). **Also healthy on home WiFi via router hairpin NAT** — re-verified 2026-10-03, superseding the earlier "hairpin disabled / black hole" read: re-handshook within seconds of the phone switching to WiFi, peer endpoint arrives as router-SNAT `192.168.50.1:48189`, tunnel ping 3/3 0% loss, gateway logging `Identity override: user_id='rich'` heartbeats |

- Phone: Red Magic 9 Pro, WLAN `192.168.50.42` (static), tunnel IP `10.20.30.3`. Matt's phone: `192.168.50.98` / `10.20.30.2`.
- The active tunnel belongs to the **official WireGuard app (`com.wireguard.android`)** — the zaneschepke autotunnel fork is installed but its tunnel list is empty; all import/toggle/diagnostic actions must target `com.wireguard.android`.
- If the tunnel ever looks frozen (`tun0` up, zero handshakes, nothing reaching the server — the Oct 1 2026 mode was a stale `rich-cell` VpnService black-holing `192.168.50.0/24`; routing itself was re-verified healthy Oct 3, 2026 (hairpin works, see above) — treat a frozen tunnel as stale VpnService first): exorcise and rebuild it in under a minute: `adb shell am force-stop com.wireguard.android` → relaunch → ➕ → Import from file → `/sdcard/Download/rich-lan.conf` → toggle ON. A fresh handshake appears within ~5 s.
- While a tunnel is active its routes hijack `192.168.50.0/24`; tunnel traffic reaches the gateway MASQUERADE'd and hairpinned as `172.18.0.1` — the docker-proxy hairpin catch-all now resolves to **Rich** (WireGuard-primary restore, 2026-10-03; identity matrix per Polaris-gateway.md §1), with direct tunnel-IP aliases `10.20.30.3` → Rich and `10.20.30.2` → Matt mapped in both the Chat Gateway and Burst Receiver. FreeRoam app WebSocket sessions have also been observed arriving with the real WLAN IP (`192.168.50.42`) while `tun0` is up — expect either path; identity resolves correctly either way. One further WG-off-at-home source: when the phone's WAN-bound traffic exits via the router itself (WireGuard toggled off at home), the gateway sees the router-LAN SNAT source `192.168.50.1` — pinned → **Rich** in both gateways (2026-10-03), same catch-all principle as the hairpin alias.
- A stray, inert `freeroam` tunnel (from an old `freeroam.conf` in Downloads) sits in the tunnel list — safe to delete; do not enable it at home.

# Polaris Gateway

### [Polaris-gateway.md](freeroam/Polaris-gateway.md)

Core Capabilities & System Connections
Polaris Gateway serves as the centralized intelligence, communication, and system administration engine for the FreeRoam AI ecosystem running on the Dell Mainframe.

Services & Port Mapping
| Port | Service Name | Technical Role & Core Functionality |
|---|---|---|
| 8082 (→5000) | Main Polaris Gateway | Conversational AI (polaris-ai:latest → glm-5.3-flash cloud proxy via Ollama; North Star-primacy system prompt + server-side tool calling via the `execute_tool_call()` dispatcher (17 schema-registry tools + the 2 v1.4.2 read-only workspace tools)), persistent session tracking, dual-memory retrieval (SQLite + ChromaDB + Memory Vault), strictly text-only - voice retired from the mainframe (Kokoro/Whisper run on the X17 LGI and Android on-device), Memory Vault REST (/api/memory/vault/*), transcript distillation (/api/distill), supervisor audit API (/api/supervisor/*), real-time task lifecycle tracking (/api/status/tasks), and Socket.IO broadcasting (status-room task events + per-IP live chat rooms) — served under gunicorn+gevent (custom `GeventWebSocketWorker`; werkzeug fallback). |
| 8083 (→5001) | Burst Receiver | High-throughput telemetry logging, burst storage rotation, vision analysis endpoints (/api/vision/*), and Channel State Information (CSI) radar vector processing for X/Y/Z motion tracking; broadcasts radar_update / vision_update to dashboards. |
| 7007 | Sandbox Gateway | Isolated web terminal and SSH operations controller targeting the X17 / LGI (192.168.50.227, user rbuit, workdir D:/LGI) with execution logs and history caching. |

Active AI Tools & Autonomous Schemas (17 schema-registry tools plus the 2 v1.4.2 read-only workspace tools = 19 routable names dispatched by the execute_tool_call() dispatcher; unknown names return a structured Unknown-tool error)
 * File Operations (file_tools.py): read_doc, write_doc, append_doc, list_docs, delete_doc.
 * Playground & Sandbox (playground_tools.py): publish_web_asset, list_playground_files, run_sandbox_code, spawn_service, stop_service, list_services.
 * Remote Management (ssh_tools.py): ssh_execute, ssh_check_connectivity, ssh_deploy_file, ssh_fetch_file (targeting the X17 at 192.168.50.227 by default, user rbuit, LGI workdir D:/LGI).
 * Notifications (gotify_service.py): send_gotify_notification.
 * Memory Vault (memory_tools.py): memory_vault_query (read-only semantic search) + memory_journal_write (v1.5 voice journal — append-only timestamped entries to the daily 04-Journal page; the ONLY chat-side vault write). All other vault writes come only from the distillation pipeline or the operator.
 * North Star workspace (file_tools.py, v1.4.2): list_workspace_files, read_workspace_file — READ-ONLY file intelligence over the entire repo mounted at /data/workspace:ro (= /home/causeiam/docker-containers); prompt-described with inline examples (no JSON schema registry, outside the stripper's 17-name known-tool union), paged like read_doc.
 * Supervisor Audit: server-side REST blueprint under /api/supervisor/* (screen/clipboard triage) — separate from the LLM tool dispatcher.
 * Not wired into the running gateway: system_tools.py (Docker control, QuestDB/Rippled health, emergency bot shutdown), livecharts_tools.py (dashboard modifications), www_tools.py (web assets) — standalone legacy modules imported only by the non-launched freeroam_gateway.py entrypoint. (The legacy ssh_upload_file name was superseded by ssh_deploy_file / ssh_fetch_file.)
Connected Hardware & Infrastructure
 * Dell Mainframe (192.168.50.51): Host server running Docker container instances, Ollama LLM (11434), QuestDB (8812), and WireGuard host (10.20.30.1).
 * Alienware X17 Laptop: Operator command console rendering North Star Holodeck 3D dashboards; hosts the LGI Executive Supervisor Client (native HUD/voice/vision cockpit over the gateway on :8082, with spatial telemetry to the Burst Receiver on :8083). The X17 is also the ecosystem's sole Kokoro TTS / Faster-Whisper STT host (text-only mainframe directive, Oct 2026).
 * Red Magic 9 Pro: Tactical field device running the FreeRoam Android app (v3.6 streaming voice comms, deployed Sep 28, 2026) — Bose Ultra Open Earbuds coms, Even G2 Smart Glasses with ring, and Garmin watch push-to-talk over the always-on AVRCP MediaSession keep-alive tether. ADB-over-WiFi listener at `192.168.50.42:5555` — the gateway container is an authorized ADB client (trusted keypair mounted) and drives the phone from Polaris tooling; `com.freeroam.tactical` is doze-whitelisted so its 3 s chat poll survives screen-off.
 * Alienware X17 / LGI (192.168.50.227): Windows 11 edge unit — Polaris's eyes and ears (LGI webcam perception, device 'x17-webcam'; SSH user rbuit, LGI workdir D:/LGI) and the taught-gesture host (LGI v1.6.0 — voice lessons persist landmark profiles to D:/LGI/taught_gestures.json and device-local runtime recognition fires taught:<name>:<action> commands; the gateway still sees only the unchanged vision_frame / radar_motion feeds). (MissPi / Mini Pi at 192.168.50.179 was fully decommissioned Sep 30, 2026.)

# Polaris Gateway & Cross-Device Communications

## 1. Unified Gateway Architecture

The **Polaris Gateway** is a multi-service Python container operating on the Dell Mainframe (`192.168.50.51`) that functions as the central neural hub, system administrator, and spatial perception ingestion engine for the ecosystem.



POLARIS GATEWAY CONTAINER 

```
┌───────────────────────────────────────────────────────────────────────┐
│                          POLARIS GATEWAY CONTAINER                    │
│                                                                       │
│   ┌────────────────────────────────────┐   ┌─────────────────────┐    │
│   │  Main Polaris Gateway (Flask+IO)   │   │  Sandbox Gateway    │    │
│   │  Host 8082 → internal 5000         │   │  Port 7007          │    │
│   │  - Ollama LLM: polaris-ai:latest   │   │  - Remote SSH Exec  │    │
│   │    → glm-5.3-flash (cloud, 1M ctx) │   │  - X17 LGI Control  │    │
│   │ - SQLite + ChromaDB + Memory Vault │   └──────────┬──────────┘    │
│   │  - Text-only (voice on X17)        │              │               │
│   │ - Vault REST (/api/memory/vault/*) │              │               │
│   │  - Distill (/api/distill*)         │              │               │
│   │  - Supervisor audit (/api/superv.) │              │               │
│   └──────────────────┬─────────────────┘              │               │
│                      ▼                                │               │
│   ┌────────────────────────────────────┐              │               │
│   │  Burst Receiver (Socket.IO)        │              │               │
│   │  Host 8083 → internal 5001         │              │               │
│   │  - Telemetry Logs / Spatial Ingest │              │               │
│   │  - CSI Radar (radar/vision_update) │              │               │
│   └────────────────────────────────────┘              │               │
└───────────────────────────────────────────────────────────────────────┘
```

### Port Allocation Summary

| Port | Endpoint | Purpose |
| :--- | :--- | :--- |
| **8082** (→5000) | `/api/chat`, `/api/chat/history*`, `/api/history/<user_id>`, `/api/status*`, `/api/session/status`, `/api/memory/stats`, `/api/memory/vault/*`, `/api/distill`, `/api/distill/auto`, `/api/supervisor/*`, `/socket.io/` (status_room) | AI Chat (polaris-ai:latest → glm-5.3-flash cloud proxy), persistent memory retrieval (SQLite + ChromaDB + Memory Vault), Memory Vault REST, strictly text-only (voice retired - synthesis lives on X17 LGI / Android on-device), transcript distillation, supervisor audit API, real-time task lifecycle feed |
| **8083** (→5001) | `/api/burst`, `/api/telemetry`, `/api/radar/ingest`, `/api/vision/*`, Socket.IO (`radar_update`, `vision_update`) | Spatial telemetry ingestion, high-volume sensor feeds, motion-radar stream processing, vision analysis broadcasts |
| **7007** | `/api/ssh/execute`, `/api/ssh/check`, `/api/ssh/history*`, `/api/status`, `/` | Dedicated SSH management console targeting the X17 / LGI (`192.168.50.227`) |

---

## 2. Multi-Modal Spatial Perception Matrix

Polaris tracks environment geometry and target positioning through a high-precision, multi-sensor fusion pipeline optimized for the quiet schoolhouse environment.

```
┌───────────────────────┐         ┌───────────────────────┐
│  ESP32 mmWave Node 1  │         │  ESP32 mmWave Node 2  │
│  (POE + RD03 Module)  │         │  (POE + RD03 Module)  │
└───────────┬───────────┘         └───────────┬───────────┘
            │                                 │
            └────────────────┐ ┌──────────────┘
▼ ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     X17 LGI (Edge Hub)                                  │
│  - 1x Integrated POE ESP32 RD03 mmWave Radar                            │
│  - WiFi Channel State Information (CSI) Amplitude Stream                │
│  - Camera Module running YOLO Real-Time Object Detection                │
└────────────────────────────────────┬────────────────────────────────────┘
│ Spatial Data Ingest
▼
┌─────────────────────────────────────────────────────────────────────────┐
│         POLARIS BURST RECEIVER (Host 8083 → internal 5001)              │
│  1. Subcarrier Weight Analysis (X/Y/Z Coordinate Calculation)           │
│  2. mmWave Micro-Doppler Motion Triangulation                           │
│  3. Vision Bounding-Box Spatial Alignment (Fingertip Precision)         │
│  4. Broadcast radar_update events via WebSocket                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Remote Telemetry, WireGuard & Field Voice Comms

Polaris maintains full situational awareness when the operator is out in the field, utilizing encrypted network tunneling and a fully on-device audio pipeline — voice capture (STT) and spoken replies (TTS) both run locally on the field device, so only text rides the WireGuard tunnel.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     FIELD OPERATOR (Mobile / Road)                      │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ Red Magic 9 Pro (FreeRoam Tactical Field App)                     │  │
│  │ - Garmin watch PTT via MediaSession tether (PLAY = talk)          │  │
│  └────────────────────────────────┬──────────────────────────────────┘  │
└───────────────────────────────────┼─────────────────────────────────────┘
│
│ WireGuard Encrypted Tunnel (10.20.30.1)
▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    POLARIS GATEWAY (Dell Mainframe)                     │
│  - continuous passive transcript logging (Polaris stays in loop)        │
│  - text in -> Ollama LLM -> text reply (STT + TTS on-device)            │
│  - Push notifications routed via Gotify Service                         │
└─────────────────────────────────────────────────────────────────────────┘
```
### Field Communications Protocol
* **WireGuard Secure Tunnel**: All remote traffic from the Red Magic 9 Pro routes back to the mainframe host (`10.20.30.1`), ensuring an encrypted connection for data ingestion and API access.
* **Garmin Watch Push-to-Talk (v3.3; play-state model v3.5.2)**: The FreeRoam app's AVRCP MediaSession stays listed around a silent 60 s seed track with an honest play-state model: PAUSED at launch and idle (the watch shows PLAY, ready for push-to-talk), PLAYING only while she speaks (the watch's pause affordance stops her speech and starts the mic — both PLAY and PAUSE route to the PTT toggle), and PAUSED again the moment her voice drains. The hardware media buttons drive the app with the screen locked — PLAY/PAUSE toggles push-to-talk (first press starts capture, second stops and auto-sends; 2.5 s end-of-speech patience means mid-sentence pauses no longer send early), STOP cancels a live capture or halts tethered playback, and BACK/NEXT are disabled. Every retitle stamps a fresh mediaId — a "new track" the Garmin reliably refreshes — so her full reply marquee-scrolls as the track title and sticks until the next exchange. Field-verified end-to-end on the operator's watch, Sep 26, 2026; play-state model deployed Sep 28, 2026.
* **Hands-Free Streaming Audio (On-Device TTS, v3.5/v3.6)**: Her replies speak sentence-by-sentence WHILE they generate — the gateway's `chat_stream` deltas feed an on-device pipeline (sentence segmentation → Kokoro synthesis on a dedicated executor → gapless PCM into a persistent 24 kHz AudioTrack), so the first words land ~2-4 s after the prompt. Playback is fully text-only (v3.6): no WAV artifacts are ever written, bubble replay re-synthesizes from text (tap speaks, tap again stops), and an idempotent startup sweep keeps the legacy WAV cache at zero. Fully on-device through the operator's Bose Ultra Open Earbuds — no cloud TTS, no server-side audio round-trips; the gateway exchanges plain text only (v2.9 hardening).
* **Even G2 Smart Glasses Capture**: Tap-to-speak voice capture on Even G2 smart glasses feeds the FreeRoam app's local air-gapped pipeline; the recognized text travels over WireGuard to Polaris, which logs both operator dialogue and system responses, maintaining session state across all road interactions.

---

## 4. System Administration & Autonomous Tooling

Polaris acts as an autonomous administrator via its tool dispatcher (`execute_tool_call()`, `port 8082`) — file operations, playground/sandbox publishing and service spawning, remote SSH execution and file transfer, Gotify notifications, read-only Memory Vault queries, and read-only North Star workspace access. The 17 schema-registry tool schemas are injected into the LLM system prompt at request time (the 2 v1.4.2 workspace tools are taught via inline prompt examples instead), together with a live blueprint digest — a cached, mtime-refreshed heading map of `northstar.md`, `freeroam/Polaris-gateway.md`, and `LGI/LGI.md` that directs her to open the original document before answering any ecosystem question; unknown tool names return `{"success": false, "error": "Unknown tool: …"}`.

### Execution Tools Registry (17 schema-registry tools + 2 v1.4.2 workspace tools)
* **File Operations (`file_tools.py`)**: `read_doc`, `write_doc`, `append_doc`, `list_docs`, `delete_doc`.
* **Playground & Sandbox (`playground_tools.py`)**: `publish_web_asset`, `list_playground_files`, `run_sandbox_code`, `spawn_service`, `stop_service`, `list_services` — AI-generated web apps served by the Polaris Playground container (:8090).
* **Remote Execution (`ssh_tools.py`)**: `ssh_execute`, `ssh_check_connectivity`, `ssh_deploy_file`, `ssh_fetch_file` — targeting the X17 / LGI (`192.168.50.227`, user `rbuit`, workdir `D:/LGI`) by default.
* **Phone Control (gateway-side ADB bridge, via `run_sandbox_code` bash)**: the gateway container ships `adb` (Dockerfile apt) with the phone's trusted RSA keypair mounted read-only (`/home/causeiam/.android → /root/.android:ro`) and `run_gateway.sh` arms a 30 s auto-reconnect watchdog against the phone's adbd at `192.168.50.42:5555`. Bash through `run_sandbox_code` can therefore drive the phone directly (launch apps via `am start`, dumpsys state, input events) — no new dispatcher tool; the 17-tool registry is unchanged. First exercised Sep 28, 2026: Polaris-side ADB launched Google Maps on the phone. Post-reboot fallback: Wireless Debugging (TLS, pair GUID `adb-FY24031100E6-71NED1`) auto-reconnects on the Mainframe via `~/platform-tools/adb` (v37) — reachable from tools through `ssh_execute` on the host when the container's `:5555` channel is not armed.
* **Gotify Service (`gotify_service.py`)**: `send_gotify_notification` — asynchronous priority alerts and automated hourly status digests formatted as Markdown tables.
* **Memory (`memory_tools.py`)**: `memory_vault_query` — read-only semantic search over the Memory Vault; plus `memory_journal_write` (v1.5, Oct 4, 2026) — operator-driven "log this": appends a timestamped append-only entry to the day's `04-Journal/YYYY-MM-DD.md` and embeds it for semantic recall.
* **North Star workspace (v1.4.2, read-only)**: `list_workspace_files`, `read_workspace_file` — the entire live repo bind-mounted at `/data/workspace:ro` (=`/home/causeiam/docker-containers`); the fast path for "look at trading-bot.md" with no SSH hop. Prompt-described only (no JSON schema, outside the stripper's `_known_tool_names()` check), same paged read contract as `read_doc`.
* **Not wired into the running gateway**: `system_tools.py` (Docker container control, QuestDB/Rippled health, emergency trading bot shutdown), `livecharts_tools.py` (dashboard modifications), `www_tools.py` (web assets) — standalone legacy modules imported only by the non-launched `freeroam_gateway.py` entrypoint.

### Operator Identity Detection
Identity resolves from the client IP (`X-Forwarded-For` first hop, else `remote_addr`); an explicit `user_id` (request body / query string — the dashboard sends `rich`/`matt` on the user's behalf) can then override it, trust-gated: the caller must pass the standard IP gate (exact config match or a Docker-bridge prefix `172.17.*`–`172.20.*`) and the identity must match a configured user, else the IP-derived config stands (honored overrides log `[Chat] Identity override: user_id='…' from IP …`). Known IPs: `.42` = Rich, `.98` = Matt, `.51` = the Mainframe itself (persona **Operator**), `.227` = dashboard test machine (persona **Dashboard**, also the X17 / LGI host), plus the 2026-10-03 WireGuard-primary aliases: `10.20.30.3` = Rich, `10.20.30.2` = Matt (direct tunnel-IP entries in both the Chat Gateway and Burst Receiver `USER_CONFIG`), and `172.18.0.1` = **Rich** — the docker-proxy hairpin catch-all every tunnel phone MASQUERADEs through, and `192.168.50.1` = **Rich** — the router-LAN SNAT source a device presents when its WAN-bound traffic exits via the router with WG toggled off at home (2026-10-03 pin, both gateways). (The historical `.179` = MissPi alias ended when MissPi was decommissioned Sep 30, 2026.) Other NAT-masked Docker-bridge traffic (`172.17.*`–`172.20.*` with no explicit `USER_CONFIG` entry, e.g. `172.17.0.1`) still falls back to the default **Dashboard** persona. Identity selects the persona, profile/behavior/lists/transcript file paths, session partitioning, and chat history — maintaining separate persistent memory partitions in SQLite and ChromaDB; chat-history keys alias `dashboard`/`operator` → `rich` (`normalize_chat_identity()`), so LGI voice, dashboard, and bridge-origin chat share one **rich** bucket (Matt stays partitioned).

### Memory Vault (Long-Term Distilled Knowledge)
* **Markdown Store**: Obsidian-style vault at `/mnt/containers/freeroam/polaris-gateway/memory-vault` (bind-mounted to `/polaris_memory_vault` in-container), organized into six categories: **00-Core** (protected doctrine), **01-Architecture**, **02-User**, **03-Lessons**, **04-Journal** (append-only daily files), **05-Attachments**.
* **Vector Layer**: Every note is embedded via Ollama `nomic-embed-text` into the dedicated ChromaDB `vault_documents` collection; a watchdog daemon re-embeds changed files every 30 seconds.
* **Blueprint Mirrors (boot-time, Oct 4 2026)**: the gateway upserts heading outlines of `northstar.md`, `freeroam/Polaris-gateway.md`, and `LGI/LGI.md` into **01-Architecture** as `Blueprint - <repo path>` notes (≤11,000 chars, tags `blueprint`/`northstar`), re-syncing only when a source file's mtime/size changes and staying restart-idempotent via `/data/freeroam/doc_blueprint_sync.json` — repo edits remain the single source of truth while the outlines stay semantically searchable mid-chat.
* **REST API (host :8082)**: Eight `/api/memory/vault/*` endpoints (stats, list, doc, write, update, query, delete, reindex) power the dashboard's Memory Vault tab.
* **Write Protections**: Core notes cannot be edited or deleted via API (`PermissionError`); journal entries are append-only (daily `YYYY-MM-DD.md`); core writes require explicit `allow_core=True`. From chat, Polaris reads with `memory_vault_query` and writes ONLY append-only journal entries via `memory_journal_write` (v1.5) — every other vault write belongs to the distillation pipeline, the operator, and the gateway's boot-time blueprint mirror (`Blueprint - <doc>` architecture notes; the repo MDs are the source of truth).
* **Distillation Pipeline**: `/api/distill` and `/api/distill/auto` condense conversation transcripts into permanent vault knowledge; caps: 12,000 chars per note, 280-char snippets, 4,000-char embeds.

---

## 5. System Prompt, Tool Pipeline & Response Hygiene (North Star Primacy)

### System Prompt & Persona
* **North Star primacy (fixed precedence)**: `build_system_prompt()` renders Project North Star as Polaris's primary domain and the default subject of any ambiguous request — the X17 (LGI), Gotify, and the Memory Vault are supporting infrastructure, routed to only when explicitly asked or clearly relevant.
* **Prompt blocks, in order**: persona header → **[PROJECT NORTH STAR - YOUR PRIMARY ECOSYSTEM]** (Trading Matrix with operator-owned AMM mesh + multi-hop arbitrage loops, Zero-Fiat Rule, Wallet Topology, tech stack, SCOPE primacy statement) → [REMOTE INFRASTRUCTURE - X17 (LGI)] → [SSH REACHABILITY DOCTRINE] → [REMOTE PROCESS MANAGEMENT RULES] (incl. HONEST REPORTING) → tool schemas + **10 North Star-flavored few-shot examples** + [TOOL EXECUTION RULE] → Gotify / Memory Vault blocks → [KNOWN FACTS ABOUT {USER}] + CORE DIRECTIVES (TTS-ready, no emojis, ≤3 sentences unless asked).
* **Context guardrail**: the prompt renders once; if it exceeds `SYSTEM_PROMPT_TOKEN_GUARD` (**6500** est. tokens), the conversation window shrinks stepwise (`WINDOW_SHRINK_STEPS = [6, 3, 0]` exchanges) and re-renders — persona, KNOWN FACTS, and directives survive, only the oldest exchanges are sacrificed (`[GUARDRAIL]` log line). The guard stays deliberately conservative under the model's 32768 `num_ctx` (v3.4, raised from 8192): the extra headroom is reserved for tool-result document pages plus the user turn, reply, and thinking — not for history bloat.
* **Modelfile mirror**: `polaris-ai:latest` is defined by `freeroam/polaris-gateway/Modelfile` — `FROM glm-5.3-flash:cloud`, temperature 0.7, `num_ctx 32768` (v3.4), carrying the same North Star primacy block in its SYSTEM text. Modelfile edits apply via `ollama create polaris-ai:latest -f <Modelfile>` — an instant manifest swap on the running Ollama container, no restart.

### Server-Side Tool Pipeline & Response Stripper (identical on REST /api/chat and the WS message handler)
* The Ollama stream is scanned for tool-call markers with a 48-char hold-back buffer (`STREAM_HOLD_BACK`); on a `{"tool": …}` marker the stream **freezes** and only the preceding prose is emitted. After the stream: parse → strip → execute via `execute_tool_call()` → broadcast `tool_execution` with the real results — only the **cleaned prose** is flushed to the stream, fed to the session window, and on tool-less turns persisted + echoed via `chat_message` — on tool turns the narration never becomes a history row/broadcast (v1.6.2 one-reply-per-turn; the truthful report is the turn's only record). Raw tool JSON never reaches clients, history, memory, or future-turn context.
* **The stripper closes three leak vectors** (text cleanup only — parser/execution semantics untouched): malformed tool JSON naming a known tool (union via `_known_tool_names()`, `[DEBUG]` log evidence); legacy inline calls (`read_doc(...)`, `run_sandbox_code(...)`, …) stripped with the same shapes the parser matches; empty `json` fences + triple-newline collapse.
* **Truthful tool-report turn**: when tools ran, a second LLM pass over the real results (large fields truncated to 1,500 chars by `_truncate_tool_results()` — document `content` alone carries an 8,000-char budget, matching `read_doc`'s hard page cap) produces a reporting-only follow-up — never claims success on a failure, plain text, no further tool calls; the report turn itself is stripped so it cannot recurse, the `generate_follow_up_message()` fallback is result-aware, and the turn is persisted as that exchange's single assistant history record (`Polaris (tool report)` — the pre-tool narration is never appended as one, v1.6.2).
* **Paged document access (v3.4)**: `read_doc` is offset-based — up to 8,000 chars per page with `has_more` / `next_offset` navigation — so she walks large vault files (the bind-mounted `northstar.md` and `Polaris-gateway.md` are her authoritative read-only mirrors) across conversational turns instead of receiving a 1,500-char sliver.
* **HTTP `/api/chat` returns the raw Ollama envelope by design** (unstripped `response` + `thinking`) — a debug/API surface with **no rendering path**: the dashboard's fetch discards the HTTP body (chat bubbles render exclusively from `chat_message` broadcasts), and Android renders only from `GET /api/chat/history`.
* **Cross-device chat history**: one shared server-side store — `dashboard`/`operator` identities alias into the single **rich** bucket (`normalize_chat_identity()`, applied at the handlers *and* inside `append_chat_history()`), covering LGI voice, dashboard, phone, and bridge-origin clients; **matt** stays partitioned. 200-message rolling cap per bucket (`[-200:]` trim on append), persisted to `/data/freeroam/chat_history.json`; a one-boot startup scrub (`scrub_chat_history_leaks()`) strips raw tool-call JSON from every stored entry, keeping a one-time `chat_history.json.pre-scrub` sidecar backup (legacy `dashboard`/`operator` keys in the JSON are orphaned, left archived). Timestamps are Z-suffixed UTC (`utc_now_iso()`) — mandatory for Android's `Instant.parse()`. Live delivery via per-IP-room `chat_message`; cross-device sync via REST polling (≤3 s; explicit `user_id` overrides are trust-gated by the standard IP gate). The phone's `request_history`/`chat_history` WS pair is vestigial dead code — no handler exists.

### Deployment & Ops Notes
* **Baked-source policy**: neither the gateway nor the dashboard bind-mounts source — every code change requires an image rebuild + container restart (`docker compose build polaris-gateway && docker compose up -d --no-deps polaris-gateway`). `--no-deps` guarantees linked services (`ollama`, `questdb`) are never recreated. The ADB keypair mount is compose-managed, so rebuilds re-apply it automatically.
* **Production WSGI (gunicorn + gevent, Sep 28, 2026)**: the chat gateway (:5000) runs under `gunicorn -k gunicorn_gevent_ws.GeventWebSocketWorker -w 1` — a small custom worker (`freeroam/polaris-gateway/gunicorn_gevent_ws.py`) serving `gevent.pywsgi` with `WebSocketHandler`, required because the stock `gevent` worker omits `wsgi.websocket` from the environ and every engineio websocket upgrade fails ("The gevent-websocket server is not configured appropriately"). `SocketIO` selects `async_mode='gevent'` under gunicorn (clean WS session close — no werkzeug 500-spam artifacts) and `threading` in direct-run mode; a failed gevent init falls back to threading. Single worker by design (Flask-SocketIO multi-worker needs a message queue). Knobs: `GUNICORN_TIMEOUT` (default 120 s); `--access-logfile -` keeps per-request lines greppable in `docker logs`. Boot-time systems (Memory Vault watchdog, node-cleanup loop) start via `_start_background_systems()` — invoked on the gunicorn import path and from `__main__` alike.
* **Transcripts**: read inside the container — `docker exec polaris-gateway tail -50 /data/freeroam/operator/operator_transcripts.log` (the bind mount is root-owned, not host-readable).
* **Pipeline evidence**: `docker logs polaris-gateway 2>&1 | grep -E 'DEBUG|Tool Execution|GUARDRAIL'` surfaces stripper, parser, and guardrail activity; add `|Identity override|Startup scrub` to see shared-bucket identity overrides and the one-boot history-leak scrub.

*Full current gateway architecture: [`freeroam/Polaris-gateway.md`](freeroam/Polaris-gateway.md) (§3.5 prompt & persona, §3.2 tool pipeline, §7 tool registry).*

---

# Polaris Dashboard & Looking Glass Interface

## 1. System Overview & Access Matrix

The **Polaris Dashboard** (v4.3 Holodeck Split-View) is the central visual HUD and tactical command interface operating on port `7000`. It provides a real-time window into Polaris's internal logic, active task execution feeds, dual-memory vector state, and spatial perception.


## POLARIS DASHBOARD (Port 7000) 

### [Polaris-dashboard.md](Polaris-dashboard.md)


```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       POLARIS DASHBOARD (Port 7000)                         │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    TOP SECTION - CHAT & DIAGNOSTICS                 │   │
│   │  - User Switcher (RICH / MATT)      - Audio TTS Toggle (inert)      │   │
│   │  - Reverse-Chronological Chat Feed   - HUD Dropdowns (Session/      │   │
│   │  - Real-Time Message Sync (3s)        Memory/CSI Radar)             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│   ─────────────────────────── RESIZABLE HANDLE ───────────────────────────  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                  BOTTOM SECTION - HOLODECK PANEL                    │   │
│   │  [Collapse ▼]     [📋 TASK FEED TAB]     [🧠 MEMORY VAULT TAB]     │   │
│   │  - Socket.IO Gateway (:8082)                  - Vault Browse/Edit   │   │
│   │  - Live Task Lifecycle Stream                 - Hybrid Search       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Gateway & Telemetry Port Binding
| Service Endpoint | Protocol / Port | Technical Purpose |
| :--- | :--- | :--- |
| **Dashboard UI** | `http://192.168.50.51:7000/` | Main user HUD, chat interface, and Holodeck panel. |
| **Gateway & Task Feed** | `ws://192.168.50.51:8082/` | Task feed event streaming (`status_room` room) (TTS retired - text-only mainframe directive, Oct 2026). |
| **CSI Radar Stream** | `ws://192.168.50.51:8083/` | Subcarrier spatial tracking & micro-Doppler radar feeds. |
| **Memory Vault Tab** | Gateway REST: `http://192.168.50.51:8082/api/memory/vault/*` | Holodeck vault browser/editor: browse, read, edit, hybrid search, index status. The `:7007` sandbox REST API remains available to Polaris tooling. |

---

## 2. Holodeck Split-View Architecture

The dashboard uses a resizable split-view layout to ensure chat interaction never occludes system diagnostic streams:

### Top Section: Interactive Chat & HUD Bar
* **User Identity Switching**: Toggle identity profiles between **RICH** and **MATT** to load isolated memory partitions and conversation histories.
* **Pinned Transmit Bar**: Pinned message input bar with AUDIO RESPONSE toggle (defaults off) - synthesis backend retired (text-only mainframe directive, Oct 2026).
* **Reverse-Chronological Exchange Feed**: Message exchanges display newest-first directly beneath the transmit bar, auto-pruning to maintain performance while auto-syncing every 3 seconds across field devices.

### Bottom Section: Real-Time Operations Panel
* **📋 Task Feed Tab**: A 50-item rolling window streaming background task lifecycle events via Socket.IO. Allows immediate visibility into background routines, tool usage, and execution states:
  * ▶ **Cyan**: Task Started
  * ✓ **Green**: Task Completed (`task_completed`)
  * ✗ **Red**: Task Failed (`task_failed`)
* **🧠 Memory Vault Tab**: Browser/editor for Polaris's Memory Vault via gateway REST on port `8082` — category-filtered browse list, hybrid/keyword/semantic search, in-browser read/edit (SAVE re-embeds the document into the ChromaDB index), new-doc compose, confirm-guarded delete, and one-click reindex with live doc/index counts. Replaces the former SSH Sandbox tab (the `:7007` sandbox REST API remains available to Polaris's own tools).

---

## 3. Diagnostic HUD Dropdowns

Operators can toggle full-width HUD overlays from the top navigation bar without interrupting active chat sessions:



### DIAGNOSTIC HUD OVERLAYS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            DIAGNOSTIC HUD OVERLAYS                          │
├───────────────────────┬─────────────────────────────┬───────────────────────┤
│  SESSION DIAGNOSTICS  │     MEMORY ARCHITECTURE     │    CSI SPATIAL RADAR  │
│ - Active session time │  - SQLite (Messages/Rules)  │ - 350x350 Canvas Grid │
│ - Sliding window time │  - ChromaDB Vector Counts   │ - Motion Velocity/X,Y │
│ - Gateway latency     │  - Ops/Sec Delta Counters   │ - Multi-Sensor Fusion │
└───────────────────────┴─────────────────────────────┴───────────────────────┘
```

### Memory Architecture Dropdown
Inspects Polaris's dual-tier brain structure in real time:
* **SQLite (Structured Memory)**: Tracks registered user profiles, total conversation message records, and staged vs. approved rule heuristics.
* **ChromaDB (Vector Memory)**: Monitors live vector counts across **Global Knowledge**, **User Memories**, and **Conversation Context** collections, complete with dynamic operations-per-second (`ops/sec`) delta calculations updated every 5 seconds. A fourth collection, **vault_documents** (the Memory Vault), is managed separately and surfaced via the dashboard's Memory Vault tab rather than this HUD.

### CSI Spatial Radar Dropdown
Provides tactical radar tracking rendered on a 350x350px circular canvas:
* **Spatial Rendering**: Plots continuous $[X, Y]$ spatial telemetry derived from WiFi Channel State Information (CSI) subcarrier deltas and mmWave radar.
* **Telemetry Overlay**: Displays real-time **Velocity** calculations, **Motion Detection** states, **Coordinates**, and **AI Confidence** percentages at 60fps.

Key Features Summary
 * Eyes into Polaris's Mind: The Memory HUD lets you watch vectors and heuristics aggregate as she learns.
 * Eyes into Polaris's Actions: The Task Feed gives you live feedback on background execution, tool calls, and automated jobs as they fire.
 * Eyes into Polaris's Vision: The CSI Radar brings spatial tracking straight into the dashboard interface.

# Looking Glass Interface (LGI) Executive Supervisor Client

### [LGI.md](LGI.md)

The **Looking Glass Interface (LGI)** is the operator's floating HUD and voice cockpit for the Polaris Executive Supervisor system. Unlike the browser-based Polaris Dashboard (`:7000`), LGI is a **native Python process running on the Alienware X17** — deliberately *not* containerized, because it requires raw hardware access: open-mic capture, webcam, screen capture, NVIDIA CUDA, and a frameless always-on-top Qt overlay. It talks **text/JSON only** over the LAN — chat/audit traffic to the Polaris Gateway (`http://192.168.50.51:8082`), spatial telemetry to the Burst Receiver (`:8083`, Socket.IO), and webcam-analyze vision passes to the Mainframe's Ollama (`:11434`). Codebase: 13 Python files + requirements.txt — 4,279 Python lines in the `LGI/` folder (incl. the Phase D probes `tts_probe.py` / `sd_play_probe.py` / `dep_probe.py`; `crash_query.ps1` travels alongside), fully self-documented in its own `LGI.md` blueprint (13 sections, v1.4 "Always Listening" — Oct 2, 2026 Phase 3 turns LGI into a full Looking-Glass peer: ONE Socket.IO feed joins the operator's identity room (chat seed + live streamed replies from every device) and the task-feed `status` room, mirrored on the HUD as a latest-5-per-side rolling window plus a read-only Dashboard Tactical Operations mirror (task feed + Memory Vault browser), with a ⚙ settings kebab for HUD font-size/transparency persisted via QSettings; 13 Python files, 4,591 lines — re-verified against the code Oct 4, 2026 (v1.5.1 + crash_analyze.ps1)) that travels with the deployment.

## 1. Mission Capabilities
* **Open-mic voice chat**: say `Polaris <question>`; the reply is spoken through local Kokoro 24 kHz TTS and mirrored on the HUD.
* **Desktop vision chat**: visual utterances ("Polaris, look at this layout") attach the latest dashcam frame (rolling dxcam capture, 1920px q70) plus optional local Tesseract OCR text; the gateway answers through its multimodal vision LLM ladder and the reply is spoken through Kokoro.
* **Webcam perception — "Her Seeing Me" (v1.2)**: a continuous webcam stack tracks people (YOLOv8n + ByteTrack, persistent IDs), reads operator attention (MediaPipe FaceMesh head pose), and recognizes gestures (palm held ~1.5 s = mic mute toggle, held fist = media play/pause). Camera-directed utterances ("Polaris, analyze this component") are answered by `polaris-ai:latest` on the live camera frame via Ollama `/api/generate`, and spatial telemetry (`radar_motion` + `vision_frame`) feeds the Mainframe's radar room — the X17 replaces MissPi as the sole spatial sensor.
* **Continuous desktop supervision**: every 30 s (and on demand via `Ctrl+Shift+S` or the HUD Force Audit button) LGI gathers a desktop-context snapshot — active window/app, clipboard snippet, screen keywords, optional screen + webcam JPEGs — and POSTs it for heuristic + LLM triage; audits are attention-gated (skipped while the operator is away, fail-open). A `SUPERVISOR_ALERT` verdict turns the SUPERVISOR LED red, raises an alert banner, and speaks the category + reason (epoch-guarded 60 s auto-clear; verbal speech is toggleable — 🔇 silent default, HUD 🔊/🔇 Advisories button, `LGI_SPEAK_ALERTS` env).
* **Ambient awareness with privacy**: all other speech is transcribed locally (faster-whisper) and shown as an ambient transcript line, but is **never sent to the network**.

## 2. Runtime Topology
```
+------------------------- Alienware X17 (192.168.50.227) -------------------+
|  lgi.py  (LGIApp — Qt main thread + 100 ms drain QTimer)                   |
|    |                                                                       |
|    +-- hud_ui.py             frameless always-on-top HUD                   |
|    |                         (MIC/SUP/POL LEDs, exchange, alert banner)    |
|    +-- audio_stt.py          open-mic VAD + faster-whisper     [daemon]    |
|    +-- audio_tts.py          Kokoro TTS 24 kHz, sounddevice    [daemon]    |
|    +-- vision_supervisor.py  screen/webcam/clipboard audits    [daemon]    |
|    +-- gateway_client.py     thread-safe HTTP client (JSON only)           |
|    +-- heartbeat             GET status every 15 s            [daemon]     |
|    +-- global hotkeys        keyboard lib -> command queue     [daemon]    |
+----------------------------- text/JSON only | LAN -------------------------+
                                              v
                          Polaris Mainframe gateway :8082
                          POST /api/chat              -> Ollama LLM
                          POST /api/supervisor/audit -> triage + vault
                          GET  /api/supervisor/status -> liveness
```

* **Qt single-thread rule**: daemon workers never touch Qt widgets. They push events onto queues (`status_q`, `command_q`, `stt_q`, `chat_q`, `supervisor_q`); a 100 ms `QTimer` drain is the *only* worker→HUD path (bounded ~100 ms UI latency, zero cross-thread Qt calls).
* **Never raise across threads**: every worker converts failures into result dicts and guarded callbacks; exceptions never propagate into the Qt loop.

### Gateway Contract & Telemetry Paths

| Endpoint | Purpose |
| :--- | :--- |
| `POST /api/chat` (:8082) | Wake-word voice chat with Polaris; desktop-vision turns attach `image_base64` / `ocr_text` / `active_window` (120 s timeout, multimodal vision ladder); reply spoken via local Kokoro TTS + mirrored on the HUD exchange. |
| `POST /api/supervisor/audit` (:8082) | 30 s desktop-context triage (heuristic + LLM) → `SUPERVISOR_ALERT` / `LOG_ONLY` / `NO_ACTION` verdicts. |
| `GET /api/supervisor/status` (:8082) | 15 s heartbeat liveness poll → POLARIS LED green/red. |
| Socket.IO → Burst Receiver (:8083) | `SpatialReporter`: `vision_connect` on (re)connect, `vision_frame` @ 2 Hz (detections + narration), `radar_motion` @ 1 Hz (`x/y/z/velocity`, `people_count`) — real coordinates pass through the receiver untouched. |
| `POST /api/generate` → Ollama (:11434) | Webcam-analyze turns: one 640px q50 JPEG + short spoken-style prompt to `polaris-ai:latest`; single-flight, never raises. |

* `gateway_client.py` (`PolarisClient`) shares one `requests.Session` behind a lock and **never raises** — every failure becomes `{"ok": False, "error": ...}`. Timeouts: 65 s chat (Ollama upstream is 60 s), 120 s desktop-vision turns, 30 s audit, 5 s status; `retries=2` with backoff. `local_vlm.py` (`LocalVLM`) is equally defensive — single-flight (concurrent requests fail fast), failures surface as `⚠ webcam:` HUD lines without crashing.
* Gateway error semantics on the HUD: **404** = X17 IP not allowlisted (fix gateway-side), **400** = missing prompt, **503** = Ollama offline — surfaced as error lines while LGI keeps running.
* X17 (`.227` → identity **RICH**) is already allowlisted; audit payloads carry the operator identity, and vault persistence decisions belong to the gateway.
* Audit payload images (screen 1280px q60, webcam 640px q50 — compressed base64 JPEGs inside JSON) are currently **ignored by the gateway**; the fields are shipped for future vault vision ingestion.

## 3. Supervision & Failure Behavior

* **Audit sensors** (each independently guarded): active window title + app (`pygetwindow`, ≤200 chars), clipboard snippet (`pyperclip`, change-gated, ≤4000 chars), screen keywords (≤12 tokens parsed from title), screen JPEG 1280px q60 (`mss`, optional), webcam JPEG 640px q50 (shared perception frame while the stack runs; legacy one-shot `CAP_DSHOW` open/release only when `LGI_WEBCAM_TRACK=0`).
* **Attention gate** (`LGI_ATTENTION_GATE=1`, default): the FaceMesh head-pose proxy marks the operator attentive/away; audits are skipped while away (fail-open when unknown). Explicit voice commands are unaffected.
* **Verdicts**: `SUPERVISOR_ALERT` → red SUP LED + banner + spoken "Supervisor alert. Category: …" + 60 s auto-clear; `LOG_ONLY` → green LED, logged; `NO_ACTION` → green LED, clear; `ok=False` → `audit failed` error line.
* **Degradation ladder — never crash**: no CUDA driver → Whisper CPU/int8 + Kokoro CPU; PyAudio or faster-whisper missing → listener disabled; `mss`/OpenCV missing → that image sensor auto-disables permanently; camera busy → CAM LED off, perception stack disabled, audits fall back to the legacy one-shot; mediapipe or ultralytics missing → that stage alone disabled (CAM LED "degraded", tracking/attention keep running); dxcam/DXGI blocked → ScreenRoller falls back to `mss` one-shot grabs (vision turns still work); Burst Receiver down → telemetry dropped silently, 10 s reconnect loop, local sensing unaffected; VLM failure → `⚠ webcam:` HUD line + spoken "I could not analyze the camera view"; Kokoro init failure → chat replies still render on the HUD; gateway unreachable → error lines + red POLARIS LED while LGI keeps running.

## 4. Privacy Guarantees

* **Wake-word gate** (`LGI_WAKE_REQUIRED=1`, default): only wake-word utterances are transmitted; ambient speech is transcribed locally for display and never leaves the machine.
* **Local voice-stop commands**: `stop talking`, `be quiet`, `quiet please`, `silence`, `stop voice`, `shut up` are matched locally — silencing the voice never touches the network.
* Ports 8082 **and** 8083 carry **text/JSON only** — never raw audio, never video streaming; the radar path carries coordinates/counts, never frames.
* **Webcam light honesty (v1.2)**: the perception stack holds the camera open while LGI runs, so the hardware LED is ON whenever tracking is enabled; `LGI_WEBCAM_TRACK=0` reverts to the legacy per-audit open/release (LED off between audits).
* **Perception frames never stream out**: the continuous webcam stream stays on the X17 — only coordinates/counts ride `radar_motion` / `vision_frame`; the rolling desktop dashcam buffer never leaves the machine either.
* **Webcam-analyze frames**: one 640px q50 JPEG per camera-directed turn goes to the configured Ollama endpoint (default the Mainframe's cloud-proxied `polaris-ai:latest`; point `LGI_VLM_URL` at a local Ollama to keep analyze frames fully on-device).
* **Sensor minimisation**: `LGI_SCREENSHOT=0` / `LGI_WEBCAM=0` strip images from audits entirely.
* Clipboard is change-gated: stale contents are never re-sent.
## 5. Voice-Restoration Hardening (Sep 30, 2026)

The silent-voice era closed with two real root causes — the output-device switch (USB/DXS) was a red herring — plus a permanent diagnostics layer:

* **pythonw import death — defused**: under a console-less boot (`pythonw`), `sys.stdout`/`sys.stderr` are `None`; the `huggingface_hub` import chain prints an stdout warning whose flush kills the chain, so the **kokoro import died silently before any audio device was ever opened** (worker never reached `worker ready`). Fixed with a console guard in `lgi.py` @58.
* **Qt-slot qFatal — defused**: a PyQt6 `qFatal`-grade abort (`0xc0000409`) inside a Qt slot terminates the process with no traceback. A custom `sys.excepthook` in `lgi.py` @93 logs and survives.
* **Diagnostics layer (live-tested)**: the TTS `_trace` tee → `%TEMP%\lgi_tts_log.txt` — **permanent** (sealed with a 1 MB truncate-on-open cap; decision closed Sep 30, 2026) and the **first-read triage surface** — voice dead → read `lgi_tts_log.txt` first; missing `worker ready` = boot killed the worker, not the device (LGI.md §10 item 11). Alongside: `%TEMP%\lgi_console.log` (stdout/stderr tee) and `D:\LGI\crash_query.ps1` (WER/dump query). The Phase D probes (`tts_probe.py`, `sd_play_probe.py`, `dep_probe.py`) + `crash_query.ps1` travel with the repo and `D:\LGI`; the four probe SchTasks and %TEMP% probe artifacts were purged after recording (LGI.md §10 item 12).
* **WATCH (LGI.md §11)**: flaky native boot crash — `cudnn64_9.dll` `0xc0000409` / ntdll heap corruption `0xc0000374` during STT init (two occurrences Sep 30: +2 s boot death and 18:22:31, each recovered by a healthy relaunch; probe artifacts + capped tee unaffected); one uncaptured alert-slot exception (18:03:21 Sep 30) will land in `lgi_console.log` if it recurs. **Oct 4 update**: the class recurred ~daily in silence (ntdll `0xc0000374` / ucrtbase `0xc0000409`) and a second finding joined it — igniting LGI while a session is already live kills the newcomer inside ~1 s (GPU-stack race, proof 15:25:20→21). Double defense shipped Oct 4 late evening: `lgi.py` v1.5.1 takes a `Global\LGI_X17_SingleInstance` mutex before any GPU/audio init (second launch self-refuses — live-tested) and `watchdog.ps1` v2.1 clears the stale session marker at relaunch plus a 3-per-10-min crash-loop brake (cooldown slow-retry while the marker persists). Freshest forensics: pristine 336 MB dump `pythonw.exe.30564.dmp` — WinDbg `!analyze -v` is the next step (Debugging Tools install = operator decision). **Postmortem prep same night: `crash_analyze.ps1` v1.0 shipped to `D:\LGI` — `-Posture` readiness audit, `-Latest`/`-All` one-command cdb `!analyze -v` drivers (reports to `D:\LGI\crash_reports\`), `-EnsureDumps` pinned full-dump capture (keep 15), `-Install` staged the debugger tooling (winget `Microsoft.WinDbg` → Windows-SDK Debuggers fallback; the fallback's single UAC click is the operator's) — the verdict on the 30564 dump lands with the first post-install run. **Verdicts same hour: no UAC needed (WinDbg MSIX ships `amd64\cdb.exe`; winget route won); 30564 = `DOUBLE_FREE ucrtbase!free_base`, 12900 + 19204 = `DOUBLE_FREE cv2.pyd!unknown_function` — the daily spontaneous class is a native double-free in OpenCV's cv2.pyd, not the voice stack; STT/cpu A/B demoted, camera-path A/B is the operator's lever.**
