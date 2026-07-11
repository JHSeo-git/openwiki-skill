# Upstream tracking

This skill is derived from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT).

- Pinned commit: `326a307203345128a60b92a356978c46e2992df3` (2026-07-11, v0.1.1)

## File mapping

Only these upstream paths carry skill-relevant behavior. Changes anywhere else (CLI/UI, provider plumbing, credentials/OAuth onboarding, connector runtimes, schedulers, tests) are intentionally out of scope.

| Upstream | This repo |
|---|---|
| `src/agent/prompt.ts` — shared skeleton + `repository` output config | `skills/openwiki/SKILL.md` Step 3 and "The user prompt to act on" (line-mapped verbatim) |
| `src/agent/prompt.ts` — `local-wiki` output config incl. `localWikiSynthesisInstruction` | `skills/openwiki-personal/SKILL.md` Step 3 and its user prompts (line-mapped verbatim) |
| `src/agent/prompt.ts` — "Wiki-first question answering" | `skills/openwiki-ask/SKILL.md` (section-mapped) |
| `src/agent/prompt.ts` — chat-mode instructions + shared security rules | `skills/openwiki-ask/SKILL.md` ground rules (loose, not line-mapped) |
| `src/agent/utils.ts` — mode-aware git evidence, snapshot, no-op, metadata | `skills/openwiki/SKILL.md` Steps 1/2/4; `skills/openwiki-personal/SKILL.md` Steps 1/2/4 |
| `src/code-mode.ts` — AGENTS.md marker snippet (workflow template is out of scope) | `skills/openwiki/SKILL.md` Step 0 |
| `src/agent/docs-only-backend.ts` — repo-run write boundary | `skills/openwiki/SKILL.md` Step 3 security rules (as prompt discipline) |
| `src/ingestion.ts` — `createSourceUpdateMessage` / `createSourceSynthesisPolicy` / `createConnectorSynthesisGuidance` | `skills/openwiki-personal/references/sources.md` |
| `src/agent/types.ts` — `OpenWikiCommand`, `OpenWikiOutputMode`, `UpdateMetadata` | both wiki skills' Step 4 schemas |
| `src/constants.ts` — path conventions (`openwiki`, `openwiki/.last-update.json`) | both wiki skills |
| `examples/*.yml` | `skills/openwiki/references/automation.md` |

Explicitly out of scope: `src/auth/*`, `src/connectors/*` (runtime; its prompt-bearing functions are mapped above via `ingestion.ts`), `src/schedules.ts`, `src/onboarding.ts` (its `INSTRUCTIONS.md` goal-file conventions — repo `openwiki/INSTRUCTIONS.md` and personal `~/.openwiki/INSTRUCTIONS.md` — are mirrored by the wiki skills' Step 1), `src/openwiki-home.ts` (directory constants only informative), `src/credentials.tsx`, `src/cli.tsx`, `src/env.ts`, `src/diagnostics.ts`, `src/startup.ts`.

## Sync procedure

1. Refresh a local upstream clone (this machine: `~/Projects/oss/openwiki`; if it is shallow, run `git fetch --unshallow` first, then `git pull`).
2. List relevant changes: `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/agent/types.ts src/agent/docs-only-backend.ts src/code-mode.ts src/ingestion.ts src/constants.ts examples/`
3. No output → report that the skills are current with upstream, and stop. Output → review each commit's diff and port it per the mapping above. The Step 3 sections of both wiki skills are verbatim, line-mapped copies — prompt changes apply as near-direct diffs; keep `[adapted]`/`[omitted]` markers intact.
4. Update the pinned commit above and add a CHANGELOG entry describing what was ported.
