# Fly Fishing River Conditions & Hatch Journal
## Project Specification, System Design, and Architecture

---

## 1. What This Project Is

A full-stack web application that gives fly anglers a single place to track river conditions, log their catches, report insect hatches, and get AI-assisted trip planning advice. Instead of cross-referencing USGS, weather apps, stocking reports, and fishing forums before every trip, a user opens one dashboard and gets a synthesized picture of what is happening on the rivers they care about.

The application has two distinct layers of value: **real-time data** (stream gauge readings, weather, stocking activity pulled from public APIs automatically) and **community knowledge** (hatch observations and catch reports logged by users). The AI layer sits on top of both and lets users query them in natural language.

**Intended users:** Serious fly anglers who fish regularly, care about conditions before making the drive, and want a personal log of their fishing history tied to the conditions that were present on each day.

**Problem statement in one sentence:** Fly anglers plan trips using five different tools and have no way to connect their personal fishing history to the conditions that were present when they caught fish.

---

## 2. Software Requirements Specification

### 2.1 Functional Requirements

#### River and Conditions Browser
- FR-1: Users can browse a list of rivers and river sections pre-seeded in the database
- FR-2: Each river section displays current gauge height (feet), discharge (CFS), and water temperature pulled from USGS
- FR-3: Each river section displays a 7-day historical trend chart for gauge height and temperature
- FR-4: Each river section displays the current weather forecast from a weather API
- FR-5: River data is cached so that page loads do not trigger live USGS API calls every time
- FR-6: A map view shows all seeded river sections as markers, colored by current fishability (green/yellow/red based on configurable thresholds per river)

#### Hatch Calendar
- FR-7: Authenticated users can submit a hatch report for a river section: insect species, intensity (light/moderate/heavy), time of day, date, and optional notes
- FR-8: The hatch calendar displays all community reports for a selected river section in a monthly calendar view
- FR-9: Reports are tagged by insect species so users can filter to see when a specific hatch has historically occurred on a given section
- FR-10: Users can view their own past reports and edit or delete them

#### Personal Catch Log
- FR-11: Authenticated users can log a catch with the following fields: fish species, river section, catch date, fishing method (dry/nymph/streamer/wet/other), fly pattern (e.g. "Parachute Adams"), fly size (hook size as integer), optional catch location as a named spot or a lat/lng pin within the section, optional photo, and notes
- FR-12: When a catch is logged, the application automatically snapshots and permanently stores the conditions present on that section on that date: gauge height, discharge, water temperature, weather summary, wind speed and direction, and any hatch reports submitted for that section on the same date
- FR-13: The catch log displays in reverse chronological order with the full conditions snapshot (conditions, fly used, location, active hatches) visible alongside each entry
- FR-14: Users can filter their log by river, section, fish species, fishing method, fly pattern, and date range
- FR-15: When logging a catch, the form auto-populates the conditions snapshot fields from current data and auto-associates any hatch reports for that section submitted within 24 hours of the catch date — the user can confirm or dismiss the association

#### Catch Log — Conditions Analysis — TRIAGED (Future Scope)
Aggregating historical catch data into a personal conditions analysis view has been deferred. The full conditions snapshot captured in FR-12 stores every field this feature needs, so no schema changes will be required when it is added. It is deferred because the feature produces no meaningful output until a user has a substantial catch history, making it difficult to build and validate early.

Deferred requirement: A conditions analysis view per river section that surfaces the user's personal productive temperature range, which fly patterns produced catches and under what conditions, and whether active hatches correlated with successful sessions.

#### Alert Subscriptions — TRIAGED (Future Scope)
Condition-based email alerts have been deferred from the initial build. The data model supports adding them later without schema changes. Deferring alerts removes the Resend dependency and simplifies the cron job to pure data ingestion.

Deferred requirement: User-defined condition threshold alerts (gauge, discharge, temperature) with email delivery via Resend when a threshold crossing is detected.

#### AI Trip Assistant
- FR-16: Authenticated users can query the AI assistant in natural language from any river section page
- FR-17: The assistant has access to current conditions, the 7-day trend for that section, recent community hatch reports, and the user's own catch history on that section
- FR-18: For the initial build, the assistant returns a complete response rather than streaming — the full response renders when ready. Streaming can be added in a later iteration.
- FR-19: The assistant can decline to make a recommendation if conditions data is stale or unavailable, rather than hallucinating

#### Authentication and User Accounts

**Complexity note:** Clerk handles the technical implementation of auth (sessions, OAuth, email verification) with minimal code. The added complexity is architectural — once users exist, every API route must verify identity and every database query must be scoped to the correct user. This is a consistent discipline applied across the whole app, not a one-time task. It is also what makes the app usable by real people rather than a local demo.

- FR-20: Users can sign up and log in with email/password or Google OAuth via Clerk
- FR-21: Unauthenticated users can view the conditions browser and hatch calendar in read-only mode
- FR-22: All write operations (log catch, submit hatch report) require authentication

---

### 2.2 Non-Functional Requirements

- NFR-1: River condition pages must load in under 2 seconds. Cached data is acceptable if no older than 1 hour for gauge readings.
- NFR-2: The application must be mobile-responsive. Anglers often check conditions from their phones on the road.
- NFR-3: The USGS and weather APIs must never be called directly from the browser — all external API calls are proxied through server-side routes to protect rate limits and avoid exposing keys.
- NFR-4: The AI assistant must not be able to access any other user's catch data. Context assembly is scoped strictly to the requesting user.
- NFR-5: All environment variables (API keys, database URLs) are stored in environment config, never committed to source control.
- NFR-6: The application must remain functional (conditions display, log viewing) even if the AI service is unavailable — the AI feature degrades gracefully.

---

### 2.3 Data Models

#### River
Represents a named river body. One river has many RiverSections.

In a relational database, the relationship is expressed through a foreign key on `RiverSection` — not through a list stored on `River`. Prisma exposes a virtual `sections` relation field on the River model that resolves to the associated sections at query time via a join. It is not a database column.

```prisma
model River {
  id          Int            @id @default(autoincrement())
  name        String
  state       String
  description String?
  sections    RiverSection[] // virtual relation — not a column, resolved by Prisma via river_id FK
  created_at  DateTime       @default(now())
}
```

#### RiverSection
A specific fishable stretch of a river with its own USGS gauge station. Belongs to one River via `river_id`. The `ideal_*` fields define the thresholds used to compute the fishability score displayed on the map.

```prisma
model RiverSection {
  id              Int            @id @default(autoincrement())
  river_id        Int
  river           River          @relation(fields: [river_id], references: [id])
  name            String
  usgs_site_code  String
  latitude_start  Float
  longitude_start Float
  latitude_end    Float
  longitude_end   Float
  target_species  String[]
  ideal_gauge_min Float?
  ideal_gauge_max Float?
  ideal_temp_min  Float?
  ideal_temp_max  Float?
  readings        StreamReading[]
  hatch_reports   HatchReport[]
  catch_logs      CatchLog[]
  created_at      DateTime       @default(now())
}
```

#### StreamReading
A point-in-time reading from USGS for a specific river section. The ingestion cron job writes these on a schedule.

Fields: `id`, `section_id`, `gauge_height_ft`, `discharge_cfs`, `temperature_c`, `reading_timestamp`, `ingested_at`

#### HatchReport
A community-submitted insect hatch observation.

Fields: `id`, `user_id`, `section_id`, `insect_species`, `intensity` (enum: light/moderate/heavy), `time_of_day` (enum: morning/midday/afternoon/evening), `report_date`, `notes`, `created_at`

#### CatchLog
A user's personal catch record with a permanently frozen conditions snapshot and full fly/location detail. The snapshot is immutable after creation — it reflects exactly what conditions were present on the day of the catch, regardless of what happens to the live data later.

Fields:
- `id`, `user_id`, `section_id`, `catch_date`, `created_at`
- `caught_species` (string)
- `method` (enum: dry/nymph/streamer/wet/other)
- `fly_pattern` (string — e.g. "Elk Hair Caddis", "Woolly Bugger #6")
- `fly_size` (integer — hook size)
- `location_name` (string, optional — user-provided spot name like "below the log jam")
- `location_lat` (float, optional)
- `location_lng` (float, optional)
- `photo_url` (string, optional)
- `notes` (string, optional)
- `conditions_snapshot` (JSON, frozen at time of entry):
  ```
  {
    gauge_height_ft: number,
    discharge_cfs: number,
    water_temp_f: number,
    weather_summary: string,
    air_temp_f: number,
    wind_speed_mph: number,
    wind_direction: string,
    sky_cover: enum (clear/partly_cloudy/overcast),
    active_hatches: [{ insect_species, intensity, time_of_day }]
  }
  ```

The `active_hatches` array is populated at log time by querying HatchReports for this section within 24 hours of the catch date. The user can confirm or dismiss suggested hatches before saving.

#### Alert — TRIAGED (Future Scope)
Schema preserved for future implementation. No Alert-related code will be built in the initial version.

Fields: `id`, `user_id`, `section_id`, `condition_type` (enum: gauge_height/discharge/temperature), `operator` (enum: above/below), `threshold_value`, `is_active`, `last_triggered_at`, `created_at`

#### User
Managed primarily by Clerk. The local `User` table stores only what the application needs beyond what Clerk provides.

Fields: `id` (matches Clerk user ID), `email`, `display_name`, `home_state`, `created_at`

---

### 2.4 External Interfaces

| Interface | Provider | Access Pattern | Used For | Status |
|---|---|---|---|---|
| Stream gauge data | USGS Water Services API | Server-side, cron + on-demand | Gauge height, discharge, temperature readings | In scope |
| Weather forecasts | OpenWeatherMap API | Server-side, cached in Redis | Current weather and forecast per river section | In scope |
| Fish stocking data | State wildlife agency APIs/CSVs | Cron, varies by state | Stocking activity near river sections | In scope (one state initially) |
| AI assistant | Anthropic Claude API (non-streaming initially) | Server-side API route | Natural language trip planning queries | In scope — streaming deferred |
| Email delivery | Resend | Server-side, triggered by cron | Alert notifications | Triaged with alerts |
| Authentication | Clerk | SDK in middleware and server components | User identity, session management | In scope |
| Object storage | Cloudflare R2 or Vercel Blob | Server-side API route | Catch log photo uploads | In scope |

---

### 2.5 Constraints and Scope Limits

- The application does not provide real-time (sub-minute) data. USGS readings are pulled on a schedule (every hour). This is appropriate — river conditions do not change meaningfully minute to minute.
- Fish stocking data is highly state-dependent. The initial build targets one state with an accessible stocking API. Expanding to additional states is additive and is not a launch blocker.
- The hatch calendar relies entirely on user-submitted data. The application does not attempt to predict hatches algorithmically.
- The AI assistant is advisory only. It does not know about private fishing access, current fish populations, or real-time conditions beyond what is in the database.
- The AI assistant returns a complete response (non-streaming) in the initial build. Streaming is an enhancement, not a core requirement, and will be added in a later iteration.
- Photo uploads are optional, limited to one photo per catch entry.
- Alert notifications are not part of the initial build. The Alert data model is defined but no Alert-related routes, UI, or email logic will be implemented until a later phase.
- Catch location (lat/lng pin) is optional. Users who do not want to record a precise location can use the free-text spot name field or leave it blank.

---

## 3. Tech Stack

### Languages and Runtime

| Layer | Language | Rationale |
|---|---|---|
| Frontend | TypeScript + React | You already know TypeScript; React is the most hireable frontend skill |
| Backend | TypeScript (Next.js API routes) | Sharing one language across the full stack eliminates context switching and lets types be shared between frontend and backend |
| Database queries | SQL via Prisma | Prisma generates TypeScript types from your schema — your queries are type-safe end to end |
| Styling | CSS via TailwindCSS | Utility-first, no context switching to separate stylesheets, excellent with component libraries |

### Core Framework and Libraries

| Package | Role |
|---|---|
| `next` (App Router) | Full-stack framework. Server components handle data fetching; client components handle interactivity |
| `react` | UI component library (bundled with Next.js) |
| `tailwindcss` | Styling |
| `shadcn/ui` | Pre-built, accessible UI components (buttons, modals, dropdowns, forms) built on Radix UI primitives — not a dependency you install, but a set of components you copy into your codebase and own |
| `prisma` | TypeScript ORM. Defines your database schema in a schema file and generates a fully-typed client |
| `@clerk/nextjs` | Authentication SDK. Handles sign-up, login, sessions, and middleware with minimal boilerplate |
| `anthropic` | Official Anthropic SDK. Used for the Claude API integration in the AI assistant |
| `leaflet` + `react-leaflet` | Interactive map for the river section browser |
| `recharts` | Charting library for gauge height and temperature trend charts |
| `ioredis` | Redis client for server-side caching of USGS and weather data |
| `zod` | Schema validation for API route inputs — ensures malformed data never reaches your database |
| `date-fns` | Date manipulation for the hatch calendar, trend charts, and condition timestamps |

### Infrastructure

| Service | Role | Why |
|---|---|---|
| Vercel | Hosting the Next.js application | Designed for Next.js, one-click GitHub deploy, automatic preview deployments for every branch, Vercel Cron for scheduled jobs (free tier) |
| Railway | PostgreSQL database + Redis | Managed PostgreSQL and Redis with a simple deployment dashboard, generous free tier, no Kubernetes required |
| Cloudflare R2 or Vercel Blob | Photo storage for catch log | S3-compatible object storage, free egress (Cloudflare) or zero-config (Vercel Blob) |
| Clerk | Authentication provider | Handles the hardest parts of auth (sessions, OAuth, email verification) so you do not build it yourself |

### Development Tooling

| Tool | Role |
|---|---|
| `eslint` + `prettier` | Code linting and formatting |
| `vitest` | Unit testing (fast, TypeScript-native, compatible with Next.js) |
| `@testing-library/react` | Component testing |
| `prisma studio` | Local GUI for browsing your database during development |
| GitHub Actions | CI pipeline — run tests and lint on every push |

### AI Development Stack

| Tool | Role in Development |
|---|---|
| Claude Code (this tool) | Primary code generation, multi-file changes, debugging, test writing |
| Cursor | In-editor completions and quick single-file work |
| Claude.ai (web) | Architecture discussions, article drafting, prompt engineering for the AI assistant feature |
| Claude API (`anthropic` SDK) | The AI feature built into the product itself |

---

## 4. System Architecture

### 4.1 High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser (User)                               │
│                                                                      │
│   River browser · Hatch calendar · Catch log · AI assistant        │
└───────────────────────────────┬─────────────────────────────────────┘
                                │  HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Next.js Application (Vercel)                      │
│                                                                      │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐   │
│  │    Server Components     │    │        API Routes             │   │
│  │                         │    │                              │   │
│  │  Fetch data at request  │    │  /api/catches    (CRUD)      │   │
│  │  time on the server.    │    │  /api/hatches    (CRUD)      │   │
│  │  No client JS needed    │    │  /api/ai         (query)     │   │
│  │  for initial render.    │    │  /api/conditions (proxy)     │   │
│  │                         │    │  /api/cron/sync  (cron job)  │   │
│  └─────────────────────────┘    └──────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Middleware (Clerk)                          │   │
│  │   Checks session token on every request. Blocks             │   │
│  │   unauthenticated access to protected routes.               │   │
│  └─────────────────────────────────────────────────────────────┘   │
└──────────┬─────────────────────────────────┬────────────────────────┘
           │                                 │
           ▼                                 ▼
┌──────────────────────┐         ┌──────────────────────────────────┐
│  PostgreSQL           │         │  Redis Cache (Railway)           │
│  (Railway)            │         │                                  │
│                      │         │  Stream readings cached by       │
│  Users               │         │  section ID, TTL = 1 hour        │
│  Rivers              │         │                                  │
│  RiverSections       │         │  Weather forecasts cached by     │
│  StreamReadings      │         │  lat/lng, TTL = 3 hours          │
│  HatchReports        │         │                                  │
│  CatchLogs           │         └──────────────────────────────────┘
└──────────────────────┘
           │
           │  (written by cron job)
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     External Data Sources                            │
│                                                                      │
│   USGS Water Services API    ·    OpenWeatherMap API                 │
│   State Stocking APIs        ·    Anthropic Claude API               │
│   Clerk (auth)               ·    Cloudflare R2 / Vercel Blob        │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 4.2 Data Flow Patterns

**Pattern 1: Viewing a river section page (read, no user action)**

```
Browser requests /rivers/[sectionId]
        │
        ▼
Next.js Server Component runs on Vercel server
        │
        ├── Query PostgreSQL for section metadata, recent hatch reports
        │
        ├── Check Redis: does a cached stream reading exist for this section?
        │       ├── YES (cache hit, < 1hr old) → use cached reading
        │       └── NO (cache miss) → fetch from USGS API → write to Redis → use reading
        │
        ├── Check Redis: does a cached weather forecast exist for this section's lat/lng?
        │       ├── YES → use cached forecast
        │       └── NO → fetch from OpenWeatherMap → write to Redis → use forecast
        │
        └── Render page HTML with all data → sent to browser
            (no client-side data fetching needed for initial load)
```

**Pattern 2: Cron job — hourly condition sync**

```
Vercel Cron triggers /api/cron/sync-conditions (once per hour)
        │
        ├── Load all RiverSections from PostgreSQL
        │
        ├── For each section:
        │       ├── Fetch latest reading from USGS Water Services API
        │       ├── Write StreamReading record to PostgreSQL
        │       └── Invalidate Redis cache for this section
        │
        └── Log sync completion with section count and any errors
```

**Pattern 3: User logs a catch**

```
User submits catch log form on the client
        │
        ▼
POST /api/catches (Next.js API route)
        │
        ├── Clerk middleware confirms user is authenticated
        ├── Zod validates request body shape and field constraints
        │
        ├── Fetch most recent StreamReading for this section from PostgreSQL
        ├── Fetch current weather for this section from Redis/OpenWeatherMap
        │
        ├── Assemble conditions_snapshot JSON from readings + weather
        │
        ├── If photo included: upload to Cloudflare R2, get URL
        │
        └── Write CatchLog record to PostgreSQL with conditions_snapshot
            → Return 201 to client → UI optimistically updates
```

**Pattern 4: User queries the AI assistant**

```
User types query and submits in the AI assistant panel
        │
        ▼
POST /api/ai (Next.js API route, complete response)
        │
        ├── Clerk middleware confirms user is authenticated
        ├── Load river section metadata and current conditions from DB/Redis
        ├── Load user's catch history for this section (last 10 entries with snapshots)
        ├── Load community hatch reports for this section (last 30 days)
        ├── Assemble structured context document from all the above
        │
        ├── Call Anthropic Claude API with:
        │       system: role definition + data context
        │       user: the angler's question
        │       stream: false
        │
        └── Await complete response → return as JSON
            → Client exits loading state and renders full response text
```

---

### 4.3 Next.js App Router Structure

The App Router uses a folder-based routing system. Understanding this structure is important before starting development.

```
app/
│
├── layout.tsx                    # Root layout — wraps entire app, applies fonts, Clerk provider
├── page.tsx                      # Landing/home page (unauthenticated accessible)
│
├── (auth)/                       # Route group — auth pages share a layout without sidebar
│   ├── sign-in/[[...sign-in]]/
│   │   └── page.tsx              # Clerk sign-in component
│   └── sign-up/[[...sign-up]]/
│       └── page.tsx              # Clerk sign-up component
│
├── (app)/                        # Route group — all authenticated app pages share a layout
│   ├── layout.tsx                # App shell layout: sidebar nav, header, user menu
│   │
│   ├── rivers/
│   │   ├── page.tsx              # River list + map overview (server component)
│   │   └── [sectionId]/
│   │       └── page.tsx          # Individual river section page (server component)
│   │
│   ├── log/
│   │   ├── page.tsx              # Catch log list view (server component)
│   │   └── new/
│   │       └── page.tsx          # New catch log form (client component)
│   │
│   └── hatches/
│       └── page.tsx              # Hatch calendar view (client component — needs interactivity)
│
└── api/                          # API routes (server-side only, never exposed to browser directly)
    ├── catches/
    │   └── route.ts              # GET (list), POST (create)
    ├── catches/[id]/
    │   └── route.ts              # PATCH (edit), DELETE
    ├── hatches/
    │   └── route.ts              # GET, POST
    ├── hatches/[id]/
    │   └── route.ts              # PATCH, DELETE
    ├── ai/
    │   └── route.ts              # POST — assembles context, calls Claude API, returns complete response
    ├── conditions/
    │   └── route.ts              # GET — on-demand condition fetch (used by map refresh)
    └── cron/
        └── sync/
            └── route.ts          # POST — triggered by Vercel Cron on schedule
```

---

### 4.4 Library Modules (`lib/`)

These are not React components or API routes — they are plain TypeScript modules used by server components and API routes. They have no UI concerns.

```
lib/
│
├── prisma.ts        # Exports a singleton Prisma client instance. Prevents connection pool
│                    # exhaustion during development hot reloads.
│
├── usgs.ts          # USGS Water Services API client.
│                    # Functions: fetchCurrentReading(siteCode), fetchHistoricalReadings(siteCode, days)
│                    # Returns typed objects — never raw API responses.
│
├── weather.ts       # OpenWeatherMap API client.
│                    # Functions: fetchCurrentWeather(lat, lng), fetchForecast(lat, lng)
│                    # Returns typed objects.
│
├── redis.ts         # Exports a singleton ioredis client.
│                    # Functions: getCachedReading(sectionId), setCachedReading(sectionId, data, ttl)
│                    # getCachedWeather(lat, lng), setCachedWeather(...)
│
├── claude.ts        # Anthropic Claude API client.
│                    # Functions: buildConditionsContext(section, conditions, hatches, catches)
│                    # sendAssistantQuery(context, userQuery) → returns complete response string
│
├── conditions.ts    # Business logic for computing fishability scores.
│                    # Functions: computeFishabilityScore(section, reading) → 'good' | 'fair' | 'poor'
│
└── storage.ts       # Cloudflare R2 / Vercel Blob client.
                     # Functions: uploadCatchPhoto(file) → url
```

---

### 4.5 Component Organization (`components/`)

```
components/
│
├── ui/              # shadcn/ui components (Button, Card, Dialog, Input, Select, etc.)
│                    # These are generated by the shadcn CLI and owned by your codebase.
│                    # Do not modify them directly — extend them in the components above.
│
├── map/
│   ├── RiverMap.tsx             # Main Leaflet map. Client component (Leaflet requires browser APIs).
│   │                            # Renders river section markers colored by fishability score.
│   └── SectionMarker.tsx        # Individual marker component with popup.
│
├── charts/
│   ├── GaugeChart.tsx           # Recharts line chart for gauge height over 7 days.
│   └── TemperatureChart.tsx     # Recharts line chart for water temperature over 7 days.
│
├── conditions/
│   ├── ConditionsPanel.tsx      # Current conditions summary card for a river section.
│   └── FishabilityBadge.tsx     # Green/yellow/red badge showing fishability score.
│
├── hatches/
│   ├── HatchCalendar.tsx        # Monthly calendar view. Client component.
│   ├── HatchReportForm.tsx      # Form for submitting a new hatch report.
│   └── HatchReportCard.tsx      # Individual report display.
│
├── catches/
│   ├── CatchLogList.tsx         # Paginated list of catch entries.
│   ├── CatchLogCard.tsx         # Individual catch display with conditions snapshot.
│   └── CatchLogForm.tsx         # Form for logging a new catch. Client component.
│
└── ai/
    └── AssistantPanel.tsx       # AI chat panel. Client component — submits query and renders complete response.
```

---

## 5. Repository and Configuration Files

### Root-Level Files

```
hatch-journal/
│
├── app/                         # Next.js App Router (see above)
├── components/                  # React components (see above)
├── lib/                         # Server-side utility modules (see above)
├── prisma/
│   ├── schema.prisma            # Database schema — single source of truth for all models
│   └── seed.ts                  # Seeds rivers and sections with real data for development
├── public/                      # Static assets (logo, favicon, map marker icons)
├── types/
│   └── index.ts                 # Shared TypeScript types used across frontend and backend
├── tests/
│   ├── lib/                     # Unit tests for lib/ modules (usgs, conditions, claude context builder)
│   └── components/              # Component tests for forms and key UI
│
├── .env.local                   # Local environment variables (gitignored)
├── .env.example                 # Documents required environment variables (committed)
├── .gitignore
├── next.config.ts               # Next.js configuration
├── tailwind.config.ts           # Tailwind configuration
├── tsconfig.json                # TypeScript configuration
├── vitest.config.ts             # Vitest test configuration
├── middleware.ts                # Clerk auth middleware — runs on every request
└── package.json
```

### Environment Variables (`.env.example`)

```bash
# Database
DATABASE_URL="postgresql://..."
REDIS_URL="redis://..."

# Auth
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="pk_..."
CLERK_SECRET_KEY="sk_..."
CLERK_WEBHOOK_SECRET="whsec_..."

# External APIs
USGS_BASE_URL="https://waterservices.usgs.gov/nwis"
OPENWEATHERMAP_API_KEY="..."
ANTHROPIC_API_KEY="sk-ant-..."

# Storage
BLOB_READ_WRITE_TOKEN="..."

# Cron security
CRON_SECRET="..."
```

---

## 6. AI Integration — Detailed Design

The Claude integration is the most article-worthy part of the project and deserves more detailed design up front than any other feature.

### What Gets Sent to Claude

When a user submits a query, the API route builds a structured context document before making any API call. This is the most important engineering decision in the AI layer — the quality of the responses depends almost entirely on the quality of the context.

**System prompt (defines role and constraints):**
```
You are a fly fishing trip planning assistant with access to real-time river
conditions, community hatch reports, and the user's personal catch history for
the river section they are viewing.

Your job is to give practical, specific advice based on the data provided.
Do not speculate beyond the data. If conditions data is missing or stale,
say so. Do not recommend fishing a section if the gauge is unsafe.

You have no knowledge of private fishing access or current fish populations
beyond what the catch history implies.
```

**User message (assembled from structured data):**
```
RIVER SECTION: Madison River — Cameron to Varney Bridge (MT)
TARGET SPECIES: Brown Trout, Rainbow Trout

CURRENT CONDITIONS (as of [timestamp]):
- Gauge height: 3.2 ft (USGS site 06043500)
- Discharge: 892 CFS
- Water temperature: 58°F
- Weather: Partly cloudy, high 67°F, wind 8mph SW
- Fishability: GOOD (ideal range: 600-1200 CFS, 50-65°F)

7-DAY TREND:
- Gauge has dropped from 4.1 ft to 3.2 ft over 7 days (falling, improving)
- Temperature has risen from 52°F to 58°F (rising, approaching ideal)

RECENT HATCH REPORTS (last 30 days, this section):
- Apr 1: PMD hatch, moderate intensity, afternoon
- Mar 28: Blue Winged Olive, heavy, morning
- Mar 28: Blue Winged Olive, moderate, afternoon
- Mar 22: Midge, light, morning

USER'S CATCH HISTORY (this section, most recent first):
- Mar 15, 2025: Brown Trout, dry fly, gauge 3.8ft, 54°F, overcast
- Feb 2, 2025: Rainbow Trout, nymph, gauge 4.2ft, 46°F, clear
- Oct 12, 2024: Brown Trout, streamer, gauge 2.9ft, 61°F, sunny

USER QUESTION: Should I fish this weekend? What should I bring?
```

### Why This Approach Is Worth Writing About

- **Context window management:** What happens when a user has 200 catch records? The context builder needs to select and summarize intelligently, not dump everything.
- **Data freshness disclosure:** Claude is told the timestamp of the data and is instructed to disclose if data is stale. Watching how it handles this in practice is interesting.
- **Domain accuracy:** Claude has general fly fishing knowledge from training, but the hatch data and catch history are authoritative local signals. When they conflict, which does Claude weight?
- **Structured vs. prose context:** This implementation uses a structured document format. Would a JSON format produce better reasoning? This is an experiment worth running and writing about.

### Response Handling (v1.0 — Non-Streaming)

The `/api/ai` route calls the Anthropic SDK with `stream: false`, awaits the complete response, and returns it as a standard JSON response body. The `AssistantPanel` client component sends the POST request, enters a loading state while waiting, and renders the full response text when it arrives.

This approach is intentionally simpler than streaming for the initial build. It avoids the complexity of `TransformStream`, `ReadableStream` handling in the browser, and custom React hooks for assembling token chunks. The trade-off is that the user sees no output until the full response is ready, which may feel slow for longer answers.

Upgrading to streaming is planned as a future iteration and a dedicated article topic. The upgrade will require:
- Switching to `stream: true` in the Anthropic SDK call
- A `TransformStream` to convert the SDK's stream format to a browser-readable stream
- A custom React hook (`useStreamingResponse`) to assemble incoming tokens into state
- Error handling for mid-stream failures

No schema or API route signature changes are required for the streaming upgrade — it is an internal implementation change only.

---

## 7. Development Phase Map

The phases below map to the build order that minimizes blocked work — each phase produces something functional before moving to the next.

| Phase | Focus | Produces |
|---|---|---|
| 0 | Study USGS API docs, explore raw JSON responses, understand the data shape | Knowledge needed to design the schema correctly |
| 1 | Project setup: Next.js, Tailwind, Clerk, Prisma, Railway databases, environment variables | A running app with auth that deploys to Vercel |
| 2 | Prisma schema, seed data (rivers + sections), Prisma Studio exploration | A populated database with real river data |
| 3 | USGS and weather lib modules, Redis caching, `/api/conditions` route | Server-side data fetching that works correctly |
| 4 | River browser UI: list view, section page with conditions panel and charts | A read-only, data-displaying app a user can navigate |
| 5 | Leaflet map integration | The river map with fishability markers |
| 6 | Catch log: form, API route, list view with conditions snapshot | Core personal logging feature end-to-end |
| 7 | Hatch calendar: form, API route, calendar view with filtering | Community hatch data entry and display |
| 8 | AI assistant: context builder in lib/claude.ts, /api/ai route, AssistantPanel component | The natural language trip planning feature |
| 9 | Polish: mobile responsiveness, error states, loading states, empty states, graceful degradation | A shippable, production-quality UI |
| 10 | Tests: lib unit tests, key component tests, GitHub Actions CI | Confidence and a CI badge for the README |
| 11 | README, custom domain, production deploy review | A live, publicly accessible application |
