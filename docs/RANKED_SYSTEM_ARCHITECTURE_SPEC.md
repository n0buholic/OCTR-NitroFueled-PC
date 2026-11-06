# TECHNICAL SPECIFICATION & IMPLEMENTATION GUIDE: CTR NITRO-FUELED NATIVE RANKED SYSTEM (PC PORT)

## 1. SYSTEM OVERVIEW & MULTI-LANGUAGE ARCHITECTURE

This document provides the complete technical specification for embedding a native Competitive Ranked System into the Crash Team Racing Nitro-Fueled (CTR NF) PC Port / PS4 Native Mod.

### Tech Stack Breakdown:

* **Backend Server (Go)**: Central authority handling user profiles, matchmaking queues (Solo FFA, 4v4, 2v2v2v2), rating calculations (MMR), database persistence, and API endpoints.

* **Client Native Injector (Native C)**: In-game memory reader/hooker that detects race end events, Lap USF tracking, position triggers, DNF/DC status, and UI rendering hooks.

* **Client Bridge / Launcher (Python)**: Local client process that communicates with the C native DLL/mod via IPC (cTypes/Sockets) and handles REST/WebSocket API authentication & state synchronization with the Go Backend Server.

## 2. DATABASE SCHEMA DESIGN (PostgreSQL / Relational)

### 2.1 Entity Relationship Overview

```text
[ players ] ───────< [ matches ] >─────── [ match_participants ]
```

### 2.2 Table Definitions

#### `players`
Stores core user identity, rating counters, and accumulated stats.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID / BIGINT` | `PRIMARY KEY` | Unique player identifier. |
| `display_name` | `VARCHAR(32)` | `NOT NULL, UNIQUE` | In-game handle. |
| `mmr_normal` | `INT` | `DEFAULT 1000, NOT NULL` | Rating for Normal Mode (Solo & Team). |
| `mmr_itemless` | `INT` | `DEFAULT 1000, NOT NULL` | Rating for Itemless Mode (Solo & Team). |
| `placements_normal` | `SMALLINT` | `DEFAULT 5, NOT NULL` | Remaining unranked placement matches for Normal. |
| `placements_itemless` | `SMALLINT` | `DEFAULT 5, NOT NULL` | Remaining unranked placement matches for Itemless. |
| `created_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Account creation timestamp. |
| `updated_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Profile update timestamp. |

#### `matches`
Logs every completed match session.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID / BIGINT` | `PRIMARY KEY` | Unique match identifier. |
| `track_id` | `VARCHAR(64)` | `NOT NULL` | Map identifier (e.g., `electron_avenue`). |
| `race_mode` | `SMALLINT` | `NOT NULL` | `0` = NORMAL, `1` = ITEMLESS. |
| `match_type` | `SMALLINT` | `NOT NULL` | `0` = SOLO_FFA, `1` = TEAM_4V4, `2` = TEAM_2V2V2V2. |
| `avg_lobby_mmr` | `INT` | `NOT NULL` | Calculated lobby average before race start. |
| `created_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Race timestamp. |

#### `match_participants`
Junction table tracking per-player results and delta outcomes per match.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `match_id` | `UUID / BIGINT` | `FOREIGN KEY (matches.id)` | Composite Primary Key (1/2). |
| `player_id` | `UUID / BIGINT` | `FOREIGN KEY (players.id)` | Composite Primary Key (2/2). |
| `finish_position` | `SMALLINT` | `NOT NULL` | Finishing order ($1 \dots 8$). |
| `team_id` | `SMALLINT` | `DEFAULT 0` | Assigned team ($0$ for Solo, $1\dots 4$ for Teams). |
| `is_dnf` | `BOOLEAN` | `DEFAULT FALSE` | Did Not Finish flag. |
| `is_dc` | `BOOLEAN` | `DEFAULT FALSE` | Disconnected / Rage Quit flag. |
| `full_lap_usf` | `BOOLEAN` | `DEFAULT FALSE` | Maintained Blue Fire full lap bonus trigger. |
| `mmr_before` | `INT` | `NOT NULL` | Rating prior to match calculation. |
| `mmr_delta` | `INT` | `NOT NULL` | Net rating gain/loss from this match. |

---

## 3. CORE RULES & STRUCT DEFINITIONS

### Core Architecture Rules:

1. **EXACTLY TWO CORE MMR RATINGS**:

   * **NORMAL MMR (`mmr_normal`)**: Handles all item-enabled races (Solo FFA, Team 4v4, Team 2v2v2v2).

   * **ITEMLESS MMR (`mmr_itemless`)**: Handles all pure racing / time-trial style races without items (Solo FFA, Team 4v4, Team 2v2v2v2).

2. **UNIFIED PLAYLIST SYSTEM**: Solo or Party/Team (4v4 or 2v2v2v2) matches affect either **NORMAL** or **ITEMLESS** rating. No separate team MMR is stored.

## 4. TIER & VISUAL SPECIFICATIONS

### 4.1 NORMAL Mode (Items On)

| Tier | Name | MMR Range | Visual Icon & Frame Specification | 
| ----- | ----- | ----- | ----- | 
| **1** | **Bronze Wumpa** | $0 - 499$ | Wooden Wumpa fruit sprite with a matte bronze rim. | 
| **2** | **Silver Nitro** | $500 - 999$ | Glowing green Nitro Vial encased in silver chrome plating. | 
| **3** | **Gold Mask** | $1000 - 1499$ | Dual Aku Aku / Uka Uka wooden mask emblem with gold leaf border. | 
| **4** | **Platinum Trophy** | $1500 - 1999$ | High-gloss platinum cup icon with cyan particle sparkles. | 
| **5** | **Aku Master** | $2000 - 2499$ | Invulnerable Aku Aku mask with floating animated feathers. | 
| **6** | **Uka Legend** | $2500 - 2999$ | Dark Uka Uka mask encircled by a purplish-red flame aura. | 
| **7** | **Chaos Grand Prix** | $3000+$ *(Top 100)* | Animated Velo Crown with revolving stars & golden particle aura. | 

### 4.2 ITEMLESS Mode (Pure Racing)

| Tier | Name | MMR Range | Visual Icon & Frame Specification | 
| ----- | ----- | ----- | ----- | 
| **1** | **Sapphire Relic** | $0 - 499$ | Deep blue Sapphire Relic with steel mechanical border. | 
| **2** | **Gold Relic** | $500 - 999$ | Polished Gold Relic badge with ticking stopwatch markings. | 
| **3** | **Platinum Relic** | $1000 - 1499$ | Platinum Relic emblem with animated cyan clockwork gears. | 
| **4** | **Sacred Fire** | $1500 - 1999$ | Bright orange Sacred Fire flame behind a speedometer crest. | 
| **5** | **Ultimate Sacred Fire** | $2000 - 2499$ | Blue Fire (USF) exhaust flame with hyper-speed trail effect. | 
| **6** | **Time Crusher** | $2500 - 2999$ | Shattered hourglass motif emitting pulsing temporal energy. | 
| **7** | **Oxide Overlord** | $3000+$ *(Top 100)* | Emperor Nitros Oxide silhouette framed by a hyper-drive aura. | 

## 5. MATHEMATICAL FORMULAS & DNF MATRIX

### 5.1 Solo FFA Delta Formula

$$
\Delta\text{MMR}_i = \text{BasePoints}(P_i) + K \cdot (\text{AvgLobbyMMR} - \text{MMR}_i)
$$

* **Base Points:** P1 $+45$, P2 $+30$, P3 $+15$, P4 $+5$, P5 $-5$, P6 $-15$, P7 $-25$, P8 $-35$.

* **K-Factor:** $0.05$ (Standard), $0.20$ (Placement Matches).

### 5.2 Team Match Scoring Formula

* **Driver Points**: Pos 1 (10pt), Pos 2 (8pt), Pos 3 (6pt), Pos 4 (5pt), Pos 5 (4pt), Pos 6 (3pt), Pos 7 (2pt), Pos 8 (1pt).

* **4v4 Format**: Winner $+30 \text{ MMR}$, Loser $-30 \text{ MMR}$.

* **2v2v2v2 Format**: Rank 1 $+35$, Rank 2 $+15$, Rank 3 $-15$, Rank 4 $-35$.

### 5.3 DNF & Disconnect Matrix

| Rule | Mode NORMAL | Mode ITEMLESS | 
| ----- | ----- | ----- | 
| **Post-Finish Timer** | **30 Seconds** | **20 Seconds** | 
| **DNF Penalty** | $-10 \text{ MMR}$ | $-20 \text{ MMR}$ | 
| **Disconnect Penalty (Solo)** | $-40 \text{ MMR}$ | $-60 \text{ MMR}$ | 
| **Disconnect Penalty (Team)** | $-50 \text{ MMR}$ | $-70 \text{ MMR}$ | 
| **USF Lap Bonus** | N/A | $+5 \text{ MMR}$ | 

## 6. AGENT INTEGRATION & EXECUTION STEPS

### 6.1 Native C Hook Specifications (`Client Mod`)

* Intercept in-game telemetry pointers at race conclusion.

* Capture player finish position, DNF status, disconnect triggers, and full-lap USF boolean flags.

* Expose binary structs for local IPC serialization.

### 6.2 Python Client Bridge Specifications (`Client Launcher`)

* Load Native C DLL/SO via dynamic binding interface (`ctypes` or direct IPC socket).

* Periodically poll or handle events for completed race payloads.

* Transform telemetry into structured JSON payloads and forward to Go Backend API via REST/WebSocket (`/api/v1/ranked/process-race`).

### 6.3 Go Backend Specifications (`Central Server`)

* Host endpoint handler for match resolution.

* Query existing database profiles (`players` table) to extract `mmr_normal` or `mmr_itemless`.

* Execute MMR calculation algorithm based on queue mode (Solo FFA vs Team 4v4 / 2v2v2v2).

* Persist new entries to `matches` and `match_participants` tables, commit updated player MMRs, and return JSON summary to client callers.