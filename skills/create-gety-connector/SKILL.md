---
name: create-gety-connector
description: Turn any data source — an API, SaaS app, database, or local files — into searchable Gety documents with a custom connector built from the gety-sample-connector template. Use when the user wants to create, adapt, or review a Gety custom connector — manifest.json, sandbox permissions, config fields, doc_link, the @gety-ai/connector-sdk poll()/onLoad lifecycle, incremental sync, and validating with the local runner, gety connector install, and gety search.
---

# Create Gety Connector

## Prerequisites

- `git` — to clone the sample repo.
- `deno` — the sample builds, tests, and runs entirely via `deno task` (no Node needed). Often not pre-installed; check `deno --version` and point the user to https://deno.com if missing.
- Gety desktop app installed with CLI integration enabled — not required to build or test the connector, only to install it and verify with `gety connector` / `gety search` / `gety doc`. Sandboxed connectors (manifest `permissions`) need Gety 0.7.0 or later.

Check these early; if `git` or `deno` is missing, ask the user to install before coding.

## Workflow

1. Inspect the connector repo before editing.
   - New connectors must start from `https://github.com/gety-ai/gety-sample-connector`; do not hand-create a connector skeleton.
   - If there is no sample-derived repo, clone the sample repo into the user's requested folder, then adapt it.
   - If the current repo is not a clone/fork of `gety-sample-connector`, ask the user to switch to or clone a sample-based repo before coding.
   - Read `references/gety-custom-connector-contract.md` before changing manifest shape, permissions, SDK lifecycle, `doc_link`, metadata, state, file/download behavior, or validation commands.

2. Clarify source semantics before coding.
   - Confirm auth method, required config fields, document identity, title/content mapping, source updated/created timestamps, deletion handling, and link behavior.
   - List every host the connector will contact (API, redirects, CDN/file hosts), every local folder it reads, and every environment variable or program it needs; these become `permissions`.
   - Explain the proposed content extraction plan and ask the user which source fields/content should become Gety `title`, `content`, metadata, and links before implementing.
   - Stop and discuss if the source needs OAuth browser login, ephemeral signed URLs, lazy remote file preview/download, binary extraction, or mixed URL/file links. The current skill scope does not cover those capabilities end to end.

3. Design the Gety document contract.
   - Choose stable `id` values and a connector-owned `doc_type` such as `linear:issue` or `github:issue`.
   - Set `content_format: "markdown"` only when `content` is genuinely markdown (e.g. an issue body); keep plain text — titles, URLs, line-separated fields — as `"plaintext"` (the default). Do not misuse `doc_type` to control markdown preview.
   - Put stable browser URLs in `metadata.url` and use manifest `doc_link: { "kind": "url", "field": "url" }`. File links must point at the user's own files; Gety refuses to open links into any connector folder (data, temp, code) or to missing files.
   - Use `doc_updated_at` for the source document update time and `metadata.created_at` for source creation time when available.

4. Implement `src/index.ts`.
   - Extend `Connector<ManifestConfig, State>` and implement `async *poll()`. `ManifestConfig` is generated from the manifest's `config.fields` into `src/gen/manifest.d.ts` by `deno task generate` (also run by `check` and `build`), so add config fields to `manifest.json` and regenerate before reading them via `this.config`.
   - Use `this.config`, `this.lastState`, and `this.signal`; `onLoad` is optional and can be async.
   - Stream pages and `yield` real update batches as they are fetched. Gety commits the state only after the whole poll succeeds, so every batch must be safe to replay.
   - Do not rely on class fields for cross-poll cache; each poll is a fresh runtime process. Persist caches in `this.dataDir`; use the per-sync temp folder (`Deno.makeTempDir()`) for scratch files.
   - Resolve `~` paths with `this.homeDir` / `this.expandHome()`, never `HOME` from the environment.
   - Let the build bundle every dependency into the entry; only `@gety-ai/connector-sdk` (Gety supplies its own SDK at run time) and `node:*` built-ins stay external. Sandboxed connectors cannot import `npm:`, `jsr:`, or `https:` modules at run time.

5. Update `manifest.json`.
   - Keep `entry` pointed at the build output, usually `dist/main.js`.
   - `capabilities` is required and non-empty; a polling/indexing data source must declare `["data-source"]`.
   - Declare minimal `permissions` (`net`, `read`, `env`, `run`, `{ "config_field": ... }`) and set `min_gety_app_version` to at least `0.7.0`. Omitting `permissions` runs the connector without a sandbox behind a red full-access warning; use `{}` if it needs nothing beyond base access.
   - Avoid high-risk grants (`run`, `"*"` or `*.com`-style wildcards in `net`, `"~"` in `read`); the desktop app makes users confirm them twice and the CLI flags them with `!` lines.
   - `schedule` supports `{ "strategy": "manual" }`, `{ "strategy": "interval", "interval_seconds": <n> }`, or `{ "strategy": "daily", "time": "HH:mm" }`; omit it for Gety's default.
   - Use lowercase/snake_case author-facing fields and valid config field types.
   - Do not add unsupported manifest fields such as `mode`, dynamic per-doc hooks, or per-doc `doc_link` (it is declared once for the whole connector type).

6. Validate locally and install in Gety.
   - Run `deno task verify` when available; otherwise run the closest equivalents: generate, format/check, lint, tests, and build.
   - If config requires secrets, tell the user to put them in `.env` themselves, using keys such as `API_KEY` or `GETY_CONFIG_API_KEY`; do not ask them to paste secrets into chat.
   - Use `deno task runner -- --reset-state` or repeated runner polls to verify emitted docs, deletes, state, and incremental behavior.
   - After the runner finishes, report the output directory (`dev/runs/<timestamp>/`) and inspect key files.
   - The sample runner may not apply the sandbox; a sync inside Gety is the real permission test.
   - Preview: `gety connector install <dir> --dry-run [--config <file>] [--set key=<json>] [--secret <field>]` — show the permission summary and its `!` risk lines to the user. Grants from `config_field` entries resolve against the config you pass.
   - Install: `gety connector install <dir> --yes [--config <file>] [--set key=<json>] [--secret <field>]` only after the user approved that summary; the CLI asks at most once and `--yes` skips the prompt, whatever the risk tier (or guide the user through Settings → Connectors → Add connector → Install from local folder, where Gety shows the permission review before the config form).
   - Check status: `gety connector show <connector_id>` or `gety connector list --kind custom`.
   - Trigger sync: `gety connector poll <connector_id>`; a permission failure appears as `permission denied: ...` on the connector's row in Gety settings.
   - After code changes, rebuild (`deno task build`) and run `gety connector restart <connector_id>`. After manifest changes (including permissions), `gety connector remove <connector_id>` and install again.
   - Verify search: `gety search <query> -c "<connector_name>"`.

7. Add an icon.
   - Once indexing is verified, give the connector a recognizable icon so it stands out in Gety's connector list and the install dialog.
   - Drop an image (PNG or SVG) in the connector root — e.g. `icon.png` — and set manifest `"icon": "icon.png"` (a relative path inside the connector root). Reuse the source's official logo when one exists.

## Anti-Patterns

- Do not fetch every remote item into memory and yield once at the end.
- Do not write ephemeral signed URLs, temporary local cache paths, or files inside the connector's own folders into `doc_link`.
- Do not store large blobs in `metadata`; put searchable/renderable text in `content`.
- Do not treat `metadata.updated_at` as a fixed contract; use `doc_updated_at`.
- Do not read `this.config`, `this.lastState`, `this.signal`, `this.dataDir`, or `this.homeDir` from the class constructor — the host injects them after construction, so in the constructor they are still defaults (`config` `{}`, `lastState` `null`, `dataDir`/`homeDir` `""`, a `signal` that never aborts). Read them in `onLoad` or `poll`.
- Do not edit generated `dist/main.js` by hand; edit `src/index.ts` and rebuild.
- Do not request `"*"`, `*.com`, `"~"`, or `run` just to make a permission error go away; add the exact host or folder that failed.
- Do not pass password fields with `--set`; Gety rejects them. Use `--secret` or `--secret-file`.

## Reference

- Load `references/gety-custom-connector-contract.md` for manifest, permissions and sandbox, SDK, document, link, state, runner, and Gety install/restart details.
