# Connectors — wiring sources with host tools

> Upstream 0.3.3 adds a built-in `custom-mcp` connector for arbitrary MCP servers. In this port that role needs no wiring: any MCP server the user has connected is already a source — synthesis rules for it live in `sources.md` (`### custom-mcp`).

Upstream OpenWiki feeds the personal wiki through built-in connectors (`src/connectors/sources/*`) that dump raw data under `~/.openwiki/connectors/`. This port has no connector runtime: **the host agent's read-only source tools are the connectors.** Pick an available tool, name the source in `~/.openwiki/INSTRUCTIONS.md` (the wiki brief), and apply the synthesis rules in [`sources.md`](sources.md). Since upstream 0.6.0, personal runs are shell-free: authenticated CLI commands, direct Git inspection, and shell-loaded credentials are no longer supported ingestion routes. Configure connector authentication outside the ingestion run; the run consumes the connector's read-only operations.

| Connector | Upstream implementation | Host-tool equivalent |
|---|---|---|
| `git-repo` | Connector-produced repository manifests | A read-only connector exposing recorded branch, HEAD, status, changed files, and recent commits. Name the repos in the wiki brief; use code mode for source-level inspection. |
| `notion` | Notion MCP server (access token) | The same Notion MCP server, connected to your agent (Claude Code: `claude mcp add ...`). |
| `x` | X API v2, OAuth user context | A read-only X MCP server or user-provided export. Session cookies/tokens belong to connector authentication, never wiki content. |
| `gmail` | Gmail REST API, OAuth | A connected read-only Gmail MCP server or mail connector. |
| `web-search` | Tavily search API (API key) | The agent's built-in web search and fetch (Claude Code: WebSearch/WebFetch; Codex: its web-search tool when enabled). No setup — list sites or feeds to watch in the wiki brief. |
| `hackernews` | Public Algolia / Firebase HN APIs (no auth) | The agent's web fetch against the same public APIs (`hn.algolia.com`, `hacker-news.firebaseio.com`). |
| `slack` | Slack Web API (`slack.com/api`), OAuth | A connected read-only Slack MCP server or connector authenticated by an approved internal app. See the Slack note below. |
| `geeknews` | — (port-only, no upstream counterpart) | The agent's web fetch of the official Atom feed `https://news.hada.io/rss/news` (no auth; no public JSON API — the site's bot integrations are push webhooks). Item pages at `news.hada.io/topic?id=...`; synthesize per the `hackernews` guidance in [`sources.md`](sources.md). |

Anything else upstream has no connector for: an MCP server the user has connected for that service, or direct dictation in chat.

Slack note:

- Scopes for a user token: `channels:history`, `groups:history`, `im:history`, `mpim:history` (+ the matching `*:read` scopes for listing) cover public/private channels, group DMs, and your own 1:1 DMs via `conversations.list`/`users.conversations` + `conversations.history`/`conversations.replies`.
- Add the `search:read` family (`search:read`, `search:read.files`, `search:read.im`, `search:read.mpim`, `search:read.private`, `search:read.public`, `search:read.users`) to enable definitive self-message search via `search.messages` — upstream's connector requests exactly these on top of the history/read set (`src/auth/providers.ts` `user_scope`), and falls back to bounded `conversations.history` scans with a warning when they are missing.
- Rate limits: the May 2025 reduction (1 request/min, 15 messages/request on history/replies) applies only to newly created, commercially distributed non-Marketplace apps — internal custom apps keep Tier 3 (50+/min, up to 999 messages/request), so a daily ingestion window is minutes, not hours.
- Configure authentication in the connector outside the personal ingestion run. A CLI-only setup needs a read-only connector before this skill can use it.
- Avoid browser-session-token tools (`xoxc` + `d` cookie, e.g. slackdump) on managed workspaces: Slack's API terms prohibit circumventing security controls, and such tools themselves warn that admins may be alerted. Employer workspaces: admin app approval and company policy come before anything Slack's ToS allows.
- Slack's API terms also restrict bulk export and LLM-training use of API data — keep the wiki a synthesis layer (notes + minimal quotes), never a full mirror. This matches the source-page discipline in `sources.md`.

## Credentials

The native CLI manages its own credential configuration, including `~/.openwiki/.env`; connected MCP servers may use their own stores. This port delegates authentication to those configured tools. Rules:

- The personal-wiki agent never reads, sources, writes, or prints credential files. No shell-based credential loading, even for a read-only API call.
- If authentication is missing, report which connector needs setup without asking for the secret value or substituting a shell/CLI route.
- Refer to credentials by env var name only. Never echo them, grep for their values, or store them in wiki pages.

Conventions:

- The wiki brief is the source registry: record per source what to track and any confidentiality boundary (e.g. company-internal repos stay local to the wiki).
- Evidence reads are read-only; the run writes nothing outside `~/.openwiki/wiki` (Step 3 rules in `SKILL.md`).
- Credential discipline applies to connector auth: the run may use an authenticated read-only MCP/source tool, but must never read, print, or document tokens, cookies, or credential files.
