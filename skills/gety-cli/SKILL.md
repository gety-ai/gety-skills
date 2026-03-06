---
name: gety-cli
description: Search and retrieve local documents via Gety CLI. Use when the user needs information from their local files, documents, or knowledge base.
---

## Usage

```
/gety-cli <natural language query>
```

### Examples

```
/gety-cli What docx files have I updated in the last 24 hours?
/gety-cli Find meeting notes about the Q1 roadmap
/gety-cli Search for PDF documents related to security review in my Work folder
/gety-cli What's in my notes about the design system?
/gety-cli Show me the latest documents I've worked on
/gety-cli What data sources do I have connected?
/gety-cli Add ~/Documents/projects as a data source
```

## Workflow

```
- [ ] Step 1: Verify gety and discover data sources ⛔ BLOCKING
- [ ] Step 2: Translate query to gety command(s)
- [ ] Step 3: Execute and iterate
- [ ] Step 4: Read specific documents (conditional)
- [ ] Step 5: Synthesize findings ⚠️ REQUIRED
```

### Step 1: Verify gety and discover data sources ⛔ BLOCKING

Run `gety connector list`.

- Has output → proceed. Keep the returned connector names for use as `-c` filter values in later steps.
- Command not found → tell user to install Gety CLI from the Gety desktop app settings. Do NOT attempt any gety commands.
- No connectors → tell user to add data sources first (via Gety desktop app or `gety connector add <path>`).

### Step 2: Translate query to gety command(s)

Parse the user's natural language query and map it to gety CLI flags:

| User intent | Maps to |
|-------------|---------|
| File type ("docx files", "PDFs") | `-e <ext>` |
| Data source ("from Work folder", "in my notes") | `-c "<connector_name>"` |
| Time range ("last 24 hours", "this week") | `--update-time-from` / `--update-time-to` (ISO 8601) |
| Recency ("recent", "latest") | `--sort-by update_time --sort-order descending` |
| Topic or keyword | `<query>` string |
| Title search ("named X", "titled X") | `--match-scope title` |
| List data sources | → `gety connector list` |
| Add data source | → `gety connector add <path>` |
| Remove data source | → `gety connector remove <id>` ⚠️ confirm with user first |

**Time calculations**: When the query mentions relative time ("last 24 hours", "this week"), compute the ISO 8601 timestamp from the current date/time.

#### Translation examples

| Query | Command |
|-------|---------|
| "What docx files have I updated in the last 24 hours?" | `gety search "" -e docx --sort-by update_time --sort-order descending --update-time-from <24h ago>` |
| "Find meeting notes about Q1 roadmap" | `gety search "Q1 roadmap meeting notes"` |
| "PDFs about security in my Work folder" | `gety search "security" -c "Folder: Work" -e pdf` |
| "What data sources do I have?" | `gety connector list` |

### Step 3: Execute and iterate

Run the translated command. If results aren't satisfactory:

- Too many irrelevant results → add filters (`-c`, `-e`, `--match-scope`)
- Too few results → broaden query, try synonyms, remove filters
- Need more results → paginate with `--offset`
- Need structured data for further processing → add `--json` and pipe to `jq`

### Step 4: Read specific documents (conditional)

When full document content is needed:

```bash
gety doc <connector_id> <doc_id>
```

- Extract `connector_id` and `doc_id` from search results
- Only read documents that add information you don't already have
- For large result sets, read the top 3-5 most relevant first

### Step 5: Synthesize findings ⚠️ REQUIRED

- Combine information from multiple documents into a coherent answer
- Cite source documents (name or path) so the user knows where info came from
- If no relevant results found, say so clearly and suggest alternative search terms
- NEVER paste raw gety output — always synthesize into natural language

## Anti-Patterns

- ❌ Dumping raw gety output without synthesis
- ❌ Reading every document from search results — only read what's needed
- ❌ Single search attempt then giving up — try alternative queries and filters
- ❌ Using commands outside the CLI surface (`search`, `doc`, `connector` only)
- ❌ Asking the user to run gety commands manually — run them directly

## Reference

Load `references/cli-capabilities.md` for full CLI options, flags, and examples.
