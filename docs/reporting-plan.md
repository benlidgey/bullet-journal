# Iteration 2 — Reporting: implementation plan (for review)

Status: **draft plan, nothing implemented yet.** Branch: `reporting`.

Goal (from README Iteration 2): extract journal entries from the store for a
date range and generate a configurable monthly **report** or **blog** as a
markdown file — with a header, an AI-generated summary grouped by tags, and a
list of entries (ordered by date or grouped by tags, tags shown optionally).
Invoked by a bash script, also callable from Claude.

---

## 1. The core constraint that shapes the design

A plain bash script **cannot read Slack or Notion** — those need the
connector (or an API token the script doesn't have). It *can* parse a local
file backend, and it *can* format a normalised list of entries.

So the work splits into three stages with a clean seam between them:

| Stage | Who does it | Notes |
|-------|-------------|-------|
| 1. Extract entries for the range | Claude (Slack/Notion) **or** the script (file backend) | Produces a normalised entries file |
| 2. Assemble the markdown report | `report.sh` (deterministic) | No Slack calls, no LLM — testable in isolation |
| 3. AI summary paragraph | Claude | Passed into the script as text |

This keeps `report.sh` deterministic and unit-testable, while Claude does the
two things only it can: pull from Slack, and write the summary.

**Normalised entries file** (the seam): NDJSON, one entry per line —
`{"date":"2026-06-19","entry":"...","tags":["work","luck"]}`. NDJSON is
chosen over TSV so commas/brackets in entry text can't break parsing.

---

## 2. New / changed files

| File | Change |
|------|--------|
| `scripts/report.sh` | **new** — the report assembler (stage 2) |
| `config.json` | add a `reporting` defaults block |
| `SKILL.md` | add a "Reporting" section + extend frontmatter triggers |
| `SKILL-selfcontained.md` | mirror the reporting section + inline reporting config |
| `README.md` | document reporting usage + the flag table |
| `tests/` | sample NDJSON fixture + expected markdown, run via `--dry-run` |

---

## 3. How the optional flags are managed (the main thing to review)

### 3.1 Three layers, highest specificity wins

```
built-in default  <  config.json "reporting" block  <  CLI flag
```

1. The script ships with sensible built-in defaults.
2. `config.json` → `reporting` overrides them (your standing preferences).
3. An explicit CLI flag overrides both (per-invocation).

Every boolean is a **paired flag** (`--show-tags` / `--no-show-tags`) so a CLI
call can force a value in *either* direction, even when the config default
already sets it. No "can't turn it back off" traps.

### 3.2 The `reporting` config block (standing defaults)

```json
"reporting": {
  "format": "report",
  "group_by": "date",
  "show_tags": true,
  "summary": true,
  "title_template": "Journal — {month_name} {year}",
  "output_dir": "reports",
  "output_template": "journal-{year}-{month}.md"
}
```

### 3.3 The flags on `report.sh`

**Source & date range**
| Flag | Default | Meaning |
|------|---------|---------|
| `--entries <path>` | — | Pre-extracted NDJSON (Slack/Notion path; Claude writes this) |
| `--from-file <path>` | — | Parse a local journal file directly (file backend) |
| `--month <YYYY-MM>` | last complete month | Convenience: sets range to that whole month |
| `--from <YYYY-MM-DD>` | — | Explicit range start (overrides `--month`) |
| `--to <YYYY-MM-DD>` | — | Explicit range end |

Exactly one of `--entries` / `--from-file` is required (the source).

**Format (the configurable/optional ones)**
| Flag | Default (config → built-in) | Meaning |
|------|------|---------|
| `--format <report\|blog>` | `report` | Template style |
| `--group-by <date\|tag>` | `date` | List ordered by date, or grouped under tag headings |
| `--show-tags` / `--no-show-tags` | show | Render `[tags]` on each entry |
| `--summary` / `--no-summary` | on | Include the AI summary paragraph |
| `--summary-file <path>` | — | File holding the AI summary text to inject |
| `--title <text>` | from `title_template` | Override the header title |

**Meta**
| Flag | Default | Meaning |
|------|---------|---------|
| `--output <path>` | from `output_template` | Output markdown file |
| `--config <path>` | next to script/skill | Where to read defaults |
| `--dry-run` | off | Print to stdout instead of writing |
| `-h`, `--help` | — | Usage |

### 3.4 Summary handling

`report.sh` never calls an LLM. When `--summary` is on:
- if `--summary-file` is given, its contents are inserted under the header;
- if not, a placeholder (`<!-- summary: pending -->`) is inserted so a human
  or Claude can fill it.

Claude's flow generates the summary first, then passes it via `--summary-file`.

---

## 4. How Claude invokes it (stage 1 + 3 orchestration)

New trigger phrases (added to the frontmatter `description`): "monthly
report", "journal report", "report:", "summarise my journal".

1. Read `config.json` (storage + reporting defaults).
2. Work out the date range (from the request; default last complete month).
3. Extract entries for the range:
   - **slack:** read the channel's messages in range via the Slack connector,
     parse each `YYYY-MM-DD — text [tags]` line into NDJSON → `entries.ndjson`.
   - **notion:** query rows in range → NDJSON.
   - **file:** skip extraction; pass `--from-file` and let the script parse.
4. Generate the AI summary paragraph (grouped by tag) → `summary.md`.
5. Run the script, e.g.:
   ```bash
   scripts/report.sh --entries entries.ndjson --month 2026-06 \
     --group-by tag --summary-file summary.md --output reports/journal-2026-06.md
   ```
6. Confirm the output path to the user.

---

## 5. Output shape (example, `--format report --group-by tag`)

```markdown
# Journal — June 2026

<!-- AI summary paragraph: themes across the month, grouped by tag -->

## work
- 2026-06-03 — shipped the onboarding revamp [work]
- 2026-06-18 — finished the skill build [work, learning]

## luck
- 2026-06-19 — found a pound coin on the pavement [luck]
```

With `--group-by date`, entries are a single date-ordered list instead of
tag-grouped sections. With `--no-show-tags`, the trailing `[...]` is dropped.

---

## 6. Open questions for you to decide

1. **Default date range** when none is given: *last complete month* (my
   recommendation, fits "monthly report") or *current month-to-date*?
2. **`blog` vs `report` format** — what's the real difference you want? My
   assumption: `blog` = prose-forward (summary prominent, entries woven in or
   lightly listed); `report` = structured list with headings. Confirm.
3. **Standalone Slack pull?** MVP keeps Slack/Notion extraction as Claude's
   job (no token to manage). If you want `report.sh` to run from cron with no
   Claude, we'd add an optional `SLACK_TOKEN` env path. In or out of scope?
4. **Script location:** `scripts/report.sh` (my pick) vs repo root.
5. **Intermediate format:** NDJSON (my pick, robust) vs TSV (simpler to eyeball).

---

## 7. Build order once approved

1. `report.sh` with flag parsing + the 3-layer precedence, file-backend
   parsing, and `--dry-run`.
2. Test fixture (`tests/sample.ndjson`) + expected output; verify via
   `--dry-run`.
3. `config.json` `reporting` block.
4. `SKILL.md` + `SKILL-selfcontained.md` reporting sections and triggers.
5. `README.md` docs + flag table.
6. End-to-end dry run against the real Slack channel via Claude.
