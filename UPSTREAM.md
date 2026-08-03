# Upstream tracking

This skill is derived from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT).

- Pinned commit: `08280e20c7702d50905a8c9c0f0a257cb18f5cf5` (2026-07-31, v0.2.5)

## File mapping

Only these upstream paths carry skill-relevant behavior. Changes anywhere else (CLI/UI, provider plumbing, credentials/OAuth onboarding, connector runtimes, schedulers, tests) are intentionally out of scope.

| Upstream | This repo |
|---|---|
| `src/agent/prompt.ts` — shared skeleton + `repository` output config | `skills/openwiki/SKILL.md` Step 3 and "The user prompt to act on" (line-mapped verbatim) |
| `src/agent/prompt.ts` — `local-wiki` output config incl. `localWikiSynthesisInstruction` | `skills/openwiki-personal/SKILL.md` Step 3 and its user prompts (line-mapped verbatim) |
| `src/agent/prompt.ts` — "Wiki-first question answering" | `skills/openwiki-ask/SKILL.md` (section-mapped) |
| `src/agent/prompt.ts` — chat-mode instructions + shared security rules | `skills/openwiki-ask/SKILL.md` ground rules (loose, not line-mapped) |
| `src/agent/utils.ts` — mode-aware git evidence (`.openwikiignore`-filtered since 0.2.5, `filterGitOutputForIgnore`), snapshot (ignores `_plan.md` since 0.2.1), no-op (interrupted runs retried since 0.2.4; ignored-path-only changes skip since 0.2.5), metadata (`status` + `language` since 0.2.4), language inheritance (`createRunContext`), plan-file cleanup (`removeTemporaryPlanFile` — the only removal path since 0.2.5) | `skills/openwiki/SKILL.md` Steps 1/2/4/5; `skills/openwiki-personal/SKILL.md` Steps 1/2/4/5 |
| `src/agent/openwiki-ignore.ts` — `.openwikiignore` pattern semantics since 0.2.5 (`OpenWikiIgnore.load`/`.parse`: gitignore-compatible globs, `!` negation with last match winning, `/` anchoring, trailing-`/` directory scoping, case-insensitive matching over canonicalized paths). Repository mode only — local-wiki runs get an inactive ruleset | `skills/openwiki/SKILL.md` Step 1 "Load `.openwikiignore`" |
| `src/language.ts` + `src/agent/translation-middleware.ts` + `src/okf/index-labels.ts` — BCP-47 validation (`resolveLanguage`), the before-agent whole-wiki translation pass with `openwiki_translation_pending` markers (`resolveTranslationPlan` / `createWikiTranslationMiddleware`, incl. the per-page translation prompt), localized index headings and derived concept `type` | both wiki skills' Step 1 language resolution, Step 2 translation pass, Step 3 "Output language" section, Step 4 label table |
| `src/okf/index-sync.ts` — before-run normalization (`migrateWikiToOkf`) and deterministic per-directory `index.md` regeneration (`synchronizeWikiIndexes`) | both wiki skills' Step 2 normalization block and Step 4 (plus the Step 3 "Index discipline" block) |
| `src/mermaid/` — after-run fence validation with degrade-to-text fallback (`validateWikiMermaid`, `degradeInvalidMermaidFences`, the no-parser heuristic) | both wiki skills' Step 4 Mermaid pass (`[adapted]` heuristic — no bundled parser) plus the Step 3 "Diagram discipline" block from `prompt.ts` |
| `skills/mermaid-diagrams/SKILL.md` — bundled diagram-authoring skill | `skills/mermaid-diagrams/SKILL.md` (verbatim; validation-ownership line `[adapted]`) |
| `src/okf/frontmatter.ts` + `src/agent/okf-middleware.ts` — OKF validation on every wiki write, minimal-front-matter derivation (`openwiki_generated`, localized `type` since 0.2.4), preserved extension fields (`openwiki_translation_pending` survives rebuilds) | both wiki skills' Step 3 "Front matter requirements (OKF)" `[adapted]` bullets and the Step 2 fallback block |
| `src/code-mode.ts` — AGENTS.md marker snippet (workflow template is out of scope); `runCodeModeConnectors` (code-mode connector orchestration since 0.2.4) | `skills/openwiki/SKILL.md` Step 0; `skills/openwiki/references/runtime-evidence.md` |
| `src/agent/docs-only-backend.ts` — repo-run write boundary; since 0.2.5 also `.openwikiignore` enforcement (reads/edits hard-denied, ls/glob/grep filtered, shell execute allowlisted to `pwd` / `git rev-parse HEAD` / `rm -f ./openwiki/_plan.md`) | `skills/openwiki/SKILL.md` Step 3 security rules and ".openwikiignore discipline" (as prompt discipline) |
| `src/ingestion.ts` — `createSourceUpdateMessage` / `createSourceSynthesisPolicy` / `createConnectorSynthesisGuidance` | `skills/openwiki-personal/references/sources.md`; the `langsmith` code-mode arm → `skills/openwiki/references/runtime-evidence.md` |
| `src/agent/types.ts` — `OpenWikiCommand`, `OpenWikiOutputMode`, `UpdateMetadata` | both wiki skills' Step 4 schemas |
| `src/constants.ts` — path conventions (`openwiki`, `openwiki/.last-update.json`) | both wiki skills |
| `examples/*.yml` | `skills/openwiki/references/automation.md` |

Next-sync note: past the 0.2.5 pin, upstream `0aa6ddc` (#547) hardens mapped `src/code-mode.ts` marker handling (malformed `OPENWIKI:START`/`END` blocks rejected instead of corrupting agent files) — Step 0 territory; and `5f8a8fb` (#371) adds after-run internal-link validation (new `src/agent/wiki-link-validator.ts`, wired through mapped `okf-middleware.ts` with prompt additions in `prompt.ts`). Both are real behavior ports, not plumbing. Port them when syncing past v0.2.5.

OKF note: this port follows upstream's code, not the [OKF spec](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) (v0.1 draft) or blog posts. Upstream 0.2.1 (#373/#376/#390) closed most of its 0.2.0 spec gaps: `timestamp` and producer-defined extension fields are now accepted and must survive round trips; generated `index.md` files carry no front matter (only the bundle-root index gets `okf_version: "0.1"`, per spec §6/§11); `index.md` and `log.md` are reserved files — excluded from indexing, never validated as concepts. Still not implemented: nothing writes a `log.md` update history (spec §7, mentioned in the 0.2 announcement post) — reserved-file handling means one would now be tolerated, but keep this port from generating it until upstream does. Watch `src/agent/prompt.ts` / `src/okf/` for it in future syncs.

Explicitly out of scope: `src/auth/*`, `src/connectors/*` (runtime; its prompt-bearing functions are mapped above via `ingestion.ts`, and port-original host-tool wiring guidance lives in `skills/openwiki-personal/references/connectors.md`), `src/schedules.ts`, `src/onboarding.ts` (its `INSTRUCTIONS.md` goal-file conventions — repo `openwiki/INSTRUCTIONS.md` and personal `~/.openwiki/INSTRUCTIONS.md` — are mirrored by the wiki skills' Step 1), `src/openwiki-home.ts` (directory constants only informative), `src/credentials.tsx`, `src/cli.tsx`, `src/env.ts`, `src/diagnostics.ts`, `src/startup.ts`, `src/telemetry/*` plus the `telemetryFile` run option in `types.ts` (anonymous CLI telemetry — this port has no runtime to report on), `src/commands.ts` (CLI command plumbing), `src/agent/skills.ts` and the middleware wiring in `src/agent/index.ts` (skill bundling/mounting — the OKF middleware's behavior is ported as the wiki skills' Steps 2/4), `skills/write-connector/` (edits the upstream CLI's own connector source code — no connector runtime exists in this port; host-tool wiring guidance lives in `skills/openwiki-personal/references/connectors.md`).

## Sync procedure

1. Refresh a local upstream clone (this machine: `~/Projects/oss/openwiki`; if it is shallow, run `git fetch --unshallow` first, then `git pull`).
2. List relevant changes: `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/agent/types.ts src/agent/docs-only-backend.ts src/agent/okf-middleware.ts src/okf/ src/code-mode.ts src/ingestion.ts src/constants.ts skills/ examples/`
3. No output → report that the skills are current with upstream, and stop. Output → review each commit's diff and port it per the mapping above. The Step 3 sections of both wiki skills are verbatim, line-mapped copies — prompt changes apply as near-direct diffs; keep `[adapted]`/`[omitted]` markers intact.
4. Update the pinned commit above and add a CHANGELOG entry describing what was ported.
