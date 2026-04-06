# Pokemon GO PVP Roster Analyzer
## Project Specification & Design Overview

---

## 1. What This Tool Is

The PVP Roster Analyzer is a command-line tool that lets a Pokemon GO player manage their roster locally and analyze it for PVP performance across all three leagues (Great League, Ultra League, Master League). It surfaces rankings, suggests teams based on type coverage criteria, and integrates a Claude-powered natural language interface so users can ask questions about their roster in plain English rather than running multiple commands and cross-referencing tab after tab.

The tool runs entirely on the user's machine. Core analysis does not require an internet connection once data is synced. The AI command requires a network connection to reach the Claude API.

**Intended user:** A Pokemon GO player who takes PVP seriously and is tired of managing a spreadsheet or toggling between PvPoke, Pokebattler, and their in-game roster to figure out who to build.

---

## 2. Software Requirements Specification

### 2.1 Functional Requirements

#### Roster Management
- FR-1: User can import a roster from a CSV or JSON file matching a defined schema
- FR-2: User can manually add a single Pokemon entry interactively via the CLI
- FR-3: User can list all roster entries with summary stats in a formatted table
- FR-4: User can delete a roster entry by name or ID
- FR-5: Roster data persists between sessions in local storage

#### PVP Analysis
- FR-6: For any Pokemon in the roster, the tool can display its PVP rank, stat product, and optimal IVs for a given league
- FR-7: The tool can display the optimal moveset (fast move + charge moves) for a given Pokemon in a given league based on current meta data
- FR-8: The tool can compare two Pokemon head-to-head in a given league, showing stat product difference, type coverage comparison, and meta ranking difference

#### Team Building
- FR-9: Given a league, the tool can suggest the best available team from the user's roster based on meta rankings
- FR-10: User can specify required type coverage (e.g., `--cover dragon,steel,water`) and the tool will filter suggestions to teams that collectively cover those types
- FR-11: Team output displays each suggested Pokemon's ranking, typing, and key coverage

#### Meta Data Sync
- FR-12: User can run a sync command to pull the latest PvPoke ranking data and game master from their respective GitHub sources
- FR-13: The tool shows what version of the data is currently cached and when it was last synced

#### AI Query Interface
- FR-14: User can ask a free-text question about their roster in natural language
- FR-15: The tool assembles structured context (roster entries, rankings, type matchups) and sends it alongside the query to the Claude API
- FR-16: The Claude response streams back to the terminal in real time
- FR-17: The AI interface has access to the full roster, current league meta rankings, and type effectiveness data

#### Output and Display
- FR-18: All tabular output is formatted using Rich (color, borders, column alignment)
- FR-19: All commands support a `--league` flag accepting `GL`, `UL`, or `ML` (defaulting to `GL`)
- FR-20: Errors are displayed with clear messages explaining what went wrong and how to fix it

---

### 2.2 Non-Functional Requirements

- NFR-1: Analysis commands (rank, compare, team) must respond in under 2 seconds on standard hardware after data is cached locally
- NFR-2: The tool must function fully offline for all non-AI and non-sync commands
- NFR-3: The roster file format must be documented and human-readable so users can edit it directly if needed
- NFR-4: Installation requires only `pip install` — no external services, databases, or configuration files required to get started
- NFR-5: The Claude API key is read from an environment variable (`ANTHROPIC_API_KEY`), never stored in a config file

---

### 2.3 Data Requirements

#### User Roster Entry
Each entry in the user's roster contains:
- Species name and Pokedex number
- Individual Values (Attack IV, Defense IV, Stamina IV) — each 0–15
- Level (1–51 in 0.5 increments)
- Fast move and charge moves (optional — defaults to meta-optimal if omitted)
- Optional nickname or notes field

#### Meta Data (Cached Locally)
- **PvPoke ranking data** — JSON file from the PvPoke repository containing ranked Pokemon lists for each league, with stat products and moveset grades
- **Pokemon GO game master** — JSON file from PokeMiners containing base stats for all Pokemon, move definitions, type matchup multipliers, and CP formula constants

#### Derived / Computed Data
- CP at a given level (computed from game master base stats and IV values)
- Stat product at league cap (computed from IVs, level, and base stats)
- PVP rank (looked up from PvPoke data by species and IV combination)

---

### 2.4 External Interfaces

| Interface | Source | Access Method | Required For |
|---|---|---|---|
| PvPoke ranking JSON | GitHub (pvpoke/pvpoke) | HTTPS GET on sync | Ranking lookups, team builder |
| Game master JSON | GitHub (PokeMiners/pokemon-go-protobuf) | HTTPS GET on sync | CP computation, move data, type chart |
| Claude API | Anthropic | HTTPS POST (streaming) | `ask` command only |

---

### 2.5 Constraints

- The tool is a local CLI application. It has no server, no web interface, and no multi-user functionality.
- The Claude API requires an API key and will incur usage costs. The `ask` command should display an estimated token count before sending if the context is large.
- PvPoke and PokeMiners data is community-maintained and may change format across updates. The sync command should validate data shape after downloading and warn if something unexpected is found.

---

## 3. Software Design

### 3.1 Architecture Overview

The tool is a single Python package installed via pip. All state is stored locally. There is no daemon, no background process, and no web server. Every command is a discrete invocation that reads from local storage, computes, outputs to the terminal, and exits.

```
┌─────────────────────────────────────────────────────────┐
│                        User (Terminal)                   │
└───────────────────────────┬─────────────────────────────┘
                            │  CLI invocation (pvp <command>)
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     CLI Layer (Typer)                    │
│   Parses commands, flags, and arguments. Routes to       │
│   the appropriate handler. Owns --help text.             │
└──────┬──────────────────────────────────────────────────┘
       │
       ├──────────────────┬──────────────────┬────────────────────┐
       ▼                  ▼                  ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐   ┌─────────────────┐
│   Roster     │  │   Analysis   │  │   Sync       │   │   AI Layer      │
│   Module     │  │   Engine     │  │   Module     │   │   (Claude API)  │
│              │  │              │  │              │   │                 │
│ import/add/  │  │ CP/IV math   │  │ Fetches and  │   │ Builds context  │
│ list/delete  │  │ Rank lookup  │  │ validates    │   │ prompt, sends   │
│              │  │ Team builder │  │ PvPoke +     │   │ to Claude,      │
│              │  │ Head-to-head │  │ game master  │   │ streams output  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘   └────────┬────────┘
       │                 │                 │                     │
       └─────────────────┴─────────────────┘                     │
                         │                                       │
                         ▼                                       │
┌─────────────────────────────────────────────────────────┐      │
│                    Data Layer                            │      │
│                                                          │      │
│  ┌─────────────────┐      ┌──────────────────────────┐  │      │
│  │  Roster Store   │      │   Meta Cache             │  │      │
│  │  (SQLite)       │      │   (Local JSON files)     │  │      │
│  │                 │      │                          │  │      │
│  │  User's Pokemon │      │  PvPoke rankings         │  │      │
│  │  IVs, moves,    │      │  Game master data        │  │      │
│  │  notes          │      │  Type chart              │  │      │
│  └─────────────────┘      └──────────────────────────┘  │      │
└─────────────────────────────────────────────────────────┘      │
                         │                                       │
                         ▼                                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Display Layer (Rich)                               │
│   Formats and renders all terminal output: tables, panels, progress     │
│   bars, streamed AI responses. No business logic lives here.            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.2 Module Breakdown

#### `cli.py` — Entry Point
Defines all Typer commands and their flags. Does no computation itself — delegates immediately to the appropriate module. This keeps the CLI layer thin and easy to test.

#### `roster.py` — Roster Module
Handles all CRUD operations for the user's Pokemon roster. Reads and writes to a local SQLite database via SQLAlchemy. Validates input on import (checks IV ranges, move names against game master). Provides a clean interface that the analysis engine consumes without knowing about the database layer.

#### `engine.py` — Analysis Engine
The computational core. Contains:
- **CP computation:** given a Pokemon's base stats (from game master), IVs, and level, compute CP using the documented formula
- **Stat product computation:** used to determine how a Pokemon performs at a league's CP cap
- **Rank lookup:** given a species and IVs, find the matching entry in the PvPoke ranking data and return its rank, percentile, and stat product
- **Team builder:** given a league and optional coverage requirements, score and rank all eligible roster entries, then select a non-overlapping team

#### `sync.py` — Sync Module
Handles fetching the PvPoke ranking JSON and game master JSON from GitHub over HTTPS, validates that the response shape matches what the tool expects, writes to local cache, and records a sync timestamp. Runs independently of everything else — if sync fails, the cached data remains usable.

#### `ai.py` — AI Layer
Builds the structured prompt context from the user's roster (serialized to a compact representation), the relevant meta data for the requested league, and the user's query. Sends the request to the Claude API using streaming and pipes the response chunks directly to the Rich display layer as they arrive. Handles API errors gracefully with clear messages.

#### `data.py` — Data Layer
SQLAlchemy models for the roster database and utility functions for reading the cached JSON files (PvPoke, game master). Acts as the single source of truth for data access — no other module reads files or queries the database directly.

#### `display.py` — Display Layer
All Rich formatting logic: table schemas, color themes, panel layouts, progress indicators. Receives plain data structures from other modules and renders them. No business logic.

---

### 3.3 Directory Structure

```
pvp-analyzer/
├── pvp/
│   ├── __init__.py
│   ├── cli.py          # Typer app, all command definitions
│   ├── roster.py       # Roster CRUD
│   ├── engine.py       # CP math, rankings, team builder
│   ├── sync.py         # Data fetching and caching
│   ├── ai.py           # Claude API integration
│   ├── data.py         # SQLAlchemy models, JSON loaders
│   └── display.py      # Rich formatting
├── tests/
│   ├── test_engine.py  # CP math, rank lookup, team builder
│   ├── test_roster.py  # Import/export, validation
│   └── fixtures/       # Pinned PvPoke and game master snapshots
├── pyproject.toml      # Package metadata, dependencies, entry point
├── README.md
└── .env.example        # Shows ANTHROPIC_API_KEY env var format
```

---

### 3.4 Technology Stack

| Technology | Role | Why This Choice |
|---|---|---|
| Python | Primary language | Best ecosystem for this kind of data tool; you have prior experience |
| Typer | CLI framework | Built on Click, excellent docs, auto-generates --help text, type-annotated commands |
| Rich | Terminal output | Best-in-class Python terminal formatting; tables, color, streaming text |
| SQLite + SQLAlchemy | Roster storage | Zero-config local database; SQLAlchemy makes it testable without touching disk |
| httpx | HTTP client | Modern, async-capable, easy to mock in tests; used for both sync and Claude API calls |
| Pydantic v2 | Data validation | Validates roster imports and game master shapes; catches malformed data early |
| pytest | Testing | Standard Python testing framework; pairs with SQLAlchemy's in-memory database for fast tests |
| Claude API (Anthropic) | AI layer | Powers the natural language `ask` command |
| GitHub Actions | CI | Runs tests on every push; can be extended later to publish to PyPI |

---

## 4. Functionality, Usage, and AI Integration

### 4.1 How a User Interacts With the Tool

The tool is invoked from the terminal using `pvp` as the root command. Every operation is a subcommand. The user never edits config files or interacts with a UI — everything happens in the terminal.

**First-time setup:**
```bash
pip install pvp-analyzer
export ANTHROPIC_API_KEY=sk-ant-...
pvp sync                   # Downloads PvPoke rankings and game master
pvp import my-roster.csv   # Loads your Pokemon into local storage
```

**Daily usage:**
```bash
pvp list                              # See your full roster in a table
pvp rank "Medicham" --league GL       # How good is your Medicham in Great League?
pvp team --league GL                  # Best team from your roster
pvp team --league GL --cover dragon,fairy,steel   # Team that covers those types
pvp compare "Medicham" "Galarian Stunfisk" --league GL
pvp ask "Do I have anything that beats Cresselia in Great League?"
pvp ask "Build me a Great League team from my roster that doesn't use Medicham"
```

---

### 4.2 Example Terminal Sessions

**Checking a single Pokemon:**
```
$ pvp rank "Medicham" --league GL

 Medicham — Great League Analysis
┌──────────────────┬────────────────┐
│ Your IVs         │ 1 / 15 / 14    │
│ Your Level       │ 40.5           │
│ Your CP          │ 1496           │
│ Rank             │ #12 / 4096     │
│ Stat Product     │ 2,431,512      │
│ Percentile       │ 99.7%          │
│ Fast Move        │ Counter        │
│ Charge Move 1    │ Ice Punch      │
│ Charge Move 2    │ Dynamic Punch  │
└──────────────────┴────────────────┘
```

**Team builder:**
```
$ pvp team --league GL --cover dragon,fairy

 Suggested Great League Team
┌──────────────────────┬────────┬──────────┬─────────────────────────────┐
│ Pokemon              │ Rank   │ Type     │ Covers                      │
├──────────────────────┼────────┼──────────┼─────────────────────────────┤
│ Medicham             │ #12    │ FIG/PSY  │ Normal, Dark, Ice           │
│ Galarian Stunfisk    │ #8     │ GRD/STL  │ Dragon, Fairy, Rock, Ice    │
│ Swampert            │ #31    │ WAT/GRD  │ Fire, Rock, Steel           │
└──────────────────────┴────────┴──────────┴─────────────────────────────┘

Coverage: Dragon ✓  Fairy ✓  Steel ✓  Water ✓  Dark ✓
```

---

### 4.3 How AI Integration Works

The `ask` command is where the Claude API comes in. Rather than exposing a raw chat interface, the tool builds a structured context document from your local data before sending anything to Claude. This means Claude is reasoning over *your actual roster* — not generic Pokemon knowledge.

**What gets sent to Claude:**

1. **Your roster** — serialized as a compact structured list of your Pokemon with their IVs, computed stat products, and PvPoke ranks for the requested league
2. **Meta context** — the current top-ranked Pokemon in the relevant league, so Claude understands the competitive environment your team needs to operate in
3. **Type chart** — so Claude can reason about coverage and weaknesses correctly
4. **Your question** — appended at the end

A simplified version of what the prompt looks like:

```
You are a Pokemon GO PVP analyst. The user's Great League roster is below.
Rank each Pokemon's stat product and current meta rank are provided.

ROSTER:
- Medicham (FIG/PSY): IVs 1/15/14, Rank #12, Stat Product 2,431,512
- Galarian Stunfisk (GRD/STL): IVs 0/14/13, Rank #8, Stat Product 2,589,344
- Swampert (WAT/GRD): IVs 2/15/15, Rank #31, Stat Product 2,198,765
- Talonflame (FIR/FLY): IVs 0/12/15, Rank #211, Stat Product 1,876,443
[... rest of roster ...]

CURRENT META TOP 10 (GL): [list]
TYPE CHART: [compact representation]

USER QUESTION: Do I have anything that can beat Cresselia reliably?
```

Claude then streams a response back to the terminal, reasoning over the actual data rather than generalizing.

**Why this is interesting to write about:**

The prompt engineering decisions here are non-obvious. How much roster data do you include before the context gets too large to be useful? How do you represent the type chart compactly? When does Claude give genuinely correct Pokemon analysis versus confidently wrong analysis? These are all rich article topics that come directly out of building this feature.

---

### 4.4 What the Tool Does Not Do

Being explicit about scope is important for keeping the project shippable:

- **No real-time game data** — the tool uses cached meta data, not a live connection to Pokemon GO servers
- **No battle simulation** — it surfaces rankings and coverage, not win-rate predictions (that is Pokebattler's job)
- **No account integration** — there is no official Pokemon GO API. The user either imports a manually formatted CSV or enters their Pokemon by hand
- **No web interface** — terminal only
- **No multi-user support** — one roster per installation

---

### 4.5 What Makes This a Good AI Article Project

The AI integration here is specific enough to generate multiple distinct articles:

- **Prompt design:** How do you structure a 200-Pokemon roster into a Claude prompt without blowing past the context window? What do you cut?
- **Accuracy evaluation:** Is Claude's Pokemon GO knowledge reliable? Where does it confabulate (e.g., wrong move effects, outdated meta reads)?
- **Context vs. knowledge:** Does providing the roster data explicitly in the prompt produce better results than asking Claude to recall it from training? (It does — and measuring that difference is a good article.)
- **Streaming UX:** What does it feel like to stream an AI response into a terminal compared to waiting for a full response? When does it feel fast and when does it feel broken?
