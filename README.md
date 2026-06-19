# bullet-journal

A generic, shareable [Claude](https://claude.ai) skill that records a
sentence or short paragraph as a **dated bullet journal entry** in a central
store. Supply your own tags, or let the skill suggest tags from a list you
configure. Submit entries from Claude on desktop, mobile, or web — they all
land in the same place.

The skill itself contains **no personal or deployment-specific detail**. You
point it at your own storage and tag list in `config.json`; everything else
is generic, so the same skill works for anyone who installs it.

## Files

| File | What it is |
|------|------------|
| `SKILL.md` | The instructions Claude follows. Generic — do not edit to customise. |
| `config.json` | The only file you edit: storage destination + your tag list. |
| `README.md` | This guide. |

## How it works

When you type something like `note: shipped the redesign #work`, Claude:

1. Reads `config.json` for your storage destination and tag list.
2. Strips the trigger prefix (`note:`, `bullet:`, etc.) to get the entry text.
3. Stamps it with today's date (`YYYY-MM-DD`).
4. Resolves tags — uses the ones you supply, or suggests up to
   `suggest_count` from your configured list when you supply none.
5. Writes one entry to your store.
6. Confirms what it stored.

Trigger phrases include: `journal this`, `log this`, `note this down`,
`jot this`, and the prefixes `note:`, `read:`, `bullet:`.

## Setup

### 1. Create your central store

The skill is backend-agnostic. The recommended store is a **Slack channel**:
you can capture entries straight from Slack on any device or via Claude,
storage lives in Slack, and you can extract entries later for reporting. A
Notion database or a local file also work.

#### Option A — Slack channel (recommended, cross-device)

1. In Slack, create a channel for your journal, e.g. `#bullet-journal`
   (a private channel is fine).
2. Connect the **Slack connector** in Claude, then **add the Claude app to
   the channel** — open the channel and `/invite` it (or via channel
   settings → Integrations). This is required: the connector can only see
   and post to channels it is a member of, so a freshly created channel
   won't be reachable until you add the app.
3. In `config.json` set:
   ```json
   "storage": {
     "type": "slack",
     "location": "#bullet-journal",
     "notes": "Post one message per entry via the Slack connector, e.g. 'YYYY-MM-DD — entry text [tag1, tag2]'."
   }
   ```

Entries post from Slack on mobile, web, and desktop, with strong built-in
search. A channel gives you a timeline feed you can scroll and search; you
can also keep several capture channels (e.g. work vs personal) and point
different installs at each. (Prefer one running document? Use a Slack
Canvas titled "Bullet Journal" and append to it instead.)

#### Option B — Notion database (cross-device)

1. In Notion, create a new page and add a **Table** database (full-page).
2. Title the database **Bullet Journal**.
3. Set up three properties (columns):
   | Property | Type | Notes |
   |----------|------|-------|
   | `Entry` | Title | Rename the default title column to `Entry`. Holds the entry text. |
   | `Date` | Date | The day the entry was recorded. |
   | `Tags` | Multi-select | Add one option per configured tag (see step 2). |
4. Add the multi-select options to `Tags` so they match your tag list, e.g.
   `work`, `luck`, `ideas`, `family`, `admin`, `learning`.
5. Connect Notion to Claude: enable the **Notion connector** in Claude, and
   share the **Bullet Journal** database with it so Claude can add rows.

#### Option C — Local file (single machine)

Skip the cloud. In `config.json` set `storage.type` to `"file"` and
`storage.location` to an absolute path, e.g.
`"/Users/you/journal.md"`. The skill appends one line per entry:
`- YYYY-MM-DD [tag1, tag2] entry text`. Single machine only — not synced.

> Note: writing to a **Google Sheet** is not currently supported — the
> Google connector can read Sheets but has no append/edit-row capability.
> Use Slack, Notion, or a file until a Sheets-writable connector is available.

### 2. Configure `config.json`

This is the only file you edit. Example:

```json
{
  "storage": {
    "type": "slack",
    "location": "#bullet-journal",
    "notes": "Post one message per entry via the Slack connector, e.g. 'YYYY-MM-DD — entry text [tag1, tag2]'."
  },
  "tags": ["work", "luck", "ideas", "family", "admin", "learning"],
  "tagging": {
    "suggest_count": 2,
    "allow_new_tags": false
  }
}
```

The shipped `config.json` also includes a `_storage_examples` block with
ready-to-copy `notion` and `file` configs — paste one into `storage` to
switch backends.

| Setting | Meaning |
|---------|---------|
| `storage.type` | `slack`, `notion`, `file`, or another backend you describe in `notes`. |
| `storage.location` | Where entries go (Slack channel, Notion DB title, or a file path). |
| `storage.notes` | Free-text instructions telling Claude how to write an entry. |
| `tags` | Your tag list — the menu the skill suggests from. |
| `tagging.suggest_count` | **Maximum** tags to suggest when you give none. Fewer is fine; the skill never pads with weak tags. |
| `tagging.allow_new_tags` | `false` = the skill never invents tags. Add new tags by editing this list yourself. |

If you supply a tag that isn't in the list, the skill still uses it, tells
you it wasn't in the list, and offers to add it to `config.json`.

### 3. Install the skill in Claude

- **Claude Code (this machine):** place the `bullet-journal` folder under a
  discovered skills directory (e.g. `~/.claude/skills/bullet-journal/`). It
  is picked up automatically.
- **Desktop / mobile / web (claude.ai):** upload the skill in Claude's skill
  settings so it travels with your account to every device. Make sure your
  storage connector (the **Slack connector** for the default setup) is
  connected on those surfaces too, so Claude can reach your channel from
  your phone and the web app.

> Cross-device only works if (a) the skill is installed on your account and
> (b) the store is a cloud backend like Slack or Notion. A local-file backend is
> single-machine only.

## Usage examples

```
note: shipped the onboarding revamp, big relief #work
```
→ Date `2026-06-17`, entry `shipped the onboarding revamp, big relief`,
tag `work` (supplied).

```
bullet: found a pound coin on the pavement today
```
→ Date `2026-06-17`, entry `found a pound coin on the pavement today`,
tag `luck` (suggested from your list).

## Roadmap

Iteration 1 handles **text entries only**. Planned next:

- **Reporting** — extract entries from the store and generate a monthly
  report or blog post from them.
- **Attachments** — use the `read:` prefix and the skill's folder layout to
  let an entry carry a file (e.g. "read this PDF" plus the PDF stored
  alongside the entry).
