---
name: bullet-journal
description: Records a sentence or short paragraph as a dated bullet journal entry in a central store. Use whenever the user wants to log, journal, note, jot, capture, or record a quick thought, observation, win, idea, or luck event — including phrases and prefixes like "journal this", "log this", "add a journal entry", "note this down", "jot this", "note:", "read:", "bullet:", or "bullet journal". Applies any tags the user supplies; if none are given, suggests tags from the configured list.
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
   - For `type: "slack"`: post one message per entry to the channel or
     Canvas named in `storage.location` via the Slack connector, e.g.
     `YYYY-MM-DD — entry text [tag1, tag2]`. For a Canvas, append to it
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

## Scope

Iteration 1 handles **text entries only**.

**Future (not yet built):** the `read:` prefix and the folder layout are
intended to let an entry carry an attachment (e.g. "read this PDF" plus the
PDF file stored alongside the entry). Do not attempt attachment handling
until that iteration is built.
