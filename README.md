# GitCheck — gitcheck.tech

See every license in your dependency chain — and where they clash.

Paste a public **GitHub, GitLab, or Bitbucket** repo and GitCheck:

1. Detects the project's **own license** (SPDX)
2. Parses dependency manifests and resolves **direct + transitive** dependencies
3. Looks up the **license of every dependency** in the chain
4. Checks compatibility **dep ↔ project license** and **dep ↔ dep**
5. **Maps the chain** visually and **highlights risks** with plain-English obligations and suggested actions
6. Exports a Markdown + JSON report

Built on [bolt.new](https://bolt.new) in **Forge mode on GLM 5.3 Flash** during the free build weekend, Sat Oct 17 12:00am PT – Sun Oct 18 11:59pm PT 2026. License/dependency data via the free [deps.dev](https://deps.dev) API plus each host's public API.

> GitCheck gives general information based on SPDX identifiers and widely used compatibility guidance. It is not legal advice.

## Build kit

The full launch kit — product spec, SPDX compatibility rules engine, data-source plan, and the starter prompts (one batch per 6-hour credit window) — lives in [`docs/bolt-build-kit.md`](docs/bolt-build-kit.md).

## Status

Pre-launch: kit under Copilot review (see Issues). Bolt project links to this repo via Bolt's GitHub integration at launch; live app will be at <https://gitcheck.tech>.
