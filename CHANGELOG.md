# Changelog

## 0.1.0 (2026-07-10)

- Upstream 0.1.0 port: re-port `openwiki` to upstream's code mode (agent no longer edits AGENTS.md/CLAUDE.md mid-run — a new Step 0 manages the upstream `<!-- OPENWIKI:START/END -->` marker snippet, migrating legacy sections; write boundary "only under openwiki/" per upstream docs-only backend; GH Actions auto-creation intentionally omitted in favor of automation.md); add `openwiki-personal` (upstream personal/local-wiki mode: `~/.openwiki/wiki`, verbatim synthesis discipline — canonical pages, confidence labels, email taxonomy — metadata without gitHead, wiki goal via `~/.openwiki/INSTRUCTIONS.md`, per-source runs in `references/sources.md` with evidence from host MCP/web-search/local tools instead of OAuth connectors); `openwiki-ask` now covers both wikis with upstream's wiki-first rules; UPSTREAM.md remapped for 0.1.0 and pin bumped `a7c556f` → `bf8f84a`.

- README: add skills.sh badge.

- Upstream sync: bump pin `39493ef` → `a7c556f` (upstream v0.0.4). Only mapped-path change is the `OPENWIKI_VERSION` constant (0.0.2 → 0.0.4); the rest is CLI plumbing (non-TTY startup handling, tool-schema recovery middleware, stream-event simplification) with no skill counterpart — `src/agent/prompt.ts`, `src/agent/utils.ts`, and `examples/` unchanged, nothing to port.

- `openwiki-ask`: add ground rules ported from upstream's `chat` mode — answering is read-only (no `openwiki/` edits unless explicitly asked; hand off to the `openwiki` skill for init/update) plus the shared secret-handling rules (never read or quote secrets or `.env` files); UPSTREAM.md and the wiki now record `openwiki-ask` as a loose analogue of upstream `chat` instead of "no upstream counterpart".

- Upstream sync: bump pin `7d35537` → `39493ef` (upstream v0.0.2). Mapped-path changes since the pin are CLI-only (`OPENWIKI_VERSION` bump, `OPENROUTER_FALLBACK_MODEL_IDS` removal — provider plumbing with no skill counterpart); `src/agent/prompt.ts`, `src/agent/utils.ts`, and `examples/` are unchanged, so nothing to port.

- `openwiki`: when neither root instruction file exists, also create `CLAUDE.md` containing `@AGENTS.md` (Claude Code reads only CLAUDE.md; the import is its documented pattern), and treat an importing CLAUDE.md as semantically current on update runs.

- Initial release: `openwiki` skill (verbatim port of the langchain-ai/openwiki agent prompt @ `7d35537`, wrapped in the CLI's runtime steps — git evidence with early no-op, content-snapshot hash, metadata write) and `openwiki-ask` skill (wiki-first Q&A), plus a keyless automation guide with scoped permissions and CI templates.
