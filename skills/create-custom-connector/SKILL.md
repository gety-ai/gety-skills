---
name: create-custom-connector
description: Turn any data source — an API, SaaS app, database, or local files — into searchable Gety documents with a custom connector built from the gety-sample-connector template. Use when the user wants to create, adapt, or review a Gety custom connector — manifest.json, config fields, doc_link, the @gety-ai/connector-sdk poll()/onLoad lifecycle, incremental sync, and validating with the local runner and gety search.
---

# Create Custom Connector

## Prerequisites

- `git` — to clone the sample repo.
- `deno` — the sample builds, tests, and runs entirely via `deno task` (no Node needed). Often not pre-installed; check `deno --version` and point the user to https://deno.com if missing.
- Gety desktop app installed with CLI integration enabled — not required to build or test the connector, only to install it and verify with `gety search` / `gety doc`.

Check these early; if `git` or `deno` is missing, ask the user to install before coding.

## Workflow

1. Inspect the connector repo before editing.
   - New connectors must start from `https://github.com/gety-ai/gety-sample-connector`; do not hand-create a connector skeleton.
   - If there is no sample-derived repo, clone the sample repo into the user's requested folder, then adapt it.
   - If the current repo is not a clone/fork of `gety-sample-connector`, ask the user to switch to or clone a sample-based repo before coding.
   - Read `references/gety-custom-connector-contract.md` before changing manifest shape, SDK lifecycle, `doc_link`, metadata, state, file/download behavior, or validation commands.

2. Clarify source semantics before coding.
   - Confirm auth method, required config fields, document identity, title/content mapping, source updated/created timestamps, deletion handling, and link behavior.
   - Explain the proposed content extraction plan and ask the user which source fields/content should become Gety `title`, `content`, metadata, and links before implementing.
   - Stop and discuss if the source needs OAuth browser login, ephemeral signed URLs, lazy remote file preview/download, binary extraction, or mixed URL/file links. The current skill scope does not cover those capabilities end to end.

3. Design the Gety document contract.
   - Choose stable `id` values and a connector-owned `doc_type` such as `linear:issue` or `github:issue`.
   - Set `content_format: "markdown"` only when `content` is genuinely markdown (e.g. an issue body); keep plain text — titles, URLs, line-separated fields — as `"plaintext"` (the default). Do not misuse `doc_type` to control markdown preview.
   - Put stable browser URLs in `metadata.url` and use manifest `doc_link: { "kind": "url", "field": "url" }`.
   - Use `doc_updated_at` for the source document update time and `metadata.created_at` for source creation time when available.

4. Implement `src/index.ts`.
   - Extend `Connector<ManifestConfig, State>` and implement `async *poll()`. `ManifestConfig` is generated from the manifest's `config.fields` into `src/gen/manifest.d.ts` by `deno task generate` (also run by `check` and `build`), so add config fields to `manifest.json` and regenerate before reading them via `this.config`.
   - Use `this.config`, `this.lastState`, and `this.signal`; `onLoad` is optional and can be async.
   - Stream pages and `yield` real update batches as they are fetched. Persist a state cursor only after the batch it represents has been yielded.
   - Do not rely on class fields for cross-poll cache; each poll is a fresh runtime process.

5. Update `manifest.json`.
   - Keep `entry` pointed at the committed build output, usually `dist/main.js`.
   - Use lowercase/snake_case author-facing fields and valid config field types.
   - Do not add unsupported manifest fields such as `permissions`, `mode`, dynamic per-doc hooks, or per-doc `doc_link` (it is declared once for the whole connector type).

6. Validate locally.
   - Run `deno task verify` when available; otherwise run the closest equivalents: generate, format/check, lint, tests, and build.
   - If config requires secrets, tell the user to put them in `.env` themselves, using keys such as `API_KEY` or `GETY_CONFIG_API_KEY`; do not ask them to paste secrets into chat.
   - Use `deno task runner -- --reset-state` or repeated runner polls to verify emitted docs, deletes, state, and incremental behavior before asking the user to install in Gety.
   - After the runner finishes, tell the user the concrete output directory, usually `dev/runs/<timestamp>/`, and mention the key files to inspect.
   - After user installs or restarts in Gety, verify with `gety search <query>` and inspect connector status if available.

7. Add an icon.
   - Once indexing is verified, give the connector a recognizable icon so it stands out in Gety's connector list and the install dialog.
   - Drop an image (PNG or SVG) in the connector root — e.g. `icon.png` — and set manifest `"icon": "icon.png"` (a relative path inside the connector root). Reuse the source's official logo when one exists.

## Anti-Patterns

- Do not fetch every remote item into memory and yield once at the end.
- Do not write ephemeral signed URLs or temporary local cache paths into `doc_link`.
- Do not store large blobs in `metadata`; put searchable/renderable text in `content`.
- Do not treat `metadata.updated_at` as a fixed contract; use `doc_updated_at`.
- Do not read `this.config`, `this.lastState`, or `this.signal` from the class constructor — the host injects them after construction, so in the constructor they are still defaults (`config` `{}`, `lastState` `null`, a `signal` that never aborts). Read them in `onLoad` or `poll`.
- Do not edit generated `dist/main.js` by hand; edit `src/index.ts` and rebuild.

## Reference

- Load `references/gety-custom-connector-contract.md` for manifest, SDK, document, link, state, runner, and Gety install/restart details.
