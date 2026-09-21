# Columnbo — conversational analytics from scratch

The goal: have AI agents build every component needed to go from raw data
to a chatbot you can ask questions of in plain English.

The chain looks like this:

1. **Raw data** — a set of parquet files.
2. **Data catalog** — an agent reads the data and writes down what each
   column means.
3. **Malloy semantic model** — an agent turns the catalog into a
   [Malloy](https://www.malloydata.dev/) model: the joins, dimensions and
   measures a question can use.
4. **Malloy Publisher** — serves the model over REST and MCP.
5. **Chat UI** — you type a question, Claude writes the Malloy query,
   Publisher runs it, and you get an answer with a chart when one helps.
6. **Feedback loop** — when a question fails because the model is missing
   something, an agent proposes a change to the model. Each change you
   accept is committed to git, so the model's history is readable and any
   change can be undone.

The first version took roughly an hour and required zero hand-written
code. The prompts that drove each step are in [docs/prompts/](docs/prompts/).
They were generated with Claude after I explained the idea.

**Why Magic: The Gathering data?** The demo uses
[MTGJSON](https://mtgjson.com/), a public dataset of Magic cards and sets.
I picked it only because I know that data well, so I can spot a wrong
answer quickly. Nothing in the approach is specific to cards.

**Where this went.** The project has since grown into **Columnbo**. There
is a live demo at <http://magic.columnbo.com>. This repo is the original
proof of concept.

![Test UI screenshot](screenshot.jpg)

The stack:

```text
[Browser UI]  ->  [Node proxy]  ->  [Claude API]       (NL → Malloy)
                       |
                       └──────>  [Malloy Publisher]  ->  [DuckDB]  ->  [MTGJSON parquet]
```

## Quick start

You need Node 18+ and an Anthropic API key. Details are in
[Prerequisites](#prerequisites).

1. Clone the repo:

   ```bash
   git clone https://github.com/peltzel/semantic-autopilot-public.git
   cd semantic-autopilot-public
   ```

2. Get the data. Download
   <https://mtgjson.com/api/v5/AllPrintingsParquetFiles.zip>, unzip it, and
   copy `cards.parquet` and `sets.parquet` into a `data/` folder in the
   repo root.

3. Put your API key in a file called `.env.local` in the repo root:

   ```bash
   ANTHROPIC_API_KEY=sk-ant-...
   ```

4. Start Malloy Publisher from the repo root:

   ```bash
   nohup npx -y @malloy-publisher/server@latest --port 4000 --server_root . --watch-env mtgjson \
     > /tmp/publisher.log 2>&1 &
   ```

5. Start the test UI:

   ```bash
   cd test-ui
   set -a; source ../.env.local; set +a
   node server.js
   ```

6. Open <http://localhost:5173> and ask a question, for example "Show me
   cards by rarity".

The sections below walk through each phase in the order it was built.

## What's in the repo

| Path                       | What it is                                                                         |
|----------------------------|------------------------------------------------------------------------------------|
| `data/`                    | MTGJSON parquet files (`cards.parquet`, `sets.parquet`). **Gitignored.**           |
| `catalog/`                 | ODCS v3 data catalogs documenting the cards and sets schemas.                      |
| `Malloy-source-files/`     | Malloy semantic model + Publisher package manifest.                                |
| `test-ui/`                 | Minimal web UI for NL → Malloy → results via Claude.                               |
| `publisher.config.json`    | Publisher server config (projects → packages).                                     |
| `docs/prompts/`            | Task prompts used to drive each phase.                                             |
| `.env.local`               | `ANTHROPIC_API_KEY=...`. **Gitignored.**                                           |
| `LICENSE`                  | MIT licence.                                                                       |

### The prompts

Each phase was driven by one prompt, given to a coding agent:

| Phase | Prompt |
|-------|--------|
| 1 — Data load | [1-setup-db.md](docs/prompts/1-setup-db.md) (currently empty) |
| 2 — Data catalog | [2-generate-data-catalog.md](docs/prompts/2-generate-data-catalog.md) |
| 3 — Malloy semantic model | [3-write-malloy-files.md](docs/prompts/3-write-malloy-files.md) |
| 4a — Malloy Publisher | [4a-install-malloy-publisher.md](docs/prompts/4a-install-malloy-publisher.md) |
| 4b — Natural-language test UI | [4b-MCP-integration-and-test-page.md](docs/prompts/4b-MCP-integration-and-test-page.md) |
| 5 — Error recovery & model evolution | [5-error-recovery-and-model-evolution.md](docs/prompts/5-error-recovery-and-model-evolution.md) |
| 6 — Git-tracked model evolution | [6-version-controlled-evolution.md](docs/prompts/6-version-controlled-evolution.md) |
| 7 — Analyst mode | [7-analyst-mode-prompt.md](docs/prompts/7-analyst-mode-prompt.md) |
| 8 — Conversational threading | [8-conversational-threading.md](docs/prompts/8-conversational-threading.md) |

The prompts are kept exactly as they were originally run, so they refer to
the project's original local folder name, "Malloy Test".

## Prerequisites

- **DuckDB CLI** (`brew install duckdb`) — for ad-hoc inspection only.
- **Node 18+** — Publisher and the test UI both run on Node.
- **MTGJSON parquet files** in `data/`. Download from
  <https://mtgjson.com/api/v5/AllPrintingsParquetFiles.zip>, unzip, and
  copy `cards.parquet` and `sets.parquet` into `data/`.
- **Anthropic API key** in `.env.local`:
  ```bash
  ANTHROPIC_API_KEY=sk-ant-...
  ```

## Phase 1 — Data load

Parquet files live in `data/`. Validate with DuckDB:

```bash
duckdb -c "SELECT COUNT(*) FROM 'data/cards.parquet';"   # → 109,733
duckdb -c "SELECT COUNT(*) FROM 'data/sets.parquet';"    # → 855
```

## Phase 2 — ODCS catalogs

Data contracts in [catalog/cards.odcs.yaml](catalog/cards.odcs.yaml) and
[catalog/sets.odcs.yaml](catalog/sets.odcs.yaml). These document the
analytically-useful columns (41 of 81 for cards, 16 of 22 for sets) with
business-friendly descriptions, examples, and the
`cards.setCode → sets.code` relationship. They feed the Malloy generation
step and the LLM system prompt.

## Phase 3 — Malloy semantic model

Generated from the ODCS catalogs into
[Malloy-source-files/](Malloy-source-files/):

- `sets.malloy` — sets dimension table with release-date parsing
- `cards.malloy` — cards source, joins to sets, exposes computed
  dimensions (`is_multicolor`, `is_creature`, `color_count`, …) and views
  (`by_rarity`, `by_set`, `color_distribution`, …)
- `queries.malloy` — smoke-test `run:` statements
- `publisher.json` — package manifest for Malloy Publisher

**Important:** List-like MTGJSON columns (`colors`, `types`, `keywords`,
…) are stored as comma-joined `VARCHAR`, not native `LIST` types. The
model uses `~ '%X%'` substring matching, not Malloy's `?` array operator.

**Paths:** the parquet paths inside `sets.malloy` / `cards.malloy` are
relative (`../data/*.parquet`). DuckDB resolves them from the working
directory, not from the model file. So run `malloy-cli` from inside
`Malloy-source-files/` (the command below does the `cd`), and start
Publisher with `--watch-env mtgjson` (see Phase 4a).

Compile / run with the Malloy CLI:

```bash
npm install -g @malloydata/cli
cd Malloy-source-files && malloy-cli run queries.malloy
```

## Phase 4a — Malloy Publisher

Runs the Malloy model as a REST + MCP service.

```bash
# from the repo root
nohup npx -y @malloy-publisher/server@latest --port 4000 --server_root . --watch-env mtgjson \
  > /tmp/publisher.log 2>&1 &
```

The `--watch-env mtgjson` flag is required. It makes
`publisher_data/mtgjson/mtgjson-analytics` a link to `Malloy-source-files/`
instead of a copy, so the relative `../data/` paths in the model still
point at the parquet files. Without the flag every query fails with
`IO Error: No files found that match the pattern "../data/..."`. If you
already have a `publisher_data/` folder from a run without the flag,
delete it first.

If port 4000 is busy, pass a different `--port`. The Publisher URL is
hard-coded in two places, so change both to match: `PUBLISHER` in
[test-ui/server.js](test-ui/server.js) and `PUBLISHER_BASE` in
[test-ui/model-enhancer.js](test-ui/model-enhancer.js).

- **Web UI / REST API:** <http://localhost:4000>
- **MCP endpoint:** <http://localhost:4040> (JSON-RPC)
- **Stop:** `pkill -f "@malloy-publisher/server"`
- **Logs:** `/tmp/publisher.log`

Query endpoint used by the test UI:

```http
POST /api/v0/projects/mtgjson/packages/mtgjson-analytics/models/cards.malloy/query
body: { "query": "run: cards -> by_rarity", "compactJson": true }
```

If Publisher gets confused after a source-file edit, delete the
copy-on-load directory and restart: `rm -rf publisher_data && <restart>`.

## Phase 4b — Natural-language test UI

Minimal browser UI in [test-ui/](test-ui/):

```bash
cd test-ui
set -a; source ../.env.local; set +a
node server.js
# open http://localhost:5173
```

Flow: browser → `server.js` calls Claude with
[test-ui/system-prompt.txt](test-ui/system-prompt.txt) (which lists every
dimension/measure/view from `cards.malloy`) → Claude emits a Malloy
query → `server.js` POSTs it to Publisher → rows render as a table.

See [test-ui/README.md](test-ui/README.md) for the detailed architecture
and known limitations.

## Phase 5 — Error recovery & model evolution

The test UI includes an automatic retry loop and an experimental model
enhancement flow.

**Retry loop** (in [test-ui/server.js](test-ui/server.js)): when a Malloy
query fails, the server feeds the failed query + error back to Claude
with the "Retry behaviour" section of `system-prompt.txt`, and tries up
to **2 more times** (`MAX_RETRIES`). Every attempt shows up in the UI as
a separate card so you can see how Claude walked back from the error.

**Model enhancement** (in [test-ui/model-enhancer.js](test-ui/model-enhancer.js)):
if all retries still fail with a "field-not-found"–style error, the UI
offers to extend the model. The flow:

1. Server calls Claude in JSON mode with the question, the failed query,
   the error, and the full current contents of `cards.malloy`. Claude
   returns `{ file, changeType, snippet, reasoning }`.
2. The UI shows the proposed snippet + reasoning with **Apply / Skip**
   buttons. Nothing touches disk until you click Apply.
3. On Apply, the server:
   - Backs up the original to `backups/<file>.<timestamp>.bak`.
   - Splices the snippet just before the closing `}` of the source
     extension, with an `-- @ai-added <ts>` marker line.
   - Mirrors the change into `publisher_data/.../cards.malloy` (Publisher
     copies packages on first load and never re-syncs from the source
     dir, so we have to write both locations). With `--watch-env` that
     folder is a link to the source folder, so this second write is
     harmless and the reload below is what matters.
   - Calls `GET /api/v0/projects/mtgjson/packages/mtgjson-analytics?reload=true`
     to make Publisher re-parse.
   - Re-runs the original question through the full ask loop.

**Safety rails:**

- Allowlist: only `cards.malloy` and `sets.malloy` can be edited.
- Hard cap: `MAX_CHANGES_PER_SESSION = 3` per server lifetime — restart
  the UI server to reset.
- Backups are unconditional and timestamped.
- All asks, proposals, and applies are appended to `logs/sessions.jsonl`
  as one JSON record per line.

**Known limitation:** the NL→Malloy system prompt is reloaded fresh on
every `/api/ask` call, but its content (the field list) is **static** —
it doesn't reflect newly-added dimensions. After a successful
enhancement, Claude often re-runs the question successfully via a
*different* path rather than using the new field. To make the new field
reachable in subsequent questions you'd need to either (a) regenerate
`system-prompt.txt` after each apply, or (b) inject "recently-added
fields" into the user message dynamically.

## Phase 6 — Git-tracked model evolution

Every applied model change now creates a real git commit on the current
branch. The flow extends Phase 5:

1. Apply writes the snippet, mirrors to `publisher_data/`, hits reload —
   *and then* runs `git add Malloy-source-files/<file> && git commit` with
   a structured message. (With `--watch-env`, `publisher_data/` is a
   link to the source folder, so the mirror and re-sync steps here are
   harmless; the reload is what matters.)
2. The UI shows the short commit hash and subject line in the
   enhancement status, and refreshes the **Model history** panel.
3. **Undo last model change** in the history panel calls `/api/rollback`,
   which:
   - Finds the most recent commit touching `Malloy-source-files/`.
   - Runs `git revert --no-edit <hash>` (creates a new revert commit;
     does not rewrite history).
   - Re-syncs `publisher_data/` from the now-reverted source files.
   - Hits the Publisher reload endpoint.

### Commit message shape

```text
Auto: add_dimension for: "<user question>"

Reasoning: <Claude's explanation>

Code added:
<the snippet>

Error that triggered this:
<the original error message>
```

This makes `git log --oneline -- Malloy-source-files` a readable
chronicle of how the model grew, and `git diff HEAD~3 -- Malloy-source-files`
shows exactly what was added over the last three changes.

### Endpoints

| Method | Path                | Purpose                                 |
|--------|---------------------|-----------------------------------------|
| GET    | `/api/history`      | Last N commits touching the model dir   |
| POST   | `/api/rollback`     | Revert most recent model commit         |
| POST   | `/api/apply`        | Apply change (now also commits)         |

### Safety

- Allowlist still enforced (`cards.malloy`, `sets.malloy` only).
- `MAX_CHANGES_PER_SESSION` cap unchanged (3 per server lifetime).
- Backups in `backups/` are still written even though git also has the
  history — they're a belt-and-suspenders for the publisher_data copy.
- Rollback uses `git revert` (additive) not `git reset` (history rewrite).
- All git operations are scoped to the repo root via `execFileSync`
  with explicit `cwd`; no shell interpolation.

## Phase 7 — Analyst mode (NYT-style visualization)

The test UI now makes **two** Claude calls per question:

1. **Query generation** (Phase 4/5 flow) — translate NL → Malloy, execute,
   retry on error.
2. **Analysis** (new) — hand the successful rows back to Claude under a
   separate analyst system prompt
   ([test-ui/analyst-prompt.md](test-ui/analyst-prompt.md)). The analyst
   returns JSON with `insight`, `analysis`, an optional `chart`, and
   optional `caveats`.

The UI leads with the insight as a large serif pull-quote. The raw
query, attempts, and rows are hidden inside a collapsible "Show query &
raw data" section — they're still there for debugging, just no longer
the headline.

### Chart philosophy

Charts are rendered with Chart.js, styled for clarity over decoration:

- Serif title that states the *finding*, not the axes ("Rares get
  reprinted more than commons", not "Card count by rarity").
- Muted achromatic bars with a single red accent on annotated points.
- Gridlines reduced to `#e5e5e5`; no legends (single-series only).
- Annotations rendered as direct-label callouts below the chart.

The analyst is instructed to **withhold** charts for single numbers,
two-item comparisons, or cases where prose tells the story better.
Empirically this works well — on the smoke-test battery:

| Question                                                | Chart? | Why                                 |
|---------------------------------------------------------|--------|-------------------------------------|
| "How many total cards are there?"                       | no     | single number                       |
| "Show me cards by rarity"                               | bar    | 6 categories worth comparing        |
| "How has the number of cards per year changed?"         | line   | 34-year trend                       |
| "Are there more red or blue cards?"                     | no     | two-item comparison → just say it   |
| "What's the relationship between mana value and rarity?"| bar    | 5 categories + a real outlier       |
| "Which 5 sets have the most printings?"                 | hbar   | long category labels                |

The analyst also flags real data quirks unprompted — the Gleemax
outlier (one card with mana_value = 1,000,000 inflating the rare
average), incomplete 2025/26 years, and the printings-vs-unique-cards
distinction all showed up as `caveats` without being asked.

## Phase 8 — Conversational threading

The UI is now a thread, not a single-shot. Each turn is rendered as a Q/A
card (newest on top), and the last 4 turns are sent back to the server
as `history` on every `/api/ask` and `/api/apply` call. The server
splices that history into both Claude calls:

- **Query generation** gets a "Recent conversation context" block
  describing previous questions, queries, and result summaries, with
  rules: pronouns ("them", "those") point at the most recent result;
  "the same"/"of those" means carry forward filters; clear topic
  switches should drop carried filters.
- **Analysis** gets the previous question + insight, with instructions
  to frame the response with continuity phrases ("Of those…", "Among
  the…") and skip context the user already heard.

History is capped at 4 turns (`HISTORY_TURNS` in [test-ui/app.js](test-ui/app.js))
to keep token usage bounded. The payload is slim — `{question, query,
rows, insight}` — chart specs and full attempt traces are not sent.

A **Clear thread** button resets `conversation` and empties the DOM
thread. Each Q/A card has a unique chart canvas id so multiple charts
can coexist. Cards render optimistically with a "Thinking…" loading
state, then swap in the analyst output when the call completes.

### Smoke test on the three canonical flows

| Flow | Result |
| --- | --- |
| **Filter down**: dogs → red dogs → avg mana of those | Each turn correctly carried filters: T2 added `and is_red`; T3 kept both and switched aggregate to `avg_mana_value`. Insights used "Of those…" framing. |
| **Re-sort**: top 10 by count → sort alphabetically | T2 kept the same group_by + aggregate, only flipped `order_by` from `card_count desc` to `card_name asc`. |
| **Topic switch**: dogs → total sets | Query layer dropped the dog filter cleanly. Analyst layer over-applied the continuity instruction and reconnected the answer back to dogs ("Those 365 Dog cards are spread across 749 sets"). Technically correct count, narratively over-eager. |

The topic-switch nuance is a known prompt-tuning trade-off: telling the
analyst to "use continuity phrases" makes follow-ups feel coherent but
can backfire on genuine topic switches. The fix would be to teach the
analyst to detect topic switches the same way the query generator
already does — left as a future improvement.

## Ports used

| Service          | Port | URL                     |
|------------------|------|-------------------------|
| Malloy Publisher | 4000 | <http://localhost:4000> |
| Publisher MCP    | 4040 | <http://localhost:4040> |
| Test UI proxy    | 5173 | <http://localhost:5173> |

## Known data quirks

- `cards.name` is **not unique** — one row per printing, so Lightning
  Bolt appears dozens of times. Use `uuid` for row identity and
  `count(name)` for distinct cards.
- `avg(manaValue)` on `rarity = 'rare'` looks inflated (~51) due to
  legacy/X-cost entries in MTGJSON. Data artifact, not a query bug.
- Many columns are nullable booleans — `WHERE x = false` silently drops
  nulls.
- Promo / The List / Secret Lair sets contain reprints with unusual
  rarities that skew rarity-distribution analysis.

## Licence

MIT — see [LICENSE](LICENSE).
