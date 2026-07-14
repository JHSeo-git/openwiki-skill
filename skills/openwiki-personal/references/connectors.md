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
| `slack` | Slack Web API (`slack.com/api`), OAuth | A custom internal Slack app with an xoxp user token, called via `curl` or the official Slack CLI's `slack api` passthrough; or Slack's official MCP server. See the Slack note below. |

Slack note (researched 2026-07-14 against Slack's primary docs):

- Scopes for a user token: `channels:history`, `groups:history`, `im:history`, `mpim:history` (+ the matching `*:read` scopes for listing) cover public/private channels, group DMs, and your own 1:1 DMs via `conversations.list`/`users.conversations` + `conversations.history`/`conversations.replies`.
- Rate limits: the May 2025 reduction (1 request/min, 15 messages/request on history/replies) applies only to newly created, commercially distributed non-Marketplace apps — internal custom apps keep Tier 3 (50+/min, up to 999 messages/request), so a daily ingestion window is minutes, not hours.
- The official Slack CLI is app-dev tooling; it has no message-export command, but `slack api <method>` is a documented generic escape hatch.
- Store the token as e.g. `OPENWIKI_SLACK_USER_TOKEN` in `~/.openwiki/.env` and load it per command — see "Credentials" below.
- Avoid browser-session-token tools (`xoxc` + `d` cookie, e.g. slackdump) on managed workspaces: Slack's API terms prohibit circumventing security controls, and such tools themselves warn that admins may be alerted. Employer workspaces: admin app approval and company policy come before anything Slack's ToS allows.
- Slack's API terms also restrict bulk export and LLM-training use of API data — keep the wiki a synthesis layer (notes + minimal quotes), never a full mirror. This matches the source-page discipline in `sources.md`.

Port-only sources (no upstream counterpart) — same pattern, evidence via host tools:

| Source | Host-tool equivalent |
|---|---|
| `geeknews` (news.hada.io — Korean HN-style dev/tech news) | The agent's web fetch of the official Atom feed `https://news.hada.io/rss/news` (no auth; there is no public JSON API — the site's bot integrations are push webhooks, not a pull API). Item pages live at `news.hada.io/topic?id=...`; follow the `hackernews` synthesis guidance in [`sources.md`](sources.md). |

Anything else upstream has no connector for: an MCP server the user has connected for that service, or direct dictation in chat.

## Credentials — `~/.openwiki/.env`

Connector credentials (a Slack user token, future connector keys) live in `~/.openwiki/.env` — the same file the upstream CLI treats as its credential source of truth (`src/env.ts`, `openWikiEnvPath`), so this port and upstream share one place. Rules:

- Name variables in upstream's style (e.g. `OPENWIKI_SLACK_USER_TOKEN=...`, matching `OPENWIKI_TAVILY_API_KEY`/`OPENWIKI_NOTION_MCP_ACCESS_TOKEN`), one `KEY=value` per line. `chmod 600 ~/.openwiki/.env`.
- The agent never reads this file — the ported prompt's security rules forbid reading `.env` files. Load it only inside the command invocation, in one shell call, so values never appear in output or the transcript: `set -a; . ~/.openwiki/.env; set +a; <connector command>`.
- Refer to credentials by env var name only. Never echo them, grep for their values, or store them in wiki pages.

Conventions:

- The wiki brief is the source registry: record per source what to track and any confidentiality boundary (e.g. company-internal repos stay local to the wiki).
- Evidence reads are read-only; the run writes nothing outside `~/.openwiki/wiki` (Step 3 rules in `SKILL.md`).
- Credential discipline applies to connector auth: the run may use an authenticated CLI or MCP server, but must never read, print, or document tokens, cookies, or credential files.
