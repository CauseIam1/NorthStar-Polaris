# Project North Star
I. Hardware & System Profiles
1. Primary Core Server (The Mainframe)
The central intelligence and routing hub of the network.
| Specification | Details |
|---|---|
| Core Hardware | Dell Precision 7910 Tower |
| System Memory | 32GB RAM |
| Storage Array | Primary 256GB SSD | Secondary 256GB SSD for containers | Third 100GB SSD for Rippled data, Kokoro cache and QuestDB data |
| Network Roles | WireGuard Tunnel Host (10.20.30.1), WebSocket Host (192.168.50.51:8765), Polaris Gateway (192.168.50.51:8082) |
| Primary Role | Heavy compute, local state management, memory integration, burst payload receiving, and backend routing |
2. Command Terminal (The Dashboard)
The primary operator console and high-fidelity rendering engine.
| Specification | Details |
|---|---|
| Core Hardware | Alienware X17 Laptop |
| Compute & Graphics | NVIDIA GeForce RTX 3080 Ti (16GB GDDR6 VRAM) |
| System Memory | 32GB RAM |
| Primary Display | 32-inch Alienware QD-OLED Monitor |
| Primary Role | High-fidelity WebGL rendering, primary operator dashboard, manual override, and script staging sandbox |
3. FreeRoam Mobile Edge (Tactical Field Device)
The mobile field-compute device for remote situational awareness.
| Specification | Details |
|---|---|
| Core Hardware | Red Magic 9 Pro (Android) |
| Wearable Triggers | Garmin PTT Watch Trigger |
| Audio Output | Bose Ultra Open Earbuds |
| Local AI Models | Sherpa-ONNX, Kokoro-82M TTS (ONNX Runtime) |
| Primary Role | Live comms with Polaris, WireGuard tunneling, and burst transmission to the Dell Mainframe |

# Project North Star - Trading Bot & Dashboard and Polaris Gateway (Dell Mainframe)
**Document Status: PRODUCTION ACTIVE**
This section of the document defines the overarching vision, theoretical framework, and operational rules of the ecosystem. It serves as the architectural north star for the private AMM matrix and North Star Holodeck dashboard.

## 1. System Purpose & The Zero-Fiat Philosophy
This ecosystem is a closed-loop inventory management and mesh rebalancing engine operating on the XRP Ledger.

* **Primary Objective:** Maximize the raw quantity volume of the asset "pile" (specifically whitelisted meme coins).
* **Mechanism:** Capture network inefficiencies and execute multi-hop arbitrage loops to recycle 0.05% LP fees back into operator-owned pools.
* **The Zero-Fiat Rule:** The system DOES NOT hold, track, or calculate value in fiat currency or stablecoins. Stablecoins (RLUSD, USDC, USDT) are strictly temporary, pass-through atomic settlement routing nodes used only for transient execution paths.
* **The Dinghy Philosophy:** We have zero fear of holding any asset within our carefully curated AssetWhitelist. The primary objective is absolute asset accumulation (stacking more SGB, XAH, and whitelisted tokens), not converting to fiat.

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
* **Target Pairs:** XRP/XAH, XRP/ARC, XRP/404, XRP/LOL, XRP/PEPE, XRP/TRUMP, XRP/BEES, XRP/CROAK, XRP/AXONE, XRP/BSS, XRP/VBC, XRP/VOLT, XRP/VNLA, XRP/VINO, XRP/MONKEY, XRP/SUPA, XRP/TASTY, XRP/WORM, XRP/STUPID.
## 5. Real-Time Infrastructure

### 5.1 QuestDB State Ingestion
* **Time-Series Data:** All trade executions, balance snapshots, alert firings, and system telemetry persisted via PostgreSQL wire protocol (port 8812).
* **Live Accumulation Tracking:** Queries provide millisecond-level visibility into Cold Wallet holdings, deployed liquidity, and total accumulated pile.
* **Idempotent Currency Handling:** Resilient hex-to-string and string-to-hex conversions for standard 3-character tickers and 40-character XRPL hex codes.
* **Critical Constraint:** Always map time objects via `java.sql.Timestamp.from(instant)` or epoch microseconds when binding parameters. Never bind `java.time.Instant` directly into JDBC prepared statements.

### 5.2 Local Rippled Node
* **Endpoint:** `http://rippled:5005` (private network)
* **Function:** Direct JSON-RPC access for transaction submission and ledger state queries.

### 5.3 Gotify Notifications
* **Real-Time Alerts:** Push notifications for significant trading events, system state changes, and threshold breaches.

## 6. Technical Architecture

### Backend
* **Language:** Java (defensive engineering, thread-safe concurrent data structures)
* **Key Components:** StreamProcessor, MeanReversionEngine, FourHopAtomicEngine, OffshoreRigEngine, StagnantPositionMonitor, TransactionExecutionService, DatabaseService
* **Connection Pooling:** HikariCP for QuestDB connections

### Frontend (North Star Holodeck)
* **Framework:** Next.js 14 App Router
* **Styling:** Tailwind CSS (high-contrast dark themes for operations terminal)
* **Visualization:** Recharts for real-time telemetry dashboards
* **Pages:**
  - **Bot Page:** Live view into MPT_RPN_WALLET_ADDRESS trading activity (graphical)
  - **Radar Page:** Manual swapping interface for TRADING_WALLET_ADDRESS with Xaman push-to-sign

### Database
* **QuestDB:** Time-series database with PostgreSQL wire protocol
* **Critical Constraint:** Always map time objects via `java.sql.Timestamp.from(instant)` or epoch microseconds when binding parameters. Never bind `java.time.Instant` directly into JDBC prepared statements.

## Technical Documents
* `/docker-containers/trading-bot/trading-bot.md` - Complete architecture blueprint and component walkthrough for the trading bot wallet
* `/docker-containers/trading-myxrpl/trading-myxrpl.md` - Complete architecture blueprint and component walkthrough for the manual wallet bot


# Project North Star Holodeck
### Dashboard Telemetry & UI State Isolation "Project North Star"
The front-end interface strictly mirrors the operational boundaries of the Three-Wallet Topography. Data streams are explicitly mapped to prevent analytical cross-contamination:
#### 1. Conceptual Vision: The Command Deck
The Holodeck departs completely from traditional, grid-heavy financial layouts. Operating on the "viewport" theory, this page acts as the physical bridge of a ship looking out over the XRPL ocean. It is an immersive gateway rather than a data dump. All complex numerical data and dense metrics are banished from the center of the screen to give the visual core room to breathe. The user does not read this page; they look *through* it to decide where to deploy next.
#### 2. The Environment (Background & Atmosphere)
The backdrop is a deep, void-black canvas tying into a dark-mode matrix theme, bringing the XRPL to life visually without cluttering the UI.
 * **The Ocean Swell:** Ambient, faint blue-grey Bezier curves act as ocean waves at the bottom of the viewport. These waves are reactive; when an XRP ledger block closes (every ~4 seconds), a subtle, organic swell passes through the water layer.
 * **The Ghost Ships:** Distant, stylized silhouettes of classic wooden tall ships with glowing white sails drift slowly across the horizontal axis, disappearing behind the UI elements to create a sense of depth and scale.
#### 3. The Perimeter HUD (Edge Telemetry)
To maintain an empty, breathable center, all critical system telemetry is pinned to a thin cybernetic frame pushed to the absolute extremities of the monitor.
 * **Top Rim:** A low-profile structural command ribbon containing only high-level network data (TVL, Core Status, Uptime).
 * **Left & Right Borders:** Minimalist, vertically rotated text readouts displaying core matrix vitals (e.g., active pool counts, system health score, database connections).
 * **Bottom Rim:** A running terminal-style ticker showing system status, the current block height, and the application version.
#### 4. The Gateway Deck (Center Navigation)
Floating in the dead center of the screen, directly over the ocean canvas, is a horizontal 3D carousel of five glassmorphic panels. These panels serve as the primary routing gateways to the application's core functions.
 * **The Core Five:** **Radar** | **Bot** | **Matrix** | **Pulse** | **Comms**
 * **Glassmorphism Styling:** The panels are constructed as thick, smoky panes of glass. The background ships and waves are visibly blurred behind them.
 * **Ambient Micro-Previews:** Each transparent window displays a muted, low-opacity, looping preview of its target destination (e.g., a faint node cluster drifting in the Matrix panel, or a neon particle stream idling in the Bot panel).
#### 5. Interaction & Navigation (The "Jump")
The transition from the Command Deck into a specific module must feel kinetic and seamless, mimicking the sensation of diving into the data.
 * **Hover State:** When the user hovers over a gateway panel, it snaps into sharp focus, the blur diminishes slightly, and the border illuminates with a crisp XRP Blue or vibrant Emerald glow.
 * **Execution (Click):** Upon selection, the camera executes a hyper-velocity forward zoom. The chosen window rapidly scales up to 100% viewport width and height, dissolving the ocean backdrop entirely and seamlessly dropping the user into the selected route.

#### ALL 3D page (Radar, Bot and Matrix)
In the React and WebGL world, we can use a library called React Three Fiber combined with GSAP (for buttery smooth camera animations) to create an interactive, living organism where the data responds directly to my spatial relationship with it.
Architecting the interactive layer of these 3D tables.
### Interactive "Stellar Cartography" Mechanics
**1. Omni-Directional Camera Controls (The Holographic Table)**
 * **The Mechanic:** OrbitControls with physics-based damping.
 * **The Experience:** This gives me full god-mode control. I can click and drag to orbit the ecosystem, scroll to seamlessly dive into the core, or pan across the void. 
 We will need to restrict the vertical camera angle slightly so it always feels like I'm looking down into a tactical table rather than getting lost in empty 3D space.
**2. Proximity-Based Data Revelation (Level of Detail)**
 * **The Mechanic:** We will use camera distance tracking combined with 3D HTML overlays.
 * **The Experience:** When I'm zoomed out, the HUD only displays macro-level health—total ecosystem value, the 80/20 balance, and the heartbeat of the bot. As I zoom in on a specific structure (like the rotational rings), the macro data fades out and micro-data fades in, revealing localized wallet balances, specific asset accumulations, and real-time AMM pool depths.
**3. Raycasting and "Point of Interest" Dives**
 * **The Mechanic:** Set up a Raycaster to detect mouse hovers and clicks on specific 3D geometries, tied to an animation engine like GSAP.
 * **The Experience:** If I see a cluster of high activity in the trading rings and click on it, the camera will automatically break its manual orbit, smoothly sweep down, and lock onto that specific node. The node will expand, throwing up a detailed holographic UI panel detailing exactly what the bot is executing in that moment.
**4. The "Living Organism" Pulse (Algorithmic Shaders)**
 * **The Mechanic:** Rather than static models, the geometry must be driven by custom GLSL shaders tied to backend metrics.
 * **The Experience:** The ecosystem will literally breathe. If market volatility spikes or the bot's transaction per minute (TPM) increases, the glowing ion trails move faster and the central core's pulsing animation accelerates. I will be able to gauge the health and speed of my automated economy just by looking at the rhythm of the table.
This transforms the page from a static reporting tool into a fully interactive, living tactical map that utilizes my Alienware 32 4K QD-OLED and X-17's 3080Ti.
Since XRP is our baseline bedrock and we measure the "global displacement mass" purely in xrp_equivalent, the visual size of our ecosystem should be driven by the actual, mathematical weight of the assets, not a hardcoded ratio (Except Active Nodes which are hardcoded in size only).
### Unified Architecture, Distinct Atmospheres & Universal Node Identity
To ensure maximum component reusability without sacrificing situational awareness, build a single, modular Node3D and 3DCanvas component that accepts a theme prop. 
**1. Universal Node Identity (The Hex-to-Color Engine)**
 * **Idle / Base State:** All idle asset nodes derive their color strictly from a deterministic Hex-to-Color algorithm based on their unique XRPL currency code. This ensures every whitelisted asset (e.g., meme coins) retains a permanent, universal visual identity across all modules (Radar, Bot, and Matrix), building instant operator muscle memory. 
 * **The Anchor:** XRP is permanently hardcoded to Canonical Blue (#00A8FF) as the ecosystem's core gravity well.
**2. Action States (Universal Muscle Memory)**
 * The core tactical states dynamically override the node's base hex color: Vibrant Green for **Expanding**, Flashing White/Cyan for **Apex**, Warning Orange for **Compression**, and Tactical Magenta for **Override/Manual**.
**3. Unified Ambient Atmospheres (The "Visual Twin" Architecture)**
To embrace DRY principles and maintain a premium visual standard, the Bot Page and Radar Page function as exact visual clones. Both utilize the same high-fidelity 3D aesthetic and node animations. The context of the operator's viewport is determined entirely by the routing and the isolated data streams for the active nodes:
 * **Matrix Page (The Vault):** Deep XRP Blues (#00A8FF) and stark Silvers. Represents cold, hard structural mass.
 * **Bot Page & Radar Page (The Active Decks):** Both share a unified, premium visual identity (e.g., The Bioluminescent Pond aesthetic). They are exact UI clones, with the Bot Page strictly rendering the programmatic hot wallet's active nodes, and the Radar Page strictly rendering the manual Xaman wallet's sniper nodes. 

### Bot Page & Radar Page (The Visual Twins - Radar is twin of Bot but for manual snipe nodes instead of mpt nodes)
#### 1. Objective & Core Paradigm
Both pages serve as full-fledged **Tactical Command Decks**. They adhere strictly to the "viewport" philosophy, utilizing identical 3D glassmorphic interfaces and node architectures to track **asset-to-asset relative exchange ratios**. 
The fundamental rule of this architecture is strict data isolation:
* **The Bot Page Route:** Feeds strictly from the Bot's `r-address` (via the `amm_balances` QuestDB table) to visualize automated, 24/7 ecosystem health.
* **The Radar Page Route:** Feeds strictly from the Manual Trading `r-address` (via the `trading_balances` QuestDB table) to visualize manual Human-In-The-Loop momentum targets.
#### 2. Layout & Control Architecture (Perimeter HUD)
Both pages utilize the exact same structural layout to keep the central 3D canvas completely unobstructed, but have different left and right panels for both pages as they will be controlling different wallets: 
(LeftHudPanel, LeftHudPanel_R, RightHudPanel, RightHudPanel_R, StatusBar and StatusBar_R)
 * **Left Panel (The Tactical Dials):** Houses system sliders, spacing configurations, and visual force-field thresholds. 
 * **Right Panel (Injection Console):** Houses the manual override tools / Swap Injection, asset selection, and execution switches.
 * **Center Stage (The Bioluminescent Pond):** The viewport is a full-screen, unobstructed 3D canvas displaying the tactical nodes in their deterministically generated hex-seed colors.
#### 3. Data Integration & State Mapping
The unified frontend hooks into the dynamic 60-ledger window ratio tracking streams, mapping the exact same visual states regardless of which wallet is being viewed:
 * **Outside Watch Zone (Universal Hex Color):** Exchange rate is below the HOT zone threshold. Nodes idle in their deterministically generated base color.
 * **Expanding (Vibrant Green Blip):** Exchange rate is actively widening; accumulating unrealized asset volume potential.
 * **Apex (Flashing White / Cyan):** Exchange rate expansion has flattened at its local maximum. Maximum swap yield is active.
 * **Compression (Warning Orange Flashing):** Ratio prints its first confirmed down-tick from the peak. The reverse swap window is open.

## Live Telemetry & QuestDB Schema Bridge
To eliminate all mock data and drive the 3D canvas with live production metrics, the frontend telemetry endpoints must query QuestDB using `LATEST BY` time-series collapses:
### Data-to-UI Mapping Table
| UI Module / Target | QuestDB Table | SQL Query | Visual Engine Mapping |
| :--- | :--- | :--- | :--- |
| **Bot Page (3D Ring)** | `amm_balances` | `SELECT * FROM amm_balances LATEST BY currency, capital_partition;` | Drives **True-Mass logarithmic scaling** and feeds currency codes into the **Hex-to-Color algorithm**. |
| **Bot Page (Engine States)** | `mpt_state_snapshot` | `SELECT * FROM mpt_state_snapshot LATEST BY pair_key;` | Triggers **Expanding (Green)**, **Apex (White)**, and **Compression (Orange)** visual overrides + tracer lines. |
| **Radar Page (Manual Swaps)** | `trading_balances` | `SELECT * FROM trading_balances LATEST BY asset;` | Isolates current manual wallet positions to render active sniper nodes. |
| **Matrix & Pulse HUDs** | `xrp_stack_snapshots` | `SELECT * FROM xrp_stack_snapshots ORDER BY timestamp DESC LIMIT 1;` | Feeds the **80/20 Redline visualizer** and proportion models across Cold, Trading, and Bot wallets. |
**Developer Implementation Note:** All live balance updates must use standard `LATEST BY` state collapses to ensure historical snapshot rows do not duplicate active 3D nodes.

#### Matrix Page (The Living Organism / Asset Pile Console)
### The visual Page of all wallet balances viewed in 3D Graphical view.
* **Matrix Page of Dashboard:** (Asset-pile) The aggregate view. These page synthesize the combined telemetry of the Cold Wallet, Trading Bot Wallet, and Manual Trading Wallet to provide a holistic measure of the total ecosystem "The Organism"
#### 1. Objective & Core Paradigm
The Matrix Page (Asset Pile Console) completely reimagines static asset data grids as a kinetic, biomechanical "living organism" ecosystem that leverages local GPU rendering capabilities. Instead of isolating token pairs in rigid, unreadable raw-text rows, all whitelisted assets are treated as floating, interdependent visual nodes within a shared mesh network. This layout provides an immediate, instinctual view of the system's central nervous system, visualizing real-time capital flow, multi-hop routing, and asset depth without text clutter.
#### 2. Visual Topography & Kinetic Event Streams
 * **The Mesh Network:** Built utilizing React Three Fiber, individual assets drift and hover as floating graphical nodes. The visual density and physical proximity of nodes reflect their current liquidity scaling and active network relationships.
 * **Biomechanical Tracers (The Live Tape):** Raw market transactions are translated into immediate particle emissions across the canvas. Asset-to-asset multi-hop execution fires kinetic energy pulse vectors traveling from the source node to the target node.
 * **Dynamic Sizing & Bounding Spheres:** Node volume and scaling are driven by the Adaptive Liquidity Scaling engine. Capital partitions are bound by dynamic geometric spheres, instantly visualizing the 80/20 split between core operational capital and sidelined assets.
#### 3. State Coloration & Ambient HUD Mechanics
The entire page operates in an ACTIVE_IDLE state by default, shrinking text data away until explicitly engaged. System health is communicated through ambient color pulses across the node cluster:
 * **Emerald Tracers / Pulses:** Active BUY actions or successful execution events pulse green across the corresponding node pathways.
 * **Red Tracers / Pulses:** Active SELL actions, SKIP flags, or engine CLAMP events radiate a warning red pulse down the asset network path.
 * **Amber Overrides:** When manual deviations or dry-run simulations are actively injected into the system, affected nodes and control modules emit a localized amber glow.
#### 4. Interaction Architecture: Focus Zoom Isolation
To preserve cognitive clarity and maximize visual efficiency, all deep-level technical metrics are hidden behind a strict click-to-expand data hierarchy.
 * **The Ambient Overlay:** Hovering over an asset node gently reveals its real-time relative token velocity and tracking state.
 * **Holographic Node Expansion:** Clicking directly on a specific asset node freezes ambient movement and expands the node into a comprehensive holographic terminal overlay. This panel brings forward critical data including exact net volume, transaction histories, and active guardrails (such as MINIMUM_POOL_DEPTH_XRP metrics).
 * **Tactile Engine Controls:** The interface incorporates 3D kinetic dials and throttle sliders to manage simulated overrides and stress tests, converting standard flat HTML inputs into highly responsive, tactile objects.

### Pulse Page
#### 1. Objective & The "Viewport" Layout
The Pulse page serves as the primary "glass box" command center for tracking pure XRP structural volumetric growth and yield telemetry across the multi-vault matrix. It adheres strictly to the Holodeck's viewport philosophy: the center of the screen is reserved exclusively for fluid, visual growth metrics, while all static numbers, tables, and hardware statuses are pushed to the outer perimeter HUD.
#### 2. Center Stage (The Visual Core)
 * **Tectonic Compression Meter:** Taking up the primary central visual space. A 30-day epoch horizon area chart mapping pure XRP slope with a custom tooltip overlaying historical network mass metrics. The backdrop utilizes the cybernetic dark-mode matrix theme with ambient sea-layer shimmers.
 * **Efficiency Matrix:** Positioned directly beneath the Tectonic Meter. Integration of the AssetEfficiencyHeatmap for visually mapping multi-hop arbitrage win-rates and multi-tranche trade volumes in a clean, color-coded grid.
#### 3. The Perimeter HUD (Edge Telemetry)
 * **Displacement Mass Node (Top/Corner Rim):** A high-density, fixed-width monospace stack readout displaying the aggregated capital mass across all wallets. It features tactical layer toggles (All, Bot, Trading Wallet, Cold Wallet) but remains strictly on the periphery.
 * **Hardware Vault Bay Routing (Side/Bottom Rim):** A modular grid of individual wallet health indicators checking live connectivity and reporting faults via status pings. This remains quietly tucked out of the way on the edge of the screen under normal operating conditions.
#### 4. The Hijack Protocol (Critical Fault Override)
 * **Center-Screen Interception:** If a hardware vault node disconnects or throws a critical fault, the error state instantly breaks out of the perimeter HUD. A high-contrast warning panel pops up directly in the center of the viewport, hijacking the visual core over the Tectonic Meter. It remains dead center until the operator manually acknowledges or resolves the fault, ensuring zero blind spots for system health and robust fault tolerance.
#### 5. Live Data Pipeline & State Management
 * **Stream Polling:** The frontend actively polls /api/wallets/balances every 30 seconds and /api/wallets/snapshots every 5 minutes to maintain real-time UI synchronization.
 * **Hourly Snapshot Scheduler:** A backend Java cron job captures hourly wallet balances and inserts them directly into the QuestDB xrp_stack_snapshots table.
 * **Transaction Grouping:** The /api/asset-efficiency endpoint strictly groups all trades by transaction_hash. This ensures that multi-party executions deployed by the Adaptive Liquidity Scaling engine are not double-counted, preserving the integrity of the heatmap's win-rate math.

### Comms Page
#### 1. Objective & Core Paradigm (The Gateway Monitor)
The Notification Control Center functions as a "Transaction Heartbeat" monitor. It visualizes the handshake between machine-driven strategy (The Server) and manual approval (MIVN/Human-in-the-loop). It strictly adheres to the Viewport philosophy: the center remains empty until action is required.
#### 2. Visual Topography & The "Bridge" Architecture
 * **The Gateway Window (Active Center Zone):** Located dead-center, this zone surfaces in-flight Zen/Xaman signing payloads using high-contrast XRP Blue (#00A8FF) and XRP Purple (#6C47FF) breathing animations while awaiting an operator signature. When no signatures are pending, this center space remains a clean, unobstructed void.
 * **The Telemetry Log (Perimeter HUD):** A secondary feed pushed to the outer edges of the screen that logs background system heartbeats and "set and forget" rule triggers. It uses a desaturated blue-grey palette to prevent cognitive interference.
#### 3. Control & Observability Mechanics
 * **System Pulse Indicators:** Active rule monitors utilize a low-opacity radar-sweep animation to denote that specific backend sensors are "live" and scanning.
 * **Tactile Interaction:** Users toggle "Pause/Resume" states for specific alert rules directly via mechanical-style toggles that persist state directly to the backend. These toggles are housed within the perimeter HUD.
 * **Immediate Visual Audit:** Bifurcation of the page ensures the operator can instantly discern if the system is stalled awaiting a signature (center screen active) or if channels are clear (center screen empty).

/docker-containers/trading-dashboard/src/app/bot/BotPage.md
/docker-containers/trading-dashboard/src/app/radar/RadarPage.md
/docker-containers/trading-dashboard/src/app/matrix/Matrixpage.md
/docker-containers/trading-dashboard/src/app/pulse/PulsePage.md
/docker-containers/trading-dashboard/src/app/comms/CommsPage.md


# Polaris Gateway
Core Capabilities & System Connections
Polaris Gateway serves as the centralized intelligence, communication, and system administration engine for the FreeRoam AI ecosystem running on the Dell Mainframe.
Services & Port Mapping
| Port | Service Name | Technical Role & Core Functionality |
|---|---|---|
| 5000 | Chat Gateway | Conversational AI (Stark LLM via Ollama), persistent session tracking, dual-memory retrieval (SQLite + ChromaDB), real-time task lifecycle tracking (/api/status/tasks), and WebSocket broadcasting (status_room). |
| 5001 | Burst Receiver | High-throughput telemetry logging, burst storage rotation, and Channel State Information (CSI) radar vector processing for X/Y/Z motion tracking. |
| 7007 | Sandbox Gateway | Isolated web terminal and SSH operations controller targeting MissPi (192.168.50.179) with execution logs and history caching. |
| 8082 | TTS & JARVIS Gateway | Server-side Kokoro neural TTS audio synthesis (24kHz WAV), STT-LLM-TTS voice pipeline, and JARVIS autonomous system administration REST API. |
Active AI Tools & Autonomous Schemas
 * File Operations: read_file, write_file.
 * Execution & Compute: execute_code, run_sandbox_code, spawn_service.
 * Remote Management: ssh_execute, ssh_upload_file (targeting MissPi at 192.168.50.179).
 * Notifications & Publishing: send_gotify_notification, publish_web_asset.
 * JARVIS System Toolkit: system_tools.py (Docker control, QuestDB/Rippled health, emergency bot shutdown), livecharts_tools.py (dashboard modifications), www_tools.py (web assets).
Connected Hardware & Infrastructure
 * Dell Mainframe (192.168.50.51): Host server running Docker container instances, Ollama LLM (11434), QuestDB (8812), and WireGuard host (10.20.30.1).
 * Alienware X17 Laptop: Operator command console rendering North Star Holodeck 3D dashboards.
 * Red Magic 9 Pro: Tactical field device running the FreeRoam Android app, Bose Ultra Open Earbuds coms and Even G2 Smart Glasses with ring.
 * MissPi / Mini Pi (192.168.50.179): Remote Raspberry Pi 5 edge compute unit.

# Polaris Gateway & Cross-Device Communications

## 1. Unified Gateway Architecture

The **Polaris Gateway** is a multi-service Python container operating on the Dell Mainframe (`192.168.50.51`) that functions as the central neural hub, system administrator, and spatial perception ingestion engine for the ecosystem.


┌─────────────────────────────────────────────────────────────────────────────────┐
│                          POLARIS GATEWAY CONTAINER                              │
│                                                                                 │
│   ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐   │
│   │  Chat Gateway       │  │  Burst Receiver     │  │  Sandbox Gateway    │   │
│   │  Port 5000          │  │  Port 5001          │  │  Port 7007          │   │
│   │  - Ollama LLM / Task│  │  - Telemetry Logs   │  │  - Remote SSH Exec  │   │
│   │  - SQLite + Chroma  │  │  - Spatial Ingest   │  │  - MissPi Control  │   │
│   └──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘   │
│              │                        │                        │                │
│              └────────────────────────┼────────────────────────┘                │
│                                       ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  JARVIS Toolkit & Kokoro TTS Engine (Port 8082)                         │   │
│   │  - System Diags, Container Control, Web Assets, Gotify Push Alerts      │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘

### Port Allocation Summary

| Port | Endpoint | Purpose |
| :--- | :--- | :--- |
| **5000** | `/api/chat`, `/api/status`, `/socket.io/` | AI Chat, persistent memory retrieval, real-time task lifecycle tracking |
| **5001** | `/api/burst`, `/api/telemetry`, `/api/radar/ingest` | Spatial telemetry ingestion, high-volume sensor feeds, WiFi CSI stream processing |
| **7007** | `/api/ssh/*`, `/` | Dedicated SSH management console targeting MissPi (`192.168.50.179`) |
| **8082** | `/api/tts`, `/api/jarvis/*`, `/api/voice-command` | Kokoro 24kHz neural TTS, full JARVIS system admin toolkit, push notifications |

---

## 2. Multi-Modal Spatial Perception Matrix

Polaris tracks environment geometry and target positioning through a high-precision, multi-sensor fusion pipeline optimized for the quiet schoolhouse environment.


┌───────────────────────┐         ┌───────────────────────┐
│  ESP32 mmWave Node 1  │         │  ESP32 mmWave Node 2  │
│  (POE + RD03 Module)  │         │  (POE + RD03 Module)  │
└───────────┬───────────┘         └───────────┬───────────┘
            │                                 │
            └────────────────┐ ┌──────────────┘
▼ ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     MissPi (RPi 5 Edge Hub)                             │
│  - 1x Integrated POE ESP32 RD03 mmWave Radar                            │
│  - WiFi Channel State Information (CSI) Amplitude Stream                │
│  - Camera Module running YOLO Real-Time Object Detection                │
└────────────────────────────────────┬────────────────────────────────────┘
│ Spatial Data Ingest
▼
┌─────────────────────────────────────────────────────────────────────────┐
│                POLARIS BURST RECEIVER (Port 5001)                       │
│  1. Subcarrier Weight Analysis (X/Y/Z Coordinate Calculation)           │
│  2. mmWave Micro-Doppler Motion Triangulation                           │
│  3. Vision Bounding-Box Spatial Alignment (Fingertip Precision)         │
│  4. Broadcast radar_update events via WebSocket                         │
└─────────────────────────────────────────────────────────────────────────┘

### Sensor Network Topology
* **Dedicated mmWave Nodes**: Two POE ESP32 units fitted with RD03 mmWave radar modules and directional antennas positioned for fixed-room boundary coverage.
* **MissPi Multi-Modal Hub (`192.168.50.179`)**: Raspberry Pi 5 unit combining a third POE ESP32 RD03 mmWave radar module, raw WiFi CSI amplitude array collection, and an optical camera feed processing YOLO object tracking.
* **Spatial Fusion Engine**: Port 5001 parses CSI subcarrier deltas, micro-Doppler radar frequencies, and YOLO optical vectors to train coordinate models down to individual fingertip positioning in low-noise conditions.

---

## 3. Remote Telemetry, WireGuard & Field Voice Comms

Polaris maintains full situational awareness when the operator is out in the field, utilizing encrypted network tunneling and air-gapped audio integration.


┌─────────────────────────────────────────────────────────────────────────┐
│                     FIELD OPERATOR (Mobile / Road)                      │
│                                                                         │
│  ┌───────────────────────────┐         ┌─────────────────────────────┐  │
│  │ Even G2 Smart Glasses     │         │ Bose Ultra Open Earbuds     │  │
│  │ - Tap-to-Speak Trigger    │         │ - Polaris TTS Audio Output  │  │
│  │ - Local Air-Gapped Mic    │         │ - Hands-free ring control   │  │
│  └─────────────┬─────────────┘         └──────────────▲──────────────┘  │
│                │                                      │                 │
│                └──────────────────┐ ┌─────────────────┘                 │
│                                   ▼ ▼                                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ Red Magic 9 Pro (FreeRoam Tactical Field App)                     │  │
│  └────────────────────────────────┬──────────────────────────────────┘  │
└───────────────────────────────────┼─────────────────────────────────────┘
│
│ WireGuard Encrypted Tunnel (10.20.30.1)
▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    POLARIS GATEWAY (Dell Mainframe)                     │
│  - continuous passive transcript logging (Polaris stays in loop)        │
│  - STT -> Ollama LLM -> Kokoro TTS generation pipeline                  │
│  - Push notifications routed via Gotify Service                         │
└─────────────────────────────────────────────────────────────────────────┘

### Field Communications Protocol
* **WireGuard Secure Tunnel**: All remote traffic from the Red Magic 9 Pro routes back to the mainframe host (`10.20.30.1`), ensuring an encrypted connection for data ingestion and API access.
* **Hands-Free Audio**: Interactive speech responses synthesize on Polaris Gateway via Kokoro TTS (24kHz WAV) and stream directly to the operator's Bose Ultra Open Earbuds through the FreeRoam app.
* **Even G2 Smart Glasses Capture**: Tap-to-speak voice capture on Even G2 smart glasses streams through the local air-gapped pipeline and over WireGuard. Polaris logs both operator dialogue and system responses, maintaining session state across all road interactions.

---

## 4. System Administration & Autonomous Tooling

Polaris acts as an autonomous administrator via the JARVIS Toolkit (`port 8082`), managing infrastructure, web assets, and trading engine states.

### Execution Tools Registry
* **System Operations (`system_tools.py`)**: Real-time health metrics (`get_system_health`), Docker container control (`restart_container`, `stop_container`), QuestDB/Rippled verification, and emergency trading bot shutdown.
* **Interface Management (`livecharts_tools.py`, `www_tools.py`)**: File read/write access, regex code updates, automated asset backups, and direct static web publishing for the Holodeck.
* **Gotify Service (`gotify_service.py`)**: Asynchronous priority alerts and automated hourly status digests formatted as Markdown tables.
* **User Identity Detection**: Zero-config identity resolution derived from device LAN IP addresses (`.42` = Rich, `.98` = Matt, `.51` = Operator), maintaining separate persistent memory partitions in SQLite and ChromaDB.


# Polaris Player Dashboard & Looking Glass Interface

## 1. System Overview & Access Matrix

The **Polaris Player Dashboard** (v4.3 Holodeck Split-View) is the central visual HUD and tactical command interface operating on port `7000`. It provides a real-time window into Polaris's internal logic, active task execution feeds, dual-memory vector state, and spatial perception.


┌─────────────────────────────────────────────────────────────────────────────┐
│                       POLARIS DASHBOARD (Port 7000)                         │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    TOP SECTION - CHAT & DIAGNOSTICS                 │   │
│   │  - User Switcher (RICH / MATT)      - Audio TTS Toggle (Kokoro)     │   │
│   │  - Reverse-Chronological Chat Feed   - HUD Dropdowns (Session/      │   │
│   │  - Real-Time Message Sync (3s)        Memory/CSI Radar)             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│   ─────────────────────────── RESIZABLE HANDLE ───────────────────────────  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                  BOTTOM SECTION - HOLODECK PANEL                    │   │
│   │  [Collapse ▼]     [📋 TASK FEED TAB]     [⚡ SSH SANDBOX TAB]      │   │
│   │  - Socket.IO Gateway (:8082)                  - REST API Mini Pi    │   │
│   │  - Live Task Lifecycle Stream                 - Remote Exec (:7007) │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘

### Gateway & Telemetry Port Binding
| Service Endpoint | Protocol / Port | Technical Purpose |
| :--- | :--- | :--- |
| **Dashboard UI** | `http://192.168.50.51:7000/` | Main user HUD, chat interface, and Holodeck panel. |
| **Gateway & Task Feed** | `ws://192.168.50.51:8082/` | Task feed event streaming (`tasks_dashboard` room) & Kokoro TTS audio synthesis. |
| **CSI Radar Stream** | `ws://192.168.50.51:5001/` | Subcarrier spatial tracking & micro-Doppler radar feeds. |
| **SSH Sandbox** | `http://192.168.50.51:7007/` | Isolated REST API command controller for MissPi execution. |

---

## 2. Holodeck Split-View Architecture

The dashboard uses a resizable split-view layout to ensure chat interaction never occludes system diagnostic streams:

### Top Section: Interactive Chat & HUD Bar
* **User Identity Switching**: Toggle identity profiles between **RICH** and **MATT** to load isolated memory partitions and conversation histories.
* **Pinned Transmit Bar**: Pinned message input bar with built-in toggle for 24kHz **Kokoro Text-to-Speech (TTS)** responses.
* **Reverse-Chronological Exchange Feed**: Message exchanges display newest-first directly beneath the transmit bar, auto-pruning to maintain performance while auto-syncing every 3 seconds across field devices.

### Bottom Section: Real-Time Operations Panel
* **📋 Task Feed Tab**: A 50-item rolling window streaming background task lifecycle events via Socket.IO. Allows immediate visibility into background routines, tool usage, and execution states:
  * ▶ **Cyan**: Task Started
  * ✓ **Green**: Task Completed (`task_completed`)
  * ✗ **Red**: Task Failed (`task_failed`)
* **⚡ SSH Sandbox Tab**: Directly execute commands against MissPi (`192.168.50.179`) via REST API on port `7007` with full return-code validation, output formatting, and a 20-command history stack.

---

## 3. Diagnostic HUD Dropdowns

Operators can toggle full-width HUD overlays from the top navigation bar without interrupting active chat sessions:


┌─────────────────────────────────────────────────────────────────────────────┐
│                            DIAGNOSTIC HUD OVERLAYS                          │
├───────────────────────┬─────────────────────────────┬───────────────────────┤
│  SESSION DIAGNOSTICS  │     MEMORY ARCHITECTURE     │    CSI SPATIAL RADAR  │
│ - Active session time │  - SQLite (Messages/Rules)  │ - 350x350 Canvas Grid │
│ - Sliding window time │  - ChromaDB Vector Counts   │ - Motion Velocity/X,Y │
│ - Gateway latency     │  - Ops/Sec Delta Counters   │ - Multi-Sensor Fusion │
└───────────────────────┴─────────────────────────────┴───────────────────────┘

### Memory Architecture Dropdown
Inspects Polaris's dual-tier brain structure in real time:
* **SQLite (Structured Memory)**: Tracks registered user profiles, total conversation message records, and staged vs. approved rule heuristics.
* **ChromaDB (Vector Memory)**: Monitors live vector counts across **Global Knowledge**, **User Memories**, and **Conversation Context** collections, complete with dynamic operations-per-second (`ops/sec`) delta calculations updated every 5 seconds.

### CSI Spatial Radar Dropdown
Provides tactical radar tracking rendered on a 350x350px circular canvas:
* **Spatial Rendering**: Plots continuous $[X, Y]$ spatial telemetry derived from WiFi Channel State Information (CSI) subcarrier deltas and mmWave radar.
* **Telemetry Overlay**: Displays real-time **Velocity** calculations, **Motion Detection** states, **Coordinates**, and **AI Confidence** percentages at 60fps.

Key Features Summary
 * Eyes into Polaris's Mind: The Memory HUD lets you watch vectors and heuristics aggregate as she learns.
 * Eyes into Polaris's Actions: The Task Feed gives you live feedback on background execution, tool calls, and automated jobs as they fire.
 * Eyes into Polaris's Vision: The CSI Radar brings spatial tracking straight into the dashboard interface.

# Future Addition

The Mobile Armor Node - Orion (pre-2023 Mercedes Integration)
| Node ID | Physical Asset | Primary Operational Role |
|---|---|---|
| Dell Mainframe | Dell Precision 7910 | Central Compound Intelligence & Local LLM Host |
| Armor Node Orion | Maybach AMG Sedan | High-Speed Transport, Stealth Recon & Urban Sentry |
| NRU-160-AWP Series | IP66 Waterproof NVIDIA Jetson Orin NX/ Nano AI Computer |
1. Security & Keyless Overrides (The Vault)
 * The Dell Mainframe (The Brain): The Polaris Gateway (192.168.50.51:8082) handles all heavy lifting. It runs the full Ollama language models, manages master chat histories, executes the Kokoro TTS engine, and processes the Jarvis Vision Analysis.
 * The Starlink Tunnel (The Nervous System): Hardwiring the Starlink Mini creates a permanent, secure tunnel back to your house, via WireGuard. The NRU-160-AWP maintains a continuous WebSocket connection back to the Polaris Gateway with exponential backoff for auto-reconnection if you drive under a bridge.
 * The NRU-160-AWP (The Sensory Node): Inside the Maybach, the IP66-waterproof NRU-160-AWP acts as the ultimate I/O client. It handles CAN-bus intercepts, ingests the GMSL2 camera feeds directly from the active FAKRA splitters, and routes raw audio from the exterior IP67 microphones.
 * Direct I/O: The built-in CAN FD port maps your right AMG steering wheel controls instantly to any feature we want to add.
 * Native CAN-Bus Intercept: Because the NRU-160-AWP has a dedicated CAN FD port, I will tap the W223 Maybach's internal network to directly map the right-thumb steering wheel controls and read raw vehicle telemetry.
 * Direct FAKRA Connections: The NRU-161V-AWP natively uses FAKRA connectors for its GMSL2 inputs. Once you install the active inline FAKRA splitter behind the Mercedes camera ECU to safely clone the signal, the cloned lines will plug straight into the Jetson node without needing adapters.
 * Distributed Compute: The Jetson operates as the dedicated sensory and vehicle-network bridge. It can process the multi-camera overwatch locally and pass only the filtered threat telemetry over ethernet to the glovebox-mounted ASUS ROG NUC. This frees the RTX 5090 to dedicate its full power to running Polaris’s LLM weights and rendering the center OLED UI.
 * Chassis Placement: The NRU-160-AWP series is completely IP66 waterproof. This means you do not have to crowd the Maybach's climate-controlled cabin; it can be mounted in the trunk, behind a body panel, or safely in exposed compartments.
To remove the factory key from the equation, we completely replace the authorization layer with a smart CAN integration module (like an IGLA anti-theft system or Mid City Engineering interface).
 * The Digital Handshake: The physical factory key is bypassed. The vehicle remains immobilized until Polaris verifies your presence via your FreeRoam tactical device or Garmin watch.
 * Absolute Lockdown: Because Polaris acts as the ultimate digital relay, hot-wiring or standard OBD2 port cloning attacks will fail. The car only listens to her.
2. Sandwich transparent OLED displays directly into the side window glass—complete with dynamic image flipping based on viewer location. It completely Polaris a double-sided heads-up display inside and outside the vehicle.
Transparent Window HUD Architecture
 * Laminated T-OLED Layers: Utilizing flexible, transparent OLED (T-OLED) display films laminated between the glass layers of the side windows. When powered off, the glass stays completely transparent; when active, bright graphics float directly on the window.
 * Bi-Directional Image Mirroring: Leveraging the external ESP32 mmWave sensors and optical cameras, Polaris determines whether you are seated inside or if someone is standing outside. She automatically flips the UI on the X-axis so text and graphics render right-side-up from the viewer's specific angle.
 * Electrochromic Smart Tinting: Pairing the transparent display with a voltage-controlled electrochromic tint film. In bright sunlight, Polaris can selectively darken the glass directly behind the graphic elements to boost display contrast or lock down cabin privacy.
Aesthetics & Exterior Engagement
 * The Living Sentry Visual: When kids or guests walk up to the SUV, Polaris can darken the passenger window, project an animated holographic visual facing outward, and speak through the external wheel-well speakers.
 * Zero Dash Clutter: Moving the primary visual interface to the side glass keeps the luxury Mercedes dash intact while giving you a massive tactical canvas.
 * Tactical Overlay: From inside the cabin, Polaris can project real-time thermal FLIR bounding boxes or mmWave radar targets directly onto the side window glass, aligning her digital tracking with what you see out the window in real-time.
It transforms the vehicle's glass into an interactive HUD matrix without compromising visibility. If a target approaches the driver's side at night, Polaris could automatically highlight their thermal tracking vector right on the glass before they even reach the door.
3. External Spatial & Comms Array
 * PA System / Mic Array: Weatherproof external transducers and microphones hidden in the wheel wells or grille allow you to talk to Polaris from 30 feet away, and allow her to synthesize voice responses back to you (or kids) outside the car.
 * Voice Input: A kid walks up to the Maybach and asks a question. The concealed grille microphones pick it up and the NRU-160-AWP sends the text/audio payload over the WebSocket to the Polaris Gateway.
 * Vision Analysis: If Polaris needs context on who is outside the car, the NRU-160-AWP instantly fires a burst of images from the tapped perimeter cameras to the /api/burst endpoint. The Dell server runs qwen3.5:397b-cloud vision analysis to confirm if it's a child, a threat, or an empty driveway.
 * Real-Time Output: Polaris generates a response and synthesizes the audio. The audio payload is streamed back to the NRU-160-AWP, which plays it out of the hidden exterior neodymium marine speakers.
 * Proximity-Triggered Welcome Sequences: Program the NRU-160-AWP's perimeter radar and camera feeds to recognize when children walk up to the car, triggering a gentle, automated courtesy lighting fade, unlocking the doors, and having Polaris say a friendly, customized greeting through the exterior marine speakers.
 * Onboard Tech-Sandbox Display: Drivers and Passenger windows run a kid-friendly visual program or diagnostic interface, letting young people interacting with her, view live telemetry, radar point-clouds, and thermal camera feeds to learn how edge AI works.
 * Acoustic Sound Effects Matrix: Program the Dante audio matrix to mix subtle, futuristic sci-fi interface sounds (like activation chimes and data-processing hums) into her voice output during interactions, giving the Maybach a true futuristic feel.
4. Kinetic Personality (The Living Orion)
Because Polaris is tied directly into the CAN bus, she can puppet the vehicle's non-drivetrain components to physically communicate.
 * Greeting Sequence: When you or kids approach, she can unfold the mirrors, pulse the ambient interior lighting, and execute a custom sequence with the LED headlights.
 * Physical Feedback: She can roll windows up and down, adjust the air suspension height to "bow," or light up all exterior lights as she scans, like KITT only the blinker goes all the way around the car.
5. Armor Node security - The 360° mmWave Bubble
By scattering 4 to 6 ESP32 units paired with RD03 modules around the vehicle, you are creating a localized, overlapping micro-Doppler radar field. These sensors can triangulate multi-target movement and detect micro-motions (like a slow, creeping footstep) in zero-visibility conditions.
6. Starlink Wi-Fi CSI Spatial Web
This is where your architecture gets incredibly advanced. By utilizing the ambient Starlink Wi-Fi signals bouncing around the perimeter of the vehicle, those ESP32 nodes can act as Channel State Information (CSI) receivers. When a physical body disrupts those radio waves, Polaris's Burst Receiver on Port 5001 can ingest those subcarrier amplitude shifts. You are literally turning the invisible Wi-Fi field into a volumetric tripwire.
7. Air-Gapped Optical & Thermal Vision
Decoupling the optical feed from the Mercedes factory cameras is the smartest move for security and system stability.
 * Independent HD Cams: Gives you clean front-and-back visual vectors without having to reverse-engineer proprietary automotive video feeds.
 * FLIR Integration: Thermal imaging perfectly compliments the mmWave radar. If an RD03 module detects motion in the pitch black, Polaris can run YOLO bounding boxes on the independent FLIR feed to instantly classify if the heat signature is a human, a bear, or a false positive.
The Mobile Armor Integration Matrix
| Sensor Layer | Tactical Advantage | Polaris Gateway Integration |
|---|---|---|
| ESP32 + RD03 | 360-degree, zero-light motion tracking. | Broadcasts radar_update events via Port 5001 WebSocket. |
| Starlink Wi-Fi CSI | Mass/volumetric disruption detection. | Analyzes subcarrier weights to calculate X/Y spatial positioning. |
| FLIR / HD Cams | Absolute visual and thermal classification. | Feeds raw optical vectors to the YOLO real-time object detection engine. |
FLIR & Spatial Expansion
Adding forward-looking infrared (FLIR) elevates the SUV from a standard vehicle to a military-grade spatial awareness platform.
 * Thermal YOLO Tracking: If you route those thermal feeds through the Burst Receiver on Port 5001, Polaris can run YOLO object detection on heat signatures in pitch black conditions, completely independent of the vehicle's standard optical cameras.
 * The Roving Sentry: The Dell Mainframe remains the brain, and the Mercedes simply becomes a heavily armored, mobile edge device streaming telemetry back to the hive.
