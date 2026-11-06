# Kernel.ai – StatAgent

Agent-assisted official employment statistics built from individual records, with quality assurance, versioned classifications and a full audit trail. Developed for the KI Challenge, Use Case 3 (StatAgent).

**Principle:** the AI plans, the program calculates, a human approves.

- Every figure is produced by plain SQL/Python over the cleaned records. No language model touches a number. Two runs with the same data and rules give the same result.
- Everything runs locally and offline (PostgreSQL in Docker, web interface on localhost).
- Rules and mappings live in the database, not in hidden constants.
- Every change to rules, code or a delivery needs the full name of an approving person and is written to `audit_log` and, for agent changes, to a Git commit with a `Freigegeben-von:` line.

> **Status:** the code has been compile-checked and the core pipeline has been run on the challenge data. The pension statistic (`pensions.py`), ÖNACE down to 5 digits and the municipality history from the official change lists are not finished. See [Limitations](#limitations).

## Contents

- [Architecture](#architecture)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Usage](#usage)
- [Project layout](#project-layout)
- [Data model](#data-model)
- [Counting rules](#counting-rules)
- [The agent](#the-agent)
- [Quality assurance](#quality-assurance)
- [Configuration](#configuration)
- [Limitations](#limitations)

## Architecture

```
roh.lieferung ──► ingest.py ──► clean_period ──► engine.py ──► stat_employees ──► sheet / questions / ministry list
 (raw, unchanged)   + qa/        (versioned per    (SQL, per        (long format,     web UI · CLI · dashboard
                                  delivery)         rule_set)        status, stand)
                         │                              │
                  validation_rule                   rule_set          audit_log ◄── every approval and agent change
                  (rules as data)                (counting rule)
```

| Step | Module | Result |
|---|---|---|
| Receive | `roh.lieferung` | 24 deliveries, stored raw |
| Clean | `ingest.py`, `qa/checks.py` | `clean_period`, `qa_finding` |
| Enrich | `rule_set` | counting rule and category level as data |
| Aggregate | `engine.py` | `stat_employees` |
| Check | `engine.reconcile`, `jump_check`, `revision_check` | quality report per month |
| Provide | `engine.export_sheet`, `engine.ask`, `engine.ministry`, `webui.py` | CSV, answers, dashboard |

## Requirements

- Docker (Docker Desktop or Docker Engine with Compose)
- Python 3.10 or newer
- The challenge package folders next to `docker-compose.yml`: `lieferungen/` (24 files `lieferung_YYYY-MM.csv.gz`) and `klassifikationen/` (`oenace_2008.csv`, `oenace_2025.csv`, `bundesland.csv`, `geschlecht.csv`, `qualifikation.csv`)

The challenge data is not part of this repository and must not be committed (see `.gitignore`).

## Quick start

```bash
# 1. Database (loads all deliveries on first start, 2-4 minutes). Wait for "fertig".
docker compose up -d
docker logs -f statagent-db

# 2. Python environment
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 3. Schema, classifications, default validation rules
python run.py --init

# 4. Inspect the real data, then adjust qa/default_rules.json if needed
python profile_data.py
python run.py --init               # only adds new rules, never overwrites approved ones

# 5. Baseline: approve rules and rule set, run all 24 months in order
python -u submission.py --stage base --by "First Last"
```

Port 5432 already in use: start with `DB_PORT=5433 docker compose up -d` and set `STATAGENT_DB_PORT=5433` for the Python commands.

Windows (cmd): `set DB_PORT=5433` and `set STATAGENT_DB_PORT=5433`. PowerShell: `$env:DB_PORT=5433`.

## Usage

### Command line

```bash
python run.py --delivery 2025-03 --ruleset v1_reference_date    # one production run (chronological order!)
python run.py --all --approve-by "First Last"                   # 2024-08 .. 2026-07 in order
python run.py --approve rules --by "First Last"                 # approve pending validation rules
python run.py --approve ruleset:v1_reference_date --by "First Last"
python run.py --approve delivery:2025-03 --by "First Last"      # makes a provisional month final
python run.py --sheet v1_reference_date --out output/aggregation_v1.csv
python run.py --ministry 2026-07                                # ministry list
python run.py --ask "Zeitreihe nach Geschlecht"                 # data question, returns SQL + definition
```

A run processes exactly one delivery and uses only its own and earlier deliveries. Published months are never changed; revision need is reported in the quality report.

### Web interface

```bash
python webui.py        # http://localhost:8080  (?lang=en for English)
```

Tabs: monthly run, quality reports and assumptions, classifications (with transition explanations), approvals and audit log, questions to the data, instructions to the agent, dashboard. German and English can be switched in the header.

### Agent

```bash
python agent.py "Dienstvertrag und freier Dienstvertrag sollen zusammen gezählt werden" --by "First Last"
python agent.py "Ganzes Monat statt Stichtag" --by "First Last"
python agent.py "Ministeriumsliste 2026-07 mit mindestens 150 Tagen" --by "First Last"
```

### Hand-in in stages

```bash
python -u submission.py --stage base    --by "First Last"   # baseline for all 24 months
git init && git add -A && git commit -m "Baseline"
python -u submission.py --stage changes --by "First Last"   # the three change orders via the agent
python submission.py --stage export     --by "First Last"   # sheets, reports, ministry CSV, audit log -> abgabe/
```

## Project layout

```
run.py                  CLI: init, approvals, runs, sheet, ministry, ask
ingest.py               raw -> cleaned, versioned layer; rules from validation_rule
engine.py               counting rules, aggregation, checks, sheet, ministry list, data requests
agent.py                language instruction -> validated plan -> approval -> execution -> log + git
webui.py                NiceGUI web interface (DE/EN)
profile_data.py         profiling of the raw layer to derive validation rules
submission.py           staged hand-in
pensions.py             pension reports: raw load and profiling only (parser pending)
common.py               DB connection, SQL literals, month arithmetic, audit log
qa/checks.py            data checks (driven by rules)
qa/report.py            quality report builder
qa/default_rules.json   default validation rules (registered unapproved)
etl/load_classifications.py   classifications, region history, transition explanations
database/                SQL model and seed data (applied by --init)
datenbank/              SQL that loads raw data on first container start
docker-compose.yml      local PostgreSQL 16
```

## Data model

Follows the task's Appendix B: one `stat_` table per statistic in long format, one shared variable table, one `klass_` table per concept.

| Table | Purpose |
|---|---|
| `roh.lieferung`, `roh.ladeprotokoll` | raw deliveries as text, load log |
| `delivery`, `clean_period` | versioned cleaned records, fingerprint (SHA-256), duplicate counter `occurrences` |
| `stat_employees` | result: period, region, ÖNACE, sex, qualification, age, variable, value, status, stand, counting rule |
| `variable_table` | one row per variable: statistic, code, label, description |
| `klass_gender`, `klass_region`, `klass_oenace`, `klass_qualification`, `klass_traeger` | classifications with versions and validity |
| `klass_transition` | code old to code new, relation 1:1, 1:n, n:1, m:n, valid from |
| `validation_rule`, `rule_set`, `special_analysis` | rules as data (approved by a named person) |
| `qa_finding`, `quality_report` | findings per delivery, report per month and rule set |
| `publication` | provisional/final status per month and rule set |
| `audit_log` | what, who, when, which instruction, who approved |

## Counting rules

| Rule set | Rule |
|---|---|
| `v1_reference_date` | a relationship counts if a period exists on the last day of the month |
| `any_day_in_month` | a relationship counts if at least one period overlaps the month; attributes from its last period; age on the last day |

Each relationship (`VVH_ID`) counts once per month. Attributes come from its last qualifying period. Exact duplicate records are folded, so no relationship is lost.

The ministry list counts persons with at least N calendar days (default 200) as `ArbeiterDV` or `ArbeiterFD` in ÖNACE section C over the last 12 months. Overlapping periods are merged with `range_agg`, so a day counts once. For every day the youngest delivery containing that day is used.

## The agent

1. An instruction in plain language is turned into a plan. If `AZURE_OPENAI_ENDPOINT` is set, a language model does this; otherwise a keyword planner is used.
2. The plan is validated against an allow-list of actions (`create_rule_set`, `update_special_analysis`, `run_ministry`, `add_validation_rule`, `ask`, `custom_select`, `clarify`).
3. A team member approves with a full name. Placeholders such as "Team" are rejected.
4. Deterministic code runs without a language model.
5. The change is written to `audit_log` and committed to Git with the instruction as message and a `Freigegeben-von:` line.

Operational code written by the agent is limited to read-only `SELECT` files in `agent_code/`.

## Quality assurance

Findings are rules in `validation_rule` with an assumption and a justification. New rules are registered unapproved and take effect only after a named approval. Example rules from delivery 2025-12:

| Finding | Records | Treatment |
|---|---|---|
| ÖNACE and state both empty | 2,141 | codes `ZZ` and `0` (unknown), record kept |
| ÖNACE 2003 codes `DA` to `DM` | 49 | mapped to section C, record kept |
| Birth year 1871 to 1907 | 11 | birth date unknown, record kept |

Each run writes a quality report with delivery data, findings and assumptions, mappings, results, reconciliation, time series comparison and revision need.

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `STATAGENT_DB_HOST` | `localhost` | database host |
| `STATAGENT_DB_PORT` | `5432` | database port |
| `STATAGENT_DB_NAME` | `statistik` | database name |
| `STATAGENT_DB_USER` | `statistik` | database user |
| `STATAGENT_DB_PASSWORD` | `statistik` | local development password only |
| `AZURE_OPENAI_ENDPOINT` | unset | enables the language-model planner |
| `AZURE_OPENAI_API_KEY` | unset | key for the endpoint |
| `STATAGENT_MODEL` | unset | deployment name, exchangeable |
| `AZURE_OPENAI_API_VERSION` | `2024-10-21` | API version |

The defaults are for a local development database only. Never reuse them elsewhere, and never commit keys.

## Limitations

- Pension statistic: `pensions.py` loads and profiles the provider Excel files; the parser into a `stat_` table is not written yet.
- ÖNACE down to 5 digits and the ÖNACE 2008 to 2025 transitions are not loaded; only section level is used.
- Municipality and district history is derived by comparing yearly lists, not yet from the official change lists.
- The reconciliation check reuses the counting condition, so it is a weak independent check.
- Unreadable end dates are treated as open-ended (documented assumption, review against the profile).
- The keyword planner and `ask` are rule-based; they cover the known phrasings, not arbitrary language.

## Licence and data

Code: add a licence of your choice before publishing. The challenge data and the regional data (CC BY 4.0, Statistik Austria) are not included and keep their own terms.
