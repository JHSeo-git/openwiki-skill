# OpenWiki Skill — Design

Date: 2026-07-07
Status: Approved in design review

## Goal

Recreate the workflow of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT) as an agent skill for Claude Code / Codex.

OpenWiki's product value lives almost entirely in its system prompt (`src/agent/prompt.ts`): discovery discipline, documentation quality rules, init/update workflows, git-evidence gathering, and metadata bookkeeping. The rest of the codebase is LLM plumbing (provider clients, deepagents runtime, SQLite checkpointing, stream parsing, credentials onboarding). This skill ports the rules into a SKILL.md package that the host agent executes with its own tools, and drops the plumbing entirely — no API keys, no separate runtime.

## Non-goals

- No standalone CLI, no direct LLM API calls, no provider/model configuration.
- No interactive chat mode — the host agent is already interactive.
- No automated test suite; verification is the manual scenarios below.

## Repository layout

```
openwiki-skill/
├── README.md                  # install (npx skills add JHSeo-git/openwiki-skill), usage, attribution
├── LICENSE                    # MIT + upstream copyright notice
├── SKILL.md                   # core skill
└── references/
    ├── init.md                # init-mode rules
    ├── update.md              # update-mode rules
    ├── agents-md-section.md   # fixed AGENTS.md/CLAUDE.md template + insertion rules
    └── ci-examples.md         # scheduled-update CI templates
```

## SKILL.md (core)

- Frontmatter: name `openwiki`; description is a short trigger phrase (generate or maintain repository wiki documentation in `openwiki/`).
- Mode detection: `openwiki/quickstart.md` exists → update; otherwise → init. An explicit user request overrides. Edge case: `openwiki/` exists without `quickstart.md` → treat as init but preserve existing files.

Shared discipline (ported from the upstream system prompt):

- Targeted discovery only: repo tree, package/config files, READMEs, entrypoints, routing, schema files, representative files per domain. Never exhaustively read every file.
- Read-only subagents (1-4) for parallel research when the harness supports them (Claude Code: Task tool); the main agent synthesizes and owns all writes.
- Treat existing docs (README, docs/, runbooks, SKILL.md files) as primary sources; when docs conflict with source, call out the stale doc and prefer source evidence.
- Never read or document secrets, credentials, or `.env` files.
- Write only under `openwiki/`; the only exceptions are top-level `AGENTS.md` / `CLAUDE.md`, and only for the reference section.
- Explain why code exists, not only what it contains. Docs target both humans and future agents.
- Temporary plan file `openwiki/_plan.md` after discovery, deleted before finishing (a harness planning tool may substitute).

Git evidence (inline command block):

```
git status --short
git rev-parse HEAD
# update with prior gitHead:
git log <lastHead>..HEAD --name-status --oneline
#   fallback: git log --since <updatedAt> --name-status --oneline
#   fallback: git log --max-count=20 --name-status --oneline
git diff --name-status HEAD
```

Non-git repositories: skip evidence and proceed from the filesystem only.

- Metadata, upstream-compatible: `openwiki/.last-update.json` — `{ updatedAt, command, gitHead, model }`. `model` is the host model id when known, else the harness name (`claude-code` / `codex`). Written only when wiki content actually changed.
- No-op rule (update): if commits since `lastHead` touch only `openwiki/` (or nothing) and the worktree is clean → report "wiki is already current" and write nothing.
- Routing: init → read `references/init.md`; update → read `references/update.md`; when touching root instruction files → `references/agents-md-section.md`; CI setup request → `references/ci-examples.md`.

## references/init.md

- Inventory order: existing docs → entrypoints → package/config → major domain dirs → tests/evals → data/schema → operational scripts.
- Use recent git history for "why": targeted `git log` / `git blame` / `git show` on high-signal files.
- `quickstart.md` first (high-level overview + links to every section), then section directories (`architecture/`, `workflows/`, `domain/`, `operations/`, ... as fits the repo).
- Budget: at most 8 pages on the initial run; small repos (≤ ~10 primary source files): quickstart plus 1-2 pages.
- Quality rules: no thin pages or stub directories; one canonical home per concept; prefer headings inside broader pages before new directories; final tree review to merge/remove low-value pages; no persistent commit-hash lists.

## references/update.md

- Read `.last-update.json` → gather git evidence → build a docs impact plan (source change → docs affected → edit needed → why) → surgical edits only.
- Diff budget: fewer than 5 changed source files → touch at most 1-2 wiki pages; more than 3 pages needs strong justification.
- No formatting-only edits; don't refresh source maps or "things to watch" sections unless materially wrong; touch quickstart only when top-level behavior, setup, or navigation changed.
- Check the root `AGENTS.md`/`CLAUDE.md` reference section for semantic staleness on every update run.
- A no-op is a valid outcome.

## references/agents-md-section.md

- The exact upstream `## OpenWiki` section template, verbatim — keeps repos interoperable with the upstream openwiki CLI.
- Rules: top-level files only; if both `AGENTS.md` and `CLAUDE.md` exist, insert in both; create `AGENTS.md` if neither exists; replace an existing section instead of duplicating; update only when semantically stale; never normalize unrelated formatting.

## references/ci-examples.md

- GitHub Actions cron workflow: checkout → install skill → run headless (`claude -p`) → open a PR when `openwiki/` changed. GitLab CI variant.
- Honest requirement note: local interactive use needs no API key, but CI automation needs `ANTHROPIC_API_KEY` or a `claude setup-token` OAuth token.
- Modeled on upstream `examples/openwiki-update.yml`.

## Harness compatibility

Skill text stays tool-neutral ("file read/search tools", "spawn read-only subagents if supported"), with Claude Code specifics as parenthetical hints. Codex consumes the same files via `npx skills` installation.

## Verification scenarios (manual)

1. Init on a small real repo → `openwiki/` created: quickstart entrypoint, page budget respected, AGENTS.md section inserted, metadata written.
2. Commit a source change, run update → surgical edit, impact plan honored, metadata refreshed.
3. Run update with no changes → no-op reported, no file churn.

## Attribution & license

MIT. README credits langchain-ai/openwiki as the source of the workflow and prompt rules; LICENSE includes the upstream MIT notice. Derived content: prompt discipline text (rewritten and condensed), AGENTS.md section template (verbatim), CI examples (adapted).
