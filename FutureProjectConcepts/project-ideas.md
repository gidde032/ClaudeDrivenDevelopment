# Project Ideas: Fly Fishing & Pokemon Go
## Grouped by Domain Category

---

## Full-Stack Web App

### Fly Fishing — River Conditions & Hatch Journal

**Problem statement:** Anglers cross-reference stream gauge levels, weather forecasts, and insect hatch timing from a half-dozen different sources every time they plan a trip. There is no single place that synthesizes all of it into an actionable picture.

**Core features:**
- Stream condition aggregator pulling from USGS and NOAA
- Community-driven hatch calendar (users log what they're seeing on the water)
- Personal fishing log with catches tied to conditions on that day
- AI assistant for natural language queries: "Should I fish the Madison this weekend?"
- River section map with current gauge and temperature overlays

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | TypeScript (full-stack) |
| Frontend | Next.js, TailwindCSS, Recharts (water level graphs), Leaflet.js (river map) |
| Backend | Next.js API routes or FastAPI (Python) |
| Database | PostgreSQL (catches, hatch reports), Redis (caching stream data) |
| External APIs | USGS Water Services API, OpenWeatherMap, state fish & wildlife stocking APIs |
| AI Agent | Claude — natural language condition summaries, go/no-go recommendations from current data |
| Auth | Clerk or Supabase Auth |
| Deployment | Railway or Render |

**Development outline:**

1. **Foundation** — Set up Next.js project, configure PostgreSQL, implement auth. Create the data models for river sections, catches, and hatch reports.
2. **Data ingestion** — Integrate USGS Water Services API and OpenWeatherMap. Build a caching layer so you're not hitting the APIs on every page load.
3. **Core UI** — Build the river map with Leaflet, gauge/temperature charts with Recharts, and the fishing log entry form.
4. **Community hatch calendar** — Users can log hatch sightings (species, intensity, date, river section). Build the calendar view and aggregation.
5. **AI layer** — Integrate Claude API. Feed it current conditions, recent hatch reports, and historical catch data from that section to power the natural language assistant.
6. **Polish & deploy** — Add alert subscriptions (email when a river hits target gauge height), mobile-responsive layout, deploy to Railway.

---

### Pokemon Go — Local Raid Coordination Hub

**Problem statement:** Raid coordination happens across fragmented Discord servers, WhatsApp chats, and Facebook groups with no structured scheduling, RSVP tracking, or team composition tooling in one place.

**Core features:**
- Raid event board showing active and upcoming raids with location
- RSVP system with push notifications when enough players commit to a lobby
- Gym map with real-time raid status
- AI-powered team composition recommendations based on who is attending and their known rosters
- Event history so communities can see participation trends over time

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | TypeScript |
| Frontend | Next.js, TailwindCSS, Mapbox GL JS (gym locations) |
| Backend | Next.js API routes + Supabase Realtime (live RSVP updates) |
| Database | PostgreSQL + PostGIS (geospatial gym data) |
| External APIs | PokeAPI (Pokemon data and typing), community-sourced gym coordinate datasets |
| AI Agent | Claude — raid viability assessments, counter recommendations based on attendee count and levels |
| Notifications | Web Push API |
| Deployment | Vercel + Supabase |

**Development outline:**

1. **Foundation** — Set up Next.js, Supabase (PostgreSQL + Realtime + Auth), and PostGIS. Define schemas for gyms, raids, and RSVPs.
2. **Gym map** — Integrate Mapbox GL JS with seeded gym location data. Display current raid status and tier as map markers.
3. **Raid board & RSVPs** — Build the raid event listing and RSVP flow. Wire up Supabase Realtime so the attendee count updates live without refresh.
4. **Push notifications** — Implement Web Push so users get notified when a lobby they've joined hits the minimum player threshold.
5. **AI layer** — Integrate Claude API with PokeAPI data. Given a boss and the attending players' rough level distribution, generate counter recommendations and a viability summary.
6. **Polish & deploy** — Add raid creation flow for community organizers, gym submission/moderation, deploy to Vercel.

---

## Developer Tool / CLI Utility

### Fly Fishing — Fly Tying Recipe Manager

**Problem statement:** Fly tiers maintain their patterns in notebooks, text files, and memory with no structured format for versioning, searching, or generating a materials shopping list before a trip to the fly shop.

**Core features:**
- Add, view, search, and tag tying patterns (hook size, materials, target species, season)
- Generate a deduplicated shopping list from a selected set of patterns
- Export individual pattern cards to PDF
- AI command for suggestions: "What would work on the Henry's Fork in late October?"
- Pattern versioning so you can track tweaks over time

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | Python |
| CLI Framework | Typer + Rich (formatted tables, color, progress bars) |
| Storage | SQLite via SQLAlchemy (local, zero-config) |
| PDF Export | WeasyPrint or ReportLab |
| AI Agent | Claude API — pattern suggestions based on species, season, and water conditions |
| Testing | pytest |
| Distribution | Published to PyPI |

**Development outline:**

1. **Data model** — Define SQLite schema for patterns (name, hook size, materials list, tags, target species, notes, version history). Set up SQLAlchemy models.
2. **Core commands** — Implement `add`, `list`, `view`, `search`, `tag`, and `delete` commands with Typer. Use Rich for clean terminal output.
3. **Shopping list** — `shop` command accepts one or more pattern names or tags, deduplicates materials across all of them, and outputs a formatted list. Add quantity tracking per material.
4. **PDF export** — `export` command renders a pattern card (materials, steps, notes, hook size) to PDF using a template. WeasyPrint lets you style this with CSS.
5. **AI command** — `suggest` command takes species, season, and optionally water conditions. Sends context to Claude API and streams a structured recommendation back to the terminal.
6. **Polish & publish** — Add `import`/`export` for JSON so patterns are portable, write docs, publish to PyPI.

---

### Pokemon Go — PVP Roster Analyzer

**Problem statement:** Serious PVP players need to cross-reference IVs, movesets, and meta rankings across hundreds of Pokemon. There is no fast CLI tool for analyzing your roster and building a team without opening three browser tabs.

**Core features:**
- Ingest a manual roster or exported data file
- Compute PVP IVs and stat products for each league (Great League, Ultra League, Master League)
- Flag top-ranked picks and surface optimal movesets from current meta data
- AI command: "Build me a Great League team from what I have that covers Dragon, Steel, and Water"
- Output formatted comparison tables for side-by-side evaluation

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | Python |
| CLI Framework | Typer + Rich |
| Data Sources | PvPoke open ranking data (JSON), PokeAPI, Pokemon GO game master JSON (PokeMiners on GitHub) |
| Data Processing | pandas (roster analysis), pure Python (CP/IV math — formulas are fully documented) |
| AI Agent | Claude — team composition analysis, meta recommendations, natural language queries against your roster |
| Storage | Local SQLite or JSON for your roster |
| Testing | pytest |

**Development outline:**

1. **Data layer** — Write parsers for the PvPoke ranking JSON and Pokemon GO game master. Implement CP and IV computation in pure Python using the community-documented formulas. Cache parsed data locally.
2. **Roster ingestion** — Define a simple CSV/JSON format for a user's roster (species, IVs, level, moves). Implement the `import` command. Support manual entry as a fallback.
3. **Analysis commands** — `rank` command shows a roster entry's PVP ranking for a given league. `team` command takes a league and outputs the top candidates with their coverage types and key matchups.
4. **Meta sync** — `sync` command pulls the latest PvPoke rankings and game master from GitHub so data stays current without reinstalling.
5. **AI command** — `ask` command sends roster data and user query to Claude. Claude reasons over the type coverage and ranking data to suggest a team and explain the tradeoffs.
6. **Polish & publish** — Add `compare` command for head-to-head Pokemon evaluation, write docs, publish to PyPI.

---

## Data Pipeline + Dashboard

### Fly Fishing — Stream Health & Fishability Index

**Problem statement:** No single tool combines stream gauge height, water temperature, weather forecasts, and fish stocking schedules into an actionable fishability score for a given river on a given day.

**Core features:**
- Automated daily ingestion from USGS, NOAA, and state wildlife agencies
- Computed fishability score per river section based on configurable thresholds (ideal gauge range, temperature range, recent stocking activity)
- Historical trend charts for each river section
- Alert subscriptions (email when a river hits your target conditions)
- AI-generated weekly river reports in plain language from the aggregated data

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | Python |
| Orchestration | Prefect (free tier) or GitHub Actions cron |
| Data Sources | USGS Water Services API, NOAA Climate API, state fish & wildlife stocking CSVs |
| Transformation | dbt (SQL-based transformation and scoring layer) |
| Storage | PostgreSQL with TimescaleDB extension (time-series data) |
| Dashboard | Streamlit (fast to build) or Evidence.dev (SQL-native, cleaner output) |
| Visualization | Plotly |
| AI Agent | Claude — generates a natural language weekly river report from structured data, answers condition questions in a chat interface |
| Deployment | Railway (pipeline + database) + Streamlit Cloud (dashboard) |

**Development outline:**

1. **Pipeline skeleton** — Set up Prefect flows for each data source. Start with USGS Water Services (most important). Write ingestion tasks that pull gauge and temperature readings into PostgreSQL on a schedule.
2. **Additional sources** — Add NOAA weather ingestion and state stocking data (these are often CSVs on state wildlife sites — may require light scraping).
3. **Scoring layer** — Use dbt to build the transformation models. Define a fishability score that weights gauge level, temperature, clarity, and days since last stocking. Make thresholds configurable per species.
4. **Dashboard** — Build Streamlit or Evidence.dev dashboard with a river selector, current conditions panel, historical charts, and the scoring breakdown.
5. **Alerts** — Add a Prefect task that runs post-ingestion and sends email alerts (via SendGrid or Resend) to subscribed users when conditions cross their thresholds.
6. **AI layer** — At the end of each weekly pipeline run, pass a structured data summary to Claude and store the generated river report. Surface it in the dashboard and optionally send it as a newsletter.

---

### Pokemon Go — Meta Shift Tracker

**Problem statement:** The Pokemon GO meta shifts with every seasonal event, new Pokemon release, and move rebalance, but there is no dashboard that tracks these changes over time and surfaces patterns from historical data.

**Core features:**
- Tracks raid boss rotations, spawn pool changes, and PVP meta ranking shifts over time
- Diffs each ingestion run against the previous snapshot to identify what changed
- Surfaces patterns across historical data (e.g., Dragon-type raids peak in January)
- AI-generated weekly "what changed this week in Pokemon GO" summaries

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | Python |
| Orchestration | Prefect |
| Data Sources | Leekduck.com (scraped — current raid and spawn data), PokeAPI, PvPoke open ranking data (JSON), PokeMiners game master on GitHub |
| Scraping | BeautifulSoup + httpx |
| Transformation | dbt or pandas + SQLAlchemy |
| Storage | PostgreSQL |
| Dashboard | Streamlit or Evidence.dev |
| Visualization | Plotly or Vega-Altair |
| AI Agent | Claude — generates structured weekly changelogs and trend summaries from diff'd snapshots |

**Development outline:**

1. **Data source inventory** — Identify each source (Leekduck, PokeAPI, PvPoke JSON, game master), understand update cadence, and decide on ingestion frequency. Most need daily checks.
2. **Ingestion pipelines** — Write Prefect flows for each source. For Leekduck, write BeautifulSoup scrapers; for others, use HTTP clients. Store raw snapshots before transformation so you can reprocess if your schema changes.
3. **Diff layer** — After each ingestion, compare the new snapshot to the previous one and write the diffs (new bosses, removed spawns, ranking changes) to a dedicated changes table. This is the core of the value.
4. **Historical analysis** — Build dbt models that aggregate changes over time: raid type distributions by month, meta ranking volatility by Pokemon, event overlap patterns.
5. **Dashboard** — Build a Streamlit or Evidence.dev dashboard with a timeline of changes, filterable by category (raids, PVP, spawns), plus historical trend charts.
6. **AI layer** — Pass each week's diff to Claude and prompt it to generate a structured changelog narrative. Store and display these in the dashboard as a browsable archive.

---

## Open-Source Library

### Fly Fishing — Python Client for Stream Condition Data

**Problem statement:** Every fly fishing app that needs USGS or NOAA stream data reimplements the same HTTP logic, response parsing, and normalization from scratch. A clean, well-typed library would eliminate that and give the community a shared foundation.

**Core features:**
- Unified Python API for USGS Water Services and NOAA streamflow forecasts
- Optional integration for state fish stocking data
- Returns normalized Pydantic models — no raw JSON handling for the consumer
- Handles pagination, retries, rate limiting, and optional local caching out of the box

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | Python |
| HTTP | httpx (async-first, also supports sync) |
| Data Models | Pydantic v2 |
| Caching | Optional diskcache integration |
| Testing | pytest + respx (httpx-compatible HTTP mocking) |
| Docs | MkDocs + mkdocstrings (auto-generated from docstrings) |
| CI | GitHub Actions (lint, test, auto-publish to PyPI on version tag) |
| Distribution | PyPI |

**Development outline:**

1. **API research** — Read the USGS Water Services and NOAA API documentation thoroughly. Understand pagination, parameter formats, available parameters (site codes, statistic codes), and rate limits.
2. **Core client** — Build the base httpx client with retry logic, timeout handling, and rate limit awareness. Keep it separate from the data-parsing layer so it can be reused.
3. **USGS module** — Implement `get_site`, `get_current_conditions`, and `get_historical_data`. Define Pydantic models for sites, readings, and time series. Test against real API responses captured with respx.
4. **NOAA module** — Implement streamflow forecast endpoints. Normalize the response format to match the same Pydantic patterns as the USGS module so consumers have a consistent interface.
5. **Caching layer** — Add opt-in diskcache integration. Useful for development and for apps that don't need real-time data on every call.
6. **Docs & CI** — Write MkDocs site with a quickstart guide and full API reference generated by mkdocstrings. Set up GitHub Actions to run tests on PRs and publish to PyPI when a version tag is pushed.

---

### Pokemon Go — Game Master Library

**Problem statement:** The Pokemon GO game master file defines all Pokemon stats, moves, and PVP mechanics, but it is a raw JSON/protobuf blob maintained by the community. There is no well-typed Python library that exposes it as clean, usable models with CP and IV computation built in.

**Core features:**
- Typed Pydantic models for Pokemon, moves, forms, and PVP mechanics
- Methods for computing CP, IVs, and PVP stat products
- Automatic updates from the community-maintained game master repo on GitHub
- Stable, versioned API so downstream tools don't break when the game master format changes

**Tech Stack:**

| Layer | Choices |
|---|---|
| Language | Python |
| Data Models | Pydantic v2 |
| Source Data | PokeMiners/pokemon-go-protobuf on GitHub (community-maintained game master) |
| Auto-Update | GitHub Actions — watches for upstream game master changes, runs tests, publishes new library version to PyPI |
| Math | Pure Python — CP formula and IV computation are fully documented by the community |
| Testing | pytest with snapshot testing (game master data pinned as fixtures to catch format changes) |
| Distribution | PyPI |

**Development outline:**

1. **Game master research** — Study the PokeMiners game master JSON structure. Identify the fields relevant to your scope: Pokemon base stats, move data, form overrides, PVP league settings. Ignore the rest initially.
2. **Pydantic models** — Define models for `Pokemon`, `Move`, `Form`, and `PvpLeague`. Use Claude to help translate the raw JSON field names (which are often protobuf-style) into clean Pythonic model definitions.
3. **Computation layer** — Implement CP calculation, IV-to-stat conversion, and PVP stat product computation as methods on the models. Validate outputs against known values from established tools like Pokebattler.
4. **Data loader** — Write a loader that fetches the game master JSON from GitHub (or a local file) and instantiates your models. Add a `sync` function that checks for the latest release and downloads it.
5. **Auto-update pipeline** — Set up a GitHub Actions workflow that runs on a schedule, checks the PokeMiners repo for a new game master version, runs the test suite against it, and if it passes, bumps the library version and publishes to PyPI automatically.
6. **Snapshot tests & docs** — Pin known-good game master snapshots as test fixtures so any future format change surfaces immediately rather than silently breaking consumers. Write a MkDocs site with usage examples.
