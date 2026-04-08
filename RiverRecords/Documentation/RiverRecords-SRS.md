# Software Requirements Specification for
# RiverRecords

**Version 1.0 approved**
**Prepared by Finn Gidden**
**gidde032**
**4/5/26**

---

## Table of Contents

1. Introduction
   - 1.1 Purpose
   - 1.2 Document Conventions
   - 1.3 Intended Audience and Reading Suggestions
   - 1.4 Product Scope
   - 1.5 References
2. Overall Description
   - 2.1 Product Perspective
   - 2.2 Product Functions
   - 2.3 User Classes and Characteristics
   - 2.4 Operating Environment
   - 2.5 Design and Implementation Constraints
   - 2.6 User Documentation
   - 2.7 Assumptions and Dependencies
3. Tech Stack
4. System Features
   - 4.1 River and Conditions Browser
   - 4.2 Community Hatch Calendar
   - 4.3 Personal Catch Log
   - 4.4 AI Trip Assistant
   - 4.5 Authentication and User Accounts
5. AI Integration - Detailed Design
6. System Architecture and Repository Structure
7. Nonfunctional Requirements
8. Development Phase Map
9. Other Requirements
- Appendix A: Glossary
- Appendix B: Analysis Models
- Appendix C: To Be Determined List
- Appendix D: Planned Future Features and Improvements

---

## Revision History

| Name | Date | Reason For Changes | Version |
|---|---|---|---|
| Finn Gidden | 4/5/26 | Initial document creation | 1.0 |

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification describes the functional and non-functional requirements for RiverRecords, version 1.0. RiverRecords is a full-stack web application designed to serve fly fishing anglers with automated river condition monitoring, community hatch reporting, a personal catch log with immutable conditions snapshots, and an AI-powered trip planning assistant.

This document covers the complete scope of the v1.0 initial build. Two feature areas: condition-based alert notifications and personal conditions analysis have been defined and scoped but explicitly deferred to future versions. Their data models are included in this document so that their future implementation requires no schema changes.

### 1.2 Document Conventions

The following conventions are used throughout this document:

- **FR-N** refers to a numbered functional requirement. Requirements are numbered sequentially within each feature section and are unique across the document.
- **NFR-N** refers to a numbered non-functional requirement.
- **REQ-N** tags within Section 4 (System Features) provide a secondary unique identifier for functional requirements as required by the IEEE SRS format. These correspond to and can be cross-referenced against FR numbers in the broader spec.
- **TRIAGED** marks features that have been formally defined but deferred from the v1.0 build. Triaged features retain their data model definitions so future implementation requires no schema migration.
- **TBD** marks items where a decision or value has not yet been finalized. All TBDs are collected in Appendix C.
- Priority ratings for each system feature use the scale: **High** (required for launch), **Medium** (important but launch does not depend on it), **Low** (enhancement).
- Higher-level requirements do not automatically inherit priority to sub-requirements. Each REQ item carries its own implicit High priority unless otherwise stated.

### 1.3 Intended Audience and Reading Suggestions

This document is written primarily for the solo developer building and maintaining RiverRecords. Secondary audiences include any collaborators, code reviewers, or future contributors who need to understand the system's intended behavior.

**For the developer (primary):** Read the document end to end before beginning implementation. Section 2 provides the product context and constraints that inform every implementation decision. Section 4 is the authoritative reference for feature behavior during development; return to it and Section 5 frequently, as they define acceptance criteria for each feature. Finally, Section 8 provides a general outline for the development of RiverRecords v1.0, ensuring that the system's creation is organized and each step ends with a functional, testable artifact.

**For technical reviewers or collaborators:** Begin with Section 1.4 (Product Scope) and Section 2.1 (Product Perspective) for context, then read Section 4 (System Features) for functional detail. Section 6 (System Architecture) provides the structural overview.

**For anyone evaluating the project:** Section 1.4 and Section 2.2 give a concise picture of what the product does. Section 3 (Tech Stack) describes the technology decisions. Appendix D lists planned future improvements.

### 1.4 Product Scope

RiverRecords is a web application for fly anglers that eliminates the need to cross-reference multiple tools before a fishing trip. It aggregates real-time river condition data from public government APIs (USGS, NOAA), hosts community-submitted insect hatch observations, and provides each user with a personal catch log that permanently records the conditions present on the day of each catch. Furthermore, each river is cached and broken into sections that allow users to examine past catch logs and hatch records; this enables anglers to precisely pinpoint their most productive areas for each river.

The application's core value proposition has two components. First, it replaces the pre-trip research workflow, as stream gauge levels, water temperature, weather, and recent hatch activity are synthesized in one place, enabling users to change plans and locations quickly. Second, it creates a personal catch log and conditions database for each angler: over time, the catch log reveals what conditions correlate with productive fishing on specific river sections, which no external tool can provide because the data is unique to each user. Once enough hatches and catches have been logged, RiverRecords can be used as a journal for past experiences, as well as a powerful predictive tool to improve fishing yield.

An AI assistant powered by the Anthropic Claude API accepts natural language queries (e.g., "should I fish the Rush this weekend?") and responds using the live conditions data, community hatch reports, and the user's own catch history as context, giving advice grounded in real, current data rather than generic recommendations.

### 1.5 References

| Document / Resource | Description | Location |
|---|---|---|
| hatch-journal-spec.md | Primary design specification and architecture document for RiverRecords | `/RiverRecords/hatch-journal-spec.md` |
| USGS Water Services REST API Documentation | API reference for stream gauge, discharge, and temperature data | https://waterservices.usgs.gov/docs/ |
| OpenWeatherMap API Documentation | API reference for weather conditions and forecasts | https://openweathermap.org/api |
| Anthropic Claude API Documentation | API reference for the Claude language model used in the AI assistant | https://docs.anthropic.com |
| Clerk Documentation | SDK and configuration reference for authentication | https://clerk.com/docs |
| Prisma Documentation | ORM schema definition, migrations, and query API | https://www.prisma.io/docs |
| Next.js App Router Documentation | Framework documentation for routing, server components, and API routes | https://nextjs.org/docs |
| IEEE Std 830-1998 | IEEE Recommended Practice for Software Requirements Specifications | IEEE Standards |

---

## 2. Overall Description

### 2.1 Product Perspective

RiverRecords is a new, self-contained product with no predecessor system. It does not replace an existing internal tool and is not a component of a larger platform. It enters a market where anglers currently rely on a fragmented collection of standalone tools, such as USGS WaterWatch, weather applications, state wildlife agency websites, and community forums such as Reddit and Fishing Reports, none of which are integrated with each other or with a user's personal fishing history.

The product is a web application hosted on Vercel, backed by a PostgreSQL database and Redis cache on Railway, and authenticated via Clerk. It communicates with four categories of external systems:

- **Government data APIs** (USGS, NOAA, state wildlife agencies) for automated condition ingestion
- **An AI API** (Anthropic Claude) for natural language query processing
- **Infrastructure services** (Railway, Vercel, Cloudflare R2, Clerk) for hosting, storage, and auth
- **The user's browser** for all UI interaction

```
┌──────────────────────────────────────────────────────────────────┐
│                      User's Browser                              │
└──────────────────────────────┬───────────────────────────────────┘
                               │ HTTPS
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│              Next.js Application (Vercel)                        │
│   Server Components · API Routes · Clerk Middleware              │
└──────┬───────────────────────────────────────┬───────────────────┘
       │                                       │
       ▼                                       ▼
┌─────────────────┐                 ┌──────────────────────┐
│  PostgreSQL     │                 │  Redis Cache         │
│  (Railway)      │                 │  (Railway)           │
└─────────────────┘                 └──────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    External Services                             │
│  USGS API · OpenWeatherMap · Anthropic Claude · Clerk · R2       │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Product Functions

The following is a high-level summary of the major function groups. Detailed requirements for each are specified in Section 4.

**River and Conditions Browser**
- Display a browsable list of pre-seeded rivers and river sections
- Show current gauge height, discharge, and water temperature from USGS for each section
- Display 7-day historical trend charts for gauge and temperature
- Show current weather forecast per section
- Render all river sections on an interactive map with color-coded fishability markers
- Cache condition data to minimize external API calls

**Community Hatch Calendar**
- Allow authenticated users to submit insect hatch observations (species, intensity, time of day)
- Display community submissions in a monthly calendar view per river section
- Support filtering by insect species to reveal historical hatch timing patterns
- Allow users to edit and delete their own submissions

**Personal Catch Log**
- Allow authenticated users to record catches with full detail: species, method, fly pattern, fly size, location, and notes
- Automatically snapshot and permanently store all conditions present at time of logging (gauge, temperature, weather, wind, active hatches)
- Display the full conditions snapshot alongside each catch entry
- Support filtering by river, section, species, method, fly pattern, and date range
- Auto-associate hatch reports from the same date for user confirmation

**AI Trip Assistant**
- Accept natural language queries from authenticated users about a specific river section
- Assemble a structured context document from current conditions, 7-day trend, recent hatch reports, and the user's personal catch history for that section
- Return a complete response from the Claude API grounded in the provided data
- Decline to make recommendations when data is missing or stale rather than speculating

**Authentication and User Accounts**
- Support sign-up and login via email/password and Google OAuth through Clerk
- Allow unauthenticated visitors read-only access to conditions and hatch data
- Require authentication for all write operations

### 2.3 User Classes and Characteristics

**Authenticated Angler (Primary)**
The core user. Fishes regularly, plans trips based on conditions, and wants a persistent log of their fishing history connected to the conditions that were present. Comfortable with web applications. Not assumed to have technical knowledge beyond normal consumer software use. This user class accesses all features including the catch log, hatch reporting, and AI assistant. The product is designed and optimized for this class.

**Unauthenticated Visitor (Secondary)**
A user who has not created an account. Can browse the river conditions browser and view community hatch reports in read-only mode. Cannot submit hatch reports, log catches, or access the AI assistant. This class is important for initial discoverability, as a visitor should be able to assess the product's value before committing to an account.

**Developer / Administrator (Tertiary)**
The sole developer building and maintaining the application. Has direct database access via Prisma Studio and can modify the river and section seed data. No dedicated admin UI is required in v1.0. Database management is performed via direct tooling (Prisma CLI, Railway dashboard).

### 2.4 Operating Environment

**Client-side:** Any modern web browser with JavaScript enabled (Chrome 114+, Firefox 115+, Safari 16+, Edge 114+). The application must be fully functional on mobile browsers given that anglers frequently check conditions from their phones in the field.

**Server-side:** Node.js 20 LTS runtime on Vercel's serverless edge infrastructure. The Next.js application is deployed as serverless functions; there is no persistent server process.

**Database:** PostgreSQL 15 hosted on Railway. Accessed exclusively through the Prisma ORM, no raw SQL is written in application code.

**Cache:** Redis 7 hosted on Railway. Accessed through the ioredis client. Used for condition data caching only, session management is handled by Clerk.

**Scheduled jobs:** Vercel Cron (free tier) triggers the condition sync endpoint on an hourly schedule.

**External API dependencies:** The application's condition data depends on the availability of USGS Water Services API and OpenWeatherMap API. The AI assistant depends on Anthropic API availability. Authentication depends on Clerk service availability. Partial degradation behavior is defined in NFR-6.

### 2.5 Design and Implementation Constraints

- **Language:** The entire application, frontend, backend, and database interaction, is written in TypeScript. No JavaScript files. No Python or other server-side language.
- **Framework:** Next.js App Router is the only framework in use. Page Router patterns are not used. All routing follows App Router conventions (server components by default, client components explicitly opted in with `"use client"`).
- **ORM:** All database interaction goes through Prisma. Raw SQL queries are not permitted in application code except in migration files.
- **External API calls from browser:** USGS, OpenWeatherMap, and Anthropic API keys must never be exposed to the client. All calls to these services are made from server components or API routes only.
- **Authentication:** Clerk is the sole authentication provider. Custom session management is not implemented.
- **Data freshness:** USGS condition data is not real-time. The system architecture explicitly accepts up to 1-hour staleness for cached readings. This is a design decision, not a deficiency.
- **Scope:** The v1.0 build targets rivers in a single US state for the fish stocking data integration. USGS and weather coverage applies nationally but seed data will be limited to a manageable initial set of river sections.
- **AI response mode:** The AI assistant returns a complete (non-streaming) response in v1.0. Streaming is explicitly deferred to a future iteration.
- **Single developer:** No CI/CD gates beyond GitHub Actions lint and test. No code review workflow. No staging environment required beyond Vercel preview deployments.

### 2.6 User Documentation

The following documentation will be delivered alongside v1.0:

**GitHub README** — The primary user-facing documentation. Contains: one-sentence product description, live application URL, feature summary, and screenshot or demo GIF. No setup instructions are needed for end users since the app is web-hosted.

**Developer README** (within the same file, separated by a heading) — Local development setup: prerequisites (Node.js version, environment variables), database setup with Prisma, and how to run the development server. Intended for any future contributor.

**In-application empty states** — Each feature section (catch log, hatch calendar) displays descriptive empty states when a user has no data yet, explaining how to get started. These serve as inline onboarding.

No formal user manual, tutorial series, or help documentation is planned for v1.0.

### 2.7 Assumptions and Dependencies

**Assumptions:**
- USGS Water Services API remains freely accessible without authentication for the gauge, discharge, and temperature endpoints used. A change to this policy would require an alternative data source.
- OpenWeatherMap's free tier (1,000 calls/day) is sufficient for the expected number of river sections and cache miss rate in the initial build. If section count or traffic grows significantly, a paid tier will be required.
- Vercel's free tier Cron supports the required 1-hour ingestion schedule. Vercel Cron on the free tier supports one cron job with a minimum interval of 1 hour, which meets the requirement exactly.
- Clerk's free tier (10,000 monthly active users) is more than sufficient for the initial user base.
- Railway's free tier provides sufficient PostgreSQL storage for the expected data volume in the initial build period.
- The Anthropic Claude API is stable and the `anthropic` SDK's interface does not change in a breaking way during development.

**Dependencies:**
- **Prisma** — schema migration and ORM. A breaking change to Prisma's schema syntax or generated client API would require migration of all database interaction code.
- **shadcn/ui** — component library. Components are copied into the codebase and owned locally, so upstream changes do not affect the build unless components are regenerated.
- **@clerk/nextjs** — authentication SDK. Clerk's Next.js integration is tightly coupled to Next.js App Router conventions. A major Next.js version change could affect compatibility.
- **USGS site codes** — the USGS gauge station site codes seeded in the database must be valid and actively reporting. Stations that go offline or are decommissioned will produce stale or absent data for their associated river sections.

---

## 3. Tech Stack

### Languages and Runtime

| Layer | Technology | Rationale |
|---|---|---|
| Frontend | TypeScript + React | Type safety across the UI layer; React is the dominant frontend tech |
| Backend | TypeScript via Next.js API routes | Single language across the full stack; types defined once can be shared between frontend and backend |
| Database queries | SQL via Prisma ORM | Prisma generates a fully type-safe client from the schema; queries are validated at compile time |
| Styling | TailwindCSS | Utility-first approach; no separate stylesheet context; composable with shadcn/ui components |

### Core Framework and Libraries

| Package | Role |
|---|---|
| `next` (App Router) | Full-stack framework. Server components fetch data at request time on the server; client components handle interactivity in the browser |
| `react` | UI rendering (bundled with Next.js) |
| `tailwindcss` | Utility-first CSS framework |
| `shadcn/ui` | Accessible component primitives (Button, Card, Dialog, Input, Select, etc.) built on Radix UI. Components are copied into the codebase via CLI and are owned locally, not a runtime dependency |
| `prisma` | TypeScript ORM. Schema is the single source of truth for all data models. Generates a type-safe client. Manages migrations |
| `@clerk/nextjs` | Authentication SDK. Manages sign-up, sign-in, OAuth, sessions, and middleware integration |
| `anthropic` | Official Anthropic SDK for Claude API calls |
| `leaflet` + `react-leaflet` | Interactive map rendering for the river section browser |
| `recharts` | Chart library for gauge height and temperature trend visualizations |
| `ioredis` | Redis client for server-side caching |
| `zod` | Runtime schema validation for all API route inputs. Prevents malformed data from reaching the database |
| `date-fns` | Date manipulation for calendar views, trend chart ranges, and condition timestamp handling |

### Infrastructure Services

| Service | Role | Rationale |
|---|---|---|
| Vercel | Application hosting, serverless functions, Cron | First-party Next.js hosting; one-click GitHub deploy; preview deployments per branch; Cron on free tier |
| Railway | PostgreSQL + Redis hosting | Managed databases with straightforward setup; generous free tier; no infrastructure management required |
| Cloudflare R2 or Vercel Blob | Catch log photo storage | S3-compatible object storage; free egress (R2) or zero-config integration (Vercel Blob) |
| Clerk | Authentication provider | Handles sessions, OAuth, and email verification; removes auth implementation burden |

### Development Tooling

| Tool | Role |
|---|---|
| `eslint` + `prettier` | Linting and consistent code formatting |
| `vitest` | Unit testing framework (TypeScript-native, fast, compatible with Next.js) |
| `@testing-library/react` | Component testing utilities |
| `prisma studio` | Local database GUI for development inspection and debugging |
| GitHub Actions | CI pipeline, runs lint and test suite on every push and pull request |

### AI Development Stack

| Tool | Role in Development |
|---|---|
| Claude Code | Primary development agent, handles code generation, multi-file changes, debugging, test scaffolding |
| Cursor | In-editor completions and targeted single-file edits |
| Claude.ai (web) | Architectural discussion, prompt engineering, and article drafting |
| Anthropic Claude API (via `anthropic` SDK) | The AI feature integrated into the product |

---

## 4. System Features

### 4.1 River and Conditions Browser

#### 4.1.1 Description and Priority

**Priority: High**

The conditions browser is the entry point to the application and the first feature a new visitor encounters. It aggregates real-time data from USGS and OpenWeatherMap for each pre-seeded river section and presents it in a unified view: list, detail, and map. This feature is entirely read-only and accessible to unauthenticated visitors, making it the primary discoverability surface for the product.

This feature has the highest priority in the build because all other features depend on river section data being present and correct. It must be functional before the hatch calendar, catch log, or AI assistant can be developed or tested meaningfully.

#### 4.1.2 Stimulus/Response Sequences

**Sequence 1: Visitor browses the river list**
1. User navigates to `/rivers`
2. System renders a list of all seeded river sections with name, state, and current fishability badge
3. User selects a river section
4. System navigates to `/rivers/[sectionId]` and renders the section detail page

**Sequence 2: Section detail page loads**
1. Next.js Server Component executes on the Vercel server
2. System queries PostgreSQL for section metadata and the 7-day StreamReading history
3. System checks Redis for a cached current reading for this section
   - Cache hit (< 1 hour old): use cached reading
   - Cache miss: fetch from USGS API, write to Redis, use returned reading
4. System checks Redis for a cached weather forecast for this section's coordinates
   - Cache hit (< 3 hours old): use cached forecast
   - Cache miss: fetch from OpenWeatherMap, write to Redis, use returned forecast
5. System renders the complete page HTML with conditions panel, trend charts, and hatch calendar; no client-side data fetching is required for the initial render
6. Browser displays the fully rendered page

**Sequence 3: User views the map**
1. User navigates to the map view
2. Browser loads the Leaflet map (client component)
3. System renders a marker for each seeded river section
4. Each marker is colored based on the section's current fishability score (green/yellow/red)
5. User clicks a marker; a popup displays the section name and current gauge reading with a link to the detail page

**Sequence 4: USGS data is unavailable**
1. Server component attempts USGS fetch (cache miss path)
2. USGS API returns an error or times out
3. System renders the section page with the most recent cached reading, marked with a staleness warning and the timestamp of the last successful reading
4. Trend charts render with whatever historical data is available

#### 4.1.3 Functional Requirements

- REQ-1 (FR-1): The system shall display a browsable list of rivers and river sections pre-seeded in the database. The list is accessible to unauthenticated users.
- REQ-2 (FR-2): Each river section detail page shall display current gauge height in feet, discharge in CFS, and water temperature in degrees Fahrenheit, sourced from the USGS Water Services API.
- REQ-3 (FR-3): Each river section detail page shall display a 7-day historical trend chart for gauge height and a separate chart for water temperature, rendered using Recharts.
- REQ-4 (FR-4): Each river section detail page shall display the current weather conditions and forecast for the section's geographic coordinates, sourced from OpenWeatherMap.
- REQ-5 (FR-5): Current stream reading data shall be cached in Redis with a TTL of 1 hour per section. Weather forecast data shall be cached with a TTL of 3 hours. Page loads shall read from cache and only call external APIs on a cache miss.
- REQ-6 (FR-6): The river browser shall include a map view rendered with Leaflet in which each seeded river section appears as a marker colored green (conditions within ideal thresholds), yellow (conditions borderline), or red (conditions outside ideal thresholds), based on the section's configured `ideal_gauge_min`, `ideal_gauge_max`, `ideal_temp_min`, and `ideal_temp_max` fields.
- REQ-7: If USGS data is unavailable at render time, the system shall display the most recently cached reading with a clearly visible staleness indicator showing the age of the data. The page shall not error or return a blank state.
- REQ-8: All USGS and OpenWeatherMap API calls shall be made exclusively from server-side code. No API keys shall be exposed to or callable from the browser.

---

### 4.2 Community Hatch Calendar

#### 4.2.1 Description and Priority

**Priority: High**

The hatch calendar is the community contribution layer of the application. It allows authenticated anglers to log insect hatch observations for specific river sections and view historical hatch reports from all users in a calendar format. Over time, this data reveals seasonal hatch timing patterns that are highly valuable for trip planning and fly selection, a service currently not being provided by any existing software.

This feature depends on the conditions browser and authentication being in place, but can be developed independently of the catch log.

#### 4.2.2 Stimulus/Response Sequences

**Sequence 1: Authenticated user submits a hatch report**
1. User navigates to a river section's hatch calendar
2. User clicks "Add Report"
3. System renders the hatch report form (insect species, intensity, time of day, date, optional notes)
4. User completes and submits the form
5. Client sends POST request to `/api/hatches`
6. Server validates user authentication via Clerk middleware
7. Server validates request body via Zod schema
8. Server writes HatchReport record to PostgreSQL
9. Server returns 201; UI updates to show the new report in the calendar

**Sequence 2: User views hatch calendar**
1. User (authenticated or not) views the hatch calendar for a section
2. System queries PostgreSQL for all HatchReports for that section in the selected month
3. System renders reports on the calendar, grouped by date
4. User filters by insect species; system re-renders showing only matching reports

**Sequence 3: User edits or deletes their own report**
1. User clicks edit or delete on a report they submitted
2. System verifies via Clerk that the requesting user matches `report.user_id`
3. Edit: system renders the form pre-populated with existing values; user updates and resubmits (PATCH to `/api/hatches/[id]`)
4. Delete: system prompts for confirmation; on confirm, sends DELETE to `/api/hatches/[id]`; record is removed and calendar updates

**Sequence 4: User attempts to edit another user's report**
1. User crafts a PATCH or DELETE request targeting a hatch report they did not create
2. Server checks that `report.user_id` matches the authenticated user's Clerk ID
3. Server returns 403 Forbidden; no change is made to the database

#### 4.2.3 Functional Requirements

- REQ-9 (FR-7): Authenticated users shall be able to submit a hatch report containing: insect species (free text), intensity (enum: light / moderate / heavy), time of day (enum: morning / midday / afternoon / evening), report date, and optional notes. All fields except notes are required.
- REQ-10 (FR-8): The hatch calendar shall display all community reports for a selected river section in a monthly calendar view, with each report visible on its corresponding date.
- REQ-11 (FR-9): Users shall be able to filter calendar reports by insect species. The filter shall update the calendar display without a full page reload.
- REQ-12 (FR-10): Users shall be able to view, edit, and delete their own previously submitted hatch reports. Editing or deleting another user's report shall return a 403 error.
- REQ-13: Hatch reports shall be associated with both a `user_id` and a `section_id`. A report without either value shall be rejected at the API route level before reaching the database.
- REQ-14: The hatch calendar shall be viewable in read-only mode by unauthenticated users. Submit, edit, and delete controls shall only be rendered for authenticated users.

---

### 4.3 Personal Catch Log

#### 4.3.1 Description and Priority

**Priority: High**

The catch log is the most differentiated feature of RiverRecords. It gives authenticated anglers a structured, searchable record of their fishing history in which every catch is permanently linked to the exact conditions present on that day. Unlike a simple journal, the conditions snapshot is automatically assembled from live USGS, weather, and community hatch data, and transforms each entry into a data point that grows more valuable as the log accumulates.

The immutability of the conditions snapshot is a deliberate design principle: the snapshot reflects what conditions were at the time of logging, not what the data shows now. This preserves the historical accuracy of the record even as live conditions change.

#### 4.3.2 Stimulus/Response Sequences

**Sequence 1: User logs a catch**
1. User navigates to the new catch form at `/log/new`
2. System pre-populates the river section selector from the user's recently viewed sections (TBD-1)
3. User selects a river section; system fetches and displays current conditions for confirmation
4. System queries HatchReports for the selected section within the past 24 hours and presents them as pre-selected checkboxes the user can confirm or dismiss
5. User fills in remaining fields: fish species, method, fly pattern, fly size, optional location (name or map pin), optional photo, optional notes
6. User submits the form
7. Client sends POST to `/api/catches`
8. Server authenticates via Clerk, validates body via Zod
9. Server fetches the most recent StreamReading for the section and current weather from Redis/OpenWeatherMap
10. Server assembles the `conditions_snapshot` JSON from readings, weather, and confirmed hatches
11. If a photo is included, server uploads to Cloudflare R2 or Vercel Blob and stores the returned URL
12. Server writes CatchLog record to PostgreSQL with the frozen snapshot
13. Server returns 201; UI navigates to the catch log list view showing the new entry

**Sequence 2: User views the catch log**
1. User navigates to `/log`
2. System queries PostgreSQL for the user's CatchLog records in reverse chronological order
3. System renders each entry as a card showing: catch species, date, river section, fly pattern and size, method, location name if present, and a collapsed conditions snapshot panel
4. User expands the conditions panel on an entry to view the full snapshot including active hatches

**Sequence 3: User filters the catch log**
1. User applies one or more filters (river, section, fish species, method, fly pattern, date range)
2. System re-queries PostgreSQL with the active filters applied
3. System re-renders the list with matching entries only

**Sequence 4: Conditions data unavailable at log time**
1. User submits a catch but the most recent StreamReading for that section is more than 24 hours old
2. Server proceeds with the write but populates `conditions_snapshot.gauge_height_ft` and related fields with the stale reading
3. The snapshot is saved with a `data_staleness_warning: true` flag so the UI can indicate the snapshot may not reflect conditions at time of catch (TBD-2)

#### 4.3.3 Functional Requirements

- REQ-15 (FR-11): Authenticated users shall be able to log a catch with the following fields: fish species (required), river section (required), catch date (required), fishing method (required, enum: dry / nymph / streamer / wet / other), fly pattern (required, free text), fly size (required, integer hook size), catch location name (optional, free text), catch location coordinates (optional, lat/lng float pair), photo (optional, one image file), notes (optional, free text).
- REQ-16 (FR-12): Upon catch submission, the system shall automatically assemble and permanently store a `conditions_snapshot` JSON object containing: gauge height in feet, discharge in CFS, water temperature in Fahrenheit, weather summary string, air temperature in Fahrenheit, wind speed in mph, wind direction string, sky cover (enum: clear / partly_cloudy / overcast), and an array of active hatch reports for that section within 24 hours of the catch date.
- REQ-17 (FR-13): The catch log list view shall display entries in reverse chronological order. Each entry shall expose the conditions snapshot including fly used, location, and active hatches.
- REQ-18 (FR-14): Users shall be able to filter their catch log by: river, river section, fish species, fishing method, fly pattern, and date range. Filters shall be combinable.
- REQ-19 (FR-15): When the catch log form is submitted, the system shall query HatchReports for the selected section within 24 hours of the catch date and present them to the user as suggested active hatches. The user shall be able to confirm or dismiss each suggestion before the catch is saved. Confirmed hatches are stored in the `conditions_snapshot.active_hatches` array.
- REQ-20: The `conditions_snapshot` field is immutable after the CatchLog record is created. No API route shall permit updating or overwriting it after initial creation.
- REQ-21: A user shall only be able to read, edit non-snapshot fields of, or delete their own CatchLog entries. Attempts to access another user's entries shall return 403.
- REQ-22: Photo uploads shall be optional. If a photo is included, it shall be stored in object storage (Cloudflare R2 or Vercel Blob) and only the returned URL shall be persisted in the database. The system shall accept one photo per catch entry.

---

### 4.4 AI Trip Assistant

#### 4.4.1 Description and Priority

**Priority: Medium**

The AI Trip Assistant is the most technically distinctive feature of the application. It accepts a natural language query from an authenticated user about a specific river section and returns a response grounded in four real data sources: the section's current conditions, the 7-day gauge and temperature trend, community hatch reports from the past 30 days, and the user's personal catch history for that section.

The assistant is not a general-purpose chatbot. It is constrained by its system prompt to advise only within the scope of the data it has been given, and it is instructed to disclose when that data is missing or stale rather than speculate.

The initial build uses a non-streaming response model: the full response is returned when ready and rendered at once. Streaming is explicitly deferred to a future iteration. (See Appendix D)

**Priority is Medium rather than High** because the application is fully functional and useful without it. The conditions browser, hatch calendar, and catch log are the core product. The AI assistant is the differentiating feature that makes the product more compelling and the article series more interesting.

#### 4.4.2 Stimulus/Response Sequences

**Sequence 1: User submits a query**
1. User is viewing a river section page and clicks the AI assistant panel
2. User types a natural language question (e.g., "Should I fish this weekend? What fly should I use?")
3. User submits the query
4. Client sends POST to `/api/ai` with the section ID and query text
5. Server authenticates via Clerk
6. Server loads: section metadata and current conditions from DB/Redis, the 7-day StreamReading history, community hatch reports for this section from the last 30 days, and the authenticated user's CatchLog entries for this section (most recent 10)
7. Server calls `buildConditionsContext()` in `lib/claude.ts` to assemble a structured context document from the loaded data
8. Server calls Anthropic Claude API with system prompt + assembled context + user query
9. Claude returns a complete response
10. Server returns the response to the client
11. Client renders the response text in the assistant panel

**Sequence 2: Conditions data is stale or missing**
1. Server assembles the context and detects that the most recent StreamReading for the section is more than 24 hours old
2. The assembled context includes a note that conditions data is stale with the last reading timestamp
3. Claude's system prompt instructs it to disclose this to the user and qualify its advice accordingly
4. Claude returns a response that acknowledges the staleness and advises the user to verify current conditions via USGS directly

**Sequence 3: Anthropic API is unavailable**
1. Server calls Anthropic API and receives an error or timeout
2. Server returns a 503 to the client
3. Client renders a clear error message: "The AI assistant is currently unavailable. River conditions and hatch data are still accessible above."
4. All other features on the page remain functional

#### 4.4.3 Functional Requirements

- REQ-23 (FR-16): Authenticated users shall be able to submit a natural language query to the AI assistant from any river section detail page.
- REQ-24 (FR-17): The system shall assemble a structured context document for each query containing: current conditions (gauge, discharge, temperature, weather, fishability score), the 7-day gauge and temperature trend, community hatch reports for the section from the last 30 days, and the authenticated user's CatchLog entries for that section (most recent 10 entries, including their conditions snapshots).
- REQ-25 (FR-18): The AI assistant shall return a complete response (non-streaming) in the v1.0 build. The response shall be rendered in the UI once the full response is received.
- REQ-26 (FR-19): The Claude system prompt shall instruct the assistant to: advise only within the scope of provided data, disclose when conditions data is stale or missing, not speculate about private fishing access or real-time fish counts, and not recommend fishing a section if gauge readings indicate unsafe conditions.
- REQ-27: The AI assistant context builder shall not include any other user's CatchLog data. Context assembly is strictly scoped to the authenticated requesting user.
- REQ-28: If the Anthropic API returns an error or is unreachable, the application shall display a graceful error message in the assistant panel. All other page features shall remain functional. The AI failure shall not produce an unhandled exception.
- REQ-29: The ANTHROPIC_API_KEY environment variable shall only be read from server-side code. It shall never be included in client-side bundles or exposed via a public environment variable prefix.

---

### 4.5 Authentication and User Accounts

#### 4.5.1 Description and Priority

**Priority: High**

Authentication is the architectural prerequisite for the hatch calendar write operations, the catch log, and the AI assistant. It is implemented using Clerk, which handles session management, OAuth, and email verification externally, reducing the authentication surface area to Clerk SDK integration in middleware and server components.

The architectural complexity of authentication is not in Clerk itself but in the discipline it imposes: every API route must verify identity, and every database query must be scoped to the correct user. This discipline must be consistently applied across all write operations from the first commit.

#### 4.5.2 Stimulus/Response Sequences

**Sequence 1: New user signs up**
1. Unauthenticated user clicks "Sign Up"
2. Clerk's hosted sign-up component handles email/password collection or Google OAuth flow
3. On successful sign-up, Clerk issues a session and redirects to the application
4. Application creates a local User record in PostgreSQL (id matching the Clerk user ID, email, display name)
5. User lands on the river browser, now in authenticated state

**Sequence 2: Returning user signs in**
1. User clicks "Sign In"
2. Clerk's hosted sign-in component handles credential verification
3. Clerk issues a session token stored in a secure, HttpOnly cookie
4. Next.js middleware (via Clerk) validates the token on every subsequent request
5. User proceeds to the application in authenticated state

**Sequence 3: Unauthenticated user attempts a write operation**
1. Unauthenticated user attempts to POST to `/api/catches` or `/api/hatches`
2. Clerk middleware detects no valid session token
3. Server returns 401 Unauthorized before the route handler executes
4. No database write occurs

**Sequence 4: User accesses a protected page without authentication**
1. Unauthenticated user navigates directly to `/log` or `/log/new`
2. Clerk middleware intercepts the request and redirects to the sign-in page
3. After successful sign-in, user is redirected back to the originally requested page

#### 4.5.3 Functional Requirements

- REQ-30 (FR-20): Users shall be able to create an account using email and password, or authenticate using Google OAuth. Both flows are handled by the Clerk hosted UI components.
- REQ-31 (FR-21): Unauthenticated users shall have read-only access to the river conditions browser (all sections, list, and map views) and the community hatch calendar. No authentication prompt shall be shown for these views unless the user attempts a write action.
- REQ-32 (FR-22): All write operations, like submitting a hatch report and creating a catch log entry, shall require a valid Clerk session. Requests without a valid session shall receive a 401 response before any business logic executes.
- REQ-33: A local User record shall be created in PostgreSQL upon first authentication. The local record's primary key shall match the Clerk user ID to enable efficient cross-reference without additional lookups.
- REQ-34: Session management, token issuance, and credential storage are delegated entirely to Clerk. The application shall not implement custom session storage, password hashing, or token generation.

---

## 5. AI Integration — Detailed Design

The Claude API integration is the most novel engineering component of RiverRecords and the one most directly connected to the article series documenting the development process. This section defines the full design of that integration.

### 5.1 Context Assembly

When a user submits a query, the `/api/ai` route assembles a structured context document before making any API call. The quality of Claude's response is determined almost entirely by the quality and structure of this context. The context builder (`lib/claude.ts`) is responsible for pulling the correct data, formatting it compactly, and constructing both the system and user messages.

**System prompt:**
```
You are a fly fishing trip planning assistant with access to real-time river
conditions, community hatch reports, and the user's personal catch history for
the river section they are viewing.

Your job is to give practical, specific advice based on the data provided.
Do not speculate beyond the data. If conditions data is missing or stale,
say so explicitly and include the age of the last reading.
Do not recommend fishing a section if gauge or discharge data indicates unsafe
wading conditions.

You have no knowledge of private fishing access or current fish populations
beyond what the catch history implies.
```

**Assembled user message (representative example):**
```
RIVER SECTION: Madison River — Cameron to Varney Bridge (MT)
TARGET SPECIES: Brown Trout, Rainbow Trout

CURRENT CONDITIONS (as of 2026-04-05T09:00:00Z):
- Gauge height: 3.2 ft (USGS site 06043500)
- Discharge: 892 CFS
- Water temperature: 58°F
- Weather: Partly cloudy, high 67°F, wind 8 mph SW
- Fishability: GOOD (ideal range: 600–1200 CFS, 50–65°F)

7-DAY TREND:
- Gauge: dropped from 4.1 ft to 3.2 ft (falling, improving)
- Temperature: risen from 52°F to 58°F (rising, approaching ideal)

RECENT HATCH REPORTS (last 30 days):
- Apr 1: PMD, moderate, afternoon
- Mar 28: Blue Winged Olive, heavy, morning
- Mar 28: Blue Winged Olive, moderate, afternoon
- Mar 22: Midge, light, morning

USER CATCH HISTORY (this section, most recent first):
- Mar 15, 2026: Brown Trout, dry fly, Parachute Adams #16, gauge 3.8 ft, 54°F, overcast
- Feb 2, 2026: Rainbow Trout, nymph, Pheasant Tail #18, gauge 4.2 ft, 46°F, clear
- Oct 12, 2025: Brown Trout, streamer, Woolly Bugger #6, gauge 2.9 ft, 61°F, sunny

USER QUESTION: Should I fish this weekend? What fly should I bring?
```

### 5.2 Context Window Considerations

The context builder must manage prompt size when a user has a large catch history for a section. The initial implementation limits catch history to the 10 most recent entries. If future usage produces longer histories, summarization logic (grouping entries by condition range rather than listing each individually) will be required. This is tracked as TBD-3.

### 5.3 API Call and Response Handling

The Anthropic SDK `messages.create()` method is called with `stream: false` in v1.0 (non-streaming). The complete response is awaited, then returned to the client as a JSON response body.

Error handling covers: missing API key (caught at startup), Anthropic API rate limit (429 — return 503 with retry guidance), network timeout (return 503 with generic unavailability message), and malformed response (log server-side, return 503).

### 5.4 Article-Worthy Engineering Decisions

The following design decisions are explicitly identified as material for the technical article series:

- **Structured context vs. JSON:** The current implementation uses a plain-text structured format for the user message. An experiment comparing this to a JSON-formatted context is planned and will be written about.
     - Determine how to send data to Claude, plain text is more natural and relevant to the model's training data, but JSON is more precise
     - Cost increase from using JSON format due to increased number of keys
- **Data freshness disclosure:** Claude is instructed to surface data staleness. Testing whether it reliably does so, and what happens when it does not.
     - Choosing how often condition data age is displayed
     - Ensuring Claude reliability notes when data is old, and finding out why it fails to do so when it does
     - Prompt engineering suggestions to maximize system capabilities and avoid data errors
- **Domain accuracy:** Claude has general fly fishing knowledge from training. Where it conflicts with the provided hatch data or catch history, which does it weigh? Testing and documenting specific cases is planned.
     - Finding the balance between user-input data and Claude's inherent fly fishing knowledge
     - Determining when/how often/ why Claude will override recent data with its general knowledge
- **Streaming upgrade:** The decision to start non-streaming and upgrade later provides a before/after implementation comparison worth documenting.
     - Start without streaming for minimum time and space complexity
     - Determine what changes in code after changing to streaming
     - Analyzing perceived and actual performance, alongside concrete data
---

## 6. System Architecture and Repository Structure

### 6.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser (User)                              │
│   River browser · Hatch calendar · Catch log · AI assistant         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Next.js Application (Vercel)                     │
│                                                                     │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐    │
│  │    Server Components    │    │        API Routes            │    │
│  │  Render data fetched    │    │  /api/catches    (CRUD)      │    │
│  │  at request time.       │    │  /api/hatches    (CRUD)      │    │
│  │  No client JS for       │    │  /api/ai         (query)     │    │
│  │  initial render.        │    │  /api/conditions (proxy)     │    │
│  └─────────────────────────┘    │  /api/cron/sync  (cron job)  │    │
│                                 └──────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  Middleware (Clerk)                          │   │
│  │  Validates session token on every request.                   │   │
│  │  Blocks unauthenticated access to protected routes.          │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────┬────────────────────────────────┬────────────────────────-┘
           │                                │
           ▼                                ▼
┌──────────────────────┐        ┌───────────────────────────────────┐
│  PostgreSQL          │        │  Redis Cache (Railway)            │
│  (Railway)           │        │  Stream readings: TTL = 1 hour    │
│  Users               │        │  Weather forecasts: TTL = 3 hrs   │
│  Rivers              │        └───────────────────────────────────┘
│  RiverSections       │
│  StreamReadings      │
│  HatchReports        │
│  CatchLogs           │
└──────────────────────┘
           │ written by cron job
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     External Data Sources                           │
│  USGS Water Services API  ·  OpenWeatherMap API                     │
│  State Stocking APIs      ·  Anthropic Claude API                   │
│  Clerk (auth)             ·  Cloudflare R2 / Vercel Blob            │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Data Flow Patterns

**Pattern 1: Viewing a river section page**
```
Browser requests /rivers/[sectionId]
  → Next.js Server Component executes on Vercel
  → Query PostgreSQL: section metadata + 7-day StreamReading history
  → Check Redis for cached current reading
      Cache hit  → use cached reading
      Cache miss → fetch USGS → write Redis → use reading
  → Check Redis for cached weather forecast
      Cache hit  → use cached forecast
      Cache miss → fetch OpenWeatherMap → write Redis → use forecast
  → Render full page HTML → send to browser (no client data fetching on load)
```

**Pattern 2: Hourly cron condition sync**
```
Vercel Cron triggers POST /api/cron/sync-conditions
  → Load all RiverSections from PostgreSQL
  → For each section:
      Fetch latest reading from USGS
      Write StreamReading record to PostgreSQL
      Invalidate Redis cache entry for this section
  → Log sync result (section count, any errors)
```

**Pattern 3: User logs a catch**
```
User submits catch form
  → POST /api/catches
  → Clerk middleware: confirm authentication
  → Zod: validate request body
  → Fetch most recent StreamReading for section from PostgreSQL
  → Fetch weather from Redis / OpenWeatherMap
  → Query HatchReports within 24 hrs for this section
  → Assemble conditions_snapshot JSON
  → If photo: upload to R2 / Vercel Blob → get URL
  → Write CatchLog to PostgreSQL with frozen snapshot
  → Return 201 → UI updates
```

**Pattern 4: User queries AI assistant**
```
User submits query in AI panel
  → POST /api/ai
  → Clerk middleware: confirm authentication
  → Load section conditions, 7-day trend, 30-day hatches, user's 10 most recent catches
  → buildConditionsContext() assembles structured prompt context
  → Call Anthropic Claude API (non-streaming, await complete response)
  → Return response JSON to client → UI renders response
```

### 6.3 Data Models

#### River
```prisma
model River {
  id          Int            @id @default(autoincrement())
  name        String
  state       String
  description String?
  sections    RiverSection[] // virtual relation, not a column
  created_at  DateTime       @default(now())
}
```

#### RiverSection
```prisma
model RiverSection {
  id              Int             @id @default(autoincrement())
  river_id        Int
  river           River           @relation(fields: [river_id], references: [id])
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
  stocking_events StockingEvent[] // TRIAGED — schema defined, not implemented in v1.0
  created_at      DateTime        @default(now())
}
```

#### StreamReading
A point-in-time reading from USGS for a specific river section. The cron job writes one record per section per hourly sync cycle. Records accumulate over time and are used for the 7-day trend charts and AI context assembly.

```prisma
model StreamReading {
  id                Int          @id @default(autoincrement())
  section_id        Int
  section           RiverSection @relation(fields: [section_id], references: [id])
  gauge_height_ft   Float
  discharge_cfs     Float
  temperature_c     Float?
  reading_timestamp DateTime     // timestamp of the reading as reported by USGS
  ingested_at       DateTime     @default(now()) // when this record was written to the DB
}
```

#### HatchReport
A community-submitted insect hatch observation. Belongs to a user and a river section. The `intensity` and `time_of_day` fields are enums to keep reporting consistent across users.

```prisma
enum HatchIntensity {
  light
  moderate
  heavy
}

enum TimeOfDay {
  morning
  midday
  afternoon
  evening
}

model HatchReport {
  id             Int           @id @default(autoincrement())
  user_id        String        // Clerk user ID
  section_id     Int
  section        RiverSection  @relation(fields: [section_id], references: [id])
  insect_species String
  intensity      HatchIntensity
  time_of_day    TimeOfDay
  report_date    DateTime
  notes          String?
  created_at     DateTime      @default(now())
}
```

#### CatchLog
A user's personal catch record with a permanently frozen conditions snapshot. The `conditions_snapshot` is a JSONB column assembled at write time and is immutable after creation; it permanently records the conditions present on the day of the catch regardless of how live data changes afterward.

```prisma
enum FishingMethod {
  dry
  nymph
  streamer
  wet
  other
}

model CatchLog {
  id                  Int          @id @default(autoincrement())
  user_id             String       // Clerk user ID
  section_id          Int
  section             RiverSection @relation(fields: [section_id], references: [id])
  catch_date          DateTime
  caught_species      String
  method              FishingMethod
  fly_pattern         String
  fly_size            Int          // hook size as integer
  location_name       String?      // free-text spot name
  location_lat        Float?
  location_lng        Float?
  photo_url           String?      // URL returned by object storage after upload
  notes               String?
  conditions_snapshot Json         // JSONB — frozen at creation, never updated
  created_at          DateTime     @default(now())
}
```

`conditions_snapshot` JSON structure (immutable after creation):
```json
{
  "gauge_height_ft": 3.2,
  "discharge_cfs": 892,
  "water_temp_f": 58,
  "weather_summary": "Partly cloudy",
  "air_temp_f": 67,
  "wind_speed_mph": 8,
  "wind_direction": "SW",
  "sky_cover": "partly_cloudy",
  "active_hatches": [
    { "insect_species": "PMD", "intensity": "moderate", "time_of_day": "afternoon" }
  ]
}
```

#### Alert (TRIAGED — schema defined, not implemented in v1.0)
Defined in the schema to avoid a future migration when alerts are implemented. No Alert-related routes, UI, or cron logic will exist in v1.0.

```prisma
enum ConditionType {
  gauge_height
  discharge
  temperature
}

enum ThresholdOperator {
  above
  below
}

model Alert {
  id               Int               @id @default(autoincrement())
  user_id          String            // Clerk user ID
  section_id       Int
  section          RiverSection      @relation(fields: [section_id], references: [id])
  condition_type   ConditionType
  operator         ThresholdOperator
  threshold_value  Float
  is_active        Boolean           @default(true)
  last_triggered_at DateTime?
  created_at       DateTime          @default(now())
}
```


### StockingEvent (TRIAGED — schema defined, not implemented in v1.0)
Defined in the schema to avoid a future migration when stocking data is implemented. No StockingEvent-related routes, UI, or cron logic will exist in v1.0. Stocking data is sourced from state wildlife agency APIs, which vary significantly in format; v1.0 targets one state and the fetcher pattern is defined in lib/ but the data is not surfaced in the UI or AI context until a future iteration.
```prisma
model StockingEvent {
  id             Int          @id @default(autoincrement())
  section_id     Int
  section        RiverSection @relation(fields: [section_id], references: [id])
  stocked_date   DateTime     // date the stocking event occurred
  species        String       // e.g. "Brown Trout", "Rainbow Trout"
  quantity       Int?         // number of fish stocked, if reported by agency
  source_agency  String       // e.g. "Montana FWP"
  source_url     String?      // direct link to agency report if available
  ingested_at    DateTime     @default(now())
}
```

#### User
The local user record stores only what the application needs beyond what Clerk provides. The `id` field is set to the Clerk user ID (a string like `user_2abc...`) rather than an auto-incremented integer, enabling cross-reference with Clerk without additional lookups.

```prisma
model User {
  id           String   @id       // matches Clerk user ID
  email        String   @unique
  display_name String?
  home_state   String?
  created_at   DateTime @default(now())
}
```

### 6.4 Repository Structure

```
river-records/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── (auth)/
│   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   └── sign-up/[[...sign-up]]/page.tsx
│   ├── (app)/
│   │   ├── layout.tsx
│   │   ├── rivers/
│   │   │   ├── page.tsx
│   │   │   └── [sectionId]/page.tsx
│   │   ├── log/
│   │   │   ├── page.tsx
│   │   │   └── new/page.tsx
│   │   └── hatches/
│   │       └── page.tsx
│   └── api/
│       ├── catches/route.ts
│       ├── catches/[id]/route.ts
│       ├── hatches/route.ts
│       ├── hatches/[id]/route.ts
│       ├── ai/route.ts
│       ├── conditions/route.ts
│       └── cron/sync/route.ts
├── components/
│   ├── ui/              (shadcn/ui primitives)
│   ├── map/             (RiverMap, SectionMarker)
│   ├── charts/          (GaugeChart, TemperatureChart)
│   ├── conditions/      (ConditionsPanel, FishabilityBadge)
│   ├── hatches/         (HatchCalendar, HatchReportForm, HatchReportCard)
│   ├── catches/         (CatchLogList, CatchLogCard, CatchLogForm)
│   └── ai/              (AssistantPanel)
├── lib/
│   ├── prisma.ts        (singleton Prisma client)
│   ├── usgs.ts          (USGS API client)
│   ├── weather.ts       (OpenWeatherMap client)
│   ├── redis.ts         (ioredis singleton + cache helpers)
│   ├── claude.ts        (context builder + API call)
│   ├── conditions.ts    (fishability score computation)
│   └── storage.ts       (photo upload)
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── types/index.ts
├── tests/
│   ├── lib/
│   └── components/
├── middleware.ts
├── .env.example
├── .env.local           (gitignored)
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── vitest.config.ts
└── package.json
```

### 6.5 Environment Variables

```bash
# Database
DATABASE_URL="postgresql://..."
REDIS_URL="redis://..."

# Auth (Clerk)
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

## 7. Nonfunctional Requirements

### Performance Requirements

- NFR-1: River section detail pages shall complete server-side rendering and deliver the initial HTML response in under 2 seconds under normal load. Cached data (within TTL) is acceptable to meet this requirement.
- NFR-2: Redis cache TTL for stream gauge readings shall be 1 hour. Cache TTL for weather forecast data shall be 3 hours. These values balance data freshness against external API rate limit consumption.
- NFR-3: The condition sync cron job shall complete a full sync of all seeded river sections within 60 seconds. If a section's USGS fetch times out, the job shall log the failure and continue to the next section without aborting the entire run.
- NFR-4: The AI assistant response shall be delivered within 15 seconds of submission under normal Anthropic API conditions. If this threshold is exceeded, the UI shall display a loading indicator to prevent the user from assuming the request failed. (TBD-4: whether to add a client-side timeout and abort.)

### Safety Requirements

- NFR-5: RiverRecords does not control or influence access to physical waterways. The AI assistant's system prompt explicitly instructs it not to recommend fishing a river section when gauge or discharge data indicates potentially unsafe wading conditions. The application bears no legal responsibility for a user's physical safety decisions, but the system shall not affirmatively recommend unsafe conditions.
- NFR-6: The application must degrade gracefully when external services are unavailable. Specifically: if USGS data is unavailable, the most recent cached reading shall be shown with a staleness label; if OpenWeatherMap is unavailable, the weather panel shall be hidden rather than shown as an error; if the Anthropic API is unavailable, the AI panel shall show a clear "unavailable" message while all other page features remain functional.

### Security Requirements

- NFR-7: All API keys and database credentials shall be stored as environment variables and shall never be committed to source control. The `.env.local` file shall be listed in `.gitignore`. The `.env.example` file with placeholder values shall be committed.
- NFR-8: The `ANTHROPIC_API_KEY`, `CLERK_SECRET_KEY`, `DATABASE_URL`, and `REDIS_URL` environment variables shall only be accessible in server-side code. They shall not be prefixed with `NEXT_PUBLIC_` and shall not appear in any client bundle.
- NFR-9: Every API route that performs a write operation shall verify the Clerk session token before executing any business logic. A missing or invalid session shall return 401 before any database interaction occurs.
- NFR-10: Every API route that reads or modifies a user-owned resource (CatchLog, HatchReport) shall verify that the requesting user's Clerk ID matches the resource's `user_id` field. Mismatches shall return 403.
- NFR-11: All database queries involving user data shall include a `where: { user_id: clerkUserId }` clause. It is not acceptable to fetch all records of a type and filter in application code, the scope restriction must occur at the database query level.
- NFR-12: Photo uploads shall be validated for file type (JPEG, PNG, WEBP only) and size (maximum 10 MB) before being passed to object storage. Unsupported file types shall return a 400 error with a descriptive message.

### Software Quality Attributes

- **Maintainability:** All external API interaction is isolated to dedicated modules in `lib/`. No USGS or OpenWeatherMap fetch calls shall appear outside `lib/usgs.ts` and `lib/weather.ts` respectively. This ensures that changes to external API behavior require changes in one place only.
- **Testability:** `lib/` modules shall be written with dependency injection or functional composition patterns that allow test doubles to substitute for real API calls. No module in `lib/` shall make a network call in its module scope (top-level await); all network calls occur inside exported functions.
- **Reliability:** The cron job shall be idempotent in that running it twice in the same hour shall not duplicate StreamReading records. Duplicate prevention shall be enforced by checking for an existing reading at the same `reading_timestamp` before inserting.
- **Usability:** The application shall be fully navigable on a mobile viewport (minimum 375px width). All interactive elements shall meet WCAG 2.1 AA touch target size requirements. shadcn/ui components are built on Radix UI primitives which meet accessibility requirements by default.
- **Portability:** The application depends on Vercel-specific features (Cron, serverless functions). If hosting migration is required in the future, the cron functionality would need to be replaced with an equivalent (e.g., GitHub Actions scheduled workflow calling a self-hosted endpoint). This dependency is accepted and documented.

### Business Rules

- BR-1: A user may only read, edit, or delete resources they own (CatchLog entries, HatchReport entries). There is no administrative override in v1.0.
- BR-2: Conditions snapshots on CatchLog entries are immutable after creation. No API route shall permit overwriting a `conditions_snapshot` field after the initial write.
- BR-3: Unauthenticated users may read river conditions and community hatch reports but may not write any data to the system.
- BR-4: The AI assistant may only use data belonging to the requesting user when assembling the catch history context. It is not permitted to include or summarize other users' catch data.
- BR-5: Stream gauge readings are stored as received from USGS without modification. The computed `fishability_score` is derived at query time from the raw reading and the section's configured thresholds, but it is never stored directly.

---

## 8. Development Phase Map

The phases below define the planned build order. Each phase produces a working, deployable artifact before the next phase begins. This order minimizes blocked work and ensures that foundational components (data models, external API integrations) are validated before dependent features are built on top of them.

| Phase | Focus | Produced Artifact |
|---|---|---|
| 0 | Study USGS Water Services API documentation; explore raw JSON response shapes; understand available parameters, site codes, and rate limits | Working knowledge required to design the schema correctly. No code produced. |
| 1 | Project setup: Next.js, TailwindCSS, Clerk, Prisma, Railway databases, environment variable configuration, GitHub repository, initial Vercel deploy | A deployable Next.js application with auth that produces a live URL |
| 2 | Prisma schema definition; database migration; seed data for rivers and sections; Prisma Studio verification | A populated PostgreSQL database with real river sections queryable locally |
| 3 | `lib/usgs.ts`, `lib/weather.ts`, `lib/redis.ts` modules; Redis caching logic; `/api/conditions` route | Server-side data fetching pipeline that correctly retrieves, caches, and returns condition data |
| 4 | River browser UI: list view, section detail page with conditions panel and trend charts | A read-only, publicly accessible application that displays real river data |
| 5 | Leaflet map integration; fishability score computation in `lib/conditions.ts`; map markers with color coding | The interactive river map with fishability indicators |
| 6 | Catch log: CatchLogForm, `/api/catches` route, conditions snapshot assembly, photo upload, CatchLogList view | Core personal logging feature, end-to-end |
| 7 | Hatch calendar: HatchReportForm, `/api/hatches` route, HatchCalendar view with filtering | Community hatch data submission and display |
| 8 | AI assistant: context builder in `lib/claude.ts`, `/api/ai` route, AssistantPanel component | Natural language trip planning feature |
| 9 | Polish: mobile responsiveness, error states, loading states, empty states, graceful degradation for all external service failures | A shippable, production-quality UI |
| 10 | Tests: unit tests for `lib/` modules, component tests for key forms and displays, GitHub Actions CI configuration | Test suite with CI badge; confidence in correctness of business logic |
| 11 | README documentation, custom domain configuration (TBD-5), final production deploy review | A live, publicly accessible application with complete documentation |

---

## 9. Other Requirements

### Database Requirements

- The PostgreSQL database shall use Prisma migrations for all schema changes. No ad-hoc schema modifications shall be made via direct SQL outside of a migration file.
- The `conditions_snapshot` field on CatchLog shall be stored as a JSONB column in PostgreSQL. JSONB (binary JSON) is preferred over JSON because it supports indexing and more efficient querying if the conditions analysis feature is added in a future build.
- The seed file (`prisma/seed.ts`) shall contain a representative set of real river sections with valid USGS site codes that can be verified against the USGS National Water Information System. Seed data shall include at minimum five river sections across at least two rivers.

### Legal and Attribution Requirements

- USGS data is provided by the United States Geological Survey and is in the public domain. No special attribution beyond identifying the data source is legally required; however, the application shall display "Data source: USGS National Water Information System" on pages displaying gauge data as a matter of transparency.
- OpenWeatherMap data under the free tier is provided under the Creative Commons Attribution-ShareAlike 4.0 license. The application shall include appropriate attribution on weather data displays.
- The application shall include a Privacy Policy page (TBD-6) disclosing what user data is collected (email via Clerk, catch records, hatch reports) and how it is used before public launch.

### Reuse and Extensibility Objectives

- The `lib/usgs.ts` module shall be designed with a clean public interface such that it could be extracted as a standalone open-source library in a future iteration without significant refactoring.
- All Prisma models for triaged features (Alert, StockingEvent) shall be defined in the schema from the start so that future implementation requires only new routes and UI, not schema migrations on a production database.

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| CFS | Cubic feet per second. The standard unit for measuring river discharge (flow volume). |
| Clerk | Third-party authentication provider used for user sign-up, sign-in, session management, and OAuth integration. |
| Conditions Snapshot | A JSON object stored permanently with each CatchLog entry. Records the gauge height, discharge, temperature, weather, and active hatches present at the time of the catch. Immutable after creation. |
| Cron Job | A scheduled task that runs automatically at a defined interval. In RiverRecords, a Vercel Cron job triggers hourly USGS data ingestion. |
| Discharge | The volume of water flowing past a point in a river per unit of time, measured in CFS. Higher discharge generally correlates with higher, faster, and more turbid water. |
| Fishability Score | A computed label (good / fair / poor) derived from comparing a river section's current gauge height and water temperature against its configured ideal ranges. Not stored in the database, instead computed at query time. |
| Gauge Height | The height of the water surface at a USGS monitoring station, measured in feet above an arbitrary datum. Used as a proxy for water volume and wading difficulty. |
| Hatch | An insect emergence event in which aquatic insects (mayflies, caddisflies, stoneflies, midges) emerge from the water in large numbers, triggering active surface feeding by trout. |
| Ideal Range | The configured gauge height and water temperature range for a river section within which conditions are considered favorable for the section's target species. Stored as `ideal_gauge_min`, `ideal_gauge_max`, `ideal_temp_min`, `ideal_temp_max` on the RiverSection model. |
| Prisma | TypeScript ORM used for database schema definition, migrations, and type-safe queries against PostgreSQL. |
| Redis | In-memory key-value data store used as a caching layer for stream condition readings and weather forecast data. |
| River Section | A defined, named stretch of a river associated with a specific USGS gauge station. The primary unit of organization in RiverRecords. |
| StreamReading | A point-in-time record of gauge height, discharge, and temperature for a river section, written by the cron ingestion job. |
| TRIAGED | A formal designation in this document for features that have been specified but deferred from the v1.0 build. Triaged features retain their data model definitions. |
| USGS | United States Geological Survey. Federal agency that operates the National Water Information System (NWIS), providing public real-time and historical stream gauge data. |
| Zod | TypeScript schema validation library used to validate API route request bodies at runtime before business logic executes. |

---

## Appendix B: Analysis Models

### B.1 Entity Relationship Overview

```
River ──< RiverSection >── StreamReading
               │
               ├──< HatchReport >── User
               │
               ├──< CatchLog >── User
               │     │
               │     └── conditions_snapshot (JSON, embedded)
               │           └── active_hatches[] (from HatchReport, frozen at log time)
               │
               └──< StockingEvent [TRIAGED]
```

One River has many RiverSections. One RiverSection has many StreamReadings, HatchReports, and CatchLogs. HatchReports and CatchLogs are owned by Users. The CatchLog's `conditions_snapshot` embeds a frozen array of hatch data, the snapshot is not a live relation but an immutable copy captured at log time recording the conditions at the time. StockingEvent is triaged — the relation is defined in the schema but no routes or UI exist in v1.0.

### B.2 Cron Job State Flow

```
Cron trigger (hourly)
      │
      ▼
Load all RiverSections
      │
      ├─ [for each section] ─────────────────────────────┐
      │                                                   │
      │   Fetch USGS reading                             │
      │        │                                         │
      │        ├── Success → Write StreamReading         │
      │        │            → Invalidate Redis cache     │
      │        │                                         │
      │        └── Failure → Log error                   │
      │                    → Skip to next section        │
      │                                                   │
      └───────────────────────────────────────────────────┘
      │
      ▼
Log sync summary (sections processed, failures)
```

### B.3 Catch Log Conditions Snapshot Assembly

```
POST /api/catches received
      │
      ├── Authenticate (Clerk)
      ├── Validate (Zod)
      │
      ├── Fetch: most recent StreamReading for section_id
      ├── Fetch: weather from Redis / OpenWeatherMap
      ├── Query: HatchReports for section within ±24 hrs of catch_date
      │
      ├── Present hatches to user for confirmation (client-side)
      │     [confirmed hatches returned in re-submission]
      │
      ├── Assemble conditions_snapshot JSON
      │
      ├── Upload photo if present → get URL
      │
      └── Write CatchLog with frozen snapshot → 201
```

---

## Appendix C: To Be Determined List

| ID | Location | Description |
|---|---|---|
| TBD-1 | Section 4.3, Sequence 1 | How the catch log form pre-populates the river section selector. Options: most recently viewed section, most recently logged section, or no pre-population. |
| TBD-2 | Section 4.3, Sequence 4 | Whether to add a `data_staleness_warning` boolean field to the `conditions_snapshot` JSON when the most recent StreamReading is older than 24 hours at log time. |
| TBD-3 | Section 5.2 | Strategy for managing context window size when a user's catch history for a section exceeds 10 entries. Options: hard limit at 10, summarize older entries, or include all with truncation. |
| TBD-4 | Section 7, NFR-4 | Whether to implement a client-side timeout and abort for the AI assistant request. Threshold and behavior to be determined. |
| TBD-5 | Section 8, Phase 11 | Custom domain selection and DNS configuration for the production deployment. |
| TBD-6 | Section 9, Legal | Privacy Policy page content and placement within the application. Required before public launch. |

---

## Appendix D: Planned Future Features and Improvements

The following features have been formally defined but deferred from the v1.0 build. They are listed here to document intent and to confirm that no schema changes will be required when they are implemented.

**Condition-Based Alert Notifications**
User-defined thresholds (gauge height, discharge, water temperature) that trigger an email notification via Resend when a threshold crossing is detected by the cron job. The Alert data model is defined in the Prisma schema. Implementation requires: Alert CRUD routes and UI, threshold evaluation logic in the cron job, and Resend email integration. No schema migration required.

**Personal Conditions Analysis View**
A per-river-section view that aggregates a user's historical catch data to surface their personal productive temperature range, which fly patterns produced catches under which conditions, and whether active hatches correlated with successful sessions. All required data is captured in the `conditions_snapshot` JSON on each CatchLog entry. Implementation requires: JSONB aggregation queries in PostgreSQL and a dedicated analysis UI. No schema migration required. This feature becomes meaningfully useful only after a user has accumulated a substantial catch history (estimated 15–20 entries per section minimum).

**AI Response Streaming**
Upgrade the AI assistant from a full-response model to a streaming response that renders tokens progressively as they arrive from the Anthropic API. Implementation requires: a `TransformStream` to convert the Anthropic SDK stream format to a browser-readable stream, a custom `useStreamingResponse` React hook, and updates to the AssistantPanel component. No schema changes required. The streaming upgrade is planned as a dedicated technical article topic.

**Expanded State Coverage for Fish Stocking Data**
The v1.0 build defines the `StockingEvent` data model and integrates a stocking data fetcher for one US state. The model is defined in the Prisma schema so no migration is required when the feature is surfaced in the UI. Expanding to additional states requires research into each state wildlife agency's data format and access method, as they vary significantly. Implementation is additive, new state-specific fetchers in `lib/` and new seed data. No schema migration required.

**Community Hatch Pattern Predictions**
Algorithmically derived hatch timing predictions based on accumulated community hatch report history, water temperature trends, and historical hatch timing by species. This is a data-availability-gated feature, as meaningful predictions require at least one full year of community data. Planned for consideration after the first fishing season of operation.
