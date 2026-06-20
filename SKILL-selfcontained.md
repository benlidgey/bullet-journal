---
name: bullet-journal
description: Records a sentence or short paragraph as a dated bullet journal entry in a central store. Use whenever the user wants to log, journal, note, jot, capture, or record a quick thought, observation, win, idea, or luck event — including phrases and prefixes like "journal this", "log this", "add a journal entry", "note this down", "jot this", "note:", "read:", "bullet:", or "bullet journal". Applies any tags the user supplies; if none are given, suggests tags from the configured list.
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
- **Tag list:** `work`, `luck`, `ideas`, `family`, `admin`, `learning`
- **Suggest count (max):** `2`  *(a maximum, not a quota)*
- **Allow new tags:** `no`  *(never invent tags; suggest only from the list)*

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

## Scope

Iteration 1 handles **text entries only**. The `read:` prefix and attachment
handling are not yet built — do not attempt them.
