# Changelog

## Unreleased

- Upstream sync: bump pin `7d35537` → `39493ef` (upstream v0.0.2). Mapped-path changes since the pin are CLI-only (`OPENWIKI_VERSION` bump, `OPENROUTER_FALLBACK_MODEL_IDS` removal — provider plumbing with no skill counterpart); `src/agent/prompt.ts`, `src/agent/utils.ts`, and `examples/` are unchanged, so nothing to port.

- `openwiki`: when neither root instruction file exists, also create `CLAUDE.md` containing `@AGENTS.md` (Claude Code reads only CLAUDE.md; the import is its documented pattern), and treat an importing CLAUDE.md as semantically current on update runs.

- Initial release: `openwiki` skill (verbatim port of the langchain-ai/openwiki agent prompt @ `7d35537`, wrapped in the CLI's runtime steps — git evidence with early no-op, content-snapshot hash, metadata write) and `openwiki-ask` skill (wiki-first Q&A), plus a keyless automation guide with scoped permissions and CI templates.
