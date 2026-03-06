---
name: gety-cli
description: Search and retrieve local documents via Gety CLI for use as context. Use when the current task needs information from user's local files, documents, or knowledge base. Actions: search files, find documents, retrieve content, lookup notes, query local knowledge, read document, fetch file content, get doc details, list data sources, manage connectors. Objects: local files, documents, meeting notes, PDF, Markdown, knowledge base, connectors, folders, local docs. Triggers: "search my files", "find in my docs", "what do my documents say about", "check my local notes", "look up in my files", "gety search", "gety doc", "local knowledge retrieval", "search local documents".
---

## Workflow

```
- [ ] Step 1: Verify gety availability ⛔ BLOCKING
- [ ] Step 2: Plan search strategy
- [ ] Step 3: Search documents
- [ ] Step 4: Read specific documents (conditional)
- [ ] Step 5: Synthesize findings ⚠️ REQUIRED
```

### Step 1: Verify gety availability ⛔ BLOCKING

Run `gety --help`.

- Has output → proceed
- Command not found → tell user to install Gety CLI from the Gety desktop app settings. Do NOT attempt any gety commands.

### Step 2: Plan search strategy

Ask yourself:
- What specific information does the task need?
- Should I filter by data source, file type, or time range?
- Is this a broad exploration or a targeted lookup?

If scope is unclear, run `gety connector list` to see available data sources.

### Step 3: Search documents

```bash
gety search "<query>" [options]
```

Key options (load `references/cli-capabilities.md` for full reference):
- `-n <limit>` — max results (default: 10)
- `--offset <n>` — pagination
- `-c "<connector_name>"` — filter by data source (repeatable / comma-separated)
- `-e <ext>` — filter by file type (e.g., `pdf,docx`)
- `--sort-by update_time --sort-order descending` — when recency matters
- `--update-time-from` / `--update-time-to` — ISO 8601 time range
- `--match-scope <title|content|semantic>` — match type filter
- `--json` — structured output for scripting and CLI pipelines (e.g., piping to `jq`)

Search strategy:
- Start with a focused query matching the user's intent
- Too many irrelevant results → add `-c` or `-e` filters
- Too few results → broaden query, try synonyms, remove filters
- Time-sensitive queries → sort by `update_time` descending
- Need more results → paginate with `--offset`

### Step 4: Read specific documents (conditional)

When full document content is needed:

```bash
gety doc <connector_id> <doc_id>
```

- Extract `connector_id` and `doc_id` from search results
- Ask: "Does reading this document add information I don't already have?"
- For large result sets, read the top 3-5 most relevant first — don't read everything

### Step 5: Synthesize findings ⚠️ REQUIRED

- Combine information from multiple documents into a coherent answer
- Cite source documents (name or path) so the user knows where the information came from
- If no relevant results found, say so clearly and suggest alternative search terms
- NEVER paste raw gety output — always synthesize into natural language

## Connector Management

When user asks to manage data sources:

```bash
gety connector list
gety connector add <path> [--name "<name>"]
gety connector remove <connector_id>
```

⚠️ Confirm with user before `connector remove` — this is destructive.

## Anti-Patterns

- ❌ Dumping raw gety output to the user without synthesis
- ❌ Reading every document from search results — only read what's needed
- ❌ Single search attempt then giving up — try alternative queries and filters
- ❌ Using commands outside the CLI surface (`search`, `doc`, `connector` only)
- ❌ Ignoring filters when the user's intent is specific to a source or file type
- ❌ Asking the user to run gety commands manually — run them directly

## Reference

Load `references/cli-capabilities.md` for full CLI options, flags, and examples.
