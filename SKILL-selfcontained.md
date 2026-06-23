---
name: bullet-journal
description: Records a sentence or short paragraph as a dated bullet journal entry in a central store, and generates reports from past entries. Use whenever the user wants to log, journal, note, jot, capture, or record a quick thought, observation, win, idea, or luck event — including phrases and prefixes like "journal this", "log this", "add a journal entry", "note this down", "jot this", "note:", "read:", "bullet:", or "bullet journal". Also use when the user asks for a journal report or summary over a period — e.g. "monthly report", "journal report", "report:", "summarise my journal", "what did I journal last month". Applies any tags the user supplies; if none are given, suggests tags from the configured list.
---

# Bullet Journal Skill (self-contained)

Capture a sentence or short paragraph as a dated bullet journal entry in a
central store. This is the **self-contained** variant: all configuration is
embedded below, so it works when uploaded as a single file (e.g. to the
Claude desktop app) with no separate `config.json`.

> If you use this variant, edit the **CONFIG** block below and ignore
> `config.json`. If you prefer the two-file approach, use `SKILL.md` +
> `config.json` instead (see README).

## CONFIG — edit these values before use

- **Storage type:** `slack`  *(or `notion` / `file`)*
- **Storage location:** `CHANNEL_ID_HERE`
  *(Slack: the channel **ID** like `C0XXXXXXXXX`, not the `#name` — find it
  in the channel's About tab or its URL after `/archives/`. Notion: the
  database title. File: an absolute path.)*
- **How to write an entry:** Slack → post one message per entry via the
  Slack connector (the Claude Slack app must be a member of the channel).
  Notion → add one row (Date, Entry, Tags). File → append a line.
- **Tag list:** `work`, `luck`, `ideas`, `family`, `admin`, `learning`, `personal`
- **Suggest count (max):** `2`  *(a maximum, not a quota)*
- **Allow new tags:** `no`  *(never invent tags; suggest only from the list)*

Reporting defaults (used by the Reporting section below):

- **Report format:** `report`  *(or `blog`)*
- **Group by:** `date`  *(or `tag`)*
- **Show tags:** `yes`
- **Summary:** `yes`
- **Title template:** `Journal — {month_name} {year}`
- **Output path template:** `reports/journal-{year}-{month}.md`

## Procedure

When this skill fires, follow these steps in order:

1. **Extract the entry text.** Take the user's sentence/paragraph. Strip any
   trigger prefix (`note:`, `read:`, `bullet:`, "journal this", "log this",
   etc.) and any inline tags (see step 3). What remains is the entry body,
   verbatim — do not rewrite, summarise, or "improve" the user's words.

2. **Stamp the date.** Use today's local date in `YYYY-MM-DD` format.

3. **Resolve tags.**
   - Tags may be supplied inline as `#tag` tokens or named explicitly.
   - **If the user supplied tags:** use exactly those. For any supplied tag
     NOT in the CONFIG tag list: still apply it, but tell the user it wasn't
     in the list and offer to add it to the CONFIG.
   - **If the user supplied no tags:** suggest up to the CONFIG suggest count,
     drawn ONLY from the CONFIG tag list, choosing genuine best fits. The
     count is a maximum, not a quota — apply fewer (even one, or none) rather
     than padding with a weak tag. Never propose a tag outside the list. Make
     clear the tags were suggested.

4. **Write one entry to the store** described in CONFIG, e.g.
   `YYYY-MM-DD — entry text [tag1, tag2]`.
   - **Slack:** post to the channel **ID** in CONFIG (the connector resolves
     channels by ID, not by `#name`). Write tags as plain bracketed text like
     `[work, luck]` — do NOT prefix tags with `#` or `@`, which Slack treats
     as channel and user references.
   - **Notion:** add one row (Date, Entry, Tags).
   - **File:** append `- YYYY-MM-DD [tag1, tag2] entry text`.

5. **Confirm.** Show the user what was stored: the date, the entry text, the
   tags, whether tags were supplied or suggested, and where it landed.

## Reporting

When the user asks for a journal report or summary over a period (see the
trigger phrases in the description), generate a markdown report from past
entries. This runs entirely in Claude — it reads the store and writes the
summary; there is no separate script.

Resolve options most-specific-wins: built-in default, then the Reporting
defaults in CONFIG, then whatever the user asks for in the request.

| Option | Default (CONFIG) | Notes |
|--------|------------------|-------|
| date range | last complete month | Honour an explicit range in the request. |
| format | `report` | `report` = structured list; `blog` = prose-forward, summary leads. |
| group by | `date` | `date` = one date-ordered list; `tag` = per-tag headings. |
| show tags | `yes` | If no, omit the trailing `[tags]`. |
| summary | `yes` | If no, skip the AI summary paragraph. |
| title | from template | `{month_name}`, `{month}` (01–12), `{year}` substituted. |
| output path | from template | Same substitutions. |

Procedure:

1. Resolve options (defaults overlaid with the request).
2. Determine the range (default: last complete calendar month).
3. Pull entries for the range from the store. Slack: read the channel's
   messages in range via the connector and parse each
   `YYYY-MM-DD — text [tags]` message into date, text, tags. Notion: query
   rows in range. File: filter lines by date.
4. Filter out non-journal messages — skip system and bot noise (in Slack:
   channel-join notices, the Claude bot's intro, "Sent using @Claude" lines).
5. Resolve each entry's date: use the `YYYY-MM-DD` prefix when present; else
   the message's own date (e.g. the Slack timestamp); else the previous
   entry's date.
6. Write the AI summary (unless summary is off): a short paragraph grouped by
   tag — themes, not a restatement of every entry.
7. Assemble the markdown: title header, summary, then entries as a single
   date-ordered list (`group_by: date`) or under per-tag headings
   (`group_by: tag`), tags shown per the show-tags setting.
8. Write the markdown file to the resolved output path when a filesystem is
   available; otherwise return the markdown for the user to save. Confirm what
   was produced and the range covered.

## Scope

Iteration 1 handles **text entries only**. Iteration 2 adds **reporting**
(above). The `read:` prefix and attachment handling are not yet built — do
not attempt them.
