---
name: bullet-journal
description: Records a sentence or short paragraph as a dated bullet journal entry in a central store, and generates reports from past entries. Use whenever the user wants to log, journal, note, jot, capture, or record a quick thought, observation, win, idea, or luck event — including phrases and prefixes like "journal this", "log this", "add a journal entry", "note this down", "jot this", "note:", "read:", "bullet:", or "bullet journal". Also use when the user asks for a journal report or summary over a period — e.g. "monthly report", "journal report", "report:", "summarise my journal", "what did I journal last month". Applies any tags the user supplies; if none are given, suggests tags from the configured list.
---

# Bullet Journal Skill

## Purpose

Capture a sentence or short paragraph as a dated bullet journal entry in a
central store. Tags come from the user when supplied; otherwise the skill
suggests them from a configured list. The skill is generic and shareable:
all personal and deployment-specific detail lives in `config.json`, never
in this file.

## Configuration

Always read `config.json` (next to this file) before acting. It defines:

- `storage.type` / `storage.location` / `storage.notes` — where entries go
  and how to write them. Do not hard-code any destination; follow these.
- `tags` — the configured tag list. This is the menu the skill suggests
  from, and the set considered "known".
- `tagging.suggest_count` — the *maximum* number of tags to suggest when the
  user gives none. Apply only as many as genuinely fit; never pad to the
  number with weak or loosely-relevant tags. Fewer is fine; zero is fine.
- `tagging.allow_new_tags` — if `false`, the skill must never *invent* tags
  on its own; suggestions come only from `tags`.
- `reporting` — default options for the reporting flow (see Reporting below):
  `format`, `group_by`, `show_tags`, `summary`, `title_template`,
  `output_path_template`. If absent, use the built-in defaults named there.

## Procedure

When this skill fires, follow these steps in order:

1. **Read `config.json`.** Load `storage`, `tags`, and `tagging`. If it is
   missing or malformed, stop and ask the user to fix it — do not guess a
   destination.

2. **Extract the entry text.** Take the user's sentence/paragraph. Strip any
   trigger prefix (`note:`, `read:`, `bullet:`, "journal this", "log this", etc.)
   and any inline tags (see step 4). What remains is the entry body, verbatim
   — do not rewrite, summarise, or "improve" the user's words.

3. **Stamp the date.** Use today's local date in `YYYY-MM-DD` format.

4. **Resolve tags.**
   - Tags may be supplied inline as `#tag` tokens or named explicitly
     ("tag it work and ideas").
   - **If the user supplied tags:** use exactly those.
     - For any supplied tag NOT in the configured `tags` list: still apply
       it, but tell the user it wasn't in the list and offer to add it to
       `config.json`. (Only edit `config.json` if they say yes.)
   - **If the user supplied no tags:** suggest up to `tagging.suggest_count`
     tags drawn ONLY from the configured `tags` list, choosing genuine best
     fits for the entry. `suggest_count` is a maximum, not a quota — apply
     fewer (even one, or none) rather than padding with a weak tag. When
     `allow_new_tags` is `false`, never propose a tag outside the list. Apply
     the suggested tags and make clear they were suggested.

5. **Write one entry to the store.** Using the method in `storage.notes`,
   create a single entry with: the date (step 3), the entry text (step 2),
   and the resolved tags (step 4).
   - For `type: "notion"`: add one row to the database in `storage.location`
     via the Notion connector (Date, Entry, Tags columns).
   - For `type: "file"`: append a line to the file at `storage.location`,
     e.g. `- YYYY-MM-DD [tag1, tag2] entry text`.
   - For `type: "slack"`: post one message per entry via the Slack
     connector, e.g. `YYYY-MM-DD — entry text [tag1, tag2]`. Use the
     channel **ID** held in `storage.location` (the connector resolves
     channels by ID, not by `#name`). Write tags as plain bracketed text
     like `[work, luck]` — do NOT prefix tags with `#` or `@`, which Slack
     interprets as channel and user references. For a Canvas, append to it
     rather than replacing it.
   - For any other `type`: follow `storage.notes`.

6. **Confirm.** Show the user what was stored: the date, the entry text, the
   tags, whether tags were supplied or suggested, and where it landed.

## Example

User: `note: shipped the onboarding revamp, big relief #work`

- Date: `2026-06-17`
- Entry: `shipped the onboarding revamp, big relief`
- Tags: `work` (supplied, in list)
- Action: add one row to the configured store; confirm to the user.

User: `bullet: bumped into an old colleague who offered to refer me`  *(no tags)*

- Date: `2026-06-17`
- Entry: `bumped into an old colleague who offered to refer me`
- Tags: `luck`, `work` (suggested from the configured list, `suggest_count: 2`)
- Action: add one row; tell the user these tags were suggested.

## Reporting

When the user asks for a journal report or summary over a period (see the
trigger phrases in the description), generate a markdown report from past
entries. This flow runs entirely in Claude (it reads the store and writes the
summary); there is no separate script.

### Resolve options

Options are resolved in order, most specific wins:

```
built-in default  <  config.json "reporting" block  <  what the user asks for
```

| Option | Built-in default | Notes |
|--------|------------------|-------|
| date range | last complete month | Honour an explicit range in the request ("for May 2026", "1–15 June", "last week"). |
| `format` | `report` | `report` = structured list with headings; `blog` = prose-forward, summary leads and runs longer, entries woven in or lightly listed. |
| `group_by` | `date` | `date` = one date-ordered list; `tag` = entries under per-tag headings. |
| `show_tags` | `true` | If `false`, omit the trailing `[tags]` on each entry. |
| `summary` | `true` | If `false`, skip the AI summary paragraph. |
| `title` | from `title_template` | `{month_name}`, `{month}` (01–12), `{year}` are substituted. |
| output path | from `output_path_template` | Same substitutions. |

### Procedure

1. **Read `config.json`** — `storage` and the `reporting` defaults.
2. **Resolve options** — defaults overlaid with anything explicit in the
   request (table above).
3. **Determine the date range** — default to the last complete calendar month.
4. **Pull entries for the range from the store:**
   - `slack`: read the channel's messages in range via the Slack connector;
     parse each `YYYY-MM-DD — text [tags]` message into date, text, tags.
   - `notion`: query rows where Date is in range.
   - `file`: read the file and keep lines whose date is in range.
5. **Filter out non-journal messages.** Keep only real entries; skip system
   and bot noise (in Slack: channel-join notices, the Claude bot's intro
   message, "Sent using @Claude" metadata lines).
6. **Resolve each entry's date.** Use the `YYYY-MM-DD` prefix when present. If
   there is no prefix, use the message's own date (e.g. the Slack message
   timestamp). If no date is available at all, use the previous entry's date.
7. **Format entry contents for reading.** Tidy entries for human reading
   without changing their meaning. In particular, render a bare URL as a
   markdown link `[text](url)`: use the entry's own wording as the link text
   when it reads well, otherwise fetch the page title from the URL; if no
   title is available, leave the plain URL.
8. **Write the AI summary** (skip if `summary` is `false`): a short paragraph
   synthesising the period, grouped by tag — themes, not a restatement of
   every entry.
9. **Assemble the markdown:** the title header, then the summary, then the
   entries — a single date-ordered list when `group_by` is `date`, or sections
   under per-tag headings when `group_by` is `tag`. Show or hide tags per
   `show_tags`.
10. **Output the markdown file** to the resolved output path when a filesystem
    is available. Where there is no writable path (e.g. the desktop/web app),
    return the markdown for the user to save. Confirm what was produced and the
    range covered.

### Report example (`format: report`, `group_by: tag`)

```markdown
# Journal — June 2026

A month mostly about shipping, with a couple of lucky breaks along the way.

## work
- 2026-06-03 — shipped the onboarding revamp [work]
- 2026-06-18 — finished the skill build [work, learning]

## luck
- 2026-06-19 — found a pound coin on the pavement [luck]
```

## Scope

Iteration 1 handles **text entries only**. Iteration 2 adds **reporting**
(above).

**Future (not yet built):**

- **Attachments** — the `read:` prefix and the folder layout, to let an entry
  carry a file (e.g. "read this PDF" plus the PDF stored alongside the entry).
- **Untagged entries in reports** — reporting currently puts untagged entries
  under an "untagged" heading when grouping by tag. A later change could
  suggest tags for them (as the capture flow does) rather than bucketing them.

Do not attempt these until they are built.
