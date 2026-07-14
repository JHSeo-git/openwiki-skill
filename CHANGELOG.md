# Changelog

## 0.1.2 (2026-07-14)

- `openwiki-personal`: add `references/connectors.md` — guidance-only map of host-tool stand-ins per upstream connector, recording upstream's actual mechanism per source (direct OAuth APIs for Slack/Gmail/X, Tavily for web-search, public HN APIs, Notion MCP, local git-repo reads) alongside the suggested host tool (built-in web search/fetch, installed `git`, a local X CLI such as `birdclaw`, MCP servers or trusted CLIs for the rest); also records port-only sources — GeekNews via its official Atom feed (`news.hada.io/rss/news`; the site has no public JSON API); linked from the skill intro and README.

- Upstream sync: bump pin `326a307` → `7c084f9` (upstream v0.1.2). Ported: deferred-documentation-area tracking in both wiki skills' Step 3 (new "Coverage self-check" block, init records undocumented areas in a quickstart `## Backlog` section, update reads/promotes/prunes backlog entries); Step 4 now runs even when a run fails after generating content (`persistRunMetadataIfChanged`) so generated content stays diffable; automation.md gains a Bitbucket Pipelines template and the GitLab template's diff/add paths now include `AGENTS.md`/`CLAUDE.md` (upstream CI-example alignment; the workflow-file path is omitted — this port generates no CI files). Out of scope: CLI-reference/chat-mode rewording (`[omitted]` here), `OPENWIKI_PROVIDER` env pins and the Step-0 workflow template (provider/CI plumbing), OAuth/env/redaction fixes and new tests (CLI plumbing).

- README: catch up to the v0.1.1 sync — upstream port version 0.1.0 → 0.1.1, marker snippet now covers root `CLAUDE.md` as well as `AGENTS.md`, and document the `INSTRUCTIONS.md` "Wiki brief" in Use.

- Skills: bump stale upstream-version references (v0.1.0 → v0.1.1) in the `openwiki` and `openwiki-personal` SKILL.md intros and Step 3 reproduction notes, missed in the 0.1.1 sync.

## 0.1.1 (2026-07-11)

- Upstream sync: bump pin `bf8f84a` → `326a307` (upstream v0.1.1). Ported: "Wiki brief" block in both modes' init/update user prompts (repo brief read from `openwiki/INSTRUCTIONS.md`, personal from `~/.openwiki/INSTRUCTIONS.md`; the personal skill's `[adapted]` goal injection is now upstream-official); two new root-agent rules treating `openwiki/INSTRUCTIONS.md` as user-authored control metadata (read, never rewrite in routine runs); Step 0 now manages the marker snippet in `/CLAUDE.md` as well (`CODE_MODE_AGENT_FILES`), keeping the `[adapted]` @AGENTS.md-import-counts-as-covered exception; automation.md GH Actions template mirrors upstream (SHA-pinned create-pull-request action, add-paths includes AGENTS.md/CLAUDE.md). Out of scope: NVIDIA NIM provider, GPT-5.6 model options, ChatGPT OAuth fix, ephemeral checkpoints, issue templates (CLI plumbing).

## 0.1.0 (2026-07-10)

- Upstream 0.1.0 port: re-port `openwiki` to upstream's code mode (agent no longer edits AGENTS.md/CLAUDE.md mid-run — a new Step 0 manages the upstream `<!-- OPENWIKI:START/END -->` marker snippet, migrating legacy sections; write boundary "only under openwiki/" per upstream docs-only backend; GH Actions auto-creation intentionally omitted in favor of automation.md); add `openwiki-personal` (upstream personal/local-wiki mode: `~/.openwiki/wiki`, verbatim synthesis discipline — canonical pages, confidence labels, email taxonomy — metadata without gitHead, wiki goal via `~/.openwiki/INSTRUCTIONS.md`, per-source runs in `references/sources.md` with evidence from host MCP/web-search/local tools instead of OAuth connectors); `openwiki-ask` now covers both wikis with upstream's wiki-first rules; UPSTREAM.md remapped for 0.1.0 and pin bumped `a7c556f` → `bf8f84a`.

- README: add skills.sh badge.

- Upstream sync: bump pin `39493ef` → `a7c556f` (upstream v0.0.4). Only mapped-path change is the `OPENWIKI_VERSION` constant (0.0.2 → 0.0.4); the rest is CLI plumbing (non-TTY startup handling, tool-schema recovery middleware, stream-event simplification) with no skill counterpart — `src/agent/prompt.ts`, `src/agent/utils.ts`, and `examples/` unchanged, nothing to port.

- `openwiki-ask`: add ground rules ported from upstream's `chat` mode — answering is read-only (no `openwiki/` edits unless explicitly asked; hand off to the `openwiki` skill for init/update) plus the shared secret-handling rules (never read or quote secrets or `.env` files); UPSTREAM.md and the wiki now record `openwiki-ask` as a loose analogue of upstream `chat` instead of "no upstream counterpart".

- Upstream sync: bump pin `7d35537` → `39493ef` (upstream v0.0.2). Mapped-path changes since the pin are CLI-only (`OPENWIKI_VERSION` bump, `OPENROUTER_FALLBACK_MODEL_IDS` removal — provider plumbing with no skill counterpart); `src/agent/prompt.ts`, `src/agent/utils.ts`, and `examples/` are unchanged, so nothing to port.

- `openwiki`: when neither root instruction file exists, also create `CLAUDE.md` containing `@AGENTS.md` (Claude Code reads only CLAUDE.md; the import is its documented pattern), and treat an importing CLAUDE.md as semantically current on update runs.

- Initial release: `openwiki` skill (verbatim port of the langchain-ai/openwiki agent prompt @ `7d35537`, wrapped in the CLI's runtime steps — git evidence with early no-op, content-snapshot hash, metadata write) and `openwiki-ask` skill (wiki-first Q&A), plus a keyless automation guide with scoped permissions and CI templates.
