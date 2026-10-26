# Gety Custom Connector Contract

## Table of Contents

- Project layout
- Manifest rules
- Permissions and sandbox
- Sensitive config
- SDK lifecycle
- WireDoc fields
- Link and preview rules
- Incremental sync and batching
- Remote files
- Validation workflow
- Gety installation and lifecycle
- Publishing (GitHub Releases, npm, versions and updates)

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
dev/gety-connector-sdk/mod.ts
scripts/build.ts
```

`manifest.json.entry` points to the build output, usually `dist/main.js`: one esbuild bundle of `src/` plus its dependencies, except two kinds of imports the build leaves external because the runtime provides them: `@gety-ai/connector-sdk` (Gety maps it to its own SDK through an import map) and Node built-ins (`node:*`). The entry must resolve inside the connector folder.

A local-folder install registers the folder itself (it does not copy it) and loads the entry from it on every poll, so source edits need a rebuild plus restart. Gety snapshots the manifest at install time and never re-reads `manifest.json`; that snapshot is also the record of the permissions the user consented to. Any change to `manifest.json` (permissions, config fields, schedule, entry, icon) therefore needs `gety connector remove <id>` and a fresh install, which shows the permission review again.

## Manifest Rules

Minimal manifest:

```json
{
  "id": "my_connector",
  "name": "My Connector",
  "version": "0.1.0",
  "min_gety_app_version": "0.6.6",
  "capabilities": ["data-source"],
  "entry": "dist/main.js",
  "description": "Indexes documents from My Connector.",
  "schedule": { "strategy": "interval", "interval_seconds": 3600 },
  "doc_link": { "kind": "url", "field": "url" },
  "permissions": { "net": ["api.example.com"] },
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

- `id` must match `^[a-z0-9][a-z0-9_-]*$`: start with a lowercase letter or digit, then lowercase letters, digits, `_`, or `-` (no leading `_`/`-`, no uppercase, no dots). Keep it stable, and for a connector others will install, prefix it with your own name (`acme-linear`, not `linear`): two connectors with the same id cannot be installed side by side. A custom connector id cannot replace a built-in connector type.
- `version` and `min_gety_app_version` are SemVer. Gety refuses a connector whose `min_gety_app_version` is newer than itself (GitHub and npm installs report `version_incompatible`). A manifest that declares `permissions` must set `min_gety_app_version` to `0.6.6` or later, the first Gety release with the sandbox: every older Gety then reports that the connector needs a newer Gety instead of running it without a sandbox or failing on an unknown field.
- `capabilities` is a required non-empty array declaring which host integration points the connector participates in. A polling/indexing data source must declare `["data-source"]` (currently the only supported value); unknown values are rejected. Capabilities are host features, not sandbox permissions.
- `entry` and `icon` must be relative paths inside the connector root. `icon` is optional (a PNG or SVG in the connector root) and is shown in Gety's connector list and install dialog.
- `schedule` declares the update strategy. Supported shapes:
  - `{ "strategy": "manual" }` — no automatic polling; updates are triggered explicitly.
  - `{ "strategy": "interval", "interval_seconds": 3600 }` — poll every `interval_seconds`; values below 60 are clamped.
  - `{ "strategy": "daily", "time": "09:00" }` — poll once per day at `time` (`HH:mm` or `HH:mm:ss`, local time).
  - Omit `schedule` to use Gety's default interval behavior.
- `config.fields[].type` can be `text`, `password`, `number`, `checkbox`, `dropdown`, or `directory`.
- Dropdown fields require ordered `options: [{ "value": "...", "label": "..." }]`.
- `permissions` is described in the next section. It may be omitted but never set to `null`.
- `sync_version` is optional: a positive whole number (1 when absent) for the format of what the connector syncs. Bump it when a release cannot continue from what an older release synced; see Versions and Updates.
- Unknown fields are rejected. `mode`, dynamic hooks, and per-doc links are not supported; do not add them.
- `schema.extra_gety_indexed_fields` makes extra `metadata` keys searchable (Gety already indexes `title` and `content`). Each entry needs both `field` and `strategy`, e.g. `"schema": { "extra_gety_indexed_fields": [{ "field": "status", "strategy": "fast_text" }] }`. Choose `strategy` by how the field should match:
  - `fast_text` — in-memory index for fast exact / substring / fuzzy (and pinyin) matching on short values. Best for titles, names, tags, folders, statuses, and similar short metadata.
  - `full_text` — SQLite FTS5 keyword search with BM25 ranking and tokenization. Best for longer text where ranked keyword search matters.
  - `semantic` — embeds the field and matches by meaning (vector similarity; requires the embedding model). Best for meaningful prose where conceptually related queries should match; overkill for short categorical values.

## Permissions and Sandbox

`permissions` decides what the connector process may do. Gety shows it to the user before every install (desktop app and `gety connector install`), and enforces it with Deno's permission flags.

### Modes

- **Sandbox:** `permissions` is an object, `{}` included. The connector gets the base access below plus exactly what it lists; everything else is denied.
- **Full access:** `permissions` is omitted or `"unrestricted"`. No sandbox; the desktop review shows a red warning and asks for an extra confirmation. Older connectors keep working this way. Every new connector should declare `permissions`; use `{}` when it needs nothing beyond base access.

### Entries

```json
"permissions": {
  "net": ["api.notion.com", "*.notion.so", "files.example.com:8443", { "config_field": "base_url" }],
  "read": [
    "~/.codex/sessions",
    { "path": "~/Library/Application Support/Google/Chrome", "os": ["macos"] },
    { "config_field": "export_dir" }
  ],
  "env": ["ACME_LOG_LEVEL"],
  "run": ["git"]
}
```

- `net`: `"*"` (any host); `host` or `host:port` (no port = any port); `*.domain` (the domain itself and every subdomain, any port); an IPv4 or `[IPv6]` literal, optionally with a port. Write a lowercase ASCII host (punycode for IDN), never a URL: no scheme, path, or user info. Every redirect hop of `fetch` is checked, so also declare redirect targets, CDN/file hosts, and auth/token hosts.
- `read`: `"~"` or `"~/dir/…"`, granting that folder and everything below. Absolute paths, empty/`.`/`..` segments, and backslashes are rejected. `{ "path": "~/…", "os": ["macos", "windows", "linux"] }` applies (and is shown) only on the listed systems. For a folder the user picks, use a `directory` config field instead of an absolute path.
- `env`: exact variable names (`^[A-Za-z_][A-Za-z0-9_]*$`, no wildcards). Reading a variable that is not granted does not fail: `Deno.env.get` returns `undefined` as if it were unset, so a missing `env` entry fails silently. `env` grants reading the variable, not access to what it points to: if a location can move (a tool that honors its own `*_HOME` variable), let the user pick it in a `directory` config field and grant it with `{ "config_field": "<id>" }` in `read`.
- `run`: program names without paths (`^[A-Za-z0-9][A-Za-z0-9._+-]*$`). A started program runs outside the sandbox with the full environment, so `run` is equivalent to full access. See Running Programs below.
- `{ "config_field": "<id>" }`: the grant comes from the user's config. In `net` it must reference a `text` field whose value is a URL or `host[:port]` without wildcards (only the host and an explicit port are granted); in `read` a `directory` field holding an absolute path. `password` fields can never be referenced. An empty value grants nothing. Gety validates the value when the user installs or saves config and re-derives grants at every start; changing config does not ask for consent again. The desktop install review resolves these entries against the field defaults; `gety connector install` (including `--dry-run`) resolves them against the merged config passed to that command (`--config`/`--set`).

### Running Programs

Deno resolves each `run` name through `PATH` when the connector starts, and only that exact absolute path may be started; the same program at another path, or through its resolved symlink target, is denied. A name not found on `PATH` grants nothing. On macOS Gety appends `/opt/homebrew/bin` and `/usr/local/bin` to `PATH` when missing, because an app started from Finder gets only launchd's minimal `PATH`. Directories that shells add at startup (nvm, volta) are not on that `PATH`, so CLIs installed with them are not found; ask users to link the program into one of the directories above, or prefer an HTTP API.

### Risk Tiers

Gety computes one tier per connector and colors the review by it: `offline` (no `net`), `network_only` (`net` without `read`/`env`), `read_and_network`, `high_risk`, `unrestricted` (full access).

A connector is `high_risk` when `run` is non-empty, `net` contains `"*"` or a single-label wildcard such as `*.com`, or `read` contains `"~"` (on the current OS). In the desktop app, `high_risk` and `unrestricted` require an extra confirmation that lists the reasons. The CLI prints the same risk lines (`!`) but asks only once on a terminal and not at all with `--yes`, so an agent must get the user's approval of the summary before passing `--yes`. Stay below `high_risk` whenever the source allows: name exact API hosts or `*.vendor.com` instead of `*` / `*.com`, read `~/.app/sub` instead of `~`, and call HTTP APIs with `fetch` instead of running a CLI.

### Base Access

Every sandboxed connector has, without declaring it (the review does not list these):

- read/write `this.dataDir`: the working directory; persists between syncs; deleted when the connector is uninstalled.
- read/write a per-sync temp folder (`TMPDIR`, `TMP`, `TEMP` point to it); deleted after each sync. `Deno.makeTempDir()` / `Deno.makeTempFile()` land there.
- system info: `homedir`, `cpus`, `systemMemoryInfo`, `osRelease`, `osUptime`, `loadavg`, `statfs`.
- read its own code.
- its own `DENO_DIR`, so `localStorage`, the Cache API, and the module cache are not shared with other connectors.

Never available in the sandbox: writing anywhere else, `ffi`, the hostname, network interfaces, user/group ids, and remote (`https:`) or `npm:` imports. A connector that truly needs them must run with full access; avoid that.

Use `this.dataDir` for state too large for `lastState` and caches that should survive between syncs; use the temp folder for scratch copies (for example, a copy of a locked SQLite file) that only one sync needs.

### What Fails in the Sandbox

| Symptom | Fix |
| --- | --- |
| `import` from `npm:`/`jsr:`/`https:` at run time, or a dynamic `import()` of one, fails | Let the build bundle the dependency into the entry (`deno task build` does this with esbuild); only `@gety-ai/connector-sdk` and `node:*` built-ins stay external. Gety runs with `--no-config`, so `deno.json` import maps do not apply at run time. |
| `permission denied: net access to "host:443" is not granted by the connector's manifest` | Add the host (and every redirect / CDN host) to `net`. |
| `getaddrinfo EPERM <host>` from `node:http`/`node:https` | Same missing `net` entry; Node APIs do not name the permission. Prefer `fetch`. |
| `Deno.env.get("HOME")` or a library's home lookup returns `undefined` or the wrong folder | Use `this.homeDir` / `this.expandHome("~/.app")`; they match the paths Gety grants for `~`. |
| `... access ... is not available to sandboxed connectors` (`write`, `sys`, `ffi`, `import`) | No manifest entry grants these: write to `this.dataDir` or the temp folder, drop identity-type `sys` calls, bundle imports. |
| `permission denied: read access to "<path>" ...` | Add the folder to `read` (`~/…`, or a `directory` config field). |

## Sensitive Config

For secrets such as API keys, tokens, passwords, and authorization headers:

- Use `config.fields` with `type: "password"` for Gety install/config UI.
- Do not ask the user to paste secrets into chat.
- Ask the user to edit `.env` themselves for local runner validation.
- Tell the user the exact environment variable names. For `api_key`, the runner accepts `GETY_CONFIG_API_KEY` or `API_KEY`; for `auth.api_key`, it accepts `GETY_CONFIG_AUTH_API_KEY` or `AUTH_API_KEY`.
- Keep `.env` out of git and use `.env.example` for non-secret placeholders only.
- With the Gety CLI, pass secrets with `--secret <field>` (no-echo prompt) or `--secret-file <field>=<path>`. `--set` rejects password fields.
- Password fields cannot be referenced from `permissions` (`config_field`).

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
- `this.lastState` is the state Gety committed after the last poll that completed successfully, or `null`.
- `this.signal` is aborted when the user disables or restarts the connector or Gety cancels the poll. Cancellation is cooperative first, then the process is killed after a grace period; release sockets and temp files when it aborts.
- `this.dataDir` is the absolute path of the persistent data folder (also the working directory). `this.homeDir` is the home directory Gety resolves `~` against, and `this.expandHome('~/x')` expands a leading `~` with it. Do not read `HOME` from the environment.
- `onLoad` is optional and may be async.
- Class fields do not persist across polls. Use state, or files in `this.dataDir`.
- A poll that sends Gety nothing for 10 minutes is treated as hung and cancelled. During long work, yield batches or log a low-frequency progress line such as `console.info('still syncing...')`.

The build leaves `@gety-ai/connector-sdk` external: installed connectors run against the SDK Gety ships (mapped by Gety's import map), which provides `dataDir`, `homeDir`, and `expandHome`, and Gety's runner injects `config`, `lastState`, `signal`, `dataDir`, and `homeDir`. The sample's `dev/gety-connector-sdk/mod.ts` is only a development copy that `deno.json` maps the import to for `deno task check`, tests, and the local runner; editing it changes nothing in Gety. If an older shim lacks `dataDir`, `homeDir`, and `expandHome`, add them so the connector type-checks and runs locally:

```ts
/** Absolute path of this connector's persistent data directory. */
protected dataDir = '';
/** Home directory Gety resolves `~` in `permissions.read` against. */
protected homeDir = '';

protected expandHome(path: string): string {
	const windows = Deno.build.os === 'windows';
	if (path === '~') return this.homeDir;
	const separators = windows ? /[\\/]/ : /\//;
	if (!path.startsWith('~') || !separators.test(path[1] ?? '')) return path;
	const rest = path.slice(2).split(separators).filter((segment) => segment !== '');
	return [this.homeDir.replace(/[\\/]+$/, ''), ...rest].join(windows ? '\\' : '/');
}
```

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

- `kind` is `url` or `file`.
- `field` is either a core doc field (`id`, `parent_id`, `title`, `content`, `doc_type`) or a direct metadata key.
- Use `"field": "url"` to read `metadata.url`.
- Do not use `"metadata.url"`; dot paths are not supported.
- Missing, empty, or non-string values mean the doc has no link.
- URL links open in the browser; Gety does not fetch URL content.
- File links must be stable local filesystem paths of the user's own files. Do not point them at temporary cache files.
- Gety refuses to open a link (file link, `file:` URL, or local path) that points into any connector's folders — its data and temp folders, Gety's connector directory, or the connector's code folder — or to a file that does not exist. "Show in folder" still works. So never write files into `this.dataDir` or the temp folder and link to them; link to the upstream service or to the user's files.

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

- Gety indexes each yielded batch as it arrives, but commits the state of the last yielded batch only after the whole poll finishes successfully. If a poll fails, the next poll starts again from the previous committed state, and some docs of the failed poll may already be indexed. Make every batch safe to replay: `upsert` by stable id converges.
- Do not advance state before yielding the corresponding docs.
- For high-water-mark APIs, keep enough state to avoid missing updates with equal timestamps.
- For local sources, prefer compact fingerprints such as relative path, mtime, and size over re-reading everything.
- Config plus state must fit the 16 MiB start frame; a poll whose next state would not fit fails and keeps the previous checkpoint. Keep state compact and put large maps in `this.dataDir`.

Batching and size:

- Large update batches are transmitted safely, so you can `yield` a page of docs at once.
- Gety keeps at most 10 MB (UTF-8 bytes) of each document's `content` and truncates the rest; do not emit huge individual docs or huge metadata blobs.
- Transmission safety does not lower memory use: building every doc before yielding still peaks. Stream pages instead.

Sync and rebuild index:

- **Sync** (scheduled syncs, the row's **More actions → Sync now**, `gety connector poll <id>`) is incremental and starts from `lastState`.
- **Rebuild index** (the row's **More actions → Rebuild index**, `gety connector rebuild <id>`, and Gety's global **Clean up & rebuild**) deletes every document of the connector, clears `lastState` and the last sync time, then syncs right away with `lastState === null`. The request is persisted: if Gety quits, the connector is disabled, or it fails to start before the delete completes, the rebuild finishes on the next start. A disabled connector refuses it with `not_running`.
- A rebuild does not clear `this.dataDir`, and neither does switching versions; only uninstalling (or reinstalling) does.

What the connector must tolerate:

- `lastState === null` can happen at any time and means only that there is no committed checkpoint: the index may be empty (first sync, rebuild, a version switch that resyncs) or partly filled (an earlier poll that failed before committing). Do a complete, replay-safe enumeration of every in-scope document. Do not skip items because a cache in `this.dataDir` says they were already sent; when `lastState` is `null`, discard such caches, or tie them to a value stored in the state so a cleared state invalidates them.
- A failed poll is replayed from the previous state (see State guidance).
- When a change alters ids, doc fields, the content mapping, or what the cursor skips, an incremental sync leaves stale documents behind. For an npm release, bump `sync_version` so the upgrade resyncs on its own; otherwise tell users to run **Rebuild index** (`gety connector rebuild <id>`) after updating.

## Remote Files

Links resolve only from the manifest `doc_link` and each doc's fields; a connector cannot resolve files or links dynamically at preview time.

For remote files (Dropbox/Drive-like):

- Put any text the connector already has into `content`, and a stable web URL into `metadata.url`.
- Represent images and PDFs as extracted/OCR text only if the connector already has that text.
- Do not yield temporary local paths, or files the connector downloaded into its own folders, as `file` links; Gety refuses to open them.

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

Check whether the sample's runner enforces the sandbox before trusting it: if `deno task runner` runs `deno run -A` and the runner only injects `config`, `lastState`, and `signal`, it does not apply `permissions` and leaves `dataDir`/`homeDir` empty. Then a missing permission only shows up after installing in Gety: poll the connector; `gety connector show <id>` reports `retrying` or `stopped` with a short code (`retrying`, `runtime_error`), and the connector's row in Gety's **Settings → Connectors** shows the full message, e.g. `permission denied: net access to "api.example.com:443" is not granted by the connector's manifest`.

## Gety Installation and Lifecycle

You can manage custom connectors either via Gety CLI (`gety connector`) or through the desktop UI. Every install asks the user to consent to the connector's permissions.

### CLI Workflow (Recommended for Agents)

1. **Dry-run validation**:
   ```bash
   gety connector install /path/to/connector --dry-run --secret api_key
   ```
   Validates the manifest and the merged config with zero side effects, and prints the permission summary (risks first, lines starting with `!`) and the permission fingerprint. With `--json` the preview carries the full `permissions` report.

2. **Install**:
   ```bash
   gety connector install /path/to/connector --yes --secret api_key
   gety connector install /path/to/connector --yes --config config.json --set 'workspace="acme"'
   ```
   Config precedence: manifest defaults < `--config <file|->` < `--set key=<json>` < `--secret`/`--secret-file`. Consent: `--yes` installs without asking; on a terminal the CLI asks `Install "<name>" with these permissions? [y/N]`; otherwise it fails with `consent_required`. Reading config or a secret from stdin requires `--yes` (else `stdin_conflict`), so review with `--dry-run` first. If the manifest's permissions change between preview and commit the install fails with `permissions_changed`; run it again. Gety registers the folder (no copy): it must stay available. The folder must not be inside Gety's connector data directory.

3. **Inspect status**:
   ```bash
   gety connector list --kind custom
   gety connector show <connector_id> --json
   ```
   `show` returns the status (`state` plus a short `detail` code such as `runtime_error`; the full error text is shown in the app's connector row), the redacted config, the permissions granted now (`config_field` entries resolved against the saved config), and the code origin.

4. **Trigger sync or rebuild**:
   ```bash
   gety connector poll <connector_id>
   ```
   Returns `accepted` without waiting; check `show` and `gety search` afterwards.

   ```bash
   gety connector rebuild <connector_id>
   ```
   Deletes the connector's documents and syncs again from `lastState === null`, without asking for confirmation; also returns `accepted` immediately. Use it after changes that alter ids, fields, or what the state cursor skips. Needs a Gety CLI protocol v7 build.

5. **Reconfigure**:
   ```bash
   gety connector configure <connector_id> --schedule daily --at 09:00
   gety connector configure <connector_id> --set 'workspace="other"'
   ```

6. **Restart after code changes**:
   After rebuilding (`deno task build`), reload the entry:
   ```bash
   gety connector restart <connector_id>
   ```
   `restart` does not re-read `manifest.json`. After manifest changes, `gety connector remove <connector_id>` and install again.

7. **Verify results**:
   ```bash
   gety search <query> -c "<connector_name>"
   ```

### UI Workflow

1. Open **Gety → Settings → Connectors**, click **Install from local folder** under **Install Connector**, and select the connector folder.
2. Gety shows the permission review first: the connector card, then the grants grouped as **Network**, **File System Read**, **Environment Variables**, and **Run Programs**. High-risk and full-access connectors need an extra confirmation in the app.
3. Fill in the configuration fields; Gety installs and starts indexing in the background. Cancelling at any step installs nothing.
4. Each loaded connector's row in **Installed Connectors** has a **Permissions** button that shows the permissions in effect, **Configure**, and a **More actions** menu: **Sync now** (enabled connectors that are running and not already syncing), **Restart** (enabled connectors only), **Manage versions** (npm installs only), **Rebuild index** (enabled connectors only), and **Delete**. A connector Gety can't load shows a direct **Delete** button instead.
5. After code edits, run `deno task build` and choose **More actions → Restart**; after manifest edits, choose **More actions → Delete** (or the direct **Delete** button of a connector Gety can't load) and install the connector again.

## Publishing

Users install a published connector with the **Install from remote** card in **Settings → Connectors**, or with `gety connector install --github …` / `--npm …`. In both cases Gety:

- never builds from source: it installs the packaged files and keeps its own copy of the code (deleted on `remove`);
- shows the permissions from a small metadata download (the `manifest.json` asset, or the registry metadata of that npm version) before the user confirms, then fetches that metadata again, downloads the package, and checks it: the `id` and `version` must match what the user reviewed, the permissions (with the config fields they reference) must have the same fingerprint as in the review (else `permissions_changed`), and the package's manifest must match the metadata fetched at commit time (GitHub: byte-for-byte; npm: the `gety` object equal as JSON). Other manifest fields are not pinned to the preview;
- records the SHA-256 of the package; `gety connector show` reports `origin` (`github` / `npm`), `tag` (GitHub), and `digest`.

Rules shared by both sources:

- The package contains the built entry and every file the connector needs at run time (icon, source map, data files). Build output goes into the package; it does not have to be committed.
- Limits: at most 100 MiB packed, 20,000 files and 500 MiB unpacked; no symbolic or hard links.
- Public sources only; Gety sends no tokens.
- One install per manifest id. Prefix `id` with your own name (`acme-linear`, not `linear`): a connector with the same id cannot be installed next to yours. Remove a local-folder install of the same id before installing the published one (otherwise `already_installed`); switching source means remove and install again.
- Raising `min_gety_app_version` does not strand users: older Gety reports `version_incompatible`, the user can still install an older version by naming it, and update checks skip versions the user's Gety cannot run.
- Publishing to both: keep `manifest.json` as the single source and generate the `gety` object in `package.json` from it in the packaging step.

### GitHub Releases

- One Release per version. The tag is the manifest `version`, optionally prefixed with `v` (`1.2.0` or `v1.2.0`).
- Two assets:
  - `manifest.json`, byte-for-byte identical to the `manifest.json` inside the zip (write both from the same file; even whitespace differences fail the install).
  - `<id>-<version>.zip`, whose root is the connector directory (`manifest.json`, the entry, runtime files) without a wrapping folder; entries stored or deflate-compressed.
- Gety installs the Release GitHub marks as Latest by default. Pre-releases install only via their Release link (`https://github.com/<owner>/<repo>/releases/tag/<tag>`) or `--version <tag>`.
- Gety only downloads from github.com (no GitHub API); private repositories and monorepo subdirectories are not supported.
- Turn on immutable releases in the repository settings so a published tag and its assets cannot change.
- The sample implements this: `deno task build && deno task package` writes both assets to `release/` and checks them (`--tag <tag>` also checks the tag; `--include <path>` adds files beyond `manifest.json`, the entry with its `.map`, and the icon). Its `.github/workflows/release.yml` publishes a Release when a `v*` tag is pushed (pre-release when the version has a pre-release part); other repositories can call that workflow, as the sample README shows.

```bash
gety connector install --github acme/gety-linear --dry-run
gety connector install --github acme/gety-linear --version v1.2.0-beta.1 --yes
gety connector install --github https://github.com/acme/gety-linear/releases/tag/v1.2.0
```

Errors: `invalid_github_input`, `release_not_found` (no public repo, no Release, or no `manifest.json` asset), `source_unavailable`, `version_incompatible`, `invalid_package` (the Release breaks the convention).

### npm

The package's `package.json` carries the complete manifest in a `gety` object:

```json
{
  "name": "@acme/gety-linear",
  "version": "1.2.0",
  "files": ["dist/main.js", "dist/main.js.map", "icon.svg", "CHANGELOG.md"],
  "gety": {
    "id": "acme-linear",
    "name": "Acme Linear",
    "version": "1.2.0",
    "min_gety_app_version": "0.6.6",
    "entry": "dist/main.js",
    "icon": "icon.svg",
    "capabilities": ["data-source"],
    "permissions": { "net": ["api.linear.app"] }
  }
}
```

- `gety` is the whole manifest; a `manifest.json` in the package is not read. Paths in it (`entry`, `icon`) are relative to the package root.
- `gety.version` must equal the package `version` (SemVer, including pre-release and build metadata), otherwise `invalid_package`.
- `files` must include the entry and every runtime file. List `CHANGELOG.md` too: npm does not pack it on its own once `files` is set, and Gety shows its sections as release notes (see Versions and Updates). Check with `npm pack --dry-run` before publishing.
- Gety installs the version the `latest` dist-tag points to, or an exact version the user names (`@acme/gety-linear@1.3.0-beta.1`, `--version 1.3.0-beta.1`, or `https://www.npmjs.com/package/@acme/gety-linear/v/1.3.0-beta.1`). Other dist-tags and ranges are not supported, so publish pre-releases under a non-`latest` tag (`npm publish --tag next`) to keep `latest` stable.
- npm never lets a version be republished, so a version's permissions are fixed once published; changing them means publishing a new version.
- The registry must list a `sha512` integrity for the tarball, and the tarball may contain only regular files and directories under `package/`.
- When the system region is mainland China, Gety reads npmmirror first and falls back to registry.npmjs.org when the mirror lacks the package or version or is unreachable. The mirror usually syncs within minutes; until then an install without a version can resolve to the mirror's older `latest`.

```bash
gety connector install --npm @acme/gety-linear --dry-run
gety connector install --npm @acme/gety-linear@1.2.0 --yes
```

Errors: `invalid_npm_input`, `package_not_found` (no such public package or version on any registry tried), plus `source_unavailable`, `version_incompatible`, `invalid_package` as for GitHub.

### Versions and Updates

Only npm installs have version management. GitHub and local-folder installs get no update checks, and `gety connector update` refuses them with `capability_not_supported`; moving them to another version means removing the connector and installing it again.

- Gety checks every npm-installed connector for a newer version each time the user opens **Settings → Connectors**, on **Check for updates**, and with `gety connector outdated`. It never updates on its own.
- The update it offers is the highest version that is newer than the installed one, not a pre-release, not deprecated, and runs on the user's Gety (its `min_gety_app_version` is not newer than the app). The `latest` dist-tag plays no part. The row's **More actions** button then shows a dot and the tooltip `Update <version> available`.
- Users can switch to any published version, newer or older, in **More actions → Manage versions** (**Switch to this version**) or with `gety connector update <id> [--version <version>]`. The list labels versions `Pre-release`, `Deprecated`, or `Needs Gety <version>`; pre-releases and deprecated versions can still be chosen explicitly, versions that need a newer Gety cannot.
- Release notes: Gety shows the section of the package's root `CHANGELOG.md` for the selected version, split by version headings (`## 1.2.0`, `## v1.2.0`, and `## [1.2.0] - 2026-07-14` all work; use one heading level for every version). A section ends at the next heading of the same or a higher level; a non-version heading such as `## Unreleased` is not shown, nor is its content, and a `---` line after a blank line counts as such a heading. An upgrade reads the notes from the new version's package, anything else from the installed copy, so keep the sections of older versions. changesets writes this format. Only the first 1 MiB is shown.
- `sync_version` (manifest, whole number, 1 when absent) is the format version of what the connector has synced: its state, its document ids, and the content of its documents. Bump it in a release that cannot continue from what an older release synced (the state changes shape, document ids change); upgrading to it then deletes the connector's documents, clears its state, and syncs from scratch. Changing `schema.extra_gety_indexed_fields` does the same without a bump, and every downgrade always does it. A manifest that sets `sync_version` needs `min_gety_app_version` `0.6.6` or later, like `permissions`.
- What carries over: `this.dataDir` and `localStorage` are kept across versions, and the state is cleared only when the switch resyncs. Settings carry over field by field: a field that keeps its `id` and type keeps the user's value (secrets included); a field whose type changes is reset to its new default, a removed field is dropped, and a new field starts at its default. When a required field ends up empty or a value is not among the new options, the user fills it in before switching. A switch asks the user to review only the permissions the new version adds.
- Withdraw a broken version with `npm deprecate <package>@<version> "<message>"`: a deprecated version is never offered as an update; users still see it, marked as deprecated with your message, and can choose it explicitly. Pointing `latest` back at an older version does not withdraw a newer one.
- Test an upgrade as users get it: install the previous published version (`gety connector install --npm <package>@<old> --yes`), let it sync, then run `gety connector update <id> --version <new> --dry-run` and check what it reports (resync, settings to fill in, new permissions) before switching.

```bash
gety connector outdated
gety connector update <connector_id> --dry-run
gety connector update <connector_id> --version 1.1.0 --yes
```
