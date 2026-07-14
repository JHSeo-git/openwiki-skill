# Connectors — wiring sources with host tools

Upstream OpenWiki feeds the personal wiki through built-in connectors (`src/connectors/sources/*`) that dump raw data under `~/.openwiki/connectors/`. This port has no connector runtime: **the host agent's own tools are the connectors.** This page is guidance only — it is not part of the ported prompt and nothing here is required. Pick the tool you already have, name the source in `~/.openwiki/INSTRUCTIONS.md` (the wiki brief), and the per-source synthesis rules in [`sources.md`](sources.md) take over.

Per upstream connector (registry order), what upstream actually uses and the suggested host-tool stand-in:

| Connector | Upstream implementation | Host-tool equivalent |
|---|---|---|
| `git-repo` | Local repository reads | Locally installed `git` plus the agent's file tools. Name the repo paths in the wiki brief. |
| `notion` | Notion MCP server (access token) | The same Notion MCP server, connected to your agent (Claude Code: `claude mcp add ...`). |
| `x` | X API v2, OAuth user context | A local X CLI — e.g. `birdclaw` (JSON output; its `bird` transport reads the browser session) — or an X MCP server. Session cookies/tokens are credentials: never print them or store them in the wiki. |
| `gmail` | Gmail REST API, OAuth | A Gmail MCP server, or an authenticated mail CLI you already trust. |
| `web-search` | Tavily search API (API key) | The agent's built-in web search and fetch (Claude Code: WebSearch/WebFetch; Codex: its web-search tool when enabled). No setup — list sites or feeds to watch in the wiki brief. |
| `hackernews` | Public Algolia / Firebase HN APIs (no auth) | The agent's web fetch against the same public APIs (`hn.algolia.com`, `hacker-news.firebaseio.com`). |
| `slack` | Slack Web API (`slack.com/api`), OAuth | A Slack MCP server, or an authenticated Slack CLI with a scoped token. |

Port-only sources (no upstream counterpart) — same pattern, evidence via host tools:

| Source | Host-tool equivalent |
|---|---|
| `geeknews` (news.hada.io — Korean HN-style dev/tech news) | The agent's web fetch of the official Atom feed `https://news.hada.io/rss/news` (no auth; there is no public JSON API — the site's bot integrations are push webhooks, not a pull API). Item pages live at `news.hada.io/topic?id=...`; follow the `hackernews` synthesis guidance in [`sources.md`](sources.md). |

Anything else upstream has no connector for: an MCP server the user has connected for that service, or direct dictation in chat.

Conventions:

- The wiki brief is the source registry: record per source what to track and any confidentiality boundary (e.g. company-internal repos stay local to the wiki).
- Evidence reads are read-only; the run writes nothing outside `~/.openwiki/wiki` (Step 3 rules in `SKILL.md`).
- Credential discipline applies to connector auth: the run may use an authenticated CLI or MCP server, but must never read, print, or document tokens, cookies, or credential files.
