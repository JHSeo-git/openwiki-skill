# openwiki-skill

[![skills.sh](https://skills.sh/b/JHSeo-git/openwiki-skill)](https://skills.sh/JHSeo-git/openwiki-skill)

Agent skills that write, maintain, and answer from OpenWiki wikis — repository documentation in `openwiki/` and a personal knowledge wiki in `~/.openwiki/wiki` — a port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) v0.2.1 for coding agents like Claude Code and Codex. The wikis are OKF-compliant (Google Knowledge Catalog OKF v0.1): every concept page opens with YAML front matter (only `type` required; producer extensions preserved), pages cross-link as an evidence-backed concept graph, each directory gets a deterministically generated `index.md`, and non-compliant pages are normalized automatically at the start of every run.

The upstream CLI drives an LLM through provider APIs. This port drops that plumbing: your coding agent already is the LLM, with filesystem and git tools, so it executes the same workflow directly — the upstream system prompt is reproduced verbatim inside the skill, with harness differences marked `[adapted]`. No API key, no runtime, no configuration.

## Skills

| Skill | What it does |
|---|---|
| [`openwiki`](skills/openwiki/SKILL.md) | Generate (init) or surgically refresh (update) a repo's `openwiki/` wiki — upstream's code mode. Auto-detects the mode; manages the marker snippet in root `AGENTS.md` and `CLAUDE.md`. |
| [`openwiki-personal`](skills/openwiki-personal/SKILL.md) | Build or maintain the personal knowledge wiki at `~/.openwiki/wiki` — upstream's personal mode, with evidence from your own MCP servers, web search, and local repos instead of built-in OAuth connectors. Per-source wiring guidance: [`references/connectors.md`](skills/openwiki-personal/references/connectors.md). |
| [`openwiki-ask`](skills/openwiki-ask/SKILL.md) | Answer questions from either wiki, wiki-first, citing pages. |

## Install

```sh
npx skills add JHSeo-git/openwiki-skill
```

Or manually: copy `skills/openwiki/`, `skills/openwiki-personal/`, and `skills/openwiki-ask/` into your agent's skills directory, e.g. `~/.claude/skills/` for Claude Code. (If you installed the `migrate-wiki-to-okf` skill from v0.2.0, remove it — upstream 0.2.1 replaced it with an automatic normalization pass inside the wiki skills.)

## Use

- "Generate documentation for this repository" → init: `openwiki/quickstart.md` entrypoint plus focused section pages — each with OKF front matter and concept-graph cross-links, each directory with a generated `index.md` — the upstream marker snippet (`<!-- OPENWIKI:START/END -->`) in root `AGENTS.md` and `CLAUDE.md`, and run metadata in `openwiki/.last-update.json`.
- "Update the wiki" → surgical update driven by a docs impact plan and a soft diff budget; no-ops cleanly (early git check + content-hash check) when nothing relevant changed.
- "Migrate the wiki to OKF" → just run an update: every run starts by normalizing non-compliant pages (minimal `type`/`title` front matter flagged `openwiki_generated: true`, bodies untouched), then enriches the flagged pages it works on.
- "How does X work?" → `openwiki-ask` answers from the wiki, citing pages and their inline source references.
- "Set up my personal wiki" / "pull today's Slack into my wiki" → `openwiki-personal` initializes or source-updates `~/.openwiki/wiki`.
- Steer either wiki with a brief: a user-authored `openwiki/INSTRUCTIONS.md` (repo) or `~/.openwiki/INSTRUCTIONS.md` (personal) is read into every run as the "Wiki brief"; the skills read it but never rewrite it.

Wikis produced here are interoperable with the upstream CLI: either tool can continue a wiki the other started.

## Automation

[skills/openwiki/references/automation.md](skills/openwiki/references/automation.md) covers keyless scheduled updates (local cron or a cloud routine under your subscription — no API key), a scoped permission allowlist for headless runs, CI templates (GitHub Actions PR flow, GitLab MR flow, Bitbucket Pipelines PR flow — CI needs `ANTHROPIC_API_KEY` or a `claude setup-token` token), and a Codex headless note.

## Upstream

System prompt, workflow, and metadata semantics derive from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT). Pinned commit and sync procedure: [UPSTREAM.md](UPSTREAM.md). The `openwiki-ask` skill and the keyless automation angle were inspired by [jatinmayekar/openwiki-for-claude-code](https://github.com/jatinmayekar/openwiki-for-claude-code) (MIT).

## License

[MIT](LICENSE)
