# GitCheck — Bolt Forge Build Weekend Kit

**Name:** GitCheck — **domain: gitcheck.tech** (Porkbun, in Demo's account, registered 2026-10-16 ~11:09pm CT, PENDINGNEWREG at check, auto-renew + WHOIS privacy on, expires 2027-10-17)
**GitHub:** repo **Mike-Demo/gitcheck.tech** — the Bolt project MUST be linked to GitHub (Bolt GitHub integration → this repo) at launch, not left Bolt-only.
**Domain link:** at publish (Window 8, not launch), connect gitcheck.tech in Bolt, then set Porkbun DNS like privatebrain.app: root ALIAS → site-dns.bolt.host + www CNAME → site-dns.bolt.host, verify via DNS-over-HTTPS (direct dig/UDP is blocked on the VM).
**Tagline:** See every license in your dependency chain — and where they clash.
**Builder:** bolt.new → **Forge agent → GLM 5.3 Flash** (default in Forge; do NOT switch to Kimi/DeepSeek, they burn allocation faster)
**Window:** Sat Oct 17, 2026 12:00am PT → Sun Oct 18, 2026 11:59pm PT
- In Demo's time: Sat 2:00am CT → Mon 1:59am CT
- **Credit resets every 6 hours = 8 windows:** Sat 12–6a, 6a–12p, 12–6p, 6p–12a PT / Sun same. Plan one prompt-batch per window; never burn a fresh window on polish before core scanning works.
**Verified grounding:** Forge is Bolt's open-model mode with GLM 5.3 Flash as default. License/dependency data source of choice is the free **deps.dev API** (Google Open Source Insights) — it returns licenses + resolved dependency graphs for npm, PyPI, Go, Maven, Cargo, NuGet, RubyGems and indexes projects from GitHub, GitLab, and Bitbucket. Pair it with each host's own repo API for the project's own license + manifest files.

> Disclaimer to ship in-app, verbatim in footer + report: "GitCheck gives general information based on SPDX identifiers and widely used compatibility guidance. It is not legal advice."

---

## 1. What it does (MVP, end-to-end)

1. User pastes a **public** repo URL: `github.com/owner/repo`, `gitlab.com/owner/repo` (incl. subgroups), or `bitbucket.org/workspace/repo`.
2. App detects host, owner, repo, default branch.
3. **Own license:** fetch LICENSE via host API (GitHub License API / repo tree; GitLab repository files; Bitbucket src) and map to an SPDX ID. If none: flag "No license found — all rights reserved by default" as a risk.
4. **Manifests:** look for package.json, requirements.txt / pyproject.toml / Pipfile, go.mod, Cargo.toml, pom.xml, build.gradle, Gemfile / *.gemspec, composer.json, *.csproj / packages.config. Parse direct dependencies + versions.
5. **License lookup + chain:** for each direct dep, query deps.dev `GetVersion` / `GetDependencies` for its license(s) and its transitive deps (cap depth 3, cap 250 nodes, dedupe). Fall back to registry metadata (npm registry, PyPI JSON) when deps.dev has no entry. Mark license `Unknown / NOASSERTION` when no source gives one — never guess.
6. **Compatibility checks (two directions):**
   - Dep license → project license: can this dep be combined under the project's own license?
   - Dep ↔ dep: flag known pairwise clashes in the same combined work (e.g. Apache-2.0 vs GPL-2.0-only).
7. **Chain map:** interactive tree/graph — project at root, direct deps, transitive deps nested; nodes colored by risk; click a node for license, obligations, and why it was flagged.
8. **Risks panel + report:** sorted High → Unknown → Medium → Low, each with dep, license, chain path (project → … → dep), obligation, and suggested action. Export **Markdown + JSON** report; copy-link to a scan (client-side state in URL/local storage is fine for v1).

## 2. Compatibility rules engine (encode exactly this; SPDX IDs)

Families:
- **Permissive:** MIT, MIT-0, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, Unlicense, CC0-1.0, 0BSD, Zlib
- **Weak / file-or-library copyleft:** LGPL-2.1-only/or-later, LGPL-3.0-only/or-later, MPL-2.0, EPL-1.0, EPL-2.0, EUPL-1.2, CDDL-1.0
- **Strong copyleft:** GPL-2.0-only/or-later, GPL-3.0-only/or-later
- **Network copyleft:** AGPL-3.0-only/or-later
- **Source-available / non-OSI (flag, don't call open source):** BSL-1.0, SSPL-1.0, Elastic-2.0, Commons-Clause variants, any "NonCommercial" (CC-BY-NC-*)
- **Unknown:** NOASSERTION, LicenseRef-*, empty

Rules (dep license checked against project license):
- Permissive dep → **Low** in any project license (keep notices). Exception: Apache-2.0 dep in GPL-2.0-only project → **High** (patent terms incompatible with GPLv2; compatible with GPLv3).
- Weak copyleft dep → **Medium** generally: obligations attach to that library/file (LGPL: dynamic linking / relinkability; MPL-2.0: disclose source of covered files). **High** if the app claims a single permissive license with no notices, or if EPL/CDDL combined into a GPL project (known incompatibility → mark Conditional/High with explanation).
- GPL dep → **High** if project license is permissive, weak copyleft, proprietary/"none", or the other GPL major version (GPL-2.0-only vs GPL-3.0 project and vice-versa; "-or-later" resolves it → downgrade to Medium with note). **Low** if project is same GPL family/version.
- AGPL dep → **High** if project is anything other than AGPL-3.0 (network-use source disclosure), **Low** if project is AGPL-3.0. Always surface the network-use obligation explicitly.
- Source-available / non-commercial dep → **High** for any project presented as open source or used commercially; label "Not an OSI open-source license."
- Unknown dep license → **Unknown risk** (separate bucket, never Low): "Cannot verify — treat as all-rights-reserved until clarified."
- Dual-licensed dep (`(MIT OR Apache-2.0)` etc.) → evaluate the **most permissive option** and note the choice; `(MIT AND GPL-3.0)` → evaluate as GPL.
- Project has **no license** + any copyleft dep → **High** overall verdict.

Overall verdict: **Compatible / Compatible with obligations / Conflicts found / Cannot verify** = worst node wins (any High → Conflicts; else any Unknown → Cannot verify; else any Medium → With obligations; else Compatible).

Each risk must show: rule fired, chain path, obligation in plain English, suggested action (e.g. "Relicense project to GPL-3.0, replace dep, or get legal review").

## 3. Data sources (all public, no keys for v1)

- GitHub: `GET api.github.com/repos/{o}/{r}`, `/license`, `/contents/{path}`, `/git/trees/{branch}?recursive=1` — 60 req/hr unauthenticated; show rate-limit state honestly; optional user-supplied token field (stored only in memory).
- GitLab: `GET gitlab.com/api/v4/projects/{urlencoded path}`, `/repository/tree`, `/repository/files/{path}/raw`.
- Bitbucket: `GET api.bitbucket.org/2.0/repositories/{ws}/{repo}`, `/src/{branch}/{path}`.
- deps.dev: `GET api.deps.dev/v3/systems/{NPM|PYPI|GO|MAVEN|CARGO|NUGET|RUBYGEMS}/packages/{name}/versions/{version}` (+ `:dependencies`). URL-encode Maven `group:artifact` and scoped npm names.
- Fallbacks: `registry.npmjs.org/{name}/{version}` (license field), `pypi.org/pypi/{name}/{version}/json` (info.license / classifiers).
- CORS note: all of the above are called client-side; if a host blocks a call, surface the exact failure and fall back to Demo Mode data — do not fake a live result.

## 4. Screens

1. **Home / Scan:** big URL input, host auto-detect chip, "Scan repo" button, 3 one-click sample repos (one per host), recent scans.
2. **Scanning state:** stepwise progress (Own license → Manifests → Resolving chain → Checking compatibility) — honest steps tied to real fetches.
3. **Results:** header verdict banner + counts; left = chain map (tree/graph toggle); right = risks list; bottom = full dependency table (name, version, license SPDX, family, direct/transitive, risk, source link); export buttons.
4. **License detail drawer:** family, obligations, conditions, why flagged.
5. **Demo Mode:** bundled sample scans (a GPL conflict, an AGPL-in-SaaS, a clean MIT stack) so the app demos perfectly with zero network.

Design: clean dev-tool look, dark header, risk colors red/amber/green/slate(unknown) used consistently everywhere, monospace for SPDX IDs and repo paths, no gradients-and-blobs filler. Accessibility: risk never conveyed by color alone (icon + label), keyboard-navigable tree.

## 5. Master prompt for Bolt — paste as Prompt 1 (Window 1, Sat 12:00–6:00am PT)

```
Build "GitCheck", a web app that scans a public code repository for open-source license risk.

User flow: paste a public GitHub, GitLab, or Bitbucket repo URL → the app detects the project's own license, parses its dependency manifests, resolves direct and transitive dependencies and each dependency's license, checks license compatibility (a) between each dependency and the project's own license and (b) between dependencies themselves, maps the whole chain visually, and highlights risks with plain-English explanations and suggested actions.

Build in this order, working end-to-end:
1. Home screen with repo URL input (auto-detect GitHub/GitLab/Bitbucket), 3 sample-repo buttons, and a Demo Mode with bundled sample data: one clean MIT project, one project with a GPL-3.0 dependency under an MIT project license (conflict), one with an AGPL-3.0 dependency (network-use risk). Demo Mode must work fully offline.
2. Results screen: verdict banner (Compatible / Compatible with obligations / Conflicts found / Cannot verify), counts by risk, an interactive dependency chain tree (project at root, direct deps, transitive nested, nodes colored red/amber/green/slate by risk with icon+label, click for detail drawer), a risks panel sorted high-first showing dependency, SPDX license, chain path, obligation and suggested action, and a full dependency table. Export the report as Markdown and JSON.
3. A compatibility rules module using SPDX IDs with these families: Permissive (MIT, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, Unlicense, CC0-1.0, Zlib), Weak copyleft (LGPL-2.1, LGPL-3.0, MPL-2.0, EPL-1.0, EPL-2.0, CDDL-1.0), Strong copyleft (GPL-2.0, GPL-3.0), Network copyleft (AGPL-3.0), Source-available/non-OSI (BSL-1.0, SSPL-1.0, Elastic-2.0, NonCommercial), Unknown (NOASSERTION/empty). Rules: permissive deps are Low except Apache-2.0 in a GPL-2.0-only project = High; weak copyleft = Medium with its obligation (LGPL linking, MPL file disclosure); GPL dep = High unless project is same GPL family (GPL-2.0-only vs GPL-3.0 project = High, "-or-later" = Medium with note); AGPL dep = High unless project is AGPL-3.0, always explain network-use disclosure; source-available/non-commercial = High and labeled "Not an OSI open-source license"; unknown license = Unknown risk, never Low. Dual license with OR = evaluate most permissive option and note it; AND = evaluate as the stricter one. No project license + any copyleft dep = High. Worst node sets the overall verdict.
4. Footer and report must state: "GitCheck gives general information based on SPDX identifiers and widely used compatibility guidance. It is not legal advice."

Design: professional dev-tool, dark header, white content, risk colors red/amber/green/slate with icons and text labels (never color alone), monospace for SPDX IDs and paths. Use React + Tailwind. Keep all scanning logic in separate modules (repoHosts, manifestParsers, licenseRules) so live fetching can be added without rewriting the UI.
```

## 6. Follow-up prompts — one batch per credit window

- **Window 2 (Sat 6a–12p PT) — Live GitHub:** "Add live scanning for GitHub repos: use the GitHub REST API (no key) to get repo info, own license via /license, file tree, and raw manifest files. Parse package.json, requirements.txt, pyproject.toml, go.mod, Cargo.toml, pom.xml, Gemfile, composer.json. Show real progress steps and honest errors (private/not found/rate-limited). Keep Demo Mode as fallback."
- **Window 3 (Sat 12–6p PT) — Chain resolution:** "Resolve licenses and transitive deps via the deps.dev v3 API (GetVersion + GetDependencies, depth 3, cap 250 nodes, dedupe), with npm registry and PyPI JSON as fallbacks. Show direct vs transitive in tree and table, and the source of each license."
- **Window 4 (Sat 6p–12a PT) — GitLab + Bitbucket:** "Add live GitLab (api/v4, URL-encoded project path incl. subgroups) and Bitbucket (2.0 API) scanning using the same manifest parsers and rules. Test each with a real public repo."
- **Window 5 (Sun 12–6a PT) — Map upgrade:** "Add graph view toggle (nodes/edges) alongside the tree, risk filters (All / High / Unknown / Copyleft / Unknown licenses), and search in the dependency table. Clicking a risk scrolls to and highlights its chain path."
- **Window 6 (Sun 6a–12p PT) — Report polish:** "Polish exports: Markdown report with verdict, risk table, full chain, obligations, and the disclaimer; JSON export matching the scan object. Add recent-scans history (localStorage) and a shareable link encoding the repo URL."
- **Window 7 (Sun 12–6p PT) — Test pass:** Test with the repos in §7, fix parser/edge cases (scoped npm packages, Maven group:artifact, monorepos with multiple manifests, no-LICENSE repos, no-manifest repos)."
- **Window 8 (Sun 6–11:59p PT) — Ship:** "Final polish, empty/error states, accessibility pass, publish. No new features."

## 7. Test repos (verify before demoing; pick live ones in Window 7)

- GitHub, permissive + big tree: a popular MIT/Apache JS or Python project.
- GitHub, copyleft clash: any permissive project depending on a GPL package (Demo Mode covers this if a live one is elusive).
- GitLab: a public project on gitlab.com with a package.json or requirements.txt.
- Bitbucket: a public workspace repo with a manifest.
- Edge: a repo with no LICENSE file; a repo with no manifest.

## 8. Risks / honesty notes

- The "free weekend / 6-hour reset" terms are as Demo stated them — I could not find a public page confirming that specific promo (Forge's earlier 50x window ran Sep 14–Oct 14). At launch, read Bolt's Forge banner/usage meter first; if the terms differ, work to the meter, not this sheet.
- Forge asks users to opt in to sharing anonymized build data for model training — decide consciously at first entry; GitCheck contains no private code/prompts, so opt-in is low-risk here, but it is Demo's call.
- GitHub unauthenticated rate limits will bite on big trees — the optional token field (memory-only) is the mitigation.
- Never present a Demo Mode result as a live scan.
