# Knowledge Base Skills

A [Claude Code](https://claude.ai/code) plugin for building and maintaining personal knowledge bases with LLMs.

Inspired by [Andrej Karpathy's tweet](https://x.com/kaborsky) on LLM Knowledge Bases — the idea that LLMs are increasingly useful not just for writing code, but for compiling, maintaining, and querying structured knowledge. Raw data goes in, a wiki comes out, and the LLM owns the entire lifecycle: ingesting sources, writing articles, linking concepts, answering questions, and continuously improving data quality. You rarely touch the wiki directly — it's the domain of the LLM.

> *"raw data from a given number of sources is collected, then compiled by an LLM into a .md wiki, then operated on by various CLIs by the LLM to do Q&A and to incrementally enhance the wiki, and all of it viewable in Obsidian."*
> — Andrej Karpathy

This plugin provides the skills to make that workflow real inside Claude Code.

## Installation

```bash
npx skills add fmhall/knowledge-base
```

## Skills

### `/wiki` — Personal Knowledge Wiki

Compiles personal data — journals, notes, messages, bookmarks, exports — into a structured knowledge wiki rendered in Obsidian. The wiki is organized as a taxonomy of markdown files (people, companies, projects, philosophies, patterns, eras, decisions, and more) with YAML frontmatter, `[[wikilinks]]`, and a backlink index.

Commands:

| Command | What it does |
|---|---|
| `/wiki ingest` | Process a new source document into the wiki |
| `/wiki absorb [date-range]` | Batch-process raw entries chronologically |
| `/wiki query <question>` | Answer questions by navigating the wiki |
| `/wiki cleanup` | Audit and enrich every article |
| `/wiki breakdown` | Find and create missing articles |
| `/wiki status` | Show wiki stats — coverage, orphans, thin pages |
| `/wiki rebuild-index` | Regenerate the master index and backlinks |
| `/wiki reorganize` | Rethink and restructure the wiki's organization |

The wiki follows Karpathy's vision closely: source documents land in `raw/`, the LLM compiles them into a `wiki/` directory of interconnected articles, and Obsidian serves as the viewing frontend. The LLM maintains index files, backlink graphs, and summaries so that even at scale (~400K+ words), it can navigate the knowledge base to answer complex queries without needing RAG infrastructure.

### `/linkedin-export` — LinkedIn Connections Import

Walks the user through exporting their LinkedIn connections data, then processes the CSV to create people pages in the wiki. Handles the quirks of LinkedIn's export format (2-line preamble, sparse emails, `.zip` files that aren't actually zipped) and triages connections by company so the user can pick which groups to import rather than creating 2,000 stubs.

| Command | What it does |
|---|---|
| `/linkedin-export` | Guide through export and process connections |
| `/linkedin-export path/to/Connections.csv` | Process an already-downloaded CSV |

### `/imessage-export` — iMessage Conversations Import

Exports iMessage conversations from the macOS Messages database using [`imessage-exporter`](https://github.com/ReagentX/imessage-exporter), then analyzes the most active conversations to create or update people pages with relationship context that only personal messages reveal — closeness, shared interests, how the user actually knows someone.

| Command | What it does |
|---|---|
| `/imessage-export` | Export last 30 days and analyze top contacts |
| `/imessage-export --days 90` | Export last 90 days |
| `/imessage-export --top 100` | Analyze top 100 conversations instead of default 50 |

### `/enrich` — Contact Enrichment

Enriches people pages in the wiki with real contact and social data — LinkedIn profiles, emails, phone numbers, Twitter/X handles — using premium enrichment APIs (Minerva, Apollo) via pay-per-call micropayments. It scans the knowledge graph for people with missing contact info, calls the APIs, validates matches, and writes the data back into the existing frontmatter schema.

| Command | What it does |
|---|---|
| `/enrich all` | Scan the knowledge graph and enrich all people with missing contact info |
| `/enrich <name>` | Enrich a specific person by name |

## AgentCash

The enrichment skill is powered by [AgentCash](https://agentcash.dev) — a micropayment layer that lets AI agents call premium, paid APIs with no API keys, no subscriptions, and no accounts. AgentCash manages a USDC wallet and handles x402 micropayments per request, so agents can access services like Apollo, Minerva, Exa, Firecrawl, and others at cost-per-call pricing (typically $0.02–$0.05 per request). This means an agent can enrich 20 contacts for about $1, without the user ever signing up for a data provider.
