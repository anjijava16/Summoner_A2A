# Agentverse Summoner A2A

A fantasy-themed **Agent-to-Agent (A2A)** multi-agent orchestration system built on **Google Cloud** and the **Google Agent Development Kit (ADK)**. A master "Summoner" agent strategically dispatches elemental familiars — Fire, Water, and Earth — to battle monsters, each with unique attack patterns, spell mechanics, and cooldown enforcement.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Architecture Diagram](#system-architecture-diagram)
- [Agents](#agents)
  - [Summoner Agent (Orchestrator)](#summoner-agent-orchestrator)
  - [Fire Elemental Familiar](#fire-elemental-familiar)
  - [Water Elemental Familiar](#water-elemental-familiar)
  - [Earth Elemental Familiar](#earth-elemental-familiar)
  - [Diagnose Agent](#diagnose-agent)
- [MCP Servers](#mcp-servers)
  - [API Tools MCP Server (Nexus of Whispers)](#api-tools-mcp-server-nexus-of-whispers)
  - [General Tools MCP Server (Arcane Forge)](#general-tools-mcp-server-arcane-forge)
  - [Database Toolbox (Summoner Librarium)](#database-toolbox-summoner-librarium)
- [Supporting Services](#supporting-services)
  - [Fake API Server (Nexus of Whispers API)](#fake-api-server-nexus-of-whispers-api)
  - [Cloud SQL (Familiar Grimoire)](#cloud-sql-familiar-grimoire)
- [A2A Protocol & Cooldown Plugin](#a2a-protocol--cooldown-plugin)
- [Interaction Flow](#interaction-flow)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup & Deployment](#setup--deployment)
- [Configuration Reference](#configuration-reference)

---

## Architecture Overview

The system implements the **Summoner's Concord** workshop — a multi-agent battle simulation where:

1. A **Summoner Agent** (master orchestrator) receives user requests describing monster encounters
2. The Summoner analyzes monster weaknesses and selects the optimal **Elemental Familiar** (Fire, Water, or Earth)
3. The chosen Familiar executes its unique attack pattern using **MCP tools** — database lookups, API calls, and spell amplification
4. A **Cooldown Plugin** enforces 60-second intervals between summons of the same familiar
5. All agents communicate via the **A2A (Agent-to-Agent) protocol** and are deployed as independent **Cloud Run** services

**Key design patterns demonstrated:**

| Pattern | Agent | ADK Type |
|---------|-------|----------|
| Sequential multi-step | Fire Elemental | `SequentialAgent` |
| Parallel execution | Water Elemental | `ParallelAgent` inside `SequentialAgent` |
| Iterative loop | Earth Elemental | `LoopAgent` |
| Remote orchestration | Summoner | `LlmAgent` with `RemoteA2aAgent` sub-agents |

---

## System Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│                         GCP PROJECT                                │
│                                                                    │
│  ┌──────────────── Cloud Run Services ───────────────────────┐     │
│  │                                                           │     │
│  │  ┌────────────────┐    A2A     ┌──────────────────┐       │     │
│  │  │   Summoner     │──────────→ │  Fire Familiar   │       │     │
│  │  │   Agent        │            │  (Sequential)    │       │     │
│  │  │  (LlmAgent)    │──┐        └──────────────────┘       │     │
│  │  │                │  │  A2A    ┌──────────────────┐       │     │
│  │  │  Port 8080     │  └──────→  │  Water Familiar  │       │     │
│  │  │                │     │      │  (Parallel)      │       │     │
│  │  └────────────────┘     │      └──────────────────┘       │     │
│  │         │               │ A2A  ┌──────────────────┐       │     │
│  │         │               └────→ │  Earth Familiar  │       │     │
│  │     Cooldown                   │  (Loop)          │       │     │
│  │     Check ↓                    └──────────────────┘       │     │
│  │  ┌────────────────┐                  │ MCP (SSE)          │     │
│  │  │ Nexus of       │           ┌──────┴──────┐             │     │
│  │  │ Whispers API   │     ┌─────┴─────┐ ┌─────┴──────┐     │     │
│  │  │ (Fake API)     │     │ API Tools │ │ General    │     │     │
│  │  │ /cooldown/*    │     │ MCP       │ │ Tools MCP  │     │     │
│  │  │ /cryosea_*     │     │ (Nexus)   │ │ (Forge)    │     │     │
│  │  └────────────────┘     └───────────┘ └────────────┘     │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                    │
│  ┌──────────────── Cloud SQL (PostgreSQL 16) ────────────────┐     │
│  │  Instance: summoner-librarium-db                          │     │
│  │  Database: familiar_grimoire                              │     │
│  │  Table: abilities (familiar_name, ability_name, damage)   │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                    │
│  ┌──────────────── Artifact Registry ────────────────────────┐     │
│  │  base-familiar:latest  │  summoner-agent:latest           │     │
│  │  api-tools-mcp:latest  │  general-tools-mcp:latest        │     │
│  │  nexus-of-whispers-api:latest                             │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                    │
│  ┌──────────────── Cloud Build Pipelines ────────────────────┐     │
│  │  agent/cloudbuild.yaml          → 3 familiars (parallel)  │     │
│  │  agent/cloudbuild-summoner.yaml → Summoner agent          │     │
│  │  mcp-servers/cloudbuild.yaml    → 2 MCP servers (parallel)│     │
│  │  prerequisite/fake_api/cloudbuild.yaml → Fake API         │     │
│  └───────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Agents

### Summoner Agent (Orchestrator)

**Path:** `agent/summoner/agent.py`
**Type:** `LlmAgent` | **Model:** Gemini 2.5 Flash

The Summoner is the master strategist that:

- Receives monster encounter descriptions from users
- Analyzes monster weaknesses to select the optimal familiar:
  - **Fire** → "Inescapable Reality" / "Revolutionary Rewrite" weaknesses
  - **Water** → "Unbroken Collaboration" weakness
  - **Earth** → "Elegant Sufficiency" weakness
- Delegates to remote Elemental Familiar agents via A2A protocol
- Tracks the last summoned familiar in state to avoid repeat summons
- Handles cooldown failures gracefully when all familiars are unavailable

**Sub-agents** (remote A2A references):
- `fire_familiar` — loaded from `FIRE_URL`
- `water_familiar` — loaded from `WATER_URL`
- `earth_familiar` — loaded from `EARTH_URL`

**Callback:** `save_last_summon_after_tool()` persists the last-used familiar into agent state.

---

### Fire Elemental Familiar

**Path:** `agent/fire/agent.py`
**Type:** `SequentialAgent` — precise, two-step "one-two punch"

**Attack pattern:**

```
Step 1: Scout Agent (Librarian)
  └─→ Queries Cloud SQL via DB Toolbox (ability_damage tool)
  └─→ Retrieves a random Fire ability (inferno_lash: 85, emberstorm: 90, Pyroclasm: 80)

Step 2: Amplifier Agent
  └─→ Calls inferno_resonance tool (General MCP)
  └─→ Multiplies base damage × 3
  └─→ Generates epic battle cry describing the attack
```

**MCP Tools Used:**
- `toolDB` — Database Toolbox (`summoner-librarium` toolset)
- `toolFunction` — General Tools MCP (`inferno_resonance`)

**Example Output:** 85 base → 255 amplified damage

---

### Water Elemental Familiar

**Path:** `agent/water/agent.py`
**Type:** `SequentialAgent` wrapping a `ParallelAgent` — simultaneous multi-pronged assault

**Attack pattern:**

```
Step 1: Channel Agent (ParallelAgent — runs simultaneously)
  ├─→ Nexus Channeler:
  │     ├─→ cryosea_shatter  → 80 damage  (API Tools MCP)
  │     └─→ moonlit_cascade  → 105 damage (API Tools MCP)
  └─→ Forge Channeler:
        └─→ leviathan_surge(20) × 3 = 60 damage (General Tools MCP)

Step 2: Power Merger Agent
  └─→ Parses all results and sums total damage → ~245 damage
```

**MCP Tools Used:**
- `toolFAPI` — API Tools MCP (`cryosea_shatter`, `moonlit_cascade`)
- `toolFunction` — General Tools MCP (`leviathan_surge`)

---

### Earth Elemental Familiar

**Path:** `agent/earth/agent.py`
**Type:** `LoopAgent` (max 2 iterations) — relentless iterative siege with energy accumulation

**Attack pattern:**

```
Iteration 1:
  ├─→ Charging Agent: seismic_charge(1) → 3 energy
  └─→ Check Agent: Reports charging status, calculates potential damage

Iteration 2:
  ├─→ Charging Agent: seismic_charge(3) → 5 energy
  └─→ Check Agent: Releases energy → 5 × ~85 = capped at 300 damage
```

**MCP Tools Used:**
- `toolFunction` — General Tools MCP (`seismic_charge`)

**Energy Rule:** Start at 1, each charge adds +2. Damage = energy × random(80–90), capped at 300.

---

### Diagnose Agent

**Path:** `mcp-servers/diagnose/agent.py`
**Type:** `LlmAgent` — diagnostic/debug orchestrator

Routes requests to two specialist sub-agents:
- **Librarian Agent** — read-only database queries (looking up, querying abilities)
- **Arcane Battlemage Agent** — spell execution (casting, multiplying power)

---

## MCP Servers

Three **Model Context Protocol** servers expose tools to the agents via SSE (Server-Sent Events) transport.

### API Tools MCP Server (Nexus of Whispers)

**Path:** `mcp-servers/api/main.py` | **Endpoint:** `/sse`

| Tool | Action | Damage |
|------|--------|--------|
| `cryosea_shatter()` | POST to Fake API `/cryosea_shatter` | 80 |
| `moonlit_cascade()` | POST to Fake API `/moonlit_cascade` | 105 |

---

### General Tools MCP Server (Arcane Forge)

**Path:** `mcp-servers/general/main.py` | **Endpoint:** `/sse`

| Tool | Action | Effect |
|------|--------|--------|
| `inferno_resonance(base_fire_damage)` | Fire amplifier | `damage × 3` |
| `leviathan_surge(base_water_damage)` | Water amplifier | `damage × 3` |
| `seismic_charge(current_energy)` | Earth energy charge | `energy + 2` |

---

### Database Toolbox (Summoner Librarium)

**Path:** `mcp-servers/db-toolbox/tools.yaml` | **Backend:** Cloud SQL PostgreSQL

| Tool | SQL Query | Purpose |
|------|-----------|---------|
| `lookup-available-ability` | `SELECT ability_name, damage_points FROM abilities WHERE familiar_name = $1` | List all abilities for a familiar |
| `ability-damage` | `SELECT damage_points FROM abilities WHERE ability_name = $1` | Get base damage for a specific ability |

**Database schema:**

```sql
CREATE TABLE abilities (
  id SERIAL PRIMARY KEY,
  familiar_name VARCHAR(50),
  ability_name VARCHAR(50) UNIQUE,
  damage_points INTEGER,
  element VARCHAR(20)
);
```

**Seeded data (Fire Elemental):**

| Ability | Damage | Element |
|---------|--------|---------|
| `inferno_lash` | 85 | Fire |
| `emberstorm` | 90 | Fire |
| `Pyroclasm` | 80 | Fire |

---

## Supporting Services

### Fake API Server (Nexus of Whispers API)

**Path:** `prerequisite/fake_api/fake_api_server.py` | **Cloud Run Service:** `nexus-of-whispers-api`

A lightweight FastAPI server providing spell-casting endpoints and cooldown state management:

| Endpoint | Method | Purpose | Response |
|----------|--------|---------|----------|
| `/` | GET | Health check | `{"message": "...active"}` |
| `/cryosea_shatter` | POST | Ice spell cast | `{"ability": "cryosea_shatter", "damage_points": 80}` |
| `/moonlit_cascade` | POST | Arcane spell cast | `{"ability": "moonlit_cascade", "damage_points": 105}` |
| `/cooldown/{familiar_name}` | GET | Check cooldown status | `{"time": "ISO_timestamp" or null}` |
| `/cooldown/{familiar_name}` | POST | Set cooldown timestamp | 204 No Content |

Cooldown state is stored in an in-memory dictionary.

---

### Cloud SQL (Familiar Grimoire)

- **Instance:** `summoner-librarium-db`
- **Engine:** PostgreSQL 16
- **Database:** `familiar_grimoire`
- **Tier:** `db-g1-small` (Enterprise edition)
- **Region:** `us-central1`
- **User:** `summoner`

Provisioned via `prepare.sh` and populated via `data_setup.sh` / `prerequisite/db_setup.py`.

---

## A2A Protocol & Cooldown Plugin

### A2A Conversion

**Path:** `agent/agent_to_a2a.py`

The `to_a2a()` function converts any ADK `BaseAgent` into a Starlette ASGI web application with full A2A protocol support:

- **Agent Card** published at `{public_url}/.well-known/agent.json` for agent discovery
- **A2A RPC endpoint** for inter-agent messaging
- **In-memory services** for task store, artifacts, sessions, memory, and credentials

### Cooldown Plugin

**Path:** `agent/cooldown_plugin.py`

Implements `BasePlugin.before_agent_callback()` to enforce a 60-second cooldown between summons of the same familiar:

1. **Filter** — Only intercepts agents ending with `_elemental_familiar`
2. **Check** — `GET /cooldown/{agent_name}` to the Fake API
3. **Evaluate** — If last summon was < 60 seconds ago, block with an error message
4. **Update** — `POST /cooldown/{agent_name}` to record the new timestamp
5. **Fail-open** — If the API is unreachable, the familiar is allowed to proceed

---

## Interaction Flow

```
User sends message: "A Glacial Serpent appears! Weakness: Unbroken Collaboration"
    │
    ▼
┌────────────────────────────────────────┐
│           Summoner Agent               │
│  1. Analyzes: "Unbroken Collaboration" │
│  2. Selects: Water Familiar            │
│  3. Checks last_summon state           │
│  4. Dispatches via A2A protocol        │
└──────────────┬─────────────────────────┘
               │ A2A RPC
               ▼
┌────────────────────────────────────────┐
│  CooldownPlugin (before_agent_callback)│
│  GET /cooldown/water_elemental_familiar│
│  → Last summon: 90s ago → ALLOWED      │
│  POST /cooldown/water_elemental_familiar│
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│       Water Familiar (Parallel)        │
│  ┌─────────────────┬────────────────┐  │
│  │ Nexus Channeler │ Forge Channeler│  │
│  │ cryosea: 80     │ leviathan: 60  │  │
│  │ moonlit: 105    │                │  │
│  └────────┬────────┴───────┬────────┘  │
│           └───────┬────────┘           │
│         Power Merger: 245 total        │
└──────────────┬─────────────────────────┘
               │ A2A Response
               ▼
User receives: "Water Familiar unleashes a combined
assault for 245 damage! The Glacial Serpent falls!"
```

---

## Tech Stack

| Category | Technology | Version |
|----------|-----------|---------|
| **AI Framework** | Google Agent Development Kit (ADK) | 1.9.0 |
| **LLM** | Gemini 2.5 Flash (via Vertex AI) | — |
| **Agent Protocol** | Agent-to-Agent (A2A) | — |
| **Tool Protocol** | Model Context Protocol (MCP) | 1.12.4 |
| **Web Framework** | Starlette / FastAPI / Uvicorn | 0.47.2 / 0.116.1 / 0.35.0 |
| **Database** | Cloud SQL PostgreSQL | 16 |
| **DB Connector** | cloud-sql-python-connector + pg8000 | 1.18.3 / 1.31.4 |
| **Container Runtime** | Docker + Cloud Run | — |
| **CI/CD** | Cloud Build | — |
| **Image Registry** | Artifact Registry | — |
| **Language** | Python | 3.12 |
| **Orchestration** | Google Cloud (GCP) | — |

---

## Project Structure

```
agentverse-agents-a2a/
│
├── README.md                           # This file
├── billing-enablement.py               # Links GCP project to billing account
├── init.sh                             # One-time GCP project initialization
├── set_env.sh                          # Environment variable configuration
├── prepare.sh                          # Provisions Cloud SQL & deploys Fake API
├── data_setup.sh                       # Populates database with ability data
│
├── agent/                              # Core agent code
│   ├── Dockerfile                      # Shared Docker image for all agents
│   ├── requirements.txt                # Python dependencies
│   ├── agent_to_a2a.py                 # A2A protocol converter (to_a2a())
│   ├── cooldown_plugin.py              # 60-second cooldown enforcement
│   ├── cloudbuild.yaml                 # Cloud Build: 3 familiars (parallel)
│   ├── cloudbuild-summoner.yaml        # Cloud Build: summoner agent
│   │
│   ├── summoner/                       # Master orchestrator (LlmAgent)
│   │   ├── __init__.py
│   │   └── agent.py
│   ├── fire/                           # Fire Familiar (SequentialAgent)
│   │   ├── __init__.py
│   │   └── agent.py
│   ├── water/                          # Water Familiar (Parallel + Sequential)
│   │   ├── __init__.py
│   │   └── agent.py
│   └── earth/                          # Earth Familiar (LoopAgent)
│       ├── __init__.py
│       └── agent.py
│
├── mcp-servers/                        # MCP tool servers
│   ├── cloudbuild.yaml                 # Cloud Build: 2 MCP servers (parallel)
│   ├── api/                            # Nexus of Whispers MCP (cryosea, moonlit)
│   │   ├── Dockerfile
│   │   ├── main.py
│   │   └── requirements.txt
│   ├── general/                        # Arcane Forge MCP (inferno, leviathan, seismic)
│   │   ├── Dockerfile
│   │   ├── main.py
│   │   └── requirements.txt
│   ├── db-toolbox/                     # Database Toolbox config
│   │   └── tools.yaml
│   └── diagnose/                       # Diagnostic debug agent
│       ├── __init__.py
│       ├── agent.py
│       └── requirements.txt
│
└── prerequisite/                       # Infrastructure prerequisites
    ├── db_setup.py                     # Cloud SQL schema + seed data
    ├── requirements-setup.txt          # DB setup dependencies
    └── fake_api/                       # Fake spell/cooldown API
        ├── Dockerfile
        ├── fake_api_server.py
        ├── cloudbuild.yaml
        └── requirements.txt
```

---

## Prerequisites

- **Google Cloud SDK** (`gcloud`) installed and authenticated
- **Python 3.12+**
- **Docker** (for local builds, optional if using Cloud Build)
- A **GCP Billing Account** (linked automatically via `init.sh`)
- Enabled GCP APIs:
  - Cloud Run
  - Cloud Build
  - Cloud SQL Admin
  - Artifact Registry
  - Vertex AI
  - Cloud Billing

---

## Setup & Deployment

### 1. Initialize GCP Project

```bash
source init.sh
```

This creates a new GCP project (`agentverse-summoner-{random_suffix}`), enables the Billing API, and links your billing account. The project ID is saved to `~/project_id.txt`.

### 2. Set Environment Variables

```bash
source set_env.sh
```

Exports all required configuration: project IDs, regions, database credentials, and service URLs.

### 3. Provision Infrastructure

```bash
source prepare.sh
```

- Creates the Cloud SQL instance (`summoner-librarium-db`, PostgreSQL 16)
- Deploys the Fake API server (`nexus-of-whispers-api`) to Cloud Run

### 4. Populate Database

```bash
source data_setup.sh
```

- Creates the `familiar_grimoire` database and `summoner` user
- Runs `prerequisite/db_setup.py` to create the `abilities` table and seed Fire Elemental data

### 5. Deploy MCP Servers

```bash
cd mcp-servers
gcloud builds submit --config=cloudbuild.yaml
```

Builds and deploys `api-tools-mcp` and `general-tools-mcp` to Cloud Run in parallel.

### 6. Deploy Elemental Familiars

```bash
cd agent
gcloud builds submit --config=cloudbuild.yaml
```

Builds a shared Docker image and deploys `fire-familiar`, `water-familiar`, and `earth-familiar` as separate Cloud Run services.

### 7. Deploy Summoner Agent

```bash
cd agent
gcloud builds submit --config=cloudbuild-summoner.yaml
```

Deploys the `summoner-agent` Cloud Run service, configured with the URLs of all familiar agents.

### 8. Interact

Send a message to the Summoner Agent's endpoint:

```
"A Molten Golem emerges from the volcano! Its weakness: Inescapable Reality!"
```

The Summoner will analyze the weakness, select Fire Familiar, and execute a 3x-amplified attack.

---

## Configuration Reference

| Parameter | Location | Default | Description |
|-----------|----------|---------|-------------|
| Project ID | `init.sh` / `~/project_id.txt` | `agentverse-summoner-{random}` | GCP project identifier |
| Region | `set_env.sh` | `us-central1` | Deployment region |
| DB Instance | `set_env.sh` | `summoner-librarium-db` | Cloud SQL instance name |
| DB Name | `set_env.sh` | `familiar_grimoire` | PostgreSQL database |
| DB User | `set_env.sh` | `summoner` | Database username |
| DB Password | `set_env.sh` | `1234qwer` | Database password |
| Cooldown Period | `cooldown_plugin.py` | 60 seconds | Minimum interval between same-familiar summons |
| LLM Model | Agent files | `gemini-2.5-flash` | Gemini model for LlmAgents |
| Cloud SQL Tier | `prepare.sh` | `db-g1-small` | Compute tier for the database |
| Port | All services | `8080` | Cloud Run service port |

---

## License

This project is part of the Google ADK / A2A demonstration ecosystem.
