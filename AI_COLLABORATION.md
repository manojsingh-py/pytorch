# AI collaboration log

I built this with an AI coding assistant (Kiro). This file records the prompts I used, which decisions came out of them, and what was verified.

## 1. Prompts

### 1.1 Original prompt

> Read the requirement and build this application from scratch. First understand the requirement and list out the flow, understand the data and suggest how you will process it. Use a machine learning model if needed.

### 1.2 Refined development prompt

The original prompt was short, so I expanded it into the brief below. It states the goal, inputs, order of work, decisions to make, constraints and definition of done. Later iterations and the final checks were run against it.

```text
ROLE
You are a senior full-stack engineer working with me on a take-home assessment.
Build a working, well-documented application from scratch, and explain your reasoning
as you go so I can review and defend every decision.

INPUTS
- Requirement: fsd-take-home-assessment/problem_statement.md
- Data:        fsd-take-home-assessment/data/crm_events.json      (Source A - CRM)
               fsd-take-home-assessment/data/calendar_events.json (Source B - Calendar)

GOAL
Build an "Event Sync Service" that:
  1. Ingests meetings from both sources.
  2. Reconciles records that refer to the same real-world meeting.
  3. Serves one unified meeting list through a REST API and a simple web UI.
  4. Shows, for every field, which source the value came from, and where the sources conflict.

WORK IN THIS ORDER (do not jump straight to code)

Step 1 - Understand the requirement
  - Restate the functional and non-functional requirements in your own words.
  - List what is being evaluated (decisions + README quality, not visual polish).
  - Call out anything the brief leaves open on purpose.

Step 2 - Describe the end-to-end flow
  - Ingest -> normalise -> remove duplicates within a source -> match across sources
    -> reconcile fields -> store -> serve via API -> display in UI.
  - Include a simple diagram (Mermaid is fine).

Step 3 - Analyse the data before writing code
  - Read every record in both files.
  - Build an expected-answer table: which CRM record pairs with which calendar
    record, which are single-source, and which are duplicates.
  - Find and list concrete examples of each problem the brief mentions:
      * no shared ID between sources
      * meetings in both sources vs only one
      * timestamps that don't match exactly (check timezones and daylight saving)
      * conflicting fields (location, format, status, time)
      * a duplicate within one source (don't confuse it with a recurring series)
      * missing or malformed fields (dates, times, emails, empty lists, placeholders)
  - Separate real conflicts from harmless wording differences
    (e.g. "Conference Room B" vs "HQ - Conference Room B").

Step 4 - Propose how you will process it, then build
  - Matching: compare at least (a) exact/rule-based, (b) weighted scoring,
    (c) supervised ML and (d) text embeddings. Recommend one and justify it given
    the data size and the lack of labelled pairs. Use ML only if it clearly helps.
  - Duplicates: how they are detected and which record is treated as current.
  - Field rules: for each field, which source is preferred and why.
  - Conflicts: never resolve silently. Pick a value so the list is usable, keep
    every source's value, and flag the disagreement with a severity (high/medium/low).
  - Bad data: never crash. Repair what is safe (and say so), keep the original
    payload, and record a data-quality issue.
  - State assumptions explicitly (e.g. timezone for timestamps without an offset).

TECHNICAL CONSTRAINTS
  - Language/framework: Python + FastAPI backend, plain HTML/JS frontend (no build step).
  - Start with a single documented command (e.g. `python run.py`) that installs
    pinned dependencies and launches the app.
  - Keep settings (weights, thresholds, timezone, internal email domains) in one config file.
  - Treat source data as untrusted: insert it into the page as text, not HTML.
  - Bind to localhost; note that there is no authentication.
  - Keep code readable and modular: ingest / normalise / match / reconcile / api.

API (minimum)
  - GET  /api/meetings        list, with filters (source, has conflicts, needs review, search)
  - GET  /api/meetings/{id}   full detail: field values, sources, conflicts, match reasons
  - GET  /api/stats           counts and summary
  - POST /api/sync            reload both sources

UI (minimum)
  - Meeting list with source badges and conflict indicators.
  - Detail view showing each field's unified value, the CRM value and the calendar
    value side by side, conflicts highlighted, and why the records were matched.

DOCUMENTATION
  - README: how to run, the flow, data findings and how each is handled,
    approach, key decisions and trade-offs, API reference, limitations,
    next steps, and a "time spent" section for me to fill in.
  - docs/AI_COLLABORATION.md: prompts used, options considered, decisions made,
    and what was verified by hand vs drafted by the AI.

DEFINITION OF DONE
  - The app starts with one command and the UI loads.
  - Output matches the expected-answer table from Step 3.
  - Every conflict and data issue found in Step 3 is visible in the API and UI.
  - README and AI log are complete and accurate.
  - Report clearly what was verified and what was not.
```

### 1.3 Follow-up prompt: tests, logging, error handling

Original wording:

> Add test cases and add some proper logging for debugging and processing. Also add try/except where needed.

Refined:

```text
GOAL
Make the service easier to trust and debug without changing its output.

TESTS (pytest)
  - Unit tests for parsing and cleanup: timezones/DST, malformed dates and times,
    email repair, name/company/domain matching, location comparison.
  - Ingestion: missing/invalid/non-array files, garbage fields, one unmappable row.
  - Matching: duplicate vs recurring series, same-day gate, wrong same-day pairs
    stay below threshold, one-to-one assignment.
  - End-to-end on the real data: reproduce the hand-built mapping of all 42 records
    and every conflict/data issue listed in the README.
  - API: every endpoint and filter, 404/422, re-sync, and failure modes.
  - Pin test dependencies separately from runtime ones.

LOGGING
  - One shared setup; level from an env var (default INFO).
  - INFO: sync lifecycle (records loaded, duplicates, match counts, summary + duration)
    and each API request with status and latency.
  - DEBUG: every data-quality issue, every candidate score with its components,
    and every conflict.

ERROR HANDLING (try/except only where it changes the outcome)
  - A bad record is logged and skipped; the rest of the file still loads.
  - A failing meeting group falls back to separate records; nothing disappears silently.
  - A failed re-sync keeps serving the last good result and reports "degraded".
  - Before any successful sync, the API returns 503 with the reason.
  - Unexpected errors: full traceback in the log, generic 500 to the client.
  - The launcher prints plain messages for common setup failures.

DEFINITION OF DONE
  - All tests pass; same 24 meetings as before; README documents tests, logging
    and error handling.
```

## 2. Requirement breakdown

The brief asks for three things: **ingest** two sources, **reconcile** records that describe the same meeting, and **serve** a unified list with visible provenance and conflicts. It also asks for a single start command, a detailed README, and AI documentation.

The brief deliberately leaves the hard choices open: how to match, which source wins, and how to treat duplicates and bad data. So the main deliverable is a set of *defensible, visible* decisions, not a clever UI.

## 3. Data analysis

We read all 42 records by hand **before** writing code and built the expected answer as a table: 17 cross-source pairs, 3 CRM-only, 4 calendar-only, and 1 calendar duplicate. The implementation was then checked against that table, not the other way round. Notable findings:

- **Timezone trap.** `CAL-A4` is the only record with `Z`. Converting 19:00Z with the correct DST offset (EDT, UTC-4, since DST began 9 Mar 2025) gives 15:00, while CRM says 14:00. The naive UTC-5 assumption would hide the conflict. We kept the correct conversion and surface the conflict.
- **Duplicate vs. recurrence.** `CAL-A5`/`CAL-A6` are one meeting: same day, 30 min apart, same client domain, and A6 was created later and adds the CIO, whom CRM said "may" attend. `CAL-A3`/`CAL-A18` look identical but are weekly occurrences. That is why same local date is a hard rule for duplicates.
- **Conflicts.** Location/format (`CRM-1002`), status (`CRM-1009` cancelled), time (`CRM-1004`, `CRM-1016`), plus many harmless wording differences that must *not* be flagged as conflicts ("Conference Room B" vs "HQ - Conference Room B").
- **Malformed data.** `"03-15/2025"`, a null time, `raj.patel[at]...`, `"external-guests"`, an end time with no seconds, an empty attendee list, and the placeholder client `"Multiple"`.

## 4. Options considered

| Decision | Options discussed | Chosen | Why |
|---|---|---|---|
| Matching | (a) exact keys (b) hand-weighted scoring (c) supervised ML (d) sentence embeddings | (b) | No labels and only 42 records, so (c) can't be trained or validated. (d) adds a ~100 MB model and opaque scores for little gain once a same-day gate and people signals exist. (b) is explainable per pair, and its weights could later be learned |
| Assignment | greedy vs Hungarian | greedy, 1:1 | Same result on this graph, simpler to explain |
| Duplicates | ignore / drop older / merge with provenance | merge, keep latest as current | No data is lost, and the merge is visible in the UI |
| Conflicts | pick a winner silently / leave unresolved / pick + flag with severity | pick + flag | The list stays usable and humans see every disagreement |
| Source of truth | one global winner / per-field rules | per-field | Calendar is better for logistics, CRM for relationship data |
| Stack | React + API / server templates / static HTML + API | FastAPI + vanilla JS | One command, no Node toolchain, the API stays the product |
| Storage | SQLite / in-memory | in-memory with atomic re-sync | Static, small inputs. Persistence is listed as a next step |
| Re-sync failure | crash / clear data / keep last good result | keep last good result, report "degraded" | A broken upstream file shouldn't blank the UI |
| Test HTTP client | `httpx` / `httpx2` (suggested by a Starlette warning) | `httpx` | Well-known package. `httpx2`'s publisher couldn't be confirmed, so the warning is filtered instead |

## 5. Iteration notes

- First full run of the pipeline matched the hand-built expected table exactly: 24 meetings, 17/3/4 split, 1 merge.
- We measured the gap between the lowest true-match score (0.75) and the highest non-match score (0.37) to make sure the 0.55 threshold isn't fragile on this data.
- A small bug (treating a dataclass like a dict in the stats counter) was caught on the first run and fixed.
- The frontend inserts all upstream text with `textContent` only, because source data is untrusted.
- Logging initially stopped records from reaching pytest's log capture. Fixed by letting the `event_sync` logger propagate (the root logger has no handlers under uvicorn, so nothing prints twice).
- The 121-test suite passed, and the reconciled output was unchanged (24 meetings).

## 6. What was verified vs. what the AI drafted

- **Verified during the session:**
  - the expected mapping of every record, which the end-to-end tests encode
  - the DST conversion
  - every flagged conflict in the README table
  - API responses and failure modes (via tests)
  - JS syntax
- **AI-drafted and reviewed:** module structure, scoring code, field rules, tests, logging, README prose.
- **Not done:** automated browser tests and visual cross-browser checks of the UI.
