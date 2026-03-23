# Agentverse Dungeon — Backend Deep Dive

The FastAPI backend that powers the turn-based RPG dungeon crawler. It manages game state, orchestrates A2A agent calls, generates quizzes, and serves the React frontend as static files.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Directory Structure](#directory-structure)
- [Module-by-Module Breakdown](#module-by-module-breakdown)
  - [main.py — Application Entry Point](#mainpy--application-entry-point)
  - [models.py — Pydantic Data Models](#modelspy--pydantic-data-models)
  - [api.py — Route Handlers & Game Logic](#apipy--route-handlers--game-logic)
  - [crud.py — Data Layer & Game Configuration](#crudpy--data-layer--game-configuration)
  - [single_agent.py — A2A Agent Integration (Current)](#single_agentpy--a2a-agent-integration-current)
  - [single_agent_old.py — A2A Agent Integration (Legacy)](#single_agent_oldpy--a2a-agent-integration-legacy)
  - [Quiz Modules](#quiz-modules)
- [API Reference](#api-reference)
- [Request / Response Lifecycle](#request--response-lifecycle)
  - [Starting a Mini-Boss Fight](#starting-a-mini-boss-fight)
  - [Starting an Ultimate Boss Fight](#starting-an-ultimate-boss-fight)
  - [Boss Turn Processing](#boss-turn-processing)
  - [Player Action Submission](#player-action-submission)
- [Agent Pipeline — How A2A Calls Work](#agent-pipeline--how-a2a-calls-work)
  - [The SequentialAgent Pattern](#the-sequentialagent-pattern)
  - [RemoteA2aAgent — Calling Deployed Agents](#remotea2aagent--calling-deployed-agents)
  - [HeroicScribeAgent — LLM-as-JSON-Parser](#heroicscribeagent--llm-as-json-parser)
  - [Event Stream Processing](#event-stream-processing)
  - [single_agent.py vs single_agent_old.py](#single_agentpy-vs-single_agent_oldpy)
- [Combat Engine Deep Dive](#combat-engine-deep-dive)
  - [Turn Order System](#turn-order-system)
  - [Damage Calculation](#damage-calculation)
  - [Game Over Conditions](#game-over-conditions)
  - [Boss Attack Generation](#boss-attack-generation)
  - [Quiz Selection & Damage Multiplier](#quiz-selection--damage-multiplier)
- [Boss Lore & Weakness System](#boss-lore--weakness-system)
- [Player Class Balance](#player-class-balance)
- [Resilience & Fallback Strategies](#resilience--fallback-strategies)
- [Data Model Relationships](#data-model-relationships)
- [CORS & Security Configuration](#cors--security-configuration)
- [Running Locally](#running-locally)
- [Key Dependencies](#key-dependencies)
- [Design Decisions & Trade-offs](#design-decisions--trade-offs)

---

## Architecture Overview

```
                    ┌──────────────────────────────────────────────┐
                    │            FastAPI Application               │
                    │                                              │
   HTTP Request ──▶ │  main.py                                     │
                    │    ├─ CORSMiddleware                         │
                    │    ├─ /api/* ──▶ api.py (APIRouter)          │
                    │    └─ /* ──▶ StaticFiles (React build)       │
                    │                                              │
                    │  api.py                                      │
                    │    ├─ POST /miniboss/start                   │
                    │    ├─ POST /ultimateboss/start               │
                    │    ├─ GET  /game/{game_id}                   │
                    │    ├─ POST /game/{game_id}/action            │
                    │    └─ GET|PUT /config                        │
                    │         │                                    │
                    │         ▼                                    │
                    │  crud.py                                     │
                    │    ├─ game_db: Dict[str, GameState]   (mem)  │
                    │    ├─ GAME_CONFIG, BOSS_WEAKNESSES           │
                    │    ├─ BOSS_DIALOGUES (14-15 phrases/boss)    │
                    │    ├─ mock_boss_attack_agent()               │
                    │    ├─ mock_player_a2a_agent()  ──────────┐   │
                    │    └─ mock_damage_quiz_agent()            │   │
                    │                                          │   │
                    │  single_agent.py                         │   │
                    │    ├─ create_heroic_action_agent() ◀─────┘   │
                    │    └─ process_player_action()                 │
                    │         │                                    │
                    │         ▼                                    │
                    │  ┌─────────────────────────────┐             │
                    │  │  InMemoryRunner              │             │
                    │  │  SequentialAgent:             │             │
                    │  │   1. RemoteA2aAgent ─────────┼─── A2A ──▶ Cloud Run
                    │  │   2. HeroicScribeAgent (LLM) │             │
                    │  └─────────────────────────────┘             │
                    │                                              │
                    │  models.py                                   │
                    │    GameState, Player, Boss, Quiz, Config     │
                    │                                              │
                    │  *_quizzes.py (4 files)                      │
                    │    shadowblade, scholar, guardian, summoner   │
                    └──────────────────────────────────────────────┘
```

---

## Directory Structure

```
backend/
├── requirements.txt          # Pinned Python dependencies
└── app/
    ├── __init__.py
    ├── main.py               # FastAPI app creation, CORS, static files
    ├── api.py                # Route handlers: start game, process turns, submit actions
    ├── crud.py               # In-memory game DB, config, boss data, agent orchestration
    ├── models.py             # Pydantic models: GameState, Player, Boss, Quiz, Config
    ├── single_agent.py       # Current A2A agent pipeline (InMemoryRunner)
    ├── single_agent_old.py   # Legacy A2A agent pipeline (uses AGENT_CARD_WELL_KNOWN_PATH)
    ├── guardian_quizzes.py   # 20 quizzes — Cloud Build, Cloud Run, IAM
    ├── scholar_quizzes.py    # 20 quizzes — RAG, BigQuery, pgvector
    ├── shadowblade_quizzes.py# 17 quizzes — Gemini CLI, MCP, ADK
    └── summoner_quizzes.py   # 20 quizzes — A2A protocol, agent architecture
```

---

## Module-by-Module Breakdown

### `main.py` — Application Entry Point

```python
app = FastAPI(title="Boss Fight Dungeon")
```

**Responsibilities:**

1. **CORS Middleware** — Allows requests from `localhost` (any port) and all `*.run.app` Cloud Run domains:
   ```python
   allow_origin_regex="https?://.*(localhost|run\.app)(:\d+)?|https?://.*\.run\.app"
   ```

2. **API Router** — Mounts all game endpoints under `/api`:
   ```python
   app.include_router(api_router, prefix="/api")
   ```

3. **Static File Serving** — Serves the React build at root `/`:
   ```python
   STATIC_FILES_DIR = os.environ.get("STATIC_FILES_DIR", "../frontend/build")
   app.mount("/", StaticFiles(directory=STATIC_FILES_DIR, html=True))
   ```
   The `html=True` flag enables SPA routing — any non-API path serves `index.html`.

4. **Shutdown Cleanup** — Clears the in-memory game database on server shutdown:
   ```python
   @app.on_event("shutdown")
   def shutdown_event():
       game_db.clear()
   ```

---

### `models.py` — Pydantic Data Models

The model hierarchy and what each field does:

```
Character (base)
├── hp: int                        Current health points
├── max_hp: int                    Maximum health points (for HP bar rendering)
└── last_damage_taken: Optional[int]  Damage from most recent hit (for DamageIndicator)

Player(Character)
├── id: str                        "player_1", "player_2", etc.
├── player_class: str              "Shadowblade" | "Scholar" | "Guardian" | "Summoner"
├── a2a_endpoint: str              Cloud Run URL of deployed agent
├── _hero_agent: PrivateAttr       InMemoryRunner instance (not serialized to JSON)
├── _agent_runner: PrivateAttr     Alternative runner reference
└── _session_id: PrivateAttr       ADK session ID for conversation continuity

Boss(Character)
├── name: str                      "Procrastination", "Mergepocalypse", etc.
└── dialog_phrases: List[str]      14-15 cycling dialogue lines per boss

Quiz
├── question: str                  The quiz question text
├── answers: List[str]             3 answer choices
├── correct_index: int             Index of correct answer (0-2)
├── damage_point: int              Damage to apply if answered correctly
└── msg: str                       Narrative message from the agent

GameState
├── game_id: str                   UUID v4
├── game_type: str                 "mini" | "ultimate"
├── game_over: bool
├── player_won: Optional[bool]
├── status_message: str            Current turn status text for UI
├── current_turn: str              "boss" or "player_N"
├── boss: Boss
├── players: List[Player]
├── last_boss_attack: Optional[str]  Most recent boss attack narrative
├── active_quiz: Optional[Quiz]    Current quiz for the active player
├── turn_order: List[str]          Cyclic sequence defining who goes when
└── turn_index: int                Current position in turn_order
```

**Why `PrivateAttr` for agent fields?**

Pydantic serializes all model fields to JSON for API responses. The `InMemoryRunner` and session objects are not JSON-serializable and should never be sent to the client. Using `PrivateAttr` keeps them as Python-only attributes:

```python
class Player(Character):
    _hero_agent: PrivateAttr = PrivateAttr(default=None)

    @property
    def hero_agent(self):
        return self._hero_agent

    @hero_agent.setter
    def hero_agent(self, agent: Any):
        self._hero_agent = agent
```

The property getter/setter pattern provides a clean interface while keeping the ADK runner hidden from serialization.

---

### `api.py` — Route Handlers & Game Logic

This is the heart of the backend — 5 endpoints + 3 helper functions.

#### Helper Functions

| Function | Purpose |
|----------|---------|
| `advance_turn(game)` | Moves `turn_index` forward by 1 (wraps around), sets `current_turn`, clears `active_quiz` |
| `check_game_over(game)` | Returns `True` if boss HP ≤ 0 (win) or player(s) HP ≤ 0 (loss) |
| `trigger_guardian_agent(player)` | Background task — pre-warms Guardian's agent with "A monster is coming" |
| `process_turn(game)` | Executes the boss's turn: damage players, advance turn, trigger next player's agent |

#### Game Over Logic — Mini vs Ultimate

```python
# Mini-Boss: ALL players must be dead (but there's only 1 player)
if game.game_type == "mini" and all(p.hp <= 0 for p in game.players):
    game.player_won = False

# Ultimate Boss: ANY player dying ends the game
if game.game_type == "ultimate" and any(p.hp <= 0 for p in game.players):
    game.player_won = False
```

This is a key design decision: Ultimate Boss mode is **more punishing** — losing a single party member is instant defeat, encouraging careful play.

#### The Guardian Pre-Trigger

```python
if player.player_class == "Guardian":
    asyncio.create_task(trigger_guardian_agent(player))
```

When a game starts with a Guardian, a **background `asyncio.Task`** immediately calls the Guardian's A2A agent with a preparation message. This achieves two things:
1. **Pre-warms the Cloud Run container** (eliminates cold start on first real turn)
2. **Seeds the conversation history** — the LLM has context before the actual boss attack arrives

The task runs in the background and doesn't block the `/start` response.

---

### `crud.py` — Data Layer & Game Configuration

#### In-Memory Database

```python
game_db: Dict[str, GameState] = {}  # key = game_id (UUID)
```

Every game is stored as a `GameState` object in a Python dictionary. No external database — all state is lost on server restart. This is intentional for a demo/workshop project.

#### Configuration System

```python
GAME_CONFIG = Config(
    player_hp={"Shadowblade": 500, "Scholar": 450, "Guardian": 950, "Summoner": 400},
    boss_hp={"Procrastination": 576, "Hype": 461, ..., "Mergepocalypse": 3200}
)
```

Config is mutable via `PUT /api/config` — the frontend or an admin tool can adjust HP values without restarting the server.

**Note:** The `boss_hp` values in config are reference values. Mini-boss fights actually use `random.randint(600, 800)`, and the Ultimate Boss is hardcoded to 3500 HP. The config values serve as a tuning reference.

#### Boss Weakness Mapping

```python
BOSS_WEAKNESSES = {
    "Procrastination": "Inescapable Reality",       # → Fire familiar
    "Hype": "Inescapable Reality",                   # → Fire familiar
    "Dogma": "Revolutionary Rewrite",                # → Fire familiar
    "Legacy": "Revolutionary Rewrite",               # → Fire familiar
    "Perfectionism": "Elegant Sufficiency",           # → Earth familiar
    "Obfuscation": "Elegant Sufficiency",             # → Earth familiar
    "Apathy": "Unbroken Collaboration",               # → Water familiar
    "Mergepocalypse": ["all four weaknesses"],        # → All familiars
}
```

These weaknesses are embedded into boss attack messages ("It whispers of the {weakness} you cannot face") and sent to the player's A2A agent. The **Summoner agent** uses these hints to select the right familiar.

#### Boss Dialogues

Each of the 7 mini-bosses has **14-15 unique dialogue phrases** — all developer/engineering themed:

| Boss | Theme | Sample Line |
|------|-------|-------------|
| Procrastination | Backlog & deadlines | "I'll just add a 'TODO' comment... and get to it later." |
| Hype | Over-engineering | "My power is in the press release, not the source code!" |
| Dogma | Rigid practices | "There is only one true framework." |
| Legacy | Tech debt | "I am etched into a thousand `// HACK:` comments." |
| Perfectionism | Code review | "A 99.9% test coverage is failure." |
| Obfuscation | Unreadable code | "I thrive in the shadows of single-letter variable names." |
| Apathy | Burnout | "I simply don't care if the build is broken." |

These cycle on the frontend every 6 seconds during the boss's turn, keeping the UI alive while A2A calls are processing.

#### Agent Orchestration Functions

**`mock_player_a2a_agent()`** — The core function that calls the player's deployed agent:

```python
async def mock_player_a2a_agent(boss_attack, agent_runner, player_id, session_id, player_class):
    # Summoner gets a 30-second delay (for cascading familiar cooldowns)
    if player_class == "Summoner":
        await asyncio.sleep(30)
        msg, dmg = await process_player_action(agent_runner, boss_attack, player_id, session_id)
    else:
        msg, dmg = await process_player_action_old(agent_runner, boss_attack, player_id, session_id)

    # Fallback if agent returns 0 damage (rate limit, parse failure)
    if dmg == 0:
        dmg = random.randint(class_specific_range)  # e.g., Summoner: 210-250
    return msg, dmg
```

Key behaviors:
1. **Summoner gets 30-second sleep** — the Summoner agent calls sub-familiars via A2A, which have 60-second cooldowns. The delay ensures the previous familiar's cooldown has expired.
2. **Summoner uses `process_player_action`** (new version) while other classes use `process_player_action_old` (legacy) — see [the comparison section below](#single_agentpy-vs-single_agent_oldpy).
3. **Zero-damage fallback** — if the A2A pipeline returns 0 (LLM rate limit, JSON parse error, network failure), the backend generates class-appropriate random damage instead of stalling the game.

**`mock_damage_quiz_agent()`** — Selects a random quiz from the player's class-specific pool and attaches the agent-generated damage:

```python
def mock_damage_quiz_agent(player_response, player_class, damage_to_boss):
    questions = class_quizzes.get(player_class, [])
    selected_quiz = random.choice(questions)
    return Quiz(**selected_quiz, damage_point=damage_to_boss, msg=player_response)
```

**`mock_boss_attack_agent()`** — Generates thematic attack narratives using templates:

```python
attack_templates = [
    f"{boss_name} looms... It whispers of the {weakness} you cannot face, dealing {damage} damage.",
    f"A wave of power emanates from {boss_name}, fueled by your fear of {weakness}...",
    f"{boss_name} declares, 'You will never achieve {weakness}!' ...",
]
return random.choice(attack_templates)
```

The boss attack message serves a dual purpose:
1. **Displayed to the player** as narrative text in the UI
2. **Sent to the player's A2A agent** as the prompt — the agent reads the weakness and damage and crafts a counter-attack

---

### `single_agent.py` — A2A Agent Integration (Current)

This module builds a **2-step agent pipeline** for each player.

#### `create_heroic_action_agent(player_agent_url)` → `InMemoryRunner`

Creates and returns a ready-to-use runner:

```python
# Step 1: Remote agent — calls the player's deployed Cloud Run service
player_agent = RemoteA2aAgent(
    name="player_agent",
    agent_card=f"{player_agent_url}/.well-known/agent.json",
)

# Step 2: Local agent — parses the narrative into structured JSON
scribe_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="HeroicScribeAgent",
    instruction="... extract {damage_point: int, message: string} ..."
)

# Chained sequentially
root_agent = SequentialAgent(
    name='HeroicActionProcessor',
    sub_agents=[player_agent, scribe_agent],
)

# Wrapped in a runner with in-memory services
runner = InMemoryRunner(app_name="HeroicScribeAgent", agent=root_agent)
```

**Why `InMemoryRunner`?**
Each player gets their own runner instance with isolated:
- **Sessions** — conversation history is per-player, per-game
- **Artifacts** — any file outputs (not used here)
- **Memory** — long-term recall (not used here)

This means multiple concurrent games don't interfere with each other.

#### `process_player_action()` — Event Stream Processing

```python
async def process_player_action(runner, prompt, user_id, session_id):
    content = types.Content(role='user', parts=[types.Part.from_text(text=prompt)])

    final_output = "Nothing"
    async for event in runner.run_async(user_id, session_id, new_message=content):
        # Log all events (text, function calls, function responses)
        if event.author == "HeroicScribeAgent" and event.content.parts[0].text:
            final_output = event.content.parts[0].text

    # Parse the JSON output
    cleaned = final_output.strip().replace('```json', '').replace('```', '').strip()
    data = json.loads(cleaned)
    return data["message"], int(data["damage_point"])
```

**The event stream contains:**
1. `player_agent` events — the RemoteA2aAgent sending/receiving from Cloud Run
2. Function call events — if the remote agent uses tools (MCP calls)
3. Function response events — tool results
4. `HeroicScribeAgent` events — the parsed JSON output

The code filters for the **last text event from `HeroicScribeAgent`** — that's the structured JSON.

**JSON Cleaning:**
LLMs sometimes wrap JSON in markdown code blocks. The cleaning step handles:
- `` ```json\n{...}\n``` `` → `{...}`
- Leading/trailing whitespace

---

### `single_agent_old.py` — A2A Agent Integration (Legacy)

Functionally identical to `single_agent.py` with two differences:

| Aspect | `single_agent.py` (current) | `single_agent_old.py` (legacy) |
|--------|---------------------------|-------------------------------|
| Agent card URL | `f"{url}/.well-known/agent.json"` (hardcoded path) | `f"{url}{AGENT_CARD_WELL_KNOWN_PATH}"` (uses ADK constant) |
| `final_output` scope | Declared **outside** the event loop | Declared **inside** the loop (reset each iteration) |
| Used by | Summoner class | Shadowblade, Scholar, Guardian |
| Function name | `process_player_action()` | `process_player_action_old()` |

**The `final_output` scope bug in the old version:**
```python
# single_agent_old.py — potential issue
async for event in runner.run_async(...):
    final_output = "Nothing"  # ← reset every iteration!
    if event.author == "HeroicScribeAgent":
        final_output = event.content.parts[0].text
```
In the old version, `final_output` is re-initialized to `"Nothing"` on every event iteration. It only works because the `HeroicScribeAgent` event is typically the last one — so the variable holds the correct value when the loop exits. The new version fixes this by declaring `final_output` before the loop.

---

### Quiz Modules

Four Python files, each exporting a list of dictionaries:

| Module | Variable | Count | Topics |
|--------|----------|-------|--------|
| `shadowblade_quizzes.py` | `shadowblade_quizzes` | 17 | Gemini CLI (`--sandbox`, `/memory show`, ReAct loop), MCP protocol (SSE transport, tool listing), ADK agents |
| `scholar_quizzes.py` | `scholar_quizzes` | 20 | RAG (retrieval-augmented generation), BigQuery (`ML.GENERATE_TEXT`, external tables), pgvector, embeddings |
| `guardian_quizzes.py` | `guardian_quizzes` | 20 | Cloud Build, Cloud Run, Load Balancer, Model Armor, IAM, Cloud Storage FUSE, vLLM vs Ollama |
| `summoner_quizzes.py` | `summoner_quizzes` | 20 | A2A protocol (agent cards, RPC), SequentialAgent, ParallelAgent, LoopAgent, MCP tools as services |

Each quiz entry follows this structure:
```python
{
    "question": "What is the primary function of...",
    "answers": ["Option A", "Option B", "Option C", "Option D"],
    "correct_index": 1  # 0-based index
}
```

When used in combat, the `damage_point` and `msg` fields are added dynamically by `mock_damage_quiz_agent()`.

---

## API Reference

### `POST /api/miniboss/start`

Start a solo mini-boss fight.

**Request Body:**
```json
{
    "player_class": "Shadowblade",
    "boss_name": "Procrastination",
    "a2a_endpoint": "https://fire-familiar-xxxxx.us-central1.run.app"
}
```

**Response:** Full `GameState` JSON with the game initialized.

**Internal Flow:**
1. Look up player HP from config
2. Generate random boss HP (600-800)
3. Create `InMemoryRunner` with player's A2A endpoint
4. Create ADK session for the player
5. If Guardian → background pre-trigger task
6. Build turn order based on class
7. Create and store game, return state

---

### `POST /api/ultimateboss/start`

Start a 4-player party fight against Mergepocalypse.

**Request Body:**
```json
{
    "a2a_endpoints": {
        "Shadowblade": "https://shadowblade-agent-xxxxx.run.app",
        "Scholar": "https://scholar-agent-xxxxx.run.app",
        "Guardian": "https://guardian-agent-xxxxx.run.app",
        "Summoner": "https://summoner-agent-xxxxx.run.app"
    }
}
```

**Response:** Full `GameState` with 4 players, Mergepocalypse boss (3500 HP), and 14-turn cycle order.

---

### `GET /api/game/{game_id}`

Fetch game state. **Side effect:** If it's the boss's turn, automatically processes the boss attack and advances to the next player.

**Response:** `GameState` — may include updated HP values, `last_boss_attack`, and `active_quiz` for the next player.

This endpoint is **polled by the frontend** after the boss's turn animation plays out. The polling triggers the boss turn → player agent call → quiz generation chain.

---

### `POST /api/game/{game_id}/action`

Submit the player's quiz answer.

**Request Body:**
```json
{
    "answer_index": 1
}
```

**Response:** Updated `GameState` with boss HP reduced and next turn state.

**Damage calculation:**
- Correct answer → `quiz.damage_point` (full agent damage)
- Wrong answer → `quiz.damage_point // 2` (integer division = half, rounded down)

---

### `GET /api/config` & `PUT /api/config`

Read or update game balance settings (player HP and boss HP values).

---

## Request / Response Lifecycle

### Starting a Mini-Boss Fight

```
Client                                    Backend
  │                                         │
  ├─ POST /api/miniboss/start ─────────────▶│
  │   {class, boss_name, a2a_endpoint}      │
  │                                         ├─ config.player_hp[class] → player HP
  │                                         ├─ random.randint(600,800) → boss HP
  │                                         ├─ create_heroic_action_agent(url)
  │                                         │    └─ RemoteA2aAgent + HeroicScribeAgent
  │                                         ├─ session_service.create_session()
  │                                         ├─ [Guardian? asyncio.create_task(pre-trigger)]
  │                                         ├─ Build turn_order
  │                                         ├─ create_new_game() → UUID
  │                                         ├─ game_db[uuid] = GameState
  │◀────────────────────────────────────────┤
  │   GameState JSON                        │
```

### Boss Turn Processing

```
Client                                    Backend
  │                                         │
  ├─ GET /api/game/{id} ──────────────────▶│
  │   (current_turn == "boss")              │
  │                                         ├─ Reset all player.last_damage_taken
  │                                         │
  │                                         ├─ [Mini-Boss]:
  │                                         │    boss_damage = randint(110,140)
  │                                         │    1/8 chance → boss_damage //= 2
  │                                         │    target = random player
  │                                         │    target.hp -= boss_damage
  │                                         │
  │                                         ├─ [Ultimate Boss]:
  │                                         │    For each alive player:
  │                                         │      damage = class-specific range
  │                                         │      player.hp -= damage
  │                                         │
  │                                         ├─ check_game_over()
  │                                         ├─ advance_turn() → current_turn = "player_N"
  │                                         │
  │                                         ├─ mock_player_a2a_agent():
  │                                         │    ├─ [Summoner] sleep(30s)
  │                                         │    ├─ RemoteA2aAgent → Cloud Run
  │                                         │    ├─ HeroicScribeAgent → JSON parse
  │                                         │    └─ fallback if dmg == 0
  │                                         │
  │                                         ├─ mock_damage_quiz_agent()
  │                                         │    └─ random.choice(class_quizzes)
  │                                         │
  │◀────────────────────────────────────────┤
  │   GameState with active_quiz            │
```

### Player Action Submission

```
Client                                    Backend
  │                                         │
  ├─ POST /api/game/{id}/action ──────────▶│
  │   {answer_index: 1}                     │
  │                                         ├─ quiz.damage_point
  │                                         ├─ correct? → full damage
  │                                         │  wrong?  → damage // 2
  │                                         ├─ boss.hp -= damage
  │                                         ├─ check_game_over()
  │                                         ├─ advance_turn()
  │                                         │
  │                                         ├─ [Next = player]:
  │                                         │    └─ Call their agent + generate quiz
  │                                         ├─ [Next = boss]:
  │                                         │    └─ Set status, clear quiz
  │                                         │
  │◀────────────────────────────────────────┤
  │   Updated GameState                     │
```

---

## Agent Pipeline — How A2A Calls Work

### The SequentialAgent Pattern

Each player gets a dedicated 2-step pipeline:

```
Boss attack message (prompt)
            │
            ▼
┌───────────────────────────┐
│ Step 1: RemoteA2aAgent    │
│ name="player_agent"       │
│                           │
│ GET {url}/.well-known/    │
│     agent.json            │
│ POST {url}/ (JSON-RPC)    │
│                           │
│ Output: Narrative text    │
│ "I unleash the flames of  │
│  Pyroclasm for 240 dmg!"  │
└───────────┬───────────────┘
            │ (output automatically
            │  passed as context)
            ▼
┌───────────────────────────┐
│ Step 2: HeroicScribeAgent │
│ model="gemini-2.5-flash"  │
│                           │
│ Instruction: Extract JSON │
│ {damage_point, message}   │
│                           │
│ Output:                   │
│ {"damage_point": 240,     │
│  "message": "I unleash    │
│  the flames of Pyroclasm"}│
└───────────┬───────────────┘
            │
            ▼
    (message, damage_point)
     returned to crud.py
```

### RemoteA2aAgent — Calling Deployed Agents

The `RemoteA2aAgent` from `google-adk` handles the full A2A protocol:

1. **Discovery** — Fetches `/.well-known/agent.json` from the target URL to learn capabilities
2. **Task Submission** — POSTs a JSON-RPC message to the agent's RPC endpoint
3. **Response Handling** — Receives the agent's text response via the A2A protocol

The agent doesn't need to know what the remote service does internally — it could be a Fire SequentialAgent, a Water ParallelAgent, or a Summoner LlmAgent. The A2A protocol abstracts away the implementation.

### HeroicScribeAgent — LLM-as-JSON-Parser

Instead of writing regex or a custom parser, the project uses **Gemini 2.5 Flash as a structured data extractor**:

```
Input:  "The Forge roars to life! Fire spell charged to deal 255 damage."
Output: {"damage_point": 255, "message": "With a roar, I channel the Forge's power..."}
```

The instruction is carefully engineered:
- **"MUST BE ONLY the raw JSON object"** — prevents markdown wrapping
- **"Convert words like 'ninety' to 90"** — handles word-form numbers
- **"first-person ('I') sentence"** — ensures consistent narrative voice
- **Example included** — few-shot prompting for reliable format

### Event Stream Processing

The `runner.run_async()` yields ADK events as an async generator:

```python
async for event in runner.run_async(user_id, session_id, new_message=content):
    # Event types:
    #   event.content.parts[0].text           → Agent text output
    #   event.content.parts[0].function_call  → Tool invocation
    #   event.content.parts[0].function_response → Tool result
    #   event.author                          → Which agent produced this
```

The code logs **all** events for debugging but only captures the final `HeroicScribeAgent` text output for parsing.

### `single_agent.py` vs `single_agent_old.py`

| | `single_agent.py` | `single_agent_old.py` |
|---|---|---|
| **Import** | No `AGENT_CARD_WELL_KNOWN_PATH` | Uses `AGENT_CARD_WELL_KNOWN_PATH` constant |
| **Agent card URL** | `f"{url}/.well-known/agent.json"` | `f"{url}{AGENT_CARD_WELL_KNOWN_PATH}"` |
| **`final_output` init** | Before the `async for` loop | Inside the loop (reset each iteration) |
| **Function name** | `process_player_action()` | `process_player_action_old()` |
| **Used by** | Summoner | Shadowblade, Scholar, Guardian |

**Why two versions?**
The Summoner uses a newer agent pattern, while other classes still use the legacy version. Both produce identical results — the difference is a minor code quality improvement in variable scoping. The separation allows the Summoner code path to evolve independently.

---

## Combat Engine Deep Dive

### Turn Order System

Turn order is a **cyclic list** that repeats indefinitely:

```python
# Mini-Boss (Shadowblade/Scholar — 2 attacks per boss turn)
turn_order = ["boss", "player_1", "player_1"]
#              ^turn 0  ^turn 1     ^turn 2    → wraps to turn 0

# Mini-Boss (Guardian/Summoner — 1 attack per boss turn)
turn_order = ["boss", "player_1"]

# Ultimate Boss (14-turn cycle)
turn_order = [
    "boss",     "player_1", "player_2", "player_4", "player_1", "player_3",
    "boss",     "player_3", "player_2", "player_1", "player_4", "player_1", "player_2", "player_1"
]
# Turns per player: Shadowblade(p1)=5, Scholar(p2)=3, Guardian(p3)=2, Summoner(p4)=2
```

**Advancement logic:**
```python
def advance_turn(game):
    game.turn_index = (game.turn_index + 1) % len(game.turn_order)
    game.current_turn = game.turn_order[game.turn_index]
    game.active_quiz = None
```

The modulo operation ensures the turn order loops forever until the game ends.

### Damage Calculation

**Boss → Player (Mini-Boss):**
```python
boss_damage = random.randint(110, 140)
if random.random() < 0.125:  # 1/8 chance
    boss_damage //= 2        # "Glancing blow" — half damage
```

**Boss → Player (Ultimate Boss AoE):**
| Class | Damage Range | Rationale |
|-------|-------------|-----------|
| Shadowblade | 60–120 | Mid-range; attacks frequently to compensate |
| Scholar | 40–80 | Low; fragile class needs protection |
| Guardian | 110–170 | High; tank is designed to absorb heavy hits |
| Summoner | 70–100 | Mid-range; glass cannon with low HP |

**Player → Boss:**
```python
damage = quiz.damage_point              # From A2A agent
if answer_index != quiz.correct_index:
    damage //= 2                         # Integer division — always rounds down
game.boss.hp = max(0, game.boss.hp - damage)  # HP cannot go negative
```

### Game Over Conditions

```python
def check_game_over(game):
    # Win: Boss dies
    if game.boss.hp <= 0:
        game.player_won = True
        return True

    # Loss (Ultimate): ANY player dies → instant game over
    if game.game_type == "ultimate" and any(p.hp <= 0 for p in game.players):
        game.player_won = False
        return True

    # Loss (Mini): ALL players dead (there's only 1, so same as "player dies")
    if game.game_type == "mini" and all(p.hp <= 0 for p in game.players):
        game.player_won = False
        return True

    return False
```

### Boss Attack Generation

**Mini-Boss** — Generates a narrative attack with weakness embedded:
```python
f"{boss_name} looms... It whispers of the {weakness} you cannot face, dealing {damage} damage."
```

The weakness text is crucial — when this message is sent to the **Summoner agent**, the LLM reads the weakness keyword and selects the correct familiar:
- "Inescapable Reality" → Fire
- "Revolutionary Rewrite" → Fire
- "Elegant Sufficiency" → Earth
- "Unbroken Collaboration" → Water

**Ultimate Boss** — Generates a generic AoE message (no per-player targeting):
```python
f"Mergepocalypse unleashes a wave of despair, striking all challengers."
```

### Quiz Selection & Damage Multiplier

```
A2A agent returns: damage_point = 250

Quiz selected: Random question from player's class pool

Player answers correctly → 250 damage to boss
Player answers wrong    → 125 damage to boss (250 // 2)
```

This creates a **2× damage multiplier** for GCP knowledge — players who understand the underlying technology are literally more powerful in the game.

---

## Boss Lore & Weakness System

All 7 mini-bosses represent **common software engineering anti-patterns**:

| Boss | Anti-Pattern | Weakness | Meaning |
|------|-------------|----------|---------|
| **Procrastination** | Delaying work, pushing tickets | Inescapable Reality | Face the truth → Fire |
| **Hype** | Over-promising, framework chasing | Inescapable Reality | Cut through noise → Fire |
| **Dogma** | Rigid practices, "that's how we do it" | Revolutionary Rewrite | Challenge assumptions → Fire |
| **Legacy** | Undocumented old code, tech debt | Revolutionary Rewrite | Modernize boldly → Fire |
| **Perfectionism** | Over-engineering, never shipping | Elegant Sufficiency | Good enough is good → Earth |
| **Obfuscation** | Unreadable code, clever hacks | Elegant Sufficiency | Clarity through patience → Earth |
| **Apathy** | No code ownership, "works on my machine" | Unbroken Collaboration | Team unity → Water |

**Mergepocalypse** (Ultimate Boss) — represents the chaos of a massive merge conflict. Weak to all strategies simultaneously.

---

## Player Class Balance

| Class | HP | Turns/Cycle (Mini) | Turns/Cycle (Ultimate) | Avg Dmg/Turn | Agent Delay | Role |
|-------|----|--------------------|----------------------|-------------|-------------|------|
| Shadowblade | 500 | 2 | 5 | 110–160 | None | Fast DPS |
| Scholar | 450 | 2 | 3 | 125–150 | None | Mid DPS |
| Guardian | 950 | 1 | 2 | 120–150 | None (pre-triggered) | Tank |
| Summoner | 400 | 1 | 2 | 210–250 | 30 seconds | Glass Cannon |

**Expected fight duration (Mini-Boss, 700 HP):**
- Shadowblade: ~3 cycles (6 turns) with quizzes
- Scholar: ~3 cycles (6 turns) with quizzes
- Guardian: ~5 cycles (5 turns) with high survival
- Summoner: ~2 cycles (2 turns) — highest damage but slowest per turn

---

## Resilience & Fallback Strategies

The backend is designed to **never stall** even when external services fail:

| Failure Mode | Detection | Fallback |
|---|---|---|
| A2A agent returns 0 damage | `if dmg == 0` | Class-specific random damage (e.g., Summoner: 210-250) |
| JSON parse failure | `except (json.JSONDecodeError, TypeError)` | Returns `("A garbled message...", 0)` → triggers 0-damage fallback |
| Remote agent unreachable | Exception bubbles from `RemoteA2aAgent` | Caught by `mock_player_a2a_agent`, returns 0 → triggers fallback |
| Frontend build not found | `except RuntimeError` | Prints warning, API still works (no static files served) |
| Rate limiting (Gemini) | Agent returns empty/error response → 0 damage | Class-specific fallback damage with "Quota limit reached" message |

---

## Data Model Relationships

```
GameState
├── game_id: str (PK)
├── game_type: "mini" | "ultimate"
├── boss: Boss
│   ├── name ──────────────────▶ BOSS_WEAKNESSES[name] → weakness string
│   ├── dialog_phrases ────────▶ BOSS_DIALOGUES[name] → List[str]
│   └── hp, max_hp, last_damage_taken
├── players: List[Player]
│   ├── player_class ──────────▶ class_quizzes[class] → List[quiz_dict]
│   ├── a2a_endpoint ──────────▶ RemoteA2aAgent target URL
│   ├── _hero_agent ───────────▶ InMemoryRunner (SequentialAgent)
│   ├── _session_id ───────────▶ ADK session for conversation continuity
│   └── hp, max_hp, last_damage_taken
├── active_quiz: Quiz (nullable)
│   ├── question, answers, correct_index
│   ├── damage_point ──────────▶ From A2A agent pipeline
│   └── msg ───────────────────▶ Agent narrative for UI
├── turn_order: List[str] ─────▶ ["boss", "player_1", ...]
├── turn_index: int ───────────▶ Current position in turn_order
└── current_turn: str ─────────▶ turn_order[turn_index]
```

---

## CORS & Security Configuration

```python
app.add_middleware(
    CORSMiddleware,
    allow_origin_regex="https?://.*(localhost|run\.app)(:\d+)?|https?://.*\.run\.app",
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"]
)
```

**What this allows:**
- `http://localhost:3000` — React development server
- `http://localhost:8000` — FastAPI development server
- `https://anything.run.app` — Any Cloud Run service
- `https://anything.us-central1.run.app` — Regional Cloud Run URLs

**What this blocks:**
- Random external domains
- Direct browser requests from non-localhost/non-Cloud-Run origins

---

## Running Locally

```bash
# 1. Navigate to backend directory
cd agentverse-ui-api-dungeon/backend

# 2. Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set required environment variables (for Vertex AI)
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="TRUE"

# 5. (Optional) Point to a custom frontend build
export STATIC_FILES_DIR="../frontend/build"

# 6. Start the server
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# Server will be available at http://localhost:8000
# API docs at http://localhost:8000/docs (Swagger UI)
```

**Without a frontend build:**
The API still works — you'll see a warning in the console but can interact via Swagger UI at `/docs` or direct HTTP calls.

---

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `fastapi` | 0.116.1 | Web framework with automatic OpenAPI docs |
| `google-adk` | 1.11.0 | Agent Development Kit — LlmAgent, SequentialAgent, RemoteA2aAgent, InMemoryRunner |
| `google-genai` | 1.31.0 | Gemini API client (for HeroicScribeAgent via Vertex AI) |
| `a2a-sdk` | 0.3.1 | Agent-to-Agent protocol SDK |
| `google-cloud-aiplatform` | 1.110.0 | Vertex AI platform client |
| `pydantic` | (via fastapi) | Data validation and serialization for all models |
| `uvicorn` | (via fastapi) | ASGI server |
| `python-dotenv` | (via single_agent) | `.env` file loading for local development |

---

## Design Decisions & Trade-offs

| Decision | Why | Trade-off |
|----------|-----|-----------|
| **In-memory `game_db`** | Zero infrastructure — no DB to provision | All games lost on restart |
| **`PrivateAttr` for runners** | Prevents serialization of non-JSON objects | Requires property boilerplate |
| **LLM as JSON parser** | Handles diverse narrative formats without brittle regex | ~0.5s latency + LLM cost per parse |
| **30s sleep for Summoner** | Simple cooldown avoidance for cascading A2A calls | Adds latency; could be event-driven |
| **`process_player_action` vs `_old`** | Incremental migration — Summoner uses newer code | Two nearly-identical files to maintain |
| **Random boss HP (600-800)** | Replayability — every fight is slightly different | Config `boss_hp` values are unused at runtime |
| **1/8 glancing blow chance** | Reduces player death rate slightly | Unpredictable — can make easy fights too easy |
| **Quiz damage ÷ 2 on wrong** | Still rewards attempting; doesn't zero out agent damage | Reduces knowledge incentive vs full penalty |
| **`asyncio.create_task` for Guardian** | Non-blocking pre-warm, doesn't delay game start | Fire-and-forget — errors silently logged |
| **CORS regex for `*.run.app`** | Works for any Cloud Run deployment URL | Slightly broad — any Cloud Run service matches |
