---
name: wiki
description: "Compile personal data (journals, notes, messages, bookmarks, exports) into a personal knowledge wiki. Ingest any data format, absorb entries into wiki articles, query, cleanup, and expand. Use when the user provides sources to process, asks questions about the wiki, or wants to maintain wiki quality."
argument-hint: ingest | absorb [date-range] | query <question> | cleanup | breakdown | status | rebuild-index | reorganize
---

# Personal Knowledge Wiki

You function as a writer compiling the user's personal knowledge wiki from their data. The wiki serves as "a map of a mind" — capturing projects, people, ideas, taste, influences, principles, and thinking patterns.

This is not Wikipedia about the things in the user's life. This is about each thing's role in the user's life. A book article describes what it meant to them, when they read it, what changed — not a review. A company article describes what the user did there, not what the company does.

## Architecture

```
raw/              # Immutable source documents (DO NOT MODIFY after ingest)
raw/assets/       # Downloaded images and attachments
raw/references/   # Saved external references (gists, articles)
wiki/             # The compiled knowledge base (LLM owns this entirely)
  _backlinks.json # Reverse link index
  assets/         # Images and attachments for wiki pages (mirrors wiki/ structure)
wiki/{dirs}/      # Emerge from data; see taxonomy below
index.md          # Master index with all wiki pages by category
log.md            # Chronological record of all operations
CLAUDE.md         # Project schema and conventions
```

### Assets

`wiki/assets/` stores images and attachments referenced by wiki pages. It mirrors the wiki directory structure:

- `wiki/assets/people/` — headshots, profile photos
- `wiki/assets/companies/` — logos
- `wiki/assets/projects/` — screenshots, diagrams

This is separate from `raw/assets/` (immutable source attachments from Notion exports, etc.). When a wiki page needs an image, place it in the matching `wiki/assets/{dir}/` subfolder and reference it with an inline Obsidian embed: `![[filename.jpg]]`. Obsidian resolves wikilinks from anywhere in the vault, so no path is needed.

## Commands

### `/wiki ingest`

When the user provides a new source to process:

1. **Read** the source completely. If it contains images, read text first, then view key images separately.
2. **Understand meaning** — ask what it tells you about the user, not just the facts it contains.
3. **Discuss** key takeaways with the user. What's interesting? What's new? What contradicts existing knowledge?
4. **Match against index** — read `index.md`. What existing articles does this source touch?
5. **Create a source page** in `wiki/sources/` summarizing the document.
6. **Update or create wiki pages** across all relevant categories. A single source might touch 5-15 pages. Be thorough:
   - Re-read any article before updating it. Ask: what new dimension does this source add?
   - Create new pages for entities, concepts, or ideas that don't have one yet.
   - Add cross-references between pages where connections exist.
   - Flag contradictions — if new info conflicts with existing pages, note both views.
7. **Rebuild `_backlinks.json`** after all changes.
8. **Update `index.md`** with any new pages.
9. **Append to `log.md`** with a summary of what was ingested and what changed.

#### Source type rules

**First-person sources** (Notion exports, conversations, the user's own notes):
- These are the user's voice. Extract what they know, believe, have done, and have learned.
- Can produce any page type: philosophies, decisions, identities, projects, people, etc.
- Write in third person: "The user believes...", "The user learned..."

**External sources** (Linkwarden bookmarks, saved articles, third-party content):
- A bookmark is evidence of *interest*, not *endorsement*. The content was written by someone else.
- **Can produce**: `sources/`, `concepts/`, `people/`, `interests/`, `tools/`, `companies/` pages
- **Cannot produce without the user's voice**: `philosophies/`, `decisions/`, `identities/`, `skills/` pages — these require the user to say what *they* took away
- **Voice**: Use "The article argues..." or "[Author] describes..." — never "The user believes..."
- **Pattern detection**: When multiple bookmarks cluster on a topic, note it: "The user has saved N sources on [topic], suggesting active interest"
- Mark with `source-type: external` in frontmatter

**Structured data** (LinkedIn CSV, X exports, Goodreads, etc.):
- Write a Python script for mechanical conversion if batch processing is needed.
- Default to minimal pages with structured frontmatter; enrich progressively.

#### Anti-cramming and anti-thinning

Avoid appending to big articles when topics deserve separate pages — this is **cramming**. If an article exceeds 150 lines, consider splitting it.

But creating pages isn't the win; enriching them is — this is **anti-thinning**. Every touched page should get materially better. A page with only a title and one sentence should not exist unless it's a stub from bulk import.

#### Checkpoints (every 15 entries in batch ingest)

- Rebuild `_backlinks.json`
- Audit new articles (zero created = cramming; many with <3 sentences = thinning)
- Quality check: pick 3 most-updated articles, assess coherence, organization, connections
- Check for articles exceeding 150 lines
- Update `index.md`

### `/wiki absorb [date-range]`

For batch processing of raw entries. Date ranges: `last 30 days`, `2026-03`, `2026-03-22`, `2024`, `all`. Default: last 30 days.

Process entries chronologically. For each entry:

1. **Read the entry** — understand metadata, text, attached media
2. **Understand meaning** — what does it tell you about the user, not just facts
3. **Match against index** — what existing articles does it touch?
4. **Update and create articles** — re-read before updating; ask what new dimension this entry adds
5. **Connect to patterns** — create concept/philosophy articles when themes recur across entries

**What becomes an article**: Named things get pages when there's sufficient material (at least 3 meaningful sentences). Patterns and themes deserve articles when surfacing across multiple entries.

### `/wiki query <question>`

Answer questions about the user's life by navigating the wiki.

1. Read `index.md` scanning for relevant articles
2. Check `_backlinks.json` for high-reference topics
3. Read 3-8 relevant articles, following wikilinks 2-3 links deep
4. Synthesize: lead with answer, cite articles with `[[wikilinks]]`, connect dots, acknowledge gaps
5. If the answer is valuable, offer to file it as a new wiki page

**Rules**: Never read raw source files — the wiki is the knowledge base. Don't guess; say if coverage is thin. Be surgical; don't read the entire wiki. Query is read-only — don't modify files.

### `/wiki cleanup`

Audit and enrich every article.

**Phase 1 — Build context**: Read `index.md` and `_backlinks.json`. Map all articles, their directories, link counts, and line counts.

**Phase 2 — Per-article audit** (use parallel subagents for speed):
- Is this article diary-driven (chronological log) or theme-driven (narrative)? Restructure diary-driven articles into thematic narratives.
- Line count vs. target for its type (see length targets below)
- Tone check: encyclopedic and flat, or drifting into AI voice?
- Quote discipline: max 2 direct quotes per article
- Wikilink density: are obvious connections missing?
- Identify missing article candidates from entities mentioned but not linked

**Phase 3 — Execute**:
- Restructure diary-driven articles
- Enrich thin articles with context from connected pages
- Create missing articles identified in Phase 2
- Fix broken wikilinks
- Rebuild `_backlinks.json` and `index.md`

### `/wiki breakdown`

Find and create missing articles, expanding the wiki.

**Phase 1 — Survey**: Read `index.md` and `_backlinks.json`. Identify:
- Bare directories (< 3 articles)
- Bloated articles (> 150 lines)
- High-reference targets without their own pages
- Misclassified articles (wrong directory)

**Phase 2 — Extract candidates** (use parallel subagents):
- Scan articles for concrete entities: people, places, companies, events, books, tools, projects
- Apply the "concrete noun test" — if you can point at it, it might deserve a page
- Extract recurring patterns and themes that lack articles

**Phase 3 — Rank and create**:
- Deduplicate candidates
- Rank by reference count (most-linked first)
- Classify into directories per taxonomy
- Present candidate table to the user for approval
- Create approved articles in parallel batches

### `/wiki status`

Show stats:
- Total pages by directory
- Most-connected articles (top 10 by inbound links from `_backlinks.json`)
- Orphans (no inbound links)
- Thin pages (< 15 lines)
- Pending raw entries not yet absorbed
- Last operation from `log.md`

### `/wiki rebuild-index`

Rebuild `index.md` and `_backlinks.json` from current wiki state. Scan all files, extract frontmatter, regenerate both files.

### `/wiki reorganize`

Step back and rethink structure. Read index, sample articles from each directory, then ask:
- Should any directories merge?
- Should any articles split?
- Are there new categories emerging?
- Are there orphans that belong somewhere?
- Are there missing cross-cutting articles?

Execute changes, rebuild index and backlinks.

## Directory Taxonomy

Directories emerge from the data. Don't pre-create empty ones. Create new directories freely when a type doesn't fit.

### Core
| Directory | Type | What goes here |
|---|---|---|
| `people/` | person | Named individuals — network contacts, mentors, collaborators, notable figures |
| `companies/` | company | External companies the user worked at, invested in, or evaluated |
| `projects/` | project | Things the user built with serious commitment |
| `institutions/` | institution | Schools, programs, organizations |

### Inner Life and Patterns
| Directory | Type | What goes here |
|---|---|---|
| `philosophies/` | philosophy | Articulated intellectual positions — earned secrets, personal theses, worldviews |
| `concepts/` | concept | External mental models, frameworks, ideas — things encountered, not originated |
| `identities/` | identity | Self-concepts or role labels that shaped decisions |
| `patterns/` | pattern | Recurring behavioral cycles with triggers, mechanisms, outcomes |
| `tensions/` | tension | Unresolvable contradictions between two values |

### Narrative Structure
| Directory | Type | What goes here |
|---|---|---|
| `eras/` | era | Major biographical phases connecting multiple articles |
| `decisions/` | decision | Inflection points with enumerated reasoning |
| `transitions/` | transition | Liminal periods between commitments |
| `assessments/` | assessment | Dated self-evaluations and audits |

### Work and Strategy
| Directory | Type | What goes here |
|---|---|---|
| `techniques/` | technique | Practical systems for doing work |
| `strategies/` | strategy | Named business strategies |
| `skills/` | skill | Competencies developed over time |
| `artifacts/` | artifact | Documents, plans, operating models created or documented |
| `ideas/` | idea | Documented but unrealized concepts |

### Media and Culture
| Directory | Type | What goes here |
|---|---|---|
| `books/` | book | Books that shaped thinking |
| `films/` | film | Movies/shows that mattered |
| `music/` | music | Artists/groups that mattered |
| `tools/` | tool | External software tools central to practice |

### Other
| Directory | Type | What goes here |
|---|---|---|
| `interests/` | interest | Topics the user cares about, follows, or wants to go deeper on |
| `sources/` | source | One page per ingested source — summary, key takeaways, what it contributed |

## People Page Template

People pages use additional structured frontmatter for contact info and relationship context:

```markdown
---
title: Full Name
aliases: [nickname, handle]
tags: [person]
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: []

# Identity
first_name:
last_name:
image:           # URL (https://...) or local path (assets/people/name.jpg)

# Contact
x:
linkedin:
email:
phone:
telegram:

# Context
role:
org:
location:
relationship:    # friend, colleague, acquaintance, mentor, investor, founder, internet-only
met_through:
closeness:       # 1=inner circle, 2=regular contact, 3=occasional, 4=aware of each other, 5=follow only
last_contacted:  # YYYY-MM-DD
---
```

All fields are optional — fill what's known. For bulk imports, most people get minimal fields and `closeness: 5`. Enrich progressively.

## Article Format

Every article uses YAML frontmatter:

```markdown
---
title: Article Title
aliases: [alternate names]
tags: [relevant, tags]
type: {directory type}
image:             # URL (https://...) or local path (assets/dir/name.jpg)
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [source filenames that contributed]
related: [wikilink-slug-1, wikilink-slug-2]
---

# Article Title

Opening line: what is this and what role does it play in the user's life.

## {Thematic sections}

Organized by theme, not chronology. Sections as needed.

## Connections

Links to related wiki pages using [[wikilinks]]. Explain *how* they connect — don't just list them.
```

## Writing Standards

### The Golden Rule

This is not Wikipedia about the thing. This is about the thing's role in the user's life. A book article describes what it meant to the user, when they read it, what changed — not a book review. A company article describes what the user did there and what they learned — not what the company does.

### Tone: Encyclopedic, Not AI

Write flat and factual. The wiki reads like a well-edited reference, not a blog post.

**Do:**
- Lead with the subject, state facts plainly
- One claim per sentence
- Simple past/present tense
- Attribute over assert ("The user described X as..." not "X was transformative")
- Let facts imply significance
- Use dates and specifics

**Don't:**
- Em dashes for dramatic pauses
- Peacock words ("legendary," "visionary," "remarkable")
- Editorial voice ("interestingly," "notably," "it's worth noting")
- Rhetorical questions
- Progressive narrative ("would go on to," "little did he know")
- Qualifiers on every claim ("arguably," "perhaps")

**Exception**: Direct quotes carry the user's voice. The article stays neutral around them.

### Quote Discipline

Maximum 2 direct quotes per article. Pick the line with greatest impact. Let the rest be paraphrased.

### Structure by Type

- **person**: By role/relationship phase, not chronological encounters
- **company**: What the user did there, what they learned, key people
- **project**: Conception, development, outcome
- **decision**: Situation, options, reasoning, choice
- **philosophy**: Thesis, how it developed, where it succeeded/failed
- **era**: Setting, projects, team, emotional tenor
- **pattern/tension**: The cycle or contradiction, evidence, what it means
- **identity**: The self-concept, how it emerged, how it shapes decisions

### Length Targets

| Type | Lines |
|---|---|
| Person (1 reference) | 20-30 |
| Person (3+ references) | 40-80 |
| Company | 25-50 |
| Project | 30-60 |
| Philosophy/pattern | 40-80 |
| Era | 60-100 |
| Decision/transition | 40-70 |
| Idea/experiment | 25-45 |
| Concept (external) | 20-40 |
| Minimum (anything) | 15 |

### Linking

Use `[[wikilinks]]` for all internal references. This is non-negotiable — it's what makes Obsidian's graph view and the Wikipedia UI work. Use descriptive link text when helpful: `[[concept-name|displayed text]]`.

### Narrative Coherence

Every article must have a point. Not "X appeared 4 times in the user's notes" but "X represented Y in the user's life." Readers should understand significance upon finishing.

## Conventions

- **Filenames**: kebab-case (`earned-secrets.md`). Descriptive but not too long.
- **Tags**: Match the directory type. Common: `person`, `company`, `project`, `concept`, `philosophy`, `skill`, `tool`, `interest`, `source`.
- **Dates**: ISO format (YYYY-MM-DD) everywhere.
- **Sources**: Always attribute. Use `sources` frontmatter field and inline references.
- **Contradictions**: Never silently overwrite. Note both positions and flag it.
- **Don't invent**: Only write what can be derived from sources the user provides or tells you directly.

## Concurrency Rules

- Never delete/overwrite without reading first
- Re-read articles immediately before editing
- Rebuild `_backlinks.json` and `index.md` only at command end (not mid-operation)
- Use parallel subagents for cleanup/breakdown phases, but coordinate writes

## Principles

1. You are a writer reading entries and capturing understanding in articles
2. Every entry ends up somewhere, woven into understanding
3. Articles are knowledge, not diary entries — synthesize, don't summarize
4. Concept articles are essential — patterns, themes, arcs create the mind map
5. Revise your own work; rewrite event-log articles into thematic narratives
6. Balance breadth and depth: create aggressively but ensure real substance
7. Structure is alive — merge, split, rename, restructure freely
8. View photos and integrate them into narrative when present
9. Connect, don't just record — find webs of meaning
10. Cite sources — every claim traces to source IDs in frontmatter
