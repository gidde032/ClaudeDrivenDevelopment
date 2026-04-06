# Project Feasibility Ranking
## Tailored to a Junior CS Student with Java, C, and TypeScript Background

---

## How This Was Evaluated

Each project was scored against four axes relevant to your specific situation:

- **Feasibility** — How much of the stack overlaps with what you already know, and how steep the remaining learning curve is
- **Shipping difficulty** — How hard it is to get something live with no prior hosting experience
- **AI integration quality** — How naturally the project pairs with an article series on AI-assisted development (both as a dev tool and as a product feature)
- **Market relevance** — How much the skills built translate to the current hiring market for SWE roles

Your TypeScript background is a significant asset. The current market heavily rewards full-stack TypeScript, and you are closer to being productive there than most juniors without that foundation. Python fluency is worth investing in alongside any project here — it is the dominant language for data, tooling, and AI work and is easy to pick up with your background.

---

## Rankings at a Glance

| Rank | Project | Domain | Difficulty |
|---|---|---|---|
| 1 | Fly Fishing — River Conditions & Hatch Journal | Full-Stack Web App | Medium |
| 2 | Pokemon Go — PVP Roster Analyzer | CLI Tool | Low-Medium |
| 3 | Fly Fishing — Fly Tying Recipe Manager | CLI Tool | Low-Medium |
| 4 | Fly Fishing — Python Client for Stream Condition Data | Open-Source Library | Medium |
| 5 | Pokemon Go — Game Master Library | Open-Source Library | Medium |
| 6 | Pokemon Go — Local Raid Coordination Hub | Full-Stack Web App | Medium-High |
| 7 | Fly Fishing — Stream Health & Fishability Index | Data Pipeline | Medium-High |
| 8 | Pokemon Go — Meta Shift Tracker | Data Pipeline | High |

---

## Detailed Analysis

---

### #1 — Fly Fishing: River Conditions & Hatch Journal
**Domain:** Full-Stack Web App

#### Feasibility Relative to Your Experience
This is the strongest match for your current skills. You already know TypeScript, which means the entire stack is written in a language you are comfortable with. Next.js is a React framework — if you have written TypeScript, you can be productive in React within a week. The concepts (components, props, state, API calls) map naturally to things you have encountered in other contexts. PostgreSQL is SQL, which you have almost certainly touched in a CS curriculum. The new concepts — auth with Clerk, deployment on Railway or Vercel, a caching layer with Redis — are each individually well-documented and have strong tutorial ecosystems. You are not learning a new paradigm anywhere; you are extending an existing one.

The map component (Leaflet.js) and the charting library (Recharts) both have thorough documentation and are configured through JSX, which you will already be writing. The USGS and OpenWeatherMap APIs return JSON over HTTP — straightforward to integrate with `fetch` or `axios`.

The primary learning investments are: Next.js patterns (app router, server vs. client components, API routes), basic database schema design and SQL queries, and deploying a web app for the first time. All three are achievable and directly transferable.

#### Skillset Expansion
- **Next.js and React** — the dominant framework combination in the industry. Knowing TypeScript already makes this an incremental step, not a leap.
- **PostgreSQL and relational schema design** — foundational database skills used in nearly every production backend.
- **REST API integration** — calling and normalizing third-party APIs is a daily task in most engineering jobs.
- **Deployment and hosting** — Vercel (for Next.js) is arguably the easiest deployment experience that exists. Your first deploy can happen in minutes. This gives you a live URL to put on your resume immediately.
- **Claude API integration** — building a user-facing AI feature (the natural language condition assistant) is a concrete, demonstrable AI skill. Not "I used AI to help me code" but "I built a product feature powered by an LLM."
- **Redis caching** — understanding when and why to cache is a backend fundamentals concept that comes up in every systems design conversation.

#### Market Relevance
TypeScript + React + Next.js + PostgreSQL is the single most in-demand skill combination for junior and new-grad software engineering roles right now, across startups and larger companies. Vercel, the company behind Next.js, has essentially made this the default full-stack web stack for a generation of developers. Knowing it well is directly actionable in interviews and job applications. A deployed, live application in this stack — one that real users can sign up for and use — is the strongest possible entry on a junior portfolio.

The AI integration layer (Claude API powering a user-facing feature) adds a second dimension that is increasingly valued: experience building products with LLMs, not just using them as coding tools.

#### Shipping Assessment
Vercel has a one-click GitHub integration. You push code, it deploys. This is genuinely the easiest hosting experience available and is what the project is designed for. Railway handles the PostgreSQL database with similar simplicity. You can have a live URL within the first week of starting. This is the lowest-friction path to a shippable product for someone without prior hosting experience.

#### AI Article Pairing
Excellent pairing. You have natural article topics at every phase: using Claude to design the schema, using Claude to build the condition-query feature, evaluating whether the AI's responses are actually useful or hallucinated, and the experience of integrating an LLM into a product versus just using it as a coding assistant.

---

### #2 — Pokemon Go: PVP Roster Analyzer
**Domain:** CLI Tool

#### Feasibility Relative to Your Experience
Python is on your resume as prior experience, and the architecture here is simple enough that the language learning curve will not dominate your time. CLI tools have no frontend, no auth, no deployment infrastructure — you are writing functions and connecting them with a command interface. The hardest part of this project is the domain logic: parsing the game master JSON and implementing the CP and IV formulas correctly. But you know this domain. You play Pokemon GO, which means you can validate your outputs intuitively, which dramatically shortens the debugging loop. Typer and Rich are both well-documented and have clear, readable APIs.

The pandas data processing layer will be new if you have not used it before, but it is one of the most-documented libraries in any language. The Claude API integration is straightforward: you build a structured prompt from your parsed roster data and send it.

#### Skillset Expansion
- **Python proficiency** — getting comfortable in Python is worth doing regardless of which project you build, and this is a gentle, well-scoped environment to do it in.
- **CLI tool design** — understanding how to structure a tool that other developers will use from the terminal, with good error messages, help text, and composable commands, is a software design skill that transfers broadly.
- **pandas** — the foundational data processing library in Python. Knowing it opens doors to data engineering, ML pipelines, and analytics work.
- **PyPI publishing** — shipping a Python package to PyPI is a concrete, repeatable skill. It requires setting up a pyproject.toml, writing a README, tagging releases, and running a CI workflow. Each of these is a real engineering practice.
- **JSON data modeling** — parsing and normalizing complex nested JSON from external sources (the game master) is a practical data engineering skill.

#### Market Relevance
CLI tools demonstrate that you understand how developers work and that you can build something useful without a GUI. This is a genuine signal to technical interviewers. Publishing to PyPI means anyone can `pip install` your tool, which shows you have shipped real software. The pandas experience is useful for data-adjacent roles. The Claude API integration shows LLM fluency in a tool-building context. This is a strong second-tier portfolio item — not as broadly marketable as a deployed web app, but very solid and specific.

#### Shipping Assessment
No hosting required. Shipping means publishing to PyPI, which is a documented process that takes an afternoon. The tool lives on the user's machine, which eliminates all the infrastructure complexity. This is the lowest-friction path to "shipped" of any project on this list.

#### AI Article Pairing
Strong pairing. The interesting article angle here is using Claude to reason over structured data in real time: "I gave Claude my roster in JSON and asked it to build me a team — here is the prompt engineering that made it actually useful, and here is where it failed." You also have the angle of using Claude during development to implement the CP math and validate it, which is a good article about AI-assisted implementation of domain-specific algorithms.

---

### #3 — Fly Fishing: Fly Tying Recipe Manager
**Domain:** CLI Tool

#### Feasibility Relative to Your Experience
Essentially identical in difficulty to the PVP Roster Analyzer, but the domain logic is simpler. There is no complex math, no external data parsing — the core challenge is designing a clean data model for fly patterns and implementing useful commands around it. Python, Typer, Rich, SQLite, and the Claude API are all the same stack. SQLAlchemy (the Python ORM for SQLite) maps naturally to what you know about objects and databases from Java.

This ranks third rather than second primarily because the Pokemon GO project gives you more interesting things to write about (domain-specific math, game data parsing, community data sources) and is a slightly richer engineering story. Both are excellent choices if you favor one interest over the other.

#### Skillset Expansion
- **Python proficiency** — same as above.
- **SQLite and SQLAlchemy** — local database design and ORM usage in Python. SQLAlchemy is the most widely used Python ORM and a common interview topic.
- **PDF generation** — a practical skill that comes up in many backend contexts (invoices, reports, exports).
- **CLI design and PyPI publishing** — same as above.

#### Market Relevance
Same profile as the PVP Analyzer. Strong for demonstrating that you can scope a problem, build a complete solution, and ship it in a usable form. The SQLAlchemy experience is particularly transferable to backend roles.

#### Shipping Assessment
Identical to the PVP Analyzer — PyPI publish, no hosting infrastructure.

#### AI Article Pairing
Good pairing. The clearest article angle is using Claude for the `suggest` command: what does it take to get a language model to give genuinely useful fly pattern recommendations rather than generic ones? Context design, prompt structure, and evaluating output quality are all writable topics.

---

### #4 — Fly Fishing: Python Client for Stream Condition Data
**Domain:** Open-Source Library

#### Feasibility Relative to Your Experience
The engineering here is clean and well-scoped: you are building a Python wrapper around two HTTP APIs, normalizing the responses into typed models, and adding reliability features (retries, caching). With your TypeScript background, the concept of typed data models maps directly to what Pydantic does in Python. httpx is one of the better-designed Python HTTP libraries and is easy to learn. The real challenge is thinking like a library author — designing a public API that is intuitive, stable, and handles edge cases gracefully. This is a design problem, not a language problem.

The CI/CD pipeline (GitHub Actions auto-publishing to PyPI on a version tag) is the most new concept here, but it is also one of the most directly teachable and most valuable skills you will build.

#### Skillset Expansion
- **Library API design** — thinking about your code as a product used by other engineers. This is a senior-level perspective and is impressive in a junior portfolio.
- **httpx and async Python** — modern Python HTTP patterns, including async/await, which maps to the async TypeScript patterns you likely already know.
- **Pydantic v2** — typed data validation in Python. This is increasingly standard in Python backends (especially FastAPI) and is a direct skill for backend roles.
- **GitHub Actions CI/CD** — automated testing and deployment pipelines. This appears in practically every engineering job description and is a gap in most junior portfolios.
- **Open-source project maintenance** — writing good documentation, handling versioning, managing a public API contract.

#### Market Relevance
Library authorship is genuinely respected because it demonstrates software design skills that CRUD app building does not. The GitHub Actions CI/CD experience is broadly applicable. Pydantic knowledge is specifically valuable for FastAPI-based Python backend roles, which are increasingly common. The USGS/NOAA API experience is reusable for any environmental data work.

The downside relative to the web app and CLI tools is that a library is less viscerally impressive to non-technical stakeholders — there is nothing to click on. But to a technical interviewer, "I designed and published a Python library with typed models, retry logic, and automated CI/CD" is a strong signal.

#### Shipping Assessment
PyPI publishing — same as the CLI tools. No hosting required. Slightly more involved than a CLI because you are writing documentation and setting up CI, but these are well-documented processes.

#### AI Article Pairing
Good pairing, with a specific angle: using Claude to help design the public API surface, and the experience of having Claude generate test fixtures from API documentation. There is also an honest article to write about whether Claude-generated tests actually covered the real edge cases, which is one of the more interesting AI-assisted testing topics.

---

### #5 — Pokemon Go: Game Master Library
**Domain:** Open-Source Library

#### Feasibility Relative to Your Experience
Very similar to the stream condition library in structure, but the source data (the Pokemon GO game master) is messier. The PokeMiners game master JSON uses protobuf field naming conventions that are verbose and inconsistently structured. Parsing it into clean models is a data modeling exercise with some real complexity. Your domain knowledge helps here — you know what the data should mean, which makes the modeling decisions clearer.

The auto-update GitHub Actions pipeline (which watches for new game master versions and auto-publishes) is the most interesting engineering feature and is also the most novel concept. It is achievable but will require learning GitHub Actions workflows from scratch.

#### Skillset Expansion
- All of the same Python library skills as #4 (Pydantic, httpx, GitHub Actions, PyPI).
- **Snapshot testing** — pinning known-good outputs as test fixtures and comparing future outputs against them. This is a practical testing pattern used in frontend and backend alike.
- **GitHub Actions automation** — specifically, cross-repo monitoring and automated release workflows. More advanced than basic CI.
- **Protobuf-adjacent data structures** — reading and normalizing protobuf-inspired JSON is a useful data engineering skill.

#### Market Relevance
Similar profile to #4. The auto-update pipeline is a distinctive feature that makes this library more interesting than a static wrapper. The Pokemon GO community is large enough that a well-maintained library here could attract actual users, which turns into organic validation of your work.

#### Shipping Assessment
Same as #4. PyPI publishing plus a GitHub Actions workflow. The workflow setup is the additional complexity.

#### AI Article Pairing
Strong pairing. Translating protobuf field names into clean Python models with Claude's help is a concrete, specific use case with interesting failure modes (Claude will sometimes get the game mechanics wrong, which is a good article).

---

### #6 — Pokemon Go: Local Raid Coordination Hub
**Domain:** Full-Stack Web App

#### Feasibility Relative to Your Experience
The language is familiar (TypeScript/Next.js), but this project has more independent complexity layers than the fly fishing web app. PostGIS adds geospatial query complexity that is nontrivial — latitude/longitude math, proximity queries, and geospatial indexing are concepts that take time to get right. Supabase Realtime (WebSocket-based live updates) adds another dimension that requires understanding async patterns and client-server state synchronization. The Web Push API is one of the more poorly documented browser APIs and has real cross-browser inconsistencies that will cost you time.

None of these are insurmountable, but each one is a meaningful obstacle on top of the foundational web app work. For a first shipping project, the compounding complexity is a real risk. You could spend two weeks on push notifications alone and still have a flaky implementation.

The fly fishing web app gives you the same TypeScript/Next.js core skills with fewer of these independent hard problems.

#### Skillset Expansion
- Everything from the fly fishing web app (#1).
- **PostGIS and geospatial queries** — valuable for location-aware applications but a specialized skill.
- **WebSocket-based real-time state** — increasingly relevant for collaborative and live applications.
- **Web Push API** — useful to know but notorious for implementation pain.

#### Market Relevance
Same strong profile as the fly fishing app. The real-time and geospatial features are additional signals but are not necessary to demonstrate core full-stack competence. If you are targeting companies building location-aware or real-time products, this has specific value.

#### Shipping Assessment
More complex than the fly fishing app. Vercel and Supabase both have good deployment stories, but getting PostGIS configured, seeding gym location data, and handling Web Push cross-browser are each non-trivial first-time tasks.

#### AI Article Pairing
Good pairing, same angles as the fly fishing app. The team composition AI feature has a clear, specific article: what information do you need to give Claude to get useful raid recommendations, and what does it get wrong?

---

### #7 — Fly Fishing: Stream Health & Fishability Index
**Domain:** Data Pipeline + Dashboard

#### Feasibility Relative to Your Experience
This project requires you to learn several tools simultaneously that have no direct analog in your current experience: Prefect (workflow orchestration), dbt (SQL transformation framework), and TimescaleDB (time-series PostgreSQL). Each of these is well-documented, but learning three new major tools at once while also writing the domain logic is a significant cognitive load for a first shipping project.

The core ideas are not algorithmically hard — you are pulling data, storing it, computing a score, and displaying it. But the infrastructure complexity (scheduling, pipeline failure handling, database extensions, dashboard deployment) is real overhead that can turn simple problems into week-long yaks. Streamlit deployment is genuinely beginner-friendly and is one of the easiest ways to put a live Python app on the internet, which partially offsets this.

If you have interest in data engineering roles specifically, this is a strong project to build. If you are more interested in general software engineering roles, the full-stack web app gives a broader and more immediately applicable skill set for a similar time investment.

#### Skillset Expansion
- **Prefect or Airflow patterns** — workflow orchestration is a core data engineering skill and appears in many data-adjacent roles.
- **dbt** — the dominant SQL transformation tool in the industry for anyone working with data warehouses or analytics.
- **TimescaleDB / time-series data** — relevant for IoT, monitoring, analytics, and financial data work.
- **Streamlit** — the fastest way to ship a data dashboard in Python. Very popular in data science and analytics contexts.
- **Data pipeline design** — thinking about ingestion, transformation, storage, and serving as distinct layers.

#### Market Relevance
High value specifically for data engineering, analytics engineering, and data-adjacent SWE roles. Lower signal for general full-stack or backend SWE roles than the web app projects. If you are interested in data, this is the right project. If you are not sure, the full-stack app gives broader optionality.

#### Shipping Assessment
Moderate complexity. Streamlit Cloud makes the dashboard easy to deploy. The pipeline itself requires a hosted PostgreSQL instance (Railway handles this) and a scheduled runner (GitHub Actions cron can substitute for Prefect if you want to simplify). The state stocking data sources vary by state and may require manual scraping work that is unpredictable.

#### AI Article Pairing
Strong pairing with a specific angle: using Claude to generate the weekly river report from structured data is a clean, demonstrable AI feature with measurable output quality. The pipeline context also gives you good material on using AI to write dbt models and data transformation logic.

---

### #8 — Pokemon Go: Meta Shift Tracker
**Domain:** Data Pipeline + Dashboard

#### Feasibility Relative to Your Experience
This is the highest-complexity project on the list for your current experience level. In addition to the pipeline tooling complexity from #7, it adds scraping (Leekduck.com) and a diff-based data architecture that requires careful design. Scrapers are inherently fragile — the target site can change its HTML structure at any time, breaking your ingestion silently. Designing a robust diff layer that correctly identifies what changed between snapshots (especially for hierarchical game data) is a nontrivial systems design problem.

The time-to-value curve is also longer than any other project. You will not have interesting historical data until the pipeline has been running for weeks or months, which means the dashboard will look sparse for most of the development period. This makes it hard to evaluate whether your work is correct and makes the shipping milestone feel distant.

#### Skillset Expansion
- Everything from #7 (Prefect, dbt, Streamlit).
- **Web scraping with BeautifulSoup and httpx** — a practical skill for data collection, but also one that introduces fragility and maintenance burden.
- **Diff-based data architecture** — a specific and interesting design pattern for change tracking.
- **Snapshot management** — versioning raw data to enable reprocessing.

#### Market Relevance
Same profile as #7 for data engineering roles. The scraping experience has some value for data collection roles. The diff architecture is an interesting system design topic for interviews.

#### Shipping Assessment
The hardest shipping story on this list. The dashboard requires historical data to be compelling, and historical data requires the pipeline to have run for a meaningful duration. Scraping introduces maintenance risk. The most likely failure mode is that the project works technically but feels incomplete because there is not yet enough data to show.

#### AI Article Pairing
Good article material specifically around using Claude to generate changelog narratives from structured diffs. The article writes itself: "I gave Claude a JSON diff of this week's Pokemon GO meta changes and asked it to write a human-readable summary — here is what the prompt looked like and what it got right and wrong."

---

## Summary Recommendation

**If you want the strongest resume impact and the clearest path to a live product:** build the Fly Fishing web app (#1). It uses TypeScript which you already know, deploys in minutes on Vercel, has a natural user-facing AI feature, and builds the exact skills that dominate the junior SWE hiring market right now.

**If you want a faster first ship and a gentler learning curve:** start with either CLI project (#2 or #3). You can publish to PyPI in a few weeks of part-time work, which gives you a concrete "shipped" item quickly. These also make excellent companion projects — build the CLI first to get comfortable with Python and shipping, then tackle the web app.

**If you are interested in data engineering:** the stream health pipeline (#7) is the right project, but go in knowing it requires learning more new tools simultaneously than the other options.

**What to avoid for now:** the two pipeline + dashboard projects are best approached after you have shipped something simpler. The Pokemon GO Meta Shift Tracker (#8) specifically has structural complexity that is likely to frustrate rather than teach at this stage.
