# openwiki-skill quickstart

## What this repository is

Agent skills that write, maintain, and answer from OpenWiki wikis — a port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) v0.1.2 for coding agents such as Claude Code and Codex. Upstream 0.1.0 split into two modes, and the port mirrors that: **code mode** documents a repository in `openwiki/`; **personal mode** maintains a local knowledge wiki at `~/.openwiki/wiki` fed by the user's own sources.

The upstream project is a CLI that drives an LLM through provider APIs (API keys, OAuth connectors, an agent runtime, a scheduler). This repo exists because a coding agent already *is* that LLM, with filesystem, git, MCP, and web-search tools attached — so the port keeps upstream's product value (its agent prompts and runtime bookkeeping) and deletes the plumbing. Result: no API key, no OAuth, no runtime, no configuration. Install and usage live in [`README.md`](../README.md); this wiki is the map of how the repo itself is built.

## Layout

| Path | Role |
|---|---|
| [`skills/openwiki/SKILL.md`](../skills/openwiki/SKILL.md) | Code mode: generate (init) or surgically refresh (update) a repo's `openwiki/` wiki. Mode auto-detection, then a 5-step runtime (below) — Step 0 manages the `<!-- OPENWIKI:START/END -->` snippet in `/AGENTS.md` and `/CLAUDE.md`. |
| [`skills/openwiki/references/automation.md`](../skills/openwiki/references/automation.md) | Scheduled updates: keyless options (subscription `claude -p` via cron, cloud routine), a scoped permission allowlist, GitHub Actions / GitLab CI / Bitbucket Pipelines templates, a Codex headless note, and personal-wiki schedules (§5). |
| [`skills/openwiki-personal/SKILL.md`](../skills/openwiki-personal/SKILL.md) | Personal mode: build or maintain `~/.openwiki/wiki`. Same prompt skeleton with upstream's local-wiki configuration (canonical pages, confidence labels, email taxonomy); evidence comes from the host's MCP servers, web search, and local repos instead of upstream's OAuth connectors. |
| [`skills/openwiki-personal/references/sources.md`](../skills/openwiki-personal/references/sources.md) | Source update runs: the per-source prompt (24h window) plus upstream's synthesis policy and per-source guidance (gmail/notion/x/hackernews/web-search/slack/git-repo), verbatim. || [`skills/openwiki-ask/SKILL.md`](../skills/openwiki-ask/SKILL.md) | Q&A over both wikis, wiki-first (upstream's "Wiki-first question answering" rules), citing pages and their inline source references. |
| [`UPSTREAM.md`](../UPSTREAM.md) | Canonical for upstream tracking: pinned upstream commit, the prompt-branch → skill mapping, and the 4-step sync procedure. |
| [`docs/superpowers/`](../docs/superpowers/) | Design specs (Korean) and implementation plans. The plans embed the skill contents or deterministic extraction commands, plus the verification scripts that check them. |
| [`CHANGELOG.md`](../CHANGELOG.md), [`LICENSE`](../LICENSE) | Release log (one bullet per change); MIT. |

## How the wiki skills work

Upstream's CLI wraps one LLM agent in runtime bookkeeping (`src/agent/utils.ts`), parameterized by output mode. Each skill reproduces its mode's wrapper as explicit steps the host agent performs directly.

`openwiki` (code mode), 5 steps:

0. **Code setup** — idempotently maintain the `<!-- OPENWIKI:START/END -->` snippet in `/AGENTS.md` and `/CLAUDE.md` (ported from `src/code-mode.ts`; legacy marker-less sections are migrated, an `@AGENTS.md`-importing CLAUDE.md counts as covered, upstream's GH Actions workflow generation is deliberately omitted).
1. **Git evidence** — reads the optional user-authored `openwiki/INSTRUCTIONS.md` brief (injected into the user prompt as "Wiki brief"; control metadata the run never rewrites), then `git --no-pager status/rev-parse/log/diff` with mode-dependent history windows, plus an *early no-op exit* (ported from `getUpdateNoopStatus`): clean worktree + only-`openwiki/` commits since the recorded head → report "already current" and stop.
2. **Content snapshot** — hash the `openwiki/` tree (excluding metadata) before working (ported from `createOpenWikiContentSnapshot`).
3. **The system prompt** — upstream `src/agent/prompt.ts` with the `repository` output configuration inlined, reproduced *verbatim*; harness differences marked `**[adapted]**`, content owned by the other skills marked `**[omitted]**`.
4. **Metadata** — recompute the hash; only if content changed, write `openwiki/.last-update.json` (`updatedAt`, `command`, `gitHead`, `model`) — even when the run failed after generating content (ported from `persistRunMetadataIfChanged`).

`openwiki-personal` (personal mode) runs the same shape without git: no evidence step or early no-op (upstream's is repository-only), the snapshot/metadata root is `~/.openwiki/wiki`, and metadata omits `gitHead` — all matching upstream's local-wiki branches of `utils.ts`. Its wiki goal lives in `~/.openwiki/INSTRUCTIONS.md` (asked once at init), and source-scoped requests follow `references/sources.md`.

The verbatim strategy is deliberate (chosen in the v2 design pivot, see `docs/superpowers/specs/`): it keeps generated wikis byte-compatible and interoperable with the upstream CLI — either tool can continue a wiki the other started — and it makes upstream syncs near-direct line diffs instead of re-interpretation.

## Changing this repo

- **Step 3 of both wiki skills is line-mapped to upstream `prompt.ts`** (each to its output-mode branch). Keep upstream sentences verbatim; mark every deviation `**[adapted]**` and every omission `**[omitted]**`. Two spans must stay byte-identical to upstream: the AGENTS.md snippet in `skills/openwiki/SKILL.md` Step 0 (vs `src/code-mode.ts`) and the synthesis/policy blocks in the personal skill and `sources.md` (vs `prompt.ts`/`ingestion.ts`) — silent unmarked edits break future syncs.
- **After editing any skill file, re-run the verification scripts** embedded in `docs/superpowers/plans/2026-07-10-openwiki-0.1.0-port.md`: frontmatter YAML parse per skill, snippet/synthesis byte-diffs against the upstream clone, section survival checks, README relative-link check.
- **When upstream moves:** follow the sync procedure in [`UPSTREAM.md`](../UPSTREAM.md) (diff the mapped paths since the pin, port per the mapping table, bump the pin, add a CHANGELOG entry).
- **Testing:** there is no automated suite; the 0.1.0 plan's Task 7 documents the manual E2E scenarios (code init → legacy-AGENTS.md migration → two-phase no-op; personal init/source-update/no-op under an isolated `HOME`; ask routing over both wikis).
- Skill frontmatter descriptions stay double-quoted single-line strings, and all skill-facing text is English.
