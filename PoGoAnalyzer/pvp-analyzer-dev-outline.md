# PVP Roster Analyzer — Development Outline
## A Phase-by-Phase Build Guide with AI-Assisted Workflow

---

## How to Use This Document

Each phase has four sections:

- **Build** — what gets created in this phase and how to prompt Claude Code to help
- **Review checklist** — what you must understand before moving on. Do not proceed until you can answer these without looking
- **Validate** — how to confirm the code actually works correctly, not just that it runs
- **Article opportunity** — observations worth capturing now that will become article material later

Work through phases in order. Later phases depend on earlier ones. Resist the urge to jump ahead.

---

## Pre-Phase: Study the Data Sources

Before writing a line of code, spend time understanding the raw data your tool will consume. You cannot design a good data layer around data you have not read.

**Do this manually:**

1. Go to the PvPoke GitHub repository and find the ranking JSON files (one per league). Download the Great League file and open it in a text editor. Understand:
   - What is the top-level structure?
   - How is each Pokemon entry keyed — by name, by Pokedex number, or something else?
   - Where are the stat product and rank fields?
   - How are moves represented?

2. Go to the PokeMiners pokemon-go-protobuf repository and find the latest game master JSON. Open it and find:
   - How are Pokemon base stats stored? (Attack, Defense, Stamina)
   - How is the CP multiplier table structured?
   - How are moves defined?
   - How is the type effectiveness chart represented?

3. Find the CP formula and IV-to-stat conversion formula in the Pokemon GO community documentation (Silph Road, GamePress, or Bulbapedia). Write them out by hand. You will implement these from scratch.

**Why this comes first:** Every design decision in the data layer will make more sense once you have seen what you are normalizing. Claude Code can generate models, but if you have not read the source data yourself, you will not know if the model is correct.

**Article capture:** Note your first impressions of the data. Is it clean? Is it human-readable? What would you have to explain to someone who had never seen it? This becomes the opening of your first article.

---

## Phase 1: Project Setup and Skeleton

### Build

Set up the Python project with the correct structure before writing any logic.

**Prompt Claude Code with:**
> "Set up a Python project called pvp-analyzer using pyproject.toml. The CLI entry point should be a command called `pvp`. Include these dependencies: typer, rich, sqlalchemy, httpx, pydantic, anthropic. Create the directory structure from this spec: [paste the directory structure from pvp-analyzer-spec.md]. Add a basic Typer app in cli.py with placeholder commands for: import-roster, add, list, rank, team, compare, sync, ask. Each command should just print 'not implemented yet'. Add a .env.example file showing the ANTHROPIC_API_KEY variable."

**Then do yourself:**
- Create a GitHub repository for the project
- Initialize git in the project folder, make an initial commit
- Create a virtual environment and install dependencies
- Run `pvp --help` and confirm all commands appear

### Review Checklist
- [ ] What does `pyproject.toml` do and why is it used instead of `setup.py`?
- [ ] What is the `[project.scripts]` section doing, and how does it connect `pvp` to your Python code?
- [ ] What is a virtual environment and why does every Python project need one?
- [ ] What does `typer.Typer()` create and how does `@app.command()` register a function as a CLI command?

### Validate
Run `pvp --help` from your terminal. Every command should appear with a description. Run `pvp rank` and confirm it prints "not implemented yet" without erroring.

### Article Opportunity
The difference between a Python script you run with `python main.py` and a proper installable CLI tool is not obvious to beginners. The pyproject.toml + entry point pattern is a good first article topic: what it means to "install" a CLI tool, and how Python packaging actually works.

---

## Phase 2: Data Layer — Models and Loaders

### Build

Define how data is stored and accessed. Two concerns: the user's roster in SQLite, and the cached meta files in JSON.

**Step 2A — SQLAlchemy roster model**

Prompt Claude Code with:
> "In data.py, define a SQLAlchemy model called RosterEntry with these fields: id (integer primary key), species (string, required), pokedex_number (integer), attack_iv (integer, 0-15), defense_iv (integer, 0-15), stamina_iv (integer, 0-15), level (float, 1-51), fast_move (string, nullable), charge_move_1 (string, nullable), charge_move_2 (string, nullable), nickname (string, nullable), notes (string, nullable). Use SQLAlchemy 2.0 style with DeclarativeBase. Add a function create_db_and_tables() that initializes the database at a path in the user's home directory (~/.pvp-analyzer/roster.db). Add basic CRUD functions: add_entry, get_all_entries, get_entry_by_id, delete_entry."

**Step 2B — JSON meta loaders**

Prompt Claude Code with:
> "In data.py, add functions to load and parse the cached meta JSON files from ~/.pvp-analyzer/cache/. Add load_pvpoke_rankings(league: str) that returns the parsed JSON for the given league (gl, ul, ml). Add load_game_master() that returns the full parsed game master. Add get_cache_metadata() that returns when each file was last synced. All functions should raise a clear, descriptive error if the cache file does not exist yet, telling the user to run pvp sync first."

**Read the generated code carefully.** For every line you do not understand, ask Claude Code to explain it before moving on.

### Review Checklist
- [ ] What is an ORM and what problem does SQLAlchemy solve over raw SQL?
- [ ] What does `DeclarativeBase` do in SQLAlchemy 2.0? How is it different from the older `Base = declarative_base()` pattern?
- [ ] Where on the filesystem will the database file be created? Why the home directory and not the project directory?
- [ ] What happens if `create_db_and_tables()` is called twice — does it overwrite the existing database?
- [ ] What is the difference between `Session` and `Engine` in SQLAlchemy?

### Validate
Write a small throwaway script (not part of the package) that imports `data.py`, calls `create_db_and_tables()`, adds a test entry, fetches it back, and prints it. Confirm the roster.db file appears in `~/.pvp-analyzer/`. Delete it and run again — confirm it recreates cleanly.

### Article Opportunity
SQLAlchemy 2.0 changed a lot. If you have used databases in Java (JDBC, JPA, Hibernate), there are direct analogues worth writing about — ORM concepts transfer, the API just looks different. "What a Java developer needs to know to use SQLAlchemy" is a specific, searchable article topic.

---

## Phase 3: Sync Module

Build sync before the engine because the engine depends on having real data to work with.

### Build

**Prompt Claude Code with:**
> "In sync.py, implement a sync() function that does the following: (1) Creates ~/.pvp-analyzer/cache/ if it doesn't exist. (2) Downloads the Great League, Ultra League, and Master League ranking JSON files from the PvPoke GitHub repository using httpx. (3) Downloads the latest game master JSON from the PokeMiners repository. (4) After each download, validates that the response is valid JSON and that it contains the top-level keys we expect — for PvPoke this should be a list of ranked entries, for the game master this should contain a key like 'itemTemplate' or similar (use whatever you found during the pre-phase data study). (5) Writes each file to the cache directory with a timestamp metadata file. (6) Prints progress using Rich as each file downloads."

**Then implement the `sync` CLI command in cli.py** by calling `sync.sync()` and handling errors with a clear user message.

### Review Checklist
- [ ] What is `httpx` and how does it differ from the built-in `urllib` or the popular `requests` library?
- [ ] What does a non-200 HTTP status code mean and how should the sync function handle it?
- [ ] What does it mean to validate JSON shape — and why is it not enough to just check that the file parses as valid JSON?
- [ ] What happens to the old cache files when sync runs again? Are they overwritten or versioned?

### Validate
Run `pvp sync` from the terminal. Confirm the files appear in `~/.pvp-analyzer/cache/`. Open them and confirm they match what you saw in the pre-phase data study. Run sync a second time and confirm it does not error or duplicate files.

### Article Opportunity
The validation step — checking that the data shape matches expectations — is where a lot of real production pipelines fail silently. This is a good angle: "Why I added shape validation to my data sync and what it caught."

---

## Phase 4: Analysis Engine — CP Math and Rank Lookup

This is the most technically precise phase. Correctness matters more than speed.

### Build

**Step 4A — Write the tests first**

Before touching engine.py, create `tests/test_engine.py` and write test cases using known correct values. Go to PvPoke, look up three or four specific Pokemon with specific IVs, and note their CP at level 40 and their stat product. These become your test cases. Write the test functions (they will fail at first — that is expected).

**Step 4B — Implement the engine**

Prompt Claude Code with:
> "In engine.py, implement the following using the Pokemon GO CP formula and IV conventions documented by the community. (1) A function compute_cp(base_attack, base_defense, base_stamina, attack_iv, defense_iv, stamina_iv, level) that returns CP as an integer, using the CP multiplier table from the game master. (2) A function compute_stat_product(base_attack, base_defense, base_stamina, attack_iv, defense_iv, stamina_iv, level) that returns the stat product as a float. (3) A function lookup_rank(species: str, attack_iv: int, defense_iv: int, stamina_iv: int, league: str) that searches the PvPoke ranking data for the matching entry and returns rank, percentile, and stat product. (4) A function get_optimal_moveset(species: str, league: str) that returns the recommended fast move and charge moves from PvPoke data. Use the data loader functions from data.py."

Run the tests. They will likely fail on the first try. Debug with Claude Code by pasting the failing test output and asking what is wrong. Repeat until all tests pass.

### Review Checklist
- [ ] Walk through the CP formula by hand with one Pokemon. Does your implementation produce the same result?
- [ ] What is a CP multiplier and why does it exist in the formula?
- [ ] What is stat product and why is it used as a proxy for PVP performance instead of CP?
- [ ] How does `lookup_rank` find the right entry — by iterating the full list or by a keyed lookup? Which is faster and does it matter at this scale?
- [ ] What does your test do if the species name does not match the game master exactly (e.g., "Galarian Stunfisk" vs. "STUNFISK_GALARIAN")?

### Validate
Cross-check every test case value against PvPoke directly in your browser. If your computed CP or rank differs from PvPoke by even one point, something is wrong. Do not move on until they match.

### Article Opportunity
This phase generates the most interesting debugging material. The CP formula has subtle details (the multiplier table uses specific breakpoints, the final CP is floored, not rounded) that will cause off-by-one errors. Document every discrepancy you find and how you resolved it. "Using AI to implement a math formula — and why I still had to validate every output manually" is a strong article.

---

## Phase 5: Roster Module

### Build

**Prompt Claude Code with:**
> "In roster.py, implement the following using the data layer from data.py: (1) import_from_file(path: str) that reads a CSV or JSON file matching the roster schema, validates each entry using Pydantic, and bulk-inserts into the database. Pydantic should enforce IV ranges 0-15, level between 1 and 51, and that species is non-empty. (2) add_interactive() that prompts the user for each field using Typer's prompt utilities and inserts one entry. (3) list_all() that returns all roster entries formatted as a Rich table with columns for species, IVs, level, and moves. (4) delete_by_id(id: int) that removes an entry after confirming with the user."

**Then wire up the CLI commands:** import-roster, add, list, and delete in cli.py by calling the corresponding roster functions.

**Also create a sample roster CSV** with five or six of your actual Pokemon so you have real data to work with for the rest of development.

### Review Checklist
- [ ] What is Pydantic doing in the import flow that plain Python code would not do automatically?
- [ ] What happens if one row in a 50-row CSV fails validation — does the whole import abort or do valid rows still get inserted? Which behavior does your implementation have, and is that the right choice?
- [ ] What does "bulk insert" mean in SQLAlchemy and why might it be faster than inserting one row at a time?
- [ ] What does the `Confirm` prompt pattern in Typer do and why is it used before destructive operations?

### Validate
Import your sample CSV. Run `pvp list` and confirm all entries appear correctly. Delete one entry, confirm it disappears. Try importing a CSV with a bad IV value (e.g., 16) and confirm the error message is clear and useful.

---

## Phase 6: Core CLI Commands — Rank and Compare

### Build

**Prompt Claude Code with:**
> "In cli.py, implement the rank command. It takes a species name as an argument and an optional --league flag (default GL). It should: look up the species in the user's roster, compute the CP using engine.py, look up the rank from PvPoke data, and display a Rich panel with species name, IVs, level, computed CP, rank, percentile, stat product, and recommended moveset. If the species is not in the roster, display a helpful error."

**Then:**
> "Implement the compare command. It takes two species names and an optional --league flag. Display a side-by-side Rich table showing both Pokemon's rank, stat product, typing, and recommended moveset. Highlight the better value in each row."

### Review Checklist
- [ ] What happens if the user has two Medicham entries — how does rank pick which one?
- [ ] How does the Rich `Panel` differ from a `Table` in terms of when you would use each?
- [ ] What is the difference between an argument and an option in Typer/CLI convention?
- [ ] If a species name contains a space (e.g., "Galarian Stunfisk"), how does the user pass it on the command line, and does your implementation handle this?

### Validate
Run `pvp rank "Medicham" --league GL` with a Medicham you know the IVs of. Compare every displayed value against PvPoke directly. Run `pvp compare` with two Pokemon and confirm the output is readable and accurate.

---

## Phase 7: Team Builder

### Build

**Prompt Claude Code with:**
> "In engine.py, implement a build_team(league: str, cover: list[str] | None) function. It should: (1) Load all roster entries. (2) For each entry, look up its PvPoke rank for the given league. (3) Score entries by rank (lower rank number = better). (4) If cover is provided, filter to ensure the selected team collectively covers all the requested types — a type is 'covered' if at least one team member has a move that is super effective against that type, or has that type as one of its own types. (5) Select a team of three non-redundant Pokemon (avoid picking two Pokemon with the same primary type if alternatives exist). (6) Return the selected team with their rank, typing, and key coverage. If no valid team can be assembled from the roster, return the best available with a note about missing coverage."

Wire up the `team` command in cli.py.

### Review Checklist
- [ ] What algorithm is the team builder using to select three Pokemon? Is it greedy (pick the best, then the next-best that adds coverage) or exhaustive (try all combinations)?
- [ ] What is the difference between a Pokemon's typing covering a threat versus a move covering a threat — and which does your implementation use?
- [ ] What edge case occurs if the user's roster has fewer than three Pokemon?
- [ ] If `--cover dragon` is specified but no roster entry covers Dragon-type, how does the function behave?

### Validate
Run the team builder with and without coverage constraints. Manually verify the output makes sense for the league meta (a team of three Magikarps should not appear as the top suggestion). Test the edge case where your roster has only two entries.

### Article Opportunity
The team builder is an algorithm problem disguised as a feature. The difference between a greedy approach and an exhaustive search, and why you chose one over the other for this scale of data, is a good technical writing topic. This also sets up the AI comparison: does Claude suggest a better team than the algorithm does?

---

## Phase 8: AI Layer

This is the phase that most directly feeds your article series. Slow down here and document everything.

### Build

**Step 8A — Design the prompt before writing the code**

Before writing any code, write out the prompt structure manually in a text file. Answer these questions:
- What information does Claude need to answer a typical question about your roster?
- How do you represent 50 Pokemon compactly without blowing past the context window?
- How much meta context (top-ranked opponents) does Claude need to give useful advice?
- What format produces the clearest responses — prose, JSON, structured sections?

Experiment by sending test prompts manually to Claude.ai (web) with real roster data. See what responses look like before automating it.

**Step 8B — Implement the AI layer**

Prompt Claude Code with:
> "In ai.py, implement an ask(query: str, league: str) function. It should: (1) Load the user's full roster from data.py and format each entry as a compact string: 'Species (Type/Type): IVs X/X/X, Rank #N in [league], Moves: FastMove / ChargeMove1 / ChargeMove2'. (2) Load the top 20 ranked Pokemon for the given league from PvPoke data for meta context. (3) Build a system prompt that describes the assistant's role and the data it has access to. (4) Build a user message that includes the formatted roster, the meta context, and the user's query. (5) Send the request to the Anthropic API using the anthropic Python SDK with streaming enabled. (6) Stream the response tokens to the terminal using Rich's Live display. Handle API errors (missing key, rate limit, network failure) with clear error messages."

Wire up the `ask` command in cli.py.

### Review Checklist
- [ ] What is the difference between a system prompt and a user message in the Claude API, and how does your implementation use each?
- [ ] What does streaming mean at the API level — what is actually being sent back from Anthropic's servers token by token?
- [ ] How does Rich's `Live` display work for streaming output, and how is it different from just printing each token as it arrives?
- [ ] How does your implementation handle the case where the roster has 100 Pokemon — does the prompt fit in Claude's context window? What is the context limit?
- [ ] Where is the API key being read from, and what happens if it is not set?

### Validate
Run several queries and evaluate the response quality critically:
- Ask something simple: "Which of my Pokemon has the highest Great League rank?"
- Ask something that requires reasoning: "Build me a team that covers Fairy and Dragon types"
- Ask something where Claude might be wrong: "Is [specific Pokemon] good against Cresselia?"

For each response, verify the factual claims. Note every error or hallucination. This is your primary article material for the AI layer.

### Article Opportunity
This phase produces at least two articles: (1) the prompt engineering decisions — what you included, what you cut, and why — and (2) the accuracy evaluation — specific examples of Claude being right, being wrong, and being confidently wrong. Both should be specific and honest. The confidently wrong examples are the most interesting.

---

## Phase 9: Display Polish and Error Handling

### Build

With all commands functional, make one pass through the display layer for consistency.

**Prompt Claude Code with:**
> "Review display.py and cli.py. Ensure: (1) Every command that can fail (species not found, cache missing, API key not set) has a clear, specific error message that tells the user what went wrong and what to do. (2) Every table has consistent column widths, color coding (rank highlighting — green for top 100, yellow for top 500, no color below), and headers. (3) Long-running operations (sync downloads, AI response waiting) have a progress indicator. Standardize all Rich formatting into display.py so no Rich calls exist in cli.py, roster.py, or engine.py directly."

### Review Checklist
- [ ] Why is it better to put all display logic in a single module rather than scattered through the codebase?
- [ ] What is the principle of separation of concerns and how does this refactor apply it?
- [ ] What makes an error message good versus bad? Look at the error messages you currently have and evaluate each one.

---

## Phase 10: Testing

### Build

**Write these tests yourself** where possible, using Claude Code to fill gaps and generate fixtures:

**`tests/test_engine.py`** — you should have started this in Phase 4. Add any missing cases:
- CP computation for at least five Pokemon with known values
- Stat product computation
- Rank lookup for a known IV combination
- Team builder: correct team selected with no coverage constraint
- Team builder: coverage constraint satisfied when possible
- Team builder: graceful behavior when constraint cannot be satisfied

**`tests/test_roster.py`**
- Valid CSV imports correctly
- Invalid IVs are rejected with a clear error
- Duplicate species are handled (two Medicham entries should both import)
- Delete removes the correct entry

**Prompt Claude Code with:**
> "In tests/, add a conftest.py that sets up an in-memory SQLite database for tests so no test writes to the real roster.db file. Add fixture JSON files to tests/fixtures/ containing a pinned snapshot of PvPoke GL rankings (just 10 entries) and a minimal game master snippet covering the species used in tests."

### Review Checklist
- [ ] What is a test fixture and why use pinned data instead of hitting the real PvPoke files during tests?
- [ ] What is an in-memory SQLite database and why is it useful for testing?
- [ ] What is `conftest.py` in pytest and what role does it play?
- [ ] What is the difference between a unit test and an integration test, and which type are your engine tests?

### Validate
Run `pytest` from the project root. All tests should pass. Check coverage if you add `pytest-cov` — aim for the engine module to be well-covered since it contains correctness-critical math.

---

## Phase 11: README and PyPI Publishing

### Build

**Write the README yourself.** It should contain:
- One-sentence description
- Installation instructions (`pip install pvp-analyzer`)
- A short demo of the key commands with example output (copy from your terminal)
- The roster CSV format with an example
- How to set the API key
- A link to your article series

**Then set up publishing:**

Prompt Claude Code with:
> "Set up GitHub Actions for this project. Create two workflows: (1) .github/workflows/test.yml that runs pytest on every push and pull request, on Python 3.11 and 3.12. (2) .github/workflows/publish.yml that triggers when a version tag is pushed (e.g., v0.1.0), builds the package with 'python -m build', and publishes to PyPI using the pypa/gh-action-pypi-publish action. Show me the pyproject.toml changes needed to include package metadata (description, author, license, classifiers)."

Create a PyPI account, generate an API token, and add it to your GitHub repository secrets.

### Review Checklist
- [ ] What does `python -m build` produce and what are the `.whl` and `.tar.gz` files it creates?
- [ ] What is a version tag in git and how is it different from a branch?
- [ ] What is a GitHub Actions secret and why does the PyPI token need to be stored there rather than in the workflow file?
- [ ] After publishing, how would someone install your tool? Run that command yourself on a clean virtual environment to confirm it works end to end.

### Validate
Publish version 0.1.0 to PyPI. Open a new terminal, create a new virtual environment, and run `pip install pvp-analyzer`. Run `pvp --help`. If the commands appear, you have shipped.

---

## Article Series Outline

Based on what you will have built and observed, here are six article topics roughly in the order they make sense to write. Write each one while the phase is fresh — do not batch them at the end.

| Article | Best Time to Write | Core Angle |
|---|---|---|
| "What I'm building and why — a Pokemon GO PVP analyzer with Claude" | After Phase 1 | Project purpose, tech stack choices, what AI-assisted development actually means |
| "How I modeled Pokemon GO's game master data in Python" | After Phase 2-3 | Data modeling decisions, what the raw data looks like versus the clean API, where Claude helped and where it was wrong about the schema |
| "Implementing CP math from community docs with AI assistance" | After Phase 4 | The CP formula, how AI translated it to code, every validation failure and how it was found, why you could not trust the output without checking it |
| "Prompt engineering for a domain-specific AI feature" | After Phase 8 | What information Claude needs, how you structured the context, what you cut, what the first prompt looked like versus the final one |
| "Where Claude was right and where it was confidently wrong" | After Phase 8 | Specific examples with exact queries and responses. Honest and specific — this is the most valuable article |
| "Shipping a Python CLI to PyPI — what I wish I'd known" | After Phase 11 | The publishing process, what pyproject.toml actually does, GitHub Actions for automated publishing, what "installed" means |

---

## Checkpoints Before Moving Between Phases

Before closing out any phase, answer these three questions in your notes:

1. **What did I build?** One or two sentences describing the output of this phase.
2. **What did I learn that surprised me?** Something you did not know before and now understand. If nothing surprised you, you went too fast.
3. **Where did the AI help and where did it steer me wrong?** Be specific. Vague answers ("Claude was great!") are not useful article material.

These notes become the raw material for the articles. The articles that resonate are the ones written by someone who clearly did the work and has specific things to say about it.
