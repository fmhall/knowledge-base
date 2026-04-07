---
name: linkedin-export
description: "Import LinkedIn connections into a knowledge base. Walks the user through exporting their data from LinkedIn, then processes the connections CSV to create people pages. Use when the user wants to import LinkedIn contacts, says 'linkedin export', 'import linkedin', 'linkedin connections', or wants to add their professional network to the wiki."
argument-hint: "[path/to/Connections.csv]"
---

# LinkedIn Export

Import LinkedIn connections into the user's knowledge base. This skill handles two phases: guiding the user through LinkedIn's data export (if they haven't already), then processing the exported CSV to create or update people pages.

## Interaction rules

Only ever ask the user **one question at a time**. Never present multiple decisions simultaneously.

## Phase 1: Get the export

If the user provides a path to a CSV file, skip to Phase 2.

Otherwise, walk them through the export:

1. Tell the user:
   > Go to **linkedin.com/mypreferences/d/download-my-data** and request your data.
   > Select **Connections** only — the full archive takes days, but Connections exports in ~10 minutes.
   > LinkedIn will email you when it's ready. Download the file and tell me where it is.

2. Wait for the user to provide the file path.

**Important**: LinkedIn delivers the export as a `.zip` file, but it is not actually compressed — it's a regular directory/file with that extension. Do NOT try to `unzip` it. If the user gives you a `.zip` path, look inside it directly or treat it as a directory. The file you need is `Connections.csv` inside the export.

## Phase 2: Discover the knowledge graph

Before processing anything, understand the user's project:

1. Read the project's CLAUDE.md, README, or top-level documentation to understand wiki structure.
2. Find where people pages live (e.g., `wiki/people/`, `people/`, `contacts/`).
3. Read 2–3 sample people pages to learn the frontmatter schema — field names, format, conventions.
4. Check if there's a raw/ directory for storing source documents.

If you can't determine the structure, ask: "Where should people pages go?"

## Phase 3: Process the CSV

### Understanding the format

The CSV has a 2-line preamble before the actual header:

```
Notes:
"When exporting your connection data, you may notice..."

First Name,Last Name,URL,Email Address,Company,Position,Connected On
Robert,Bench,https://www.linkedin.com/in/robert-bench-7a535b11,,Radius,Co-founder & CEO,27 Mar 2026
```

- **Row 0**: "Notes:" label
- **Row 1**: LinkedIn's privacy notice (quoted string)
- **Row 2**: Blank
- **Row 3**: Actual CSV header — `First Name,Last Name,URL,Email Address,Company,Position,Connected On`
- **Row 4+**: Data rows

Skip the first 3 lines when parsing. The actual CSV header is `First Name,Last Name,URL,Email Address,Company,Position,Connected On`.

Email addresses are sparse — LinkedIn only includes them for connections who opted in.

### Copy to raw/

Copy the Connections.csv to the project's `raw/` directory (e.g., `raw/linkedin-connections.csv`) so there's an immutable source record. Never modify the raw copy after this point.

### Triage connections

Don't blindly create a page for every connection — 2,000 pages of near-empty stubs aren't useful. Instead:

1. **Parse all rows** and count the total.
2. **Check existing pages** — scan the wiki for people who already exist (match on name, LinkedIn URL, or company+name).
3. **Group by company/organization** — this is the most useful lens for the user.
4. **Present a summary** to the user:
   > Found N connections. M already have wiki pages.
   > Top companies: [Company A] (X), [Company B] (Y), [Company C] (Z)
   > Which companies or groups should I import? Or should I import all?

5. Wait for the user to tell you which subset to import.

### Create people pages

For each person to import:

1. **Check for duplicates** — search existing wiki pages by name and LinkedIn URL. If the person already exists, update their page (fill in missing fields only, never overwrite).

2. **Create the page** using the project's existing frontmatter schema. Map LinkedIn fields:
   - `First Name` + `Last Name` → title, first_name, last_name
   - `URL` → linkedin field
   - `Email Address` → email field (if present)
   - `Company` → org field
   - `Position` → role field
   - `Connected On` → note in body text ("Connected on LinkedIn (DD Mon YYYY)")

3. **Set sensible defaults** for fields the CSV doesn't provide:
   - `relationship`: `colleague` (default for professional network)
   - `closeness`: `4` (aware of each other — appropriate for LinkedIn connections)
   - `met_through`: infer from shared company if obvious, otherwise leave blank
   - `sources`: reference the CSV file (e.g., `[linkedin-connections.csv]`)

4. **Write the body** — keep it minimal. A one-liner with their role, company, and connection date. Don't invent biography. Example:
   ```
   General Partner at a16z. Connected on LinkedIn (27 Jan 2026).

   ## Connections

   - [[company-page|Company Name]] — Worked together
   ```

5. **Link to company pages** if they exist in the wiki. Don't create company pages — just note the company name if no page exists.

### Update the index

After creating pages, update the project's index file (if one exists) and log the operation to the project's log (if one exists). Follow the project's existing log format.

## Phase 4: Report

Summarize what happened:

- Total connections in CSV
- Pages created (with count by company/group)
- Pages updated (existing pages that got new LinkedIn data)
- Pages skipped (already complete, or user chose not to import)
- Any notable findings (e.g., "12 connections have email addresses")
