---
name: gety-cli
description: Search and retrieve local documents via Gety CLI. Use when the user needs information from their local files, documents, or knowledge base.
argument-hint: <natural language query>
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

Run `gety connector list` (or `gety connector list --kind all` to see both folders and custom connectors).

- Has output → proceed. Keep the returned connector names for use as `-c` filter values in later steps.
- Command not found → tell user to install Gety CLI from the Gety desktop app settings. Do NOT attempt any gety commands.
- No connectors → tell user to add data sources first (via `gety connector add <path>` for folders or `gety connector install <dir>` for custom connectors).

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
| List data sources | → `gety connector list` (`--kind fs` / `--kind custom` / `--kind all`) |
| Inspect connector details | → `gety connector show <id>` |
| Add folder data source | → `gety connector add <path> [--name <name>]` |
| Install custom connector | → `gety connector install <directory> [--config <file>]` |
| Configure connector | → `gety connector configure <id> [options]` |
| Restart connector | → `gety connector restart <id>` |
| Sync / poll connector | → `gety connector poll <id>` |
| Enable / disable connector | → `gety connector enable <id>` / `gety connector disable <id>` |
| Remove data source | → `gety connector remove <id>` ⚠️ confirm with user first |

**Time calculations**: When the query mentions relative time ("last 24 hours", "this week"), compute the ISO 8601 timestamp from the current date/time.

#### Translation examples

| Query | Command |
|-------|---------|
| "What docx files have I updated in the last 24 hours?" | `gety search "" -e docx --sort-by update_time --sort-order descending --update-time-from <24h ago>` |
| "Find meeting notes about Q1 roadmap" | `gety search "Q1 roadmap meeting notes"` |
| "PDFs about security in my Work folder" | `gety search "security" -c "Folder: Work" -e pdf` |
| "What data sources do I have?" | `gety connector list --kind all` |
| "Check custom connector status" | `gety connector show <connector_id>` |

### Step 3: Execute and iterate

Run the translated command. If results aren't satisfactory:

- Too many irrelevant results → add filters (`-c`, `-e`, `--match-scope`)
- Too few results → broaden query, try synonyms, remove filters
- Need more results → paginate with `--offset`
- Need structured data for further processing → add `--json` and pipe to `jq`. Keep the snippet text in what you print (for example `.snippets[].highlighted_text.text`), not only titles: a file's name often says nothing about what's inside it.

#### Search for coverage, not just a first hit

Gety's keyword search matches every word of a multi-word query literally, and it does not merge word forms (for example "renew" and "renewal", or singular and plural). So:

- After a phrase query, also search the key concept as a single short keyword, and try its other word forms and common synonyms.
- When the question asks for a complete answer ("who", "which", "all", "every", "list"), don't stop at the first document that looks like the answer. Run at least one more search for later notes, emails, follow-ups or corrections on the same topic, and check the snippet of every plausible result before ruling it out.
- When the question asks "why" or "how did it change", look for where the decision was recorded (meeting notes, decision logs, memos, emails), not only the document that holds the numbers.

### Step 4: Read specific documents (conditional)

When document content is needed:

```bash
gety doc <connector_id> <doc_id>
```

- Extract `connector_id` and `doc_id` from search results
- Only read documents that add information you don't already have
- For large result sets, read the top 3-5 most relevant first

#### ⚠️ Handling large documents (Context Window Protection)

`gety doc` outputs full document content. Large files (lengthy PDFs, books, logs, large data dumps) can produce massive outputs that saturate context and waste tokens.

**Best practices for large files:**

1. **Save to a temporary file, then search with tools (Recommended)**:
   Redirect output to a file and use `grep`, `rg`, or `head` to inspect relevant sections:
   ```bash
   gety doc <connector_id> <doc_id> > /tmp/gety_doc.txt
   grep -i -C 3 "keyword" /tmp/gety_doc.txt
   head -n 50 /tmp/gety_doc.txt
   ```
2. **Use page ranges for paged documents**:
   If the document has pages (e.g. PDF), limit with `--page-start` and `--page-end`:
   ```bash
   gety doc <connector_id> <doc_id> --page-start 1 --page-end 5
   ```
3. **Avoid reading if snippets suffice**:
   If search results from `gety search` already provide enough context, do not fetch the full document.

### Step 5: Synthesize findings ⚠️ REQUIRED

- Combine information from multiple documents into a coherent answer
- When comparing versions (v1 vs v2, before vs after, plan vs actual), compare the underlying numbers line by line yourself. A summary or variance table inside a document may leave items out, and labels (such as a percentage written next to an amount) may not match the amounts; when they disagree, trust the amounts and say so.
- Before answering a "who / which / all" question, ask yourself whether a later note could add to or change the list, and search for it if you haven't.
- Cite source documents (name or path) so the user knows where info came from
- If no relevant results found, say so clearly and suggest alternative search terms
- NEVER paste raw gety output — always synthesize into natural language

## Anti-Patterns

- ❌ Dumping raw gety output without synthesis
- ❌ Dumping full content of large documents into conversation context — save to a file and search with `grep` or paginate with `--page-start`/`--page-end`
- ❌ Reading every document from search results — only read what's needed
- ❌ Single search attempt then giving up — try alternative queries and filters
- ❌ Using commands outside the CLI surface (`search`, `doc`, `connector` only)
- ❌ Asking the user to run gety commands manually — run them directly

## Reference

Load `references/cli-capabilities.md` for full CLI options, flags, and examples.
