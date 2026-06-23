# Iteration 2 — Reporting: implementation plan (for review, v2)

Status: **draft plan, nothing implemented yet.** Branch: `reporting`.

Goal: extract journal entries from the store for a date range and generate a
configurable monthly **report** or **blog** as a markdown file — with a
header, an AI-generated summary grouped by tags, and a list of entries
(ordered by date or grouped by tags, tags shown optionally).

### Decisions locked in (v2)

- Default date range: **last complete month**.
- `report` = structured list with headings; `blog` = prose-forward (summary
  prominent, entries woven in / lightly listed).
- **No bash script.** Reporting is driven entirely by Claude, so it can use
  the Slack connector to pull entries and write the AI summary in one place.
- Output is a **markdown** file.

> This overrides the README Iteration 2 line "invoked by a bash script." That
> line needs updating when we implement (noted in §5).

---

## 1. Why all-in-Claude

A bash script can't read Slack (no credentials), and the summary needs an
LLM. Splitting extraction/formatting across a script and Claude added a
handoff file (NDJSON) and complexity for no real gain. Keeping the whole flow
in Claude means one place reads the store, summarises, and writes the file.

Trade-off, stated honestly: we lose a standalone, unit-testable formatter and
cron-without-Claude. If automation is wanted later, a **scheduled Claude
task** can run the same instruction monthly (see §6) — no script needed.

---

## 2. New / changed files

| File | Change |
|------|--------|
| `config.json` | add a `reporting` defaults block |
| `SKILL.md` | add a "Reporting" section (procedure) + extend frontmatter triggers |
| `SKILL-selfcontained.md` | mirror the reporting section + inline reporting config |
| `README.md` | document reporting + update the Iteration 2 spec (drop "bash script") |

No new script, no intermediate file.

---

## 3. How the options are managed (the main thing to review)

Without CLI flags, the "options" are resolved by Claude at report time from
two layers:

```
built-in default  <  config.json "reporting" block  <  what you ask for in the request
```

1. The skill defines sensible built-in defaults.
2. `config.json` → `reporting` sets your standing preferences.
3. Anything you say in the request overrides for that run
   ("report for May, grouped by tag, hide the tags").

### 3.1 The `reporting` config block (standing defaults)

```json
"reporting": {
  "format": "report",
  "group_by": "date",
  "show_tags": true,
  "summary": true,
  "title_template": "Journal — {month_name} {year}",
  "output_path_template": "reports/journal-{year}-{month}.md"
}
```

### 3.2 The options and how each is expressed in a request

| Option | Default (config → built-in) | Say something like… |
|--------|------------------------------|---------------------|
| Date range | last complete month | "for May 2026", "from 1–15 June", "last week" |
| `format` | `report` | "as a blog", "blog post version" |
| `group_by` | `date` | "grouped by tag", "ordered by date" |
| `show_tags` | `true` | "without tags", "hide the tags" |
| `summary` | `true` | "skip the summary", "no summary" |
| `title` | from `title_template` | "title it 'June review'" |
| output path | from `output_path_template` | "save it to notes/june.md" |

Claude resolves these in order: start from built-in, apply `config.reporting`,
then apply anything explicit in the request. If the request is silent on an
option, the config/default stands.

---

## 4. The reporting procedure (added to SKILL.md)

New trigger phrases (added to frontmatter `description`): "monthly report",
"journal report", "report:", "summarise my journal".

When the reporting flow fires:

1. **Read config** — storage block + `reporting` defaults.
2. **Resolve options** — defaults overlaid with the request (§3).
3. **Determine the range** — default last complete month.
4. **Pull entries** for the range from the store:
   - **slack:** read the channel's messages in range via the Slack connector;
     parse each `YYYY-MM-DD — text [tags]` line into date / text / tags.
   - **notion:** query rows in range.
   - **file:** read the journal file and filter by date.
5. **Write the AI summary** — a paragraph synthesising the month, grouped by
   tag (skipped if `summary` is off).
6. **Assemble the markdown** — header, summary, then the entry list ordered by
   date or grouped under tag headings, tags shown per `show_tags`.
7. **Output** — write the markdown file to the resolved output path (Claude
   Code / filesystem). Where there's no writable path (desktop/web app),
   return the markdown for the user to save. Confirm what was produced.

---

## 5. Output shape

`--format report --group-by tag`:

```markdown
# Journal — June 2026

A month mostly about shipping: the onboarding revamp landed, and a couple
of lucky breaks. (AI-generated, grouped by theme.)

## work
- 2026-06-03 — shipped the onboarding revamp [work]
- 2026-06-18 — finished the skill build [work, learning]

## luck
- 2026-06-19 — found a pound coin on the pavement [luck]
```

`report` + `group_by: date` → single date-ordered list under the summary.
`blog` → the summary leads and runs longer, entries referenced in prose or a
light list. `show_tags: false` drops the trailing `[...]`.

README Iteration 2 currently says "invoked by a bash script" — that line gets
rewritten to "invoked from Claude" when we implement.

---

## 6. Optional later: automation

If you want the monthly report to run without prompting, a **scheduled Claude
task** (cron-style) can fire the same "generate last month's journal report"
instruction on the 1st of each month. Out of scope for this iteration; noted
so the door stays open.

---

## 7. Testing

No unit-testable script, so verification is a real run:

1. Seed a handful of dated entries in the Slack channel (a known fixture).
2. Run the report for that range via Claude.
3. Check the markdown: correct header, summary present and grouped by tag,
   entries filtered to range, grouping/ordering and tag display match the
   options.
4. Re-run with overrides ("as a blog", "grouped by tag", "no tags") to confirm
   the option precedence works.

---

## 8. Build order once approved

1. `config.json` `reporting` block.
2. `SKILL.md` reporting procedure + triggers.
3. `SKILL-selfcontained.md` mirror + inline reporting config.
4. `README.md` docs + rewrite the Iteration 2 spec line.
5. End-to-end run against the real Slack channel; iterate on summary quality.
