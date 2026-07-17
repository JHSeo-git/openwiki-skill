# Upstream tracking

This skill is derived from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT).

- Pinned commit: `d4e94ab513ab13908c6b61346b23dc17bbd59b1f` (2026-07-16, v0.2.0 — "OKF + telemetry")

## File mapping

Only these upstream paths carry skill-relevant behavior. Changes anywhere else (CLI/UI, provider plumbing, credentials/OAuth onboarding, connector runtimes, schedulers, tests) are intentionally out of scope.

| Upstream | This repo |
|---|---|
| `src/agent/prompt.ts` — shared skeleton + `repository` output config | `skills/openwiki/SKILL.md` Step 3 and "The user prompt to act on" (line-mapped verbatim) |
| `src/agent/prompt.ts` — `local-wiki` output config incl. `localWikiSynthesisInstruction` | `skills/openwiki-personal/SKILL.md` Step 3 and its user prompts (line-mapped verbatim) |
| `src/agent/prompt.ts` — "Wiki-first question answering" | `skills/openwiki-ask/SKILL.md` (section-mapped) |
| `src/agent/prompt.ts` — chat-mode instructions + shared security rules | `skills/openwiki-ask/SKILL.md` ground rules (loose, not line-mapped) |
| `src/agent/utils.ts` — mode-aware git evidence, snapshot, no-op, metadata | `skills/openwiki/SKILL.md` Steps 1/2/5; `skills/openwiki-personal/SKILL.md` Steps 1/2/5 |
| `src/agent/index-middleware.ts` — deterministic per-directory `index.md` regeneration after each run | both wiki skills' Step 4 (and the Step 3 "Index discipline" block) |
| `src/agent/frontmatter-validator.ts` — OKF front matter rules enforced on every wiki write | both wiki skills' Step 3 "Front matter requirements (OKF)" `[adapted]` self-check bullet |
| `skills/migrate-wiki-to-okf/SKILL.md` — bundled OKF migration skill | `skills/migrate-wiki-to-okf/SKILL.md` (root/wrapper resolution `[adapted]`) |
| `src/code-mode.ts` — AGENTS.md marker snippet (workflow template is out of scope) | `skills/openwiki/SKILL.md` Step 0 |
| `src/agent/docs-only-backend.ts` — repo-run write boundary | `skills/openwiki/SKILL.md` Step 3 security rules (as prompt discipline) |
| `src/ingestion.ts` — `createSourceUpdateMessage` / `createSourceSynthesisPolicy` / `createConnectorSynthesisGuidance` | `skills/openwiki-personal/references/sources.md` |
| `src/agent/types.ts` — `OpenWikiCommand`, `OpenWikiOutputMode`, `UpdateMetadata` | both wiki skills' Step 4 schemas |
| `src/constants.ts` — path conventions (`openwiki`, `openwiki/.last-update.json`) | both wiki skills |
| `examples/*.yml` | `skills/openwiki/references/automation.md` |

OKF note: upstream implements a deliberate strict subset of the [OKF spec](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) (v0.1 draft), and this port follows the code, not the spec or the 0.2 blog post. Differences that look like omissions but are upstream decisions: the validator allows exactly five front matter fields (`timestamp` and producer-defined extras are rejected; the migrate skill says "Never add `timestamp`"); generated `index.md` files carry front matter and fixed `# Files`/`# Directories` sections (spec §6 index files carry none, except an optional bundle-root `okf_version`); and `log.md` (spec §7 optional update-history file, mentioned in the 0.2 announcement post) is not implemented — no code writes it, and `EXCLUDED_FILES` does not reserve it, so a spec-conformant front-matter-less `log.md` would make upstream's own index middleware throw. Do not add `log.md` here until upstream ships it: an upstream CLI continuing a wiki containing one would fail. Watch `src/agent/prompt.ts` / `src/agent/index-middleware.ts` for it in future syncs.

Explicitly out of scope: `src/auth/*`, `src/connectors/*` (runtime; its prompt-bearing functions are mapped above via `ingestion.ts`, and port-original host-tool wiring guidance lives in `skills/openwiki-personal/references/connectors.md`), `src/schedules.ts`, `src/onboarding.ts` (its `INSTRUCTIONS.md` goal-file conventions — repo `openwiki/INSTRUCTIONS.md` and personal `~/.openwiki/INSTRUCTIONS.md` — are mirrored by the wiki skills' Step 1), `src/openwiki-home.ts` (directory constants only informative), `src/credentials.tsx`, `src/cli.tsx`, `src/env.ts`, `src/diagnostics.ts`, `src/startup.ts`, `src/telemetry/*` plus the `telemetryFile` run option in `types.ts` (anonymous CLI telemetry — this port has no runtime to report on), `src/commands.ts` (CLI command plumbing), `src/agent/skills.ts` and the middleware wiring in `src/agent/index.ts` (skill bundling/mounting — here `migrate-wiki-to-okf` installs like any other skill, and the middleware's behavior is ported as the wiki skills' Step 4), `skills/write-connector/` (edits the upstream CLI's own connector source code — no connector runtime exists in this port; host-tool wiring guidance lives in `skills/openwiki-personal/references/connectors.md`).

## Sync procedure

1. Refresh a local upstream clone (this machine: `~/Projects/oss/openwiki`; if it is shallow, run `git fetch --unshallow` first, then `git pull`).
2. List relevant changes: `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/agent/types.ts src/agent/docs-only-backend.ts src/agent/index-middleware.ts src/agent/frontmatter-validator.ts src/code-mode.ts src/ingestion.ts src/constants.ts skills/ examples/`
3. No output → report that the skills are current with upstream, and stop. Output → review each commit's diff and port it per the mapping above. The Step 3 sections of both wiki skills are verbatim, line-mapped copies — prompt changes apply as near-direct diffs; keep `[adapted]`/`[omitted]` markers intact.
4. Update the pinned commit above and add a CHANGELOG entry describing what was ported.
