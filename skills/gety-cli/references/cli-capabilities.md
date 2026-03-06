# Gety CLI Reference

## Commands

| Command | Purpose |
|---------|---------|
| `gety search <query>` | Search indexed documents |
| `gety doc <connector_id> <doc_id>` | Get full document content |
| `gety connector list` | List all data source connectors |
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
gety doc <connector_id> <doc_id>
```

Returns full document details and content. The `connector_id` and `doc_id` come from search results.

### Example

```bash
gety doc folder_1 doc_123
```

## gety connector

```bash
gety connector list
gety connector add <path> [--name <name>]
gety connector remove <connector_id>
```

### Examples

```bash
gety connector list
gety connector add /path/to/docs --name "Folder: Docs"
gety connector remove folder_1
```
