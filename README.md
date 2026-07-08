# openwiki-skill

Agent skills that write, maintain, and answer from repository documentation in `openwiki/` — a port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) for coding agents like Claude Code and Codex.

The upstream CLI drives an LLM through provider APIs. This port drops that plumbing: your coding agent already is the LLM, with filesystem and git tools, so it executes the same workflow directly — the upstream system prompt is reproduced verbatim inside the skill, with harness differences marked `[adapted]`. No API key, no runtime, no configuration.

## Skills

| Skill | What it does |
|---|---|
| [`openwiki`](skills/openwiki/SKILL.md) | Generate (init) or surgically refresh (update) the `openwiki/` wiki. Auto-detects the mode. |
| [`openwiki-ask`](skills/openwiki-ask/SKILL.md) | Answer repository questions using the wiki as the primary source, citing pages. |

## Install

```sh
npx skills add JHSeo-git/openwiki-skill
```

Or manually: copy `skills/openwiki/` and `skills/openwiki-ask/` into your agent's skills directory, e.g. `~/.claude/skills/` for Claude Code.

## Use

- "Generate documentation for this repository" → init: `openwiki/quickstart.md` entrypoint plus focused section pages, an `## OpenWiki` section in root `AGENTS.md`/`CLAUDE.md` (byte-compatible with the upstream CLI), and run metadata in `openwiki/.last-update.json`.
- "Update the wiki" → surgical update driven by a docs impact plan and a soft diff budget; no-ops cleanly (early git check + content-hash check) when nothing relevant changed.
- "How does X work?" → `openwiki-ask` answers from the wiki, citing pages and their inline source references.

Wikis produced here are interoperable with the upstream CLI: either tool can continue a wiki the other started.

## Automation

[skills/openwiki/references/automation.md](skills/openwiki/references/automation.md) covers keyless scheduled updates (local cron or a cloud routine under your subscription — no API key), a scoped permission allowlist for headless runs, CI templates (GitHub Actions PR flow, GitLab MR flow — CI needs `ANTHROPIC_API_KEY` or a `claude setup-token` token), and a Codex headless note.

## Upstream

System prompt, workflow, and metadata semantics derive from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT). Pinned commit and sync procedure: [UPSTREAM.md](UPSTREAM.md). The `openwiki-ask` skill and the keyless automation angle were inspired by [jatinmayekar/openwiki-for-claude-code](https://github.com/jatinmayekar/openwiki-for-claude-code) (MIT).

## License

[MIT](LICENSE)
