# openwiki-skill quickstart

## What this repository is

Agent skills that write, maintain, and answer from repository documentation kept in an `openwiki/` directory — a port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) for coding agents such as Claude Code and Codex.

The upstream project is a CLI that drives an LLM through provider APIs (API keys, model config, an agent runtime). This repo exists because a coding agent already *is* that LLM, with filesystem and git tools attached — so the port keeps upstream's product value (its agent prompt and runtime bookkeeping) and deletes the plumbing. Result: no API key, no runtime, no configuration. Install and usage live in [`README.md`](../README.md); this wiki is the map of how the repo itself is built.

## Layout

| Path | Role |
|---|---|
| [`skills/openwiki/SKILL.md`](../skills/openwiki/SKILL.md) | The main skill: generate (init) or surgically refresh (update) a repo's `openwiki/` wiki. Mode auto-detection, then a 4-step runtime (below). |
| [`skills/openwiki/references/automation.md`](../skills/openwiki/references/automation.md) | Scheduled updates: keyless options (subscription `claude -p` via cron, cloud routine), a scoped permission allowlist for headless runs, GitHub Actions / GitLab CI templates, a Codex headless note. |
| [`skills/openwiki-ask/SKILL.md`](../skills/openwiki-ask/SKILL.md) | Companion skill: answer repo questions using the wiki as primary source, citing pages and their inline source references. Original to this repo (no upstream counterpart). |
| [`UPSTREAM.md`](../UPSTREAM.md) | Canonical for upstream tracking: pinned upstream commit, file mapping, and the 4-step sync procedure. |
| [`docs/superpowers/`](../docs/superpowers/) | Design spec (v2, Korean) and the implementation plan. The plan embeds byte-exact copies of every skill file plus the verification scripts that check them. |
| [`CHANGELOG.md`](../CHANGELOG.md), [`LICENSE`](../LICENSE) | Release log (one bullet per change); MIT. |

## How the openwiki skill works — and why it has 4 steps

Upstream's CLI wrapped its LLM agent in runtime bookkeeping (`src/agent/utils.ts`). The skill reproduces that wrapper as explicit steps so the host agent performs it directly:

1. **Git evidence** — `git --no-pager status/rev-parse/log/diff` with mode-dependent history windows, plus an *early no-op exit* for update runs (ported from upstream `getUpdateNoopStatus`): clean worktree + only-`openwiki/` commits since the recorded head → report "already current" and stop.
2. **Content snapshot** — hash the `openwiki/` tree (excluding metadata) before working (ported from `createOpenWikiContentSnapshot`).
3. **The system prompt** — upstream `src/agent/prompt.ts` reproduced *verbatim*; the agent acts on it. Harness differences are marked inline with `**[adapted]**`, dropped upstream content with `**[omitted]**`.
4. **Metadata** — recompute the hash; only if content actually changed, write `openwiki/.last-update.json` (`updatedAt`, `command`, `gitHead`, `model` — upstream's exact schema).

The verbatim strategy is deliberate (chosen in the v2 design pivot, see `docs/superpowers/specs/`): it keeps generated wikis byte-compatible and interoperable with the upstream CLI — either tool can continue a wiki the other started — and it makes upstream syncs near-direct line diffs instead of re-interpretation.

## Changing this repo

- **Step 3 of `skills/openwiki/SKILL.md` is line-mapped to upstream `prompt.ts`.** Keep upstream sentences verbatim; mark every deviation `**[adapted]**` and every omission `**[omitted]**`. The `## OpenWiki` section template inside it must stay byte-identical to upstream — a silent unmarked edit breaks future syncs (this exact defect was caught and fixed once already; see commit history around the "mark adapted metadata line" fix).
- **After editing any skill file, re-run the verification scripts** embedded in `docs/superpowers/plans/2026-07-07-openwiki-skill.md`: frontmatter YAML parse, template byte-diff against the upstream clone, 16-section survival check, README relative-link check.
- **When upstream moves:** follow the sync procedure in [`UPSTREAM.md`](../UPSTREAM.md) (diff the mapped paths since the pin, port, bump the pin, add a CHANGELOG entry).
- **Testing:** there is no automated suite; the plan's Task 6 documents the manual E2E scenarios (fixture repo → init → surgical update → no-op → ask) used to verify skill behavior.
- Skill frontmatter descriptions stay double-quoted single-line strings, and all skill-facing text is English.
