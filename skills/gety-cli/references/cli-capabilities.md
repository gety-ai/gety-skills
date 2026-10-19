# Gety CLI Reference

## Commands

| Command | Purpose |
|---------|---------|
| `gety search <query>` | Search indexed documents |
| `gety doc <connector_id> <doc_id>` | Get full document content (supports pagination) |
| `gety connector list` | List connectors (`--kind fs\|custom\|all`) |
| `gety connector show <connector_id>` | Show connector details, status, and config |
| `gety connector install <directory>` | Install a custom connector from a local directory, `--github <input>`, or `--npm <input>` |
| `gety connector configure <connector_id>` | Reconfigure connector settings or schedule |
| `gety connector restart <connector_id>` | Restart a connector worker process |
| `gety connector poll <connector_id>` | Trigger an immediate sync (custom) or index update (folder); returns `accepted` |
| `gety connector rebuild <connector_id>` | Delete the connector's documents and index again from scratch; returns `accepted`, no confirmation |
| `gety connector enable <connector_id>` | Enable a connector |
| `gety connector disable <connector_id>` | Disable a connector |
| `gety connector add <path>` | Add a folder connector |
| `gety connector remove <connector_id>` | Remove a connector |

## Global Flags

| Flag | Effect |
|------|--------|
| `--json` | Structured JSON output |
| `--help` | Show help |
| `--version` | Show version |

## gety search

```bash
gety search <query> [options]
```

### Options

| Flag | Description |
|------|-------------|
| `-n, --limit <n>` | Max result count |
| `--offset <n>` | Pagination offset |
| `--no-semantic-search` | Disable semantic search |
| `-c, --connector <name>` | Filter by connector name (repeatable or comma-separated) |
| `-e, --ext <ext>` | Filter by extension (repeatable or comma-separated) |
| `--match-scope <scope>` | `title`, `content`, or `semantic` |
| `--sort-by <field>` | `default` or `update_time` |
| `--sort-order <order>` | `ascending` or `descending` |
| `--update-time-from <iso8601>` | Update-time range start |
| `--update-time-to <iso8601>` | Update-time range end |

### Examples

```bash
gety search "meeting notes"
gety search "roadmap" -n 20 --offset 20
gety search "security review" -c "Folder: Work" -e pdf,docx
gety search "design system" --match-scope title,content --sort-by update_time --sort-order descending
gety search "Q1 report" --json
```

## gety doc

```bash
gety doc <connector_id> <doc_id> [options]
```

Returns full document details and content. The `connector_id` and `doc_id` come from search results.

### Options

| Flag | Description |
|------|-------------|
| `--page-start <n>` | First page to return (1-based) |
| `--page-end <n>` | Last page to return (1-based) |

### Best Practices for Large Documents

When fetching documents that could be large (PDFs, large docs, logs), **do not dump full content directly into the conversation context**. Instead:

1. **Save to a file and search with `grep`**:
   ```bash
   gety doc folder_1 doc_123 > /tmp/doc.txt
   grep -i -C 3 "renewal clause" /tmp/doc.txt
   ```
2. **Paginate if page numbers are known**:
   ```bash
   gety doc folder_1 doc_123 --page-start 1 --page-end 3
   ```

## gety connector

Manage filesystem and custom connectors.

### Subcommands

```bash
# List connectors
gety connector list [--kind fs|custom|all]

# Inspect a connector
gety connector show <connector_id>

# Install a custom connector: local directory (linked, not copied), GitHub Release, or npm package
gety connector install <directory> [--config <file|->] [--set <key=json>...] [--secret <key>...] [--secret-file <key=path>...] [--yes] [--dry-run]
gety connector install --github <owner/repo|repo link|Release link> [--version <tag|version>] [config flags] [--yes] [--dry-run]
gety connector install --npm <name|@scope/name|name@version|npmjs.com link> [--version <version>] [config flags] [--yes] [--dry-run]

# Reconfigure a connector
gety connector configure <connector_id> [--config <file|->] [--set <key=json>...] [--schedule manual|interval|daily] [--every <duration>] [--at <HH:MM>]

# Control execution
gety connector restart <connector_id>
gety connector poll <connector_id>
gety connector rebuild <connector_id>
gety connector enable <connector_id>
gety connector disable <connector_id>

# Manage filesystem folders
gety connector add <path> [--name <name>]
gety connector remove <connector_id>
```

### Install & Configure Options

| Flag | Description |
|------|-------------|
| `--dry-run` | (`install` only) Show the preview and the permission summary without installing; GitHub/npm previews download only metadata |
| `--yes` | (`install` only) Grant the permissions shown without asking. Without it the CLI asks once on a terminal and fails with `consent_required` elsewhere. For an actual install, `--config -` or `--secret` require `--yes`; `--dry-run` does not. Get the user's approval of the `--dry-run` summary first |
| `--github <input>` | (`install` only) Public GitHub repo: `owner/repo` or repo link (latest Release), or a Release link |
| `--npm <input>` | (`install` only) Public npm package: `name`, `@scope/name`, `name@1.2.0`, or an npmjs.com package link (latest version unless one is named) |
| `--version <tag\|version>` | (`install` with `--github`/`--npm` only) Release tag or version for GitHub; exact version for npm. Not with an input that already names one |
| `--config <file>` | JSON config file path, or `-` for stdin |
| `--set <key=json>` | Set a dotted non-secret config property (e.g. `--set 'workspace="acme"'`). Password fields are rejected here |
| `--secret <key>` | Prompt for a password field value on TTY (stderr) |
| `--secret-file <key=path>` | Read a password field from a file (strips trailing newline, max 1MB) |
| `--schedule <strategy>` | Schedule strategy: `manual`, `interval`, or `daily` |
| `--every <duration>` | Interval duration for `--schedule interval` (e.g. `60s`, `5m`, `1h`, min 60s) |
| `--at <HH:MM>` | Daily wall-clock time for `--schedule daily` (24h format) |

### Examples

```bash
# List all connectors
gety connector list --kind all

# Show connector detail
gety connector show linear-connector

# Review a custom connector's preview and permissions before install
gety connector install ./my-connector --dry-run --secret api_key

# Install after the user approved the permissions; password via no-echo prompt
gety connector install ./my-connector --yes --secret api_key

# Install from a GitHub Release or an npm package
gety connector install --github acme/gety-linear --dry-run
gety connector install --npm @acme/gety-linear@1.2.0 --yes

# Change schedule to daily sync at 09:00
gety connector configure my_connector_id --schedule daily --at 09:00

# Trigger immediate sync
gety connector poll my_connector_id

# Delete a connector's documents and index it again from scratch
gety connector rebuild my_connector_id

# Restart connector after code changes
gety connector restart my_connector_id

# Add a folder data source
gety connector add /path/to/docs --name "Folder: Docs"

# Remove a connector
gety connector remove folder_1
```
