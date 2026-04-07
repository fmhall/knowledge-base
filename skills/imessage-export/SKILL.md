---
name: imessage-export
description: "Export and import iMessage conversations into a knowledge base. Uses imessage-exporter to pull recent conversations from the macOS Messages database, then analyzes them to create or update people pages. Use when the user wants to import iMessage data, says 'imessage export', 'import messages', 'text messages', 'imessage', or wants to add personal contacts from their messaging history."
argument-hint: "[--days N] [--top N]"
---

# iMessage Export

Export iMessage conversations from the macOS Messages database and process them into knowledge base people pages. This skill handles the full pipeline: installing the exporter tool, running the export with a date filter, analyzing conversations to identify the user's most active contacts, and creating/updating people pages with relationship context that only personal messages can reveal.

## Interaction rules

Only ever ask the user **one question at a time**. Never present multiple decisions simultaneously.

## Step 0: Check prerequisites

### macOS only

This skill requires macOS with a local Messages database. If the user isn't on macOS, tell them and stop.

### Install imessage-exporter

Check if `imessage-exporter` is installed:

```bash
which imessage-exporter
```

If not installed, the user needs Rust/Cargo:

```bash
which cargo
```

If Cargo is available:
```bash
cargo install imessage-exporter
```

If Cargo is not available:
```bash
brew install rust
cargo install imessage-exporter
```

Wait for installation to complete before proceeding.

### Full Disk Access

`imessage-exporter` needs access to `~/Library/Messages/chat.db`. If the export fails with a permission error, tell the user:

> The terminal app needs Full Disk Access to read the Messages database.
> Go to **System Settings → Privacy & Security → Full Disk Access** and enable it for your terminal app (Terminal, iTerm2, WezTerm, etc.).
> You may need to restart the terminal after granting access.

## Step 1: Discover the knowledge graph

Before processing anything, understand the user's project:

1. Read the project's CLAUDE.md, README, or top-level documentation.
2. Find where people pages live and read 2–3 samples to learn the frontmatter schema.
3. Check for a `raw/` directory where source exports should go.

## Step 2: Run the export

Export recent conversations to the project's `raw/` directory:

```bash
imessage-exporter \
  -f txt \
  -o raw/imessage-export \
  -s YYYY-MM-DD \
  -e YYYY-MM-DD
```

**Date range**: Default to the last 30 days. Calculate the start date as today minus 30 days. If the user passed `--days N`, use that instead. The `-s` flag is the start date (inclusive) and `-e` is the end date (exclusive).

**Format**: Always use `-f txt`. Text format is compact, parseable, and sufficient for extracting relationship signals. HTML is unnecessary overhead.

**Attachments**: Do not use `-c` (copy method). Attachments aren't useful for people page creation and bloat the export significantly.

**Custom name**: Optionally use `-m` with the user's first name so their messages show their name instead of "Me" — makes the export easier to read. Ask the user their first name if you don't already know it.

### Handle existing exports

If `raw/imessage-export/` already exists from a prior run, ask the user:

> There's an existing iMessage export in raw/imessage-export/. Should I overwrite it with a fresh export, or work with the existing data?

### Verify the export

After the export completes, check what was produced:

```bash
ls raw/imessage-export/ | wc -l
```

Report the count: "Exported N conversations."

## Step 3: Identify top contacts

The export produces one `.txt` file per conversation. File names are either:
- Phone numbers: `+12025551234.txt`
- Email addresses: `user@example.com.txt`
- Group chats: `Group Name - ID.txt` (e.g., `Fam Text - 34.txt`)

### Sort by activity

Sort conversations by file size (larger = more messages = more active relationship):

```bash
ls -lS raw/imessage-export/*.txt | head -50
```

### Analyze top conversations

Read the top conversations (default: top 50 by size, or `--top N` if specified). For each conversation:

1. **Identify the person** — extract their name from the message headers. Messages show sender names like `John Doe` before each message.
2. **Skip group chats initially** — focus on 1-on-1 conversations first. Group chats are useful for confirming relationships but create noise.
3. **Extract relationship signals**:
   - How frequently do they message?
   - What do they talk about? (work, personal, shared interests)
   - Are there references to meeting up, shared history, or mutual friends?
   - What's the tone? (casual/close vs. professional/distant)

### Cross-reference with existing wiki

Check each identified person against existing wiki pages. Classify:
- **Already exists, can be enriched** — existing page missing personal details that messages reveal
- **New person, worth a page** — active enough relationship to warrant a wiki entry
- **Skip** — spammy, one-off, or automated messages (delivery notifications, 2FA, etc.)

### Present findings to the user

Show a summary:

> Analyzed N conversations from the last 30 days.
> Found M people worth adding/updating:
>
> **New people** (X):
> - [Name] — [brief context from messages]
> - ...
>
> **Existing pages to update** (Y):
> - [Name] — [what new info messages reveal]
> - ...
>
> Want me to proceed with all of these, or pick specific ones?

Wait for the user's direction.

## Step 4: Create/update people pages

For each person the user approves:

### New pages

Create using the project's frontmatter schema. What iMessage data provides:

- **Name**: From message headers (first name, sometimes full name)
- **Phone number**: From the filename (if phone-based conversation)
- **Email**: From the filename (if email-based conversation)
- **Relationship**: Infer from message content and tone:
  - Daily casual messages → `friend` (closeness 2–3)
  - Weekly professional check-ins → `colleague` (closeness 3–4)
  - Family group chat member → `family` (closeness 1–2)
- **Met through**: Infer from context if possible
- **Closeness**: Base on message frequency and depth:
  - `1` — inner circle, daily contact, deep personal sharing
  - `2` — close friend, weekly contact, personal topics
  - `3` — regular contact, mix of personal and professional
  - `4` — occasional contact, mostly professional
  - `5` — rare contact
- **Sources**: `[imessage-export]`

**Body content**: Write a brief relationship description based on what the messages reveal. Focus on the nature of the relationship, shared context, and how the user knows this person — not message-by-message summaries. Be discreet with personal content.

Example:
```markdown
Close friend from Vanderbilt. Regular contact about crypto, startups, and personal life.
Active in the "Hall Boys!" group chat with several mutual friends.

## Connections

- [[some-mutual-friend]] — Both in Hall Boys group chat
```

### Updating existing pages

For existing pages, only add information the messages reveal that isn't already captured:

- Upgrade `closeness` if messages show a closer relationship than the current rating
- Add `phone` if the page is missing it and the conversation filename has a number
- Add relationship context to the body (e.g., "Also connected via iMessage; regular personal contact")
- Update `relationship` if messages reveal a different dynamic (e.g., colleague → friend)

**Never overwrite existing data.** If messages contradict existing info, note both and flag for the user.

### Privacy

iMessage conversations are deeply personal. When writing page content:

- **Don't quote messages** directly unless they're mundane (e.g., "let's grab lunch")
- **Don't include sensitive topics** — health, finances, relationship drama, complaints about third parties
- **Summarize relationship dynamics**, not conversation content
- **When in doubt, be vague** — "Regular personal contact" is better than exposing private details

## Step 5: Update index and log

After creating/updating pages, update the project's index and log files following the project's existing conventions.

## Step 6: Report

Summarize:

- Conversations exported: N
- Conversations analyzed: M (top by activity)
- New people pages created: X (list names)
- Existing pages updated: Y (list names with what changed)
- Skipped: Z (automated messages, unknown contacts, etc.)
