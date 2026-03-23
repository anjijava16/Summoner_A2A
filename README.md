# Agentverse — The Summoner's Concord

A multi-agent AI  Google's **ADK (Agent Development Kit)**, **A2A (Agent-to-Agent) Protocol**, **MCP (Model Context Protocol)**, and a turn-based RPG dungeon crawler game — all running serverlessly on **Google Cloud**.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture Diagram](#architecture-diagram)
- [Repository Structure](#repository-structure)
- [Part 1: agentverse-agents-a2a](#part-1-agentverse-agents-a2a)
  - [Setup & Initialization](#setup--initialization)
  - [Elemental Familiar Agents](#elemental-familiar-agents)
  - [MCP Servers (Tool Providers)](#mcp-servers-tool-providers)
  - [Cooldown System](#cooldown-system)
  - [Cloud Build & Deployment](#cloud-build--deployment)
- [Part 2: agentverse-ui-api-dungeon](#part-2-agentverse-ui-api-dungeon)
  - [Backend (FastAPI)](#backend-fastapi)
  - [Frontend (React)](#frontend-react)
  - [Game Mechanics](#game-mechanics)
  - [Agent Integration in Combat](#agent-integration-in-combat)
- [Complete Game Loop](#complete-game-loop)
- [Google Cloud Services](#google-cloud-services)
- [A2A Protocol Explained](#a2a-protocol-explained)
- [How to Run](#how-to-run)

---

## Project Overview

The project consists of **two interconnected projects**:

| Project | Purpose |
|---------|---------|
| **agentverse-agents-a2a** | Multi-agent backend — 4 AI familiars (Fire, Water, Earth, Summoner) deployed as independent Cloud Run services communicating via A2A, with MCP servers providing tools |
| **agentverse-ui-api-dungeon** | Turn-based RPG game — React frontend + FastAPI backend where players fight bosses using their AI agents' abilities, with quiz-based damage mechanics |

**Core Technologies:**
- **Google ADK** — Agent orchestration framework (LlmAgent, SequentialAgent, ParallelAgent, LoopAgent)
- **A2A Protocol** — Inter-agent communication via `.well-known/agent.json` cards and RPC endpoints
- **MCP (Model Context Protocol)** — Standardized tool exposure via SSE transport
- **Gemini 2.5 Flash** — LLM backbone via Vertex AI
- **Cloud Run** — Serverless deployment for all services
- **Cloud SQL (PostgreSQL)** — Ability database ("Librarium of Knowledge")

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      agentverse-ui-api-dungeon                             │
│  ┌──────────────┐    ┌──────────────────────────────────────────┐  │
│  │  React App   │───▶│  FastAPI Backend (/api)                  │  │
│  │  - HomePage  │    │  - /miniboss/start   - /ultimateboss     │  │
│  │  - Combat    │    │  - /game/{id}        - /game/{id}/action │  │
│  │  - Quizzes   │    │                                          │  │
│  └──────────────┘    │  HeroicScribeAgent (parses agent output) │  │
│                      └────────────────┬─────────────────────────┘  │
└───────────────────────────────────────┼─────────────────────────────┘
                                        │ A2A Protocol
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     agentverse-agents-a2a                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Summoner Agent (Orchestrator)                   │   │
│  │  LlmAgent + RemoteA2aAgent sub-agents                       │   │
│  │  Analyzes boss weakness → selects best familiar              │   │
│  └───────┬──────────────┬──────────────┬───────────────────────┘   │
│          │ A2A          │ A2A          │ A2A                        │
│          ▼              ▼              ▼                            │
│  ┌──────────────┐ ┌──────────┐ ┌──────────────┐                   │
│  │ Fire Familiar│ │  Water   │ │ Earth Familiar│                   │
│  │ Sequential:  │ │ Parallel:│ │ Loop (2x):    │                   │
│  │ scout→amplify│ │ 3 spells │ │ charge→check  │                   │
│  └──────┬───────┘ └────┬─────┘ └──────┬────────┘                  │
│         │              │              │                             │
│         ▼              ▼              ▼                             │
│  ┌──────────────────────────────────────────────┐                  │
│  │           MCP Servers (Tool Providers)        │                  │
│  │  general-tools-mcp: inferno_resonance,        │                  │
│  │    leviathan_surge, seismic_charge             │                  │
│  │  api-tools-mcp: cryosea_shatter,              │                  │
│  │    moonlit_cascade                             │                  │
│  └──────────────────────────────────────────────┘                  │
│              │                                                      │
│              ▼                                                      │
│  ┌───────────────────┐  ┌──────────────────────┐                   │
│  │ Nexus of Whispers │  │ Cloud SQL (Librarium) │                  │
│  │ Cooldown API +    │  │ abilities table       │                  │
│  │ External spells   │  │ (familiar, dmg, elem) │                  │
│  └───────────────────┘  └──────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
Google_A2A_exp/
├── README.md                          ← You are here
│
├── agentverse-agents-a2a/              ← Multi-agent backend
│   ├── init.sh                        # GCP project creation + billing
│   ├── set_env.sh                     # Environment variable definitions
│   ├── prepare.sh                     # Cloud SQL + fake API provisioning
│   ├── data_setup.sh                  # Database population
│   ├── billing-enablement.py          # Billing account linker
│   │
│   ├── agent/                         # Agent implementations
│   │   ├── agent_to_a2a.py            # A2A wrapper (exposes agent as A2A service)
│   │   ├── cooldown_plugin.py         # Shared cooldown plugin
│   │   ├── cloudbuild.yaml            # Deploys fire/water/earth to Cloud Run
│   │   ├── cloudbuild-summoner.yaml   # Deploys summoner to Cloud Run
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── earth/agent.py             # LoopAgent: seismic charge accumulator
│   │   ├── fire/agent.py              # SequentialAgent: DB scout → amplifier
│   │   ├── water/agent.py             # ParallelAgent: 3-spell fan-out → merge
│   │   └── summoner/agent.py          # LlmAgent: orchestrator with RemoteA2aAgents
│   │
│   ├── mcp-servers/                   # MCP tool servers
│   │   ├── cloudbuild.yaml            # Deploys MCP servers to Cloud Run
│   │   ├── api/main.py                # cryosea_shatter, moonlit_cascade
│   │   ├── general/main.py            # inferno_resonance, leviathan_surge, seismic_charge
│   │   ├── db-toolbox/tools.yaml      # Cloud SQL ability queries (YAML-based)
│   │   └── diagnose/agent.py          # Master agent combining DB + spell tools
│   │
│   └── prerequisite/
│       ├── fake_api/fake_api_server.py # Nexus of Whispers: cooldown + spell APIs
│       └── db_setup.py                # Populates abilities table in Cloud SQL
│
└── agentverse-ui-api-dungeon/                ← RPG game application
    ├── Dockerfile                     # Multi-stage: React build + Python backend
    ├── cloudbuild.yaml                # Cloud Build deployment
    ├── start.sh                       # Entrypoint: uvicorn on port 8000
    ├── designdoc.md                   # Full application design specification
    ├── spec.md                        # Game design spec
    ├── balance.md                     # Combat balance tuning numbers
    │
    ├── backend/
    │   ├── requirements.txt
    │   └── app/
    │       ├── main.py                # FastAPI app + CORS + static files
    │       ├── api.py                 # Game endpoints (start, action, state)
    │       ├── crud.py                # Game logic, boss attacks, A2A agent calls
    │       ├── models.py              # Pydantic models (GameState, Player, Boss, Quiz)
    │       ├── single_agent.py        # HeroicScribeAgent—wraps RemoteA2aAgent
    │       ├── guardian_quizzes.py     # 20 GCP infrastructure quizzes
    │       ├── scholar_quizzes.py      # 20 RAG/data pipeline quizzes
    │       ├── shadowblade_quizzes.py  # 17 Gemini CLI/MCP/ADK quizzes
    │       └── summoner_quizzes.py     # 20 A2A protocol/agent quizzes
    │
    └── frontend/
        ├── package.json
        ├── public/assets/images/      # Boss & player sprites, attack effects
        └── src/
            ├── App.js                 # Router + global state management
            ├── styles.css             # Combat animations & layout
            ├── pages/
            │   ├── HomePage.js        # Main menu
            │   ├── MiniBossPage.js    # Single-player boss setup
            │   └── UltimateBossPage.js# 4-player ultimate boss setup
            ├── components/
            │   ├── CombatScreen.js    # Main battle loop UI
            │   ├── QuizModal.js       # Draggable quiz overlay
            │   ├── PreCombatScreen.js # Pre-fight confirmation
            │   ├── Boss.js            # Boss sprite + HP bar
            │   ├── Player.js          # Player sprite + HP bar
            │   ├── HpBar.js           # HP progress bar
            │   ├── DialogBubble.js    # Speech bubble component
            │   ├── DamageIndicator.js # Floating damage numbers
            │   ├── AttackEffect.js    # Visual attack animations
            │   ├── StatusDisplay.js   # Status message bar
            │   └── MainMenu.js        # Menu component
            └── contexts/
                └── BackgroundContext.js# Global background image state
```

---

## Part 1: agentverse-agents-a2a

### Setup & Initialization

The setup is a 4-script pipeline:

| Script | What it does |
|--------|-------------|
| `init.sh` | Creates GCP project, enables billing via `billing-enablement.py`, saves project ID to `~/project_id.txt` |
| `set_env.sh` | **Must be sourced** (`source ./set_env.sh`). Exports all env vars: `PROJECT_ID`, `REGION`, database creds, service URLs, A2A base URL |
| `prepare.sh` | Provisions Cloud SQL instance (`summoner-librarium-db`) + deploys Nexus of Whispers API to Cloud Run |
| `data_setup.sh` | Creates DB/user in Cloud SQL, runs `db_setup.py` to populate the `abilities` table |

### Elemental Familiar Agents

Each familiar is a distinct ADK agent architecture showcasing different orchestration patterns:

#### Fire Elemental — `SequentialAgent`
```
scout_agent → amplifier_agent
```
- **scout_agent**: Queries Cloud SQL via Toolbox to find fire abilities and their base damage
- **amplifier_agent**: Calls `inferno_resonance` MCP tool (base × 3) and crafts a battle narrative
- Example flow: DB lookup finds `inferno_lash` (85 dmg) → amplified to 255 damage

#### Water Elemental — `ParallelAgent` + `SequentialAgent`
```
channel_agent (parallel) → power_merger
  ├─ nexus_channeler: cryosea_shatter (80) + moonlit_cascade (105)
  └─ forge_channeler: leviathan_surge (20 × 3 = 60)
```
- Three spells execute simultaneously via ParallelAgent
- `power_merger` sums all damage values into a combined attack (e.g., 245 total)

#### Earth Elemental — `LoopAgent`
```
Loop 2x: charging_agent → check_agent
```
- **charging_agent**: Calls `seismic_charge` tool (current_energy + 2 each iteration)
- **check_agent**: Reports charge level and potential damage (energy × 80-90, cap 300)
- Iterative power buildup: 1 → 3 → 5 energy, then unleashes

#### Summoner — `LlmAgent` (Orchestrator)
```
master_summoner_agent
  ├─ RemoteA2aAgent → fire-familiar (Cloud Run)
  ├─ RemoteA2aAgent → water-familiar (Cloud Run)
  └─ RemoteA2aAgent → earth-familiar (Cloud Run)
```
- Analyzes boss weakness from description to select the best familiar
- Enforces 60-second cooldown per familiar
- Tracks `last_summon` in conversation state to avoid consecutive repeats
- Boss weakness mapping:
  - *Inescapable Reality* → Fire
  - *Revolutionary Rewrite* → Fire
  - *Elegant Sufficiency* → Earth
  - *Unbroken Collaboration* → Water

### MCP Servers (Tool Providers)

MCP servers expose tools via Server-Sent Events (SSE) at `/sse` endpoints:

| Server | Tools | Purpose |
|--------|-------|---------|
| **general-tools-mcp** | `inferno_resonance(base)` → base × 3 | Fire damage amplifier |
| | `leviathan_surge(base)` → base × 3 | Water damage amplifier |
| | `seismic_charge(energy)` → energy + 2 | Earth energy accumulator |
| **api-tools-mcp** | `cryosea_shatter()` → 80 dmg | Water external spell (calls Nexus API) |
| | `moonlit_cascade()` → 105 dmg | Water external spell (calls Nexus API) |
| **db-toolbox** (YAML) | `lookup-available-ability(familiar_name)` | SQL: SELECT abilities by familiar |
| | `ability-damage(ability_name)` | SQL: SELECT damage by ability name |

### Cooldown System

Each familiar has a **60-second cooldown** enforced via the Nexus of Whispers API:

```
Before Agent Executes:
  1. GET /cooldown/{agent_name} → returns last_used timestamp
  2. If (now - last_used) < 60 seconds → REJECT with "exhausted" message
  3. If available → POST /cooldown/{agent_name} with current timestamp
  4. Agent proceeds normally
```

Implemented as `CoolDownPlugin` (shared) or `check_cool_down` callback (per-agent), configured via `before_agent_callback` on the root agent.

### Cloud Build & Deployment

| Config | Deploys |
|--------|---------|
| `agent/cloudbuild.yaml` | `fire-familiar`, `water-familiar`, `earth-familiar` (parallel) |
| `agent/cloudbuild-summoner.yaml` | `summoner-agent` (needs familiar URLs) |
| `mcp-servers/cloudbuild.yaml` | `api-tools-mcp`, `general-tools-mcp` |
| `prerequisite/fake_api/cloudbuild.yaml` | `nexus-of-whispers-api` |

All services deploy to **Cloud Run** in `us-central1` with `min-instances=1`.

---

## Part 2: agentverse-ui-api-dungeon

### Backend (FastAPI)

**Endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/miniboss/start` | Start 1-player fight (class + boss + agent URL) |
| POST | `/api/ultimateboss/start` | Start 4-player party fight (4 agent URLs) |
| GET | `/api/game/{game_id}` | Fetch game state (auto-processes boss turn) |
| POST | `/api/game/{game_id}/action` | Submit quiz answer (applies damage) |
| GET/PUT | `/api/config` | Read/update game balance settings |

**Key Backend Components:**

- **`single_agent.py`** — Creates a `SequentialAgent` per player:
  - Step 1: `RemoteA2aAgent` calls the player's deployed agent (e.g., `fire-familiar.run.app`)
  - Step 2: `HeroicScribeAgent` (Gemini 2.5 Flash) parses the narrative into `{"damage_point": int, "message": string}`

- **`crud.py`** — Game logic layer:
  - 7 mini-bosses with unique weaknesses and dialogue phrases
  - Boss attack generation with thematic messages
  - A2A agent invocation with response parsing
  - Quiz selection from class-specific question banks

- **`models.py`** — Pydantic models: `GameState`, `Player`, `Boss`, `Quiz`, `Config`

### Frontend (React)

**Routing:**
| Path | Component | Purpose |
|------|-----------|---------|
| `/` | `HomePage` | Main menu with Mini-Boss / Ultimate Boss options |
| `/mini-boss` | `MiniBossPage` | Enter A2A endpoint, auto-detect class, random boss |
| `/ultimate-boss` | `UltimateBossPage` | Enter 4 agent endpoints |
| `/pre-combat` | `PreCombatScreen` | Preview boss/player stats before fight |
| `/combat` | `CombatScreen` | Main battle loop with animations |

**CombatScreen Features:**
- Animated boss attacks (shake, pulsate, desaturate, flip effects cycling every 6 seconds)
- Boss dialogue cycling from predefined phrase lists
- Draggable `QuizModal` (via react-draggable) with 3 answer choices
- Floating damage numbers via `DamageIndicator`
- HP bars with color coding
- Game-over screen with victory/defeat message

### Game Mechanics

**Player Classes:**

| Class | HP | Role | Quiz Topics | Turns (Ultimate) |
|-------|----|------|-------------|-------------------|
| Shadowblade | 500 | DPS | Gemini CLI, MCP, ADK | 5 |
| Scholar | 450 | Mid-DPS | RAG, BigQuery, pgvector | 3 |
| Guardian | 950 | Tank | Cloud Build, Cloud Run, IAM | 2 |
| Summoner | 400 | Glass Cannon | A2A, Agent architecture | 2 |

**Mini-Bosses (7):**

| Boss | Weakness | HP Range |
|------|----------|----------|
| Procrastination | Inescapable Reality | 600-800 |
| Hype | Inescapable Reality | 600-800 |
| Dogma | Revolutionary Rewrite | 600-800 |
| Legacy | Revolutionary Rewrite | 600-800 |
| Perfectionism | Elegant Sufficiency | 600-800 |
| Obfuscation | Elegant Sufficiency | 600-800 |
| Apathy | Unbroken Collaboration | 600-800 |

**Ultimate Boss:** Mergepocalypse (3500 HP, weak to all 4 strategies)

**Combat Flow:**
1. Boss attacks (AoE in ultimate, single-target in mini)
2. Player's A2A agent generates a narrative response
3. `HeroicScribeAgent` parses response → damage value
4. Class-specific quiz appears (GCP knowledge questions)
5. Correct answer → full damage to boss; Wrong → half damage
6. Repeat until boss HP ≤ 0 (win) or player(s) HP ≤ 0 (lose)

### Agent Integration in Combat

```
Boss attacks → attack description sent to player's A2A agent
                         │
                         ▼
            ┌─────────────────────────┐
            │ RemoteA2aAgent          │
            │ (calls deployed agent)  │
            │ e.g., fire-familiar     │
            │ Returns: narrative text │
            └────────────┬────────────┘
                         │
                         ▼
            ┌─────────────────────────┐
            │ HeroicScribeAgent       │
            │ (Gemini 2.5 Flash)      │
            │ Parses → JSON:          │
            │ {damage_point, message} │
            └────────────┬────────────┘
                         │
                         ▼
            ┌─────────────────────────┐
            │ Quiz Selection          │
            │ Random class-specific   │
            │ question + agent damage │
            │ → sent to frontend      │
            └─────────────────────────┘
```

---

## Complete Game Loop

**Mini-Boss Fight Example (Shadowblade vs Procrastination):**

1. **Frontend** → User enters agent URL (`shadowblade-agent.run.app`), class auto-detected
2. **Backend** → Creates game: Boss HP 600-800, Player HP 500, turn order `[boss, player, player]`
3. **Boss Turn** → Generates attack: *"Procrastination looms... dealing 120 damage"*
4. **Player Turn** → `RemoteA2aAgent` calls Shadowblade agent → receives narrative
5. **Scribe Agent** → Parses to `{"damage_point": 95, "message": "I strike with fury!"}`
6. **Quiz** → Random Shadowblade quiz (Gemini CLI / MCP topic) with 95 damage attached
7. **User answers** → Correct: 95 damage to boss. Wrong: 47 damage. Boss HP decreases
8. **Repeat** until boss or player reaches 0 HP

**Note:** Shadowblade/Scholar get 2 consecutive player turns per cycle (`[boss, player, player]`).

---

## Google Cloud Services

| Service | Usage |
|---------|-------|
| **Cloud Run** | Hosts all agents, MCP servers, game backend, Nexus API |
| **Cloud SQL (PostgreSQL)** | `familiar_grimoire` database with abilities table |
| **Cloud Build** | CI/CD for building Docker images and deploying to Cloud Run |
| **Artifact Registry** | Docker image storage (`agentverse-repo`) |
| **Vertex AI (Gemini 2.5 Flash)** | LLM backbone for all agent reasoning |
| **Cloud IAM** | Service account permissions for agent-to-service communication |
| **Cloud Billing API** | Automated billing account linking during setup |

---

## A2A Protocol Explained

**A2A (Agent-to-Agent)** is a standardized protocol for independent agent services to communicate:

```
Agent A                                  Agent B (Cloud Run)
   │                                         │
   ├── GET /.well-known/agent.json ──────────▶│  (Agent Card: name, capabilities, RPC URL)
   │◀─────────────────────────────────────────┤
   │                                         │
   ├── POST /rpc ────────────────────────────▶│  (Send message/task)
   │   {message: "Attack the monster"}       │
   │◀─────────────────────────────────────────┤  (Response with narrative)
   │   {result: "I unleash inferno!"}        │
```

In this project:
- **`agent_to_a2a.py`** wraps any ADK agent into an A2A-compatible Starlette application
- **`RemoteA2aAgent`** (from `google-adk`) connects to remote A2A services as sub-agents
- The **Summoner** orchestrates Fire/Water/Earth via A2A
- The **Dungeon backend** calls player agents via A2A through `HeroicScribeAgent`

---

## How to Run

### Prerequisites
- Google Cloud project with billing enabled
- `gcloud` CLI authenticated
- Python 3.11+, Node.js 20+

### Setup (agentverse-agents-a2a)
```bash
cd agentverse-agents-a2a

# 1. Create GCP project and enable billing
./init.sh

# 2. Set environment variables (MUST source, not execute)
source ./set_env.sh

# 3. Provision infrastructure (Cloud SQL + Nexus API)
./prepare.sh

# 4. Populate database
./data_setup.sh

# 5. Deploy MCP servers
gcloud builds submit mcp-servers/ --config mcp-servers/cloudbuild.yaml \
  --substitutions=...

# 6. Deploy familiar agents
gcloud builds submit agent/ --config agent/cloudbuild.yaml \
  --substitutions=...

# 7. Deploy summoner (after familiars are up)
gcloud builds submit agent/ --config agent/cloudbuild-summoner.yaml \
  --substitutions=...
```

### Local Development (agentverse-ui-api-dungeon)
```bash
cd agentverse-ui-api-dungeon

# Backend
cd backend && pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000

# Frontend (separate terminal)
cd frontend && npm install && npm start
```

### Local Agent Testing
```bash
cd agentverse-agents-a2a/mcp-servers
source .venv/bin/activate
# or use .venv/bin/adk directly
.venv/bin/adk run earth
```
