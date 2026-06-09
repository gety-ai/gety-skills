# Gety Custom Connector Contract

## Table of Contents

- Project layout
- Manifest rules
- Sensitive config
- SDK lifecycle
- WireDoc fields
- Link and preview rules
- Incremental sync and batching
- Remote files
- Validation workflow

## Project Layout

Start by cloning the sample repository:

```bash
git clone https://github.com/gety-ai/gety-sample-connector <connector-folder>
# Prefer SSH (and have GitHub SSH keys set up)? Use this instead:
# git clone git@github.com:gety-ai/gety-sample-connector.git <connector-folder>
```

Use only a clone or fork of `gety-sample-connector` for new connector work. Do not hand-create a connector skeleton, because the sample contains the build, generated config type, local runner, and verification wiring expected by this skill.

Use the current sample shape:

```text
manifest.json
src/index.ts
src/index.test.ts
src/gen/manifest.d.ts
dist/main.js
dist/main.js.map
deno.json
dev/runner.ts
scripts/build.ts
```

`manifest.json.entry` should point to the committed build artifact, usually `dist/main.js`. Gety loads the entry from the installed folder on every poll, so source edits require rebuild plus Restart in Gety. Any change to `manifest.json` (including adding an `icon`) requires uninstalling and reinstalling the connector in Gety, because Gety reads the manifest only at install time.

## Manifest Rules

Minimal manifest:

```json
{
  "id": "my_connector",
  "name": "My Connector",
  "version": "0.1.0",
  "min_app_version": "0.5.1",
  "entry": "dist/main.js",
  "description": "Indexes documents from My Connector.",
  "schedule": { "interval": 3600 },
  "doc_link": { "kind": "url", "field": "url" },
  "config": {
    "fields": [
      {
        "id": "api_key",
        "title": "API Key",
        "type": "password",
        "required": true,
        "description": "Create this token in the service settings."
      }
    ]
  }
}
```

Rules:

- `id` must match `^[a-z0-9][a-z0-9_-]*$`: start with a lowercase letter or digit, then lowercase letters, digits, `_`, or `-` (no leading `_`/`-`, no uppercase, no dots). Prefer stable service names such as `linear` or `github`.
- `version` and `min_app_version` are SemVer. Use the sample's current `min_app_version` unless the connector needs a newer Gety contract.
- `entry` and `icon` must be relative paths inside the connector root. `icon` is optional (a PNG or SVG in the connector root) and is shown in Gety's connector list and install dialog.
- `schedule.interval` is seconds; values below 60 are clamped. Omit the field for Gety's default or set a service-appropriate interval.
- `config.fields[].type` can be `text`, `password`, `number`, `checkbox`, `dropdown`, or `directory`.
- Dropdown fields require ordered `options: [{ "value": "...", "label": "..." }]`.
- Unknown fields are rejected. `permissions`, `mode`, dynamic hooks, and per-doc links are not supported; do not add them.
- `schema.extra_indexed_fields` makes extra `metadata` keys searchable (Gety already indexes `title` and `content`). Each entry needs both `field` and `strategy`, e.g. `"schema": { "extra_indexed_fields": [{ "field": "status", "strategy": "fast_text" }] }`. Choose `strategy` by how the field should match:
  - `fast_text` — in-memory index for fast exact / substring / fuzzy (and pinyin) matching on short values. Best for titles, names, tags, folders, statuses, and similar short metadata.
  - `full_text` — SQLite FTS5 keyword search with BM25 ranking and tokenization. Best for longer text where ranked keyword search matters.
  - `semantic` — embeds the field and matches by meaning (vector similarity; requires the embedding model). Best for meaningful prose where conceptually related queries should match; overkill for short categorical values.

## Sensitive Config

For secrets such as API keys, tokens, passwords, and authorization headers:

- Use `config.fields` with `type: "password"` for Gety install/config UI.
- Do not ask the user to paste secrets into chat.
- Ask the user to edit `.env` themselves for local runner validation.
- Tell the user the exact environment variable names. For `api_key`, the runner accepts `GETY_CONFIG_API_KEY` or `API_KEY`; for `auth.api_key`, it accepts `GETY_CONFIG_AUTH_API_KEY` or `AUTH_API_KEY`.
- Keep `.env` out of git and use `.env.example` for non-secret placeholders only.

## SDK Lifecycle

Use the SDK surface:

```ts
import { Connector, type PollResult, del, upsert } from '@gety-ai/connector-sdk';
import type { ManifestConfig } from './gen/manifest.d.ts';

type State = {
  cursor?: string;
};

export default class MyConnector extends Connector<ManifestConfig, State> {
  async onLoad(): Promise<void> {
    // Optional. Runs once per fresh runtime process before poll().
  }

  async *poll(): AsyncGenerator<PollResult, void, unknown> {
    let cursor = this.lastState?.cursor;

    for await (const page of fetchPages(this.config.api_key, cursor, this.signal)) {
      yield {
        updates: page.items.map((item) => upsert(toDoc(item))),
        state: { cursor: page.next_cursor },
      };
      cursor = page.next_cursor;
    }
  }
}
```

Lifecycle facts:

- Each scheduled poll starts a fresh connector runtime process.
- `this.config` is injected from Gety's install/config form.
- `this.lastState` is the host-persisted state from the last successful poll progress.
- `this.signal` is aborted when the user disables or restarts the connector.
- `onLoad` is optional and may be async.
- Class fields do not persist across polls. Use state or an explicit on-disk cache only when needed.

## WireDoc Fields

Use `upsert(doc)` and `del(id)`.

```ts
upsert({
  id: issue.id,
  title: issue.title,
  content: markdown,
  content_format: 'markdown',
  doc_type: 'linear:issue',
  doc_updated_at: issue.updatedAt,
  original_file_size: markdown.length,
  metadata: {
    url: issue.url,
    created_at: issue.createdAt,
    team: issue.teamName,
    status: issue.status,
  },
});
```

Field guidance:

- Only `id` and `title` are required; every other WireDoc field is optional and dropped from the wire when unset.
- `id` must be stable across polls.
- `content` is the text Gety indexes and previews.
- `content_format` is `plaintext` (the default) or `markdown`; it controls content rendering, not source type. Use `markdown` only for real markdown — plain text marked `markdown` collapses single line breaks and interprets characters like `#`/`*`.
- `doc_type` should usually be `<namespace>:<type>`, such as `linear:issue`. Use `file:<ext>` only when the source really represents a file-like document and the UI should use file-extension presentation.
- `doc_updated_at` must be RFC 3339 with `Z` or an offset. It should be the source document update time, not the indexing time.
- `metadata.created_at`, if present, must be an RFC 3339 source creation timestamp; a non-string or non-RFC-3339 value fails the poll. It is shown in preview display.
- `metadata.url` is the conventional stable browser URL used by `doc_link.field = "url"`.
- `hide_from_search` can hide docs while preserving them for parent/relationship use.

## Link and Preview Rules

`doc_link` is declared once in manifest for the whole connector type:

```json
{ "kind": "url", "field": "url" }
```

Resolution rules:

- `field` is either a core doc field (`id`, `parent_id`, `title`, `content`, `doc_type`) or a direct metadata key.
- Use `"field": "url"` to read `metadata.url`.
- Do not use `"metadata.url"`; dot paths are not supported.
- Missing, empty, or non-string values mean the doc has no link.
- URL links open in the browser; Gety does not fetch URL content.
- File links must be stable local filesystem paths. Do not point them at temporary cache files.

Preview/icon implications:

- `content_format: "markdown"` enables markdown rendering for text/source preview when no file link exists.
- Open/reveal/copy actions are controlled only by `doc_link`, not by `content_format`.
- A Linear connector should keep `doc_type: "linear:issue"` and set `content_format: "markdown"` instead of pretending the issue is a local `.md` file.

## Incremental Sync and Batching

Prefer real source boundaries:

- Remote APIs: yield per page or per durable cursor boundary.
- Local enumerations: yield per file/chunk boundary that can be retried safely.
- Deletes: emit `del(id)` when the source proves the document disappeared or is no longer in scope.

State guidance:

- State should be a durable checkpoint after the yielded updates.
- Do not advance state before yielding the corresponding docs.
- If a poll fails after a yielded batch, Gety can retry from the last committed state.
- For high-water-mark APIs, keep enough state to avoid missing updates with equal timestamps.
- For local sources, prefer compact fingerprints such as relative path, mtime, and size over re-reading everything.

Batching and size:

- Large update batches are transmitted safely, so you can `yield` a page of docs at once.
- Gety limits document content to about 10 MB; do not emit huge individual docs or huge metadata blobs.
- Transmission safety does not lower memory use: building every doc before yielding still peaks. Stream pages instead.

## Remote Files

Links resolve only from the manifest `doc_link` and each doc's fields; a connector cannot resolve files or links dynamically at preview time.

For remote files (Dropbox/Drive-like):

- Put any text the connector already has into `content`, and a stable web URL into `metadata.url`.
- Represent images and PDFs as extracted/OCR text only if the connector already has that text.
- Do not yield temporary local paths as `file` links for remote files.

Cached source-file preview, lazy download, host-side extraction, and download/reveal of remote files are not supported. If the user needs those, say so plainly and stop.

## Validation Workflow

Run the strongest available command:

```bash
deno task verify
```

If `verify` is unavailable, run its equivalents (`verify` itself runs `fix && check && test && build`):

```bash
deno task fix     # fmt + lint --fix; use `deno task fmt:check` and `deno task lint` for read-only
deno task check   # runs generate, then type-checks
deno task test
deno task build
```

Use the local runner before installing in Gety:

```bash
cp .env.example .env
deno task runner -- --reset-state
deno task runner -- --polls 2
```

The runner persists incremental state at `dev/.runner/state.json` (override with `--state`); `--reset-state` ignores it. The `state.before.json` and `state.after.json` under `dev/runs/<timestamp>/` are per-run snapshots, not the live state.

If `.env` needs secrets, tell the user to edit it themselves before running the runner. Do not request secret values in chat.

Inspect runner output under `dev/runs/` for:

- stable IDs
- expected markdown/plaintext content
- correct `metadata.url`
- RFC 3339 timestamps
- delete operations
- state before/after behavior

After every runner run, report the concrete output path back to the user. The sample runner writes data under:

```text
dev/runs/<timestamp>/
  summary.json
  state.before.json
  state.after.json
  updates.json
  deletes.json
  docs/
    0001-<doc_type>__<doc_id>.md
    0001-<doc_type>__<doc_id>.json
```

The content file is `.md` for `content_format: "markdown"` and `.txt` otherwise (omitted when the doc has no content); the `.json` sidecar is always written.

Gety lifecycle is UI-driven:

1. User installs from the local connector folder in Gety Custom Connectors.
2. User fills config fields.
3. Gety starts indexing in the background.
4. After code edits, rebuild and ask the user to click Restart.
5. Verify indexed results with `gety search <query>` and, when needed, `gety doc <connector-id> <doc-id>`.
