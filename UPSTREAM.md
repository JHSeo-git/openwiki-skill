# Upstream tracking

This skill is derived from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT).

- Pinned commit: `a7c556f1dd81b1680ce268fc8822694b8476f6b8` (2026-07-08, v0.0.4)

## File mapping

Only these upstream paths carry skill-relevant behavior. Changes anywhere else (CLI/UI, provider plumbing, credentials onboarding, tests) are intentionally out of scope.

| Upstream | This repo |
|---|---|
| `src/agent/prompt.ts` (system prompt, user prompts, AGENTS.md template) | `skills/openwiki/SKILL.md` — Step 3 and "The user prompt to act on" are line-mapped verbatim ports |
| `src/agent/utils.ts` (git evidence, content snapshot, no-op detection, metadata) | `skills/openwiki/SKILL.md` — Steps 1, 2, 4 |
| `src/constants.ts` (path conventions) | `skills/openwiki/SKILL.md` |
| `examples/*.yml` | `skills/openwiki/references/automation.md` |

`skills/openwiki-ask/SKILL.md` is a loose analogue of upstream's `chat` command: its ground rules port the chat-mode instructions and shared security rules from `src/agent/prompt.ts` (not line-mapped); the wiki-first steps and citation rules are original to this repo.

## Sync procedure

1. Refresh a local upstream clone (this machine: `~/Projects/oss/openwiki`; if it is shallow, run `git fetch --unshallow` first, then `git pull`).
2. List relevant changes: `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/constants.ts examples/`
3. No output → report that the skill is current with upstream, and stop. Output → review each commit's diff and port it. Because Step 3 of `skills/openwiki/SKILL.md` is a verbatim, line-mapped copy of the prompt, prompt changes apply as near-direct diffs; keep `[adapted]`/`[omitted]` markers intact.
4. Update the pinned commit above and add a CHANGELOG entry describing what was ported.
