# openwiki-skill Implementation Plan (v2 — verbatim porting strategy)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `openwiki` + `openwiki-ask` agent skills — a verbatim port of langchain-ai/openwiki's system prompt wrapped in the runtime steps its CLI performed, executable by Claude Code / Codex with no LLM API plumbing.

**Architecture:** Markdown-only skill package. `skills/openwiki/SKILL.md` = mode resolution + Step 1 git evidence (with early no-op) + Step 2 content-snapshot hash + Step 3 upstream system prompt reproduced verbatim (`[adapted]`/`[omitted]` markers) + Step 4 metadata write + upstream user-prompt blocks. `skills/openwiki/references/automation.md` = keyless scheduling, scoped permissions, CI templates. `skills/openwiki-ask/SKILL.md` = wiki-first Q&A. Repo meta: `README.md`, `UPSTREAM.md`, `CHANGELOG.md`. Spec: `docs/superpowers/specs/2026-07-07-openwiki-skill-design.md` (v2).

**Tech Stack:** Markdown + YAML frontmatter. Verification via `python3` (PyYAML available on this machine), `grep`, `git`. No build, no dependencies, no automated test suite (per spec).

## Global Constraints

- Repo: `/Users/a78256/Projects/openwiki-skill`, branch `main`. `LICENSE` (plain GitHub MIT) and `docs/` are already committed — do not touch them.
- All skill-facing text is English. Conversation with the user stays Korean.
- Upstream pin (copy exactly): `7d355379423172049308ba166ec2eff02c1c2e7d` (2026-07-06). Upstream clone lives at `/Users/a78256/Projects/oss/openwiki`.
- Step 3 of `skills/openwiki/SKILL.md` must stay line-mappable to upstream `src/agent/prompt.ts`: upstream sentences verbatim; every deviation carries an inline `**[adapted]**` marker; dropped upstream content carries `**[omitted]**`. `${OPEN_WIKI_DIR}` → `openwiki`, `${UPDATE_METADATA_PATH}` → `openwiki/.last-update.json`.
- Output conventions are upstream-compatible and exact: wiki dir `openwiki/`, entrypoint `openwiki/quickstart.md`, metadata `openwiki/.last-update.json` with keys `updatedAt`, `command`, `gitHead`, `model`.
- The AGENTS.md section template must match upstream `src/agent/prompt.ts` byte-for-byte (Task 1 verifies with a diff script).
- SKILL.md frontmatter: `description` value double-quoted, single line; YAML-parse it before committing (user rule).
- Every commit message ends with: `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`
- Run all commands from `/Users/a78256/Projects/openwiki-skill` unless a step says otherwise.

---

### Task 1: skills/openwiki/SKILL.md (verbatim port)

**Files:**
- Create: `skills/openwiki/SKILL.md`

**Interfaces:**
- Consumes: upstream `src/agent/prompt.ts` and `src/agent/utils.ts` semantics (verbatim/ported)
- Produces: `skills/openwiki/SKILL.md` — routes to `references/automation.md` (Task 2); its metadata format and no-op behavior are exercised by the E2E test (Task 6)

- [ ] **Step 1: Write the file**

Create `skills/openwiki/SKILL.md` with exactly this content:

`````markdown
---
name: openwiki
description: "Generate or maintain repository wiki documentation in openwiki/. Auto-detects init (no wiki yet) vs update (wiki exists). Use when asked to create, initialize, refresh, or update a repo's wiki or codebase documentation."
---

# OpenWiki — repository wiki agent

Port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki): the upstream system prompt reproduced verbatim (Step 3), wrapped in the runtime steps the upstream CLI performed around it (Steps 1, 2, 4 — from `src/agent/utils.ts`). You are the agent; the current repository is the target. No CLI, no API key — you do the work with your own tools.

Harness adaptations are marked **[adapted]**; upstream content with no equivalent here is marked **[omitted]**. Everything else is upstream text — keep it that way so upstream syncs stay line-mappable (see `UPSTREAM.md` in this skill's source repo).

## Mode resolution

- The user explicitly asks to initialize / build from scratch → **init**.
- The user explicitly asks to update / refresh → **update**.
- Otherwise auto-detect: `openwiki/quickstart.md` exists → **update**; it does not → **init**. (If `openwiki/` exists without `quickstart.md`, run init but read and preserve what is already there.)
- Any other instruction in the user's request (e.g. "focus on the API routes") is an **additional user instruction** — append it to the user prompt as shown at the end of this file.

## Model tier

Upstream defaults to frontier coding models. Documentation quality depends on it — run this skill on a frontier tier, not a small/fast model.

## Step 1 — Collect git evidence (before any write; all git read-only; use `git --no-pager`)

Read `openwiki/.last-update.json` if it exists to recover `gitHead` and `updatedAt`. Then run:

Always:

```bash
git --no-pager status --short
git --no-pager rev-parse HEAD
git --no-pager diff --name-status HEAD
```

History, by mode:

- init, or update with no prior metadata:

```bash
git --no-pager log --max-count=20 --name-status --oneline
```

- update with a recorded `gitHead`:

```bash
git --no-pager log <gitHead>..HEAD --name-status --oneline
```

- update with only `updatedAt`:

```bash
git --no-pager log --since <updatedAt> --name-status --oneline
```

**Early no-op exit** (update mode, only when the user gave no additional instruction — ported from upstream `getUpdateNoopStatus`): if the worktree is clean (ignoring `openwiki/.last-update.json` itself) and the commits since the recorded `gitHead` are absent or touch only `openwiki/` paths → report that the wiki is already current and stop here.

Not a git repository → skip the commands and infer changes from filesystem timestamps, source inspection, and existing docs instead.

Keep the collected output — it is the "Git context" / "Git change summary" block used by the user prompt at the end of this file.

## Step 2 — Snapshot the wiki (before the work; ported from upstream `createOpenWikiContentSnapshot`)

```bash
find openwiki -type f ! -name '.last-update.json' 2>/dev/null | LC_ALL=C sort | xargs shasum -a 256 2>/dev/null | shasum -a 256
```

Record the hash. (`shasum -a 256` covers macOS and most Linux; on minimal Linux images substitute `sha256sum` in both places.) You will recompute it in Step 4.

## Step 3 — System prompt (act as this agent)

> Reproduced from upstream `src/agent/prompt.ts`. **[adapted]** markers cover three things: (a) DeepAgents virtual-filesystem tools and `/`-rooted virtual paths become your native file tools on real repo-relative paths; (b) the DeepAgents task tool becomes your harness's read-only subagents (Claude Code: the Task tool) — if your harness has none (e.g. Codex), skip subagents, work sequentially, and be extra disciplined about targeted reads; (c) metadata recording moves from the CLI to Step 4. **[omitted]** covers the upstream "OpenWiki CLI reference" section and chat mode — there is no CLI here and the host agent is already interactive.

You are OpenWiki, an expert technical writer, software architect, and product analyst.

Your job is to inspect the current codebase and produce documentation in the openwiki/ directory that is excellent for both humans and future coding agents.

**[adapted]** Use only the tools available to you. Prefer your native discovery tools — glob/grep-style search for targeted discovery, short targeted file reads, and your file write/edit tools for changes. Use git through the shell when it provides useful history. Do not invent files, modules, APIs, business rules, or behavior. Ground every important claim in source files, existing docs, or git evidence you have inspected.

Run discipline:

- **[adapted]** Filesystem tools operate on real repo-relative paths such as README.md, agent/..., server/..., and openwiki/quickstart.md.
- **[adapted]** Do not write outside the target repository. Keep shell commands rooted in the target repository directory.
- Do not exhaustively read every file. Inspect the repository tree, package/config files, README-style files, entrypoints, routing files, database/schema files, and representative files for each major domain.
- Do not call glob with **/* from the repository root. Use targeted discovery by directory and extension. Prefer shell commands like rg --files with excludes for .git, node_modules, dist, build, cache directories, and existing generated wiki output.
- Prefer grep/glob and short targeted reads over full-file reads when files are large.
- Create a strong first-pass wiki that is accurate and navigable, then stop. The wiki can be refined in later update runs.
- Keep the initial documentation set focused: quickstart plus the smallest set of section pages needed to explain the repo clearly.
- Do not run commands that search outside the target repository.

Subagent discipline:

- **[adapted]** You may use your harness's read-only subagents (Claude Code: the Task tool) to parallelize research during init and update runs when the repository has multiple substantial domains. If your harness has no subagents, skip this section and research sequentially.
- Default to 1-2 subagents for large or unfamiliar repositories. Use 3-4 subagents only when the repository is clearly small/medium, the domains are naturally independent, or the user explicitly asks for deeper research.
- Subagents must only inspect and summarize. They must not create, edit, delete, or move files, and they must not write to openwiki/.
- Give each subagent a narrow brief such as existing docs, runtime architecture, data/storage, UI/API surface, integrations, tests/evals, or business workflows.
- Ask each subagent to return concise findings with source paths and notable open questions. The main agent must synthesize the final docs and is responsible for all writes.
- Treat subagent reports as internal discovery notes. Do not paste subagent reports into the final user-facing response; the final response should summarize completed documentation changes and important caveats.

Planning discipline:

- After discovery and before writing final documentation, create a temporary openwiki/_plan.md file that lists the intended wiki pages, source evidence for each page, and remaining questions.
- **[adapted]** Write the plan to openwiki/_plan.md with your file tools.
- **[adapted]** Before completing the run, delete openwiki/_plan.md (for example `rm -f openwiki/_plan.md`).
- Do not leave openwiki/_plan.md in the final wiki.

Git discipline:

- Use git heavily where it helps explain why code exists, not just what code exists.
- During init, inspect recent commit history and use git log, git show, or git blame selectively on important files to understand how major workflows, entrypoints, and business rules evolved.
- During update, always inspect commits added since the previous successful OpenWiki run. Prefer the gitHead recorded in openwiki/.last-update.json; fall back to the last updatedAt timestamp if no gitHead exists.
- Use git status and git diff to account for uncommitted local changes, especially if they touch existing docs or important source files.
- Do not over-index on ancient history. Focus on recent commits and high-signal history for important files.

Existing documentation discipline:

- Treat existing README files, docs/ trees, root documentation files, runbooks, and SKILL.md files as primary source material.
- Summarize and link to existing docs when they are still useful instead of duplicating them wholesale.
- If existing docs conflict with source code or git history, call out the likely stale documentation and prefer current source evidence.

Root agent instruction files:

- Unless the user explicitly asks you not to, always make sure the repository's top-level agent instruction files reference the OpenWiki quickstart.
- Only consider top-level /AGENTS.md and /CLAUDE.md for this step. Do not edit nested AGENTS.md or CLAUDE.md files.
- If /AGENTS.md or /CLAUDE.md exists, add or update the OpenWiki reference section there. If both exist, ensure the same section is added to both (duplicated).
- If neither exists, create top-level /AGENTS.md containing only the OpenWiki reference section.
- During update runs, inspect any existing OpenWiki reference section in /AGENTS.md and/or /CLAUDE.md and refresh it only if the section is missing or semantically stale. This check is required even when the wiki itself is otherwise current.
- Preserve surrounding instructions in existing files. Replace/update an existing OpenWiki reference section instead of adding duplicates.
- Do not edit /AGENTS.md or /CLAUDE.md only to normalize formatting, blank lines, wrapping, or punctuation if the existing OpenWiki section is already semantically correct.
- Use this exact section structure every time:

```markdown
## OpenWiki

This repository has documentation located in the /openwiki directory.

Start here:
- [OpenWiki quickstart](openwiki/quickstart.md)

OpenWiki includes repository overview, architecture notes, workflows, domain concepts, operations, integrations, testing guidance, and source maps.

When working in this repository, read the OpenWiki quickstart first, then follow its links to the relevant architecture, workflow, domain, operation, and testing notes.
```

**[omitted]** (Upstream places an "OpenWiki CLI reference" here — there is no CLI in this port.)

Security and privacy rules:

- Do not read or document secret values, credentials, private keys, tokens, .env files, or other sensitive material.
- Do not read .env files. .env.example and other sample configuration files may be read only if they contain placeholders, not live secrets.
- If a secret-bearing file appears relevant, document only that such configuration exists and where non-sensitive setup should be described.
- Keep all documentation under openwiki/.
- Do not modify source code outside openwiki/. The only allowed exceptions are top-level /AGENTS.md and /CLAUDE.md, and only for the OpenWiki reference section described above.

Documentation goals:

- Someone with zero knowledge of the repository should be able to start at openwiki/quickstart.md and understand what the project is, how it is organized, what it does, and where to go next.
- A future agent should be able to use the docs to make high-quality code changes with less source exploration.
- Capture both technical details and business/product logic.
- Explain why important code exists, not only what files contain.
- Prefer clear Markdown with stable links between pages.
- Organize the docs like human documentation, not a raw file inventory.
- Include change-oriented guidance for future agents: where to start, what to watch out for, and which tests or checks are relevant when changing each major area.
- Keep the docs concise enough to maintain. Avoid repeating the same concept across pages; give each concept one canonical home and link to it from other pages when needed.
- Use git history for discovery, but do not include persistent commit hash lists in documentation unless a specific historical decision is important for future work.

Section quality rules:

- Do not create a directory unless it represents a real documentation area.
- A section directory should usually contain multiple substantive pages. A single-file directory is acceptable only when that page is substantial, has a clear domain boundary, and is likely to grow.
- Avoid thin pages. If a page would mostly be a stub, source map, or short note, merge it into openwiki/quickstart.md or a broader section page instead.
- Prefer headings inside broader pages before creating many small directories.
- Each page should provide real explanatory value: what the area does, why it exists, where to start, what to watch out for, and key source references.
- Before finishing an init or update run, review the openwiki/ tree. Merge, move, or remove low-value single-file directories and stub pages so the wiki remains easy to navigate and maintain.
- For small repositories with about 10 or fewer primary source files, prefer openwiki/quickstart.md plus at most 1-2 supporting pages. Avoid one-file section directories unless the boundary is clearly useful and likely to grow.
- Avoid splitting content into separate topic pages unless there is enough distinct, repository-specific behavior to justify the split.

Required documentation structure:

- openwiki/quickstart.md must be the entrypoint.
- openwiki/quickstart.md must include a high-level repository overview and links to every major section.
- **[adapted]** Write documentation with your file tools at real repo-relative paths, for example openwiki/quickstart.md.
- When the repository is large enough to need section directories, create one directory per major section, for example architecture/, workflows/, domain/, api/, data-models/, operations/, integrations/, testing/, or similar names that fit the repo.
- Each section directory should contain focused Markdown pages; if a directory would contain only one short page, prefer a broader page or a heading in openwiki/quickstart.md.
- Include source-file references inline where they help readers verify or continue exploring.
- Source Map sections are optional. Add one only when it materially improves navigation for that page. Prefer inline source references for short pages.
- Track the last successful documentation update in openwiki/.last-update.json.

Mode-specific behavior — init:

- This is an initial documentation run.
- Assume openwiki/ does not yet contain useful documentation.
- Build the documentation structure from scratch.
- First build a repository inventory: existing docs, graph/app entrypoints, package/config files, major domain folders, tests/evals, data/schema files, skill/playbook files, and operational scripts.
- Use git evidence during init to understand how important files and workflows came to be. Prefer recent commits and targeted git blame/show on high-signal files.
- If the repo already has substantial docs, create a wiki that functions as an opinionated map and synthesis layer over those docs.
- Create openwiki/quickstart.md first, then the linked section pages.
- Use at most 8 documentation pages on the initial run unless the repository is clearly tiny.
- Do not try to document every source file. Document the main architecture, workflows, domain concepts, data models, integrations, operations, tests, and known extension points at the right level of detail.
- **[adapted]** Record successful run metadata in openwiki/.last-update.json yourself, per Step 4.

Mode-specific behavior — update:

- This is a maintenance update run.
- Inspect the existing openwiki/ documentation before editing.
- Read openwiki/.last-update.json if it exists.
- Always use git-oriented repository evidence to understand recent changes. Inspect commits added since the previous successful run using the recorded gitHead when available. If shell execution is unavailable, use filesystem timestamps, source inspection, and existing docs to infer what changed.
- Before editing, build a docs impact plan from the changed source files: source change -> docs affected -> edit needed -> why. If a page cannot be tied to a relevant source, workflow, product, or existing-doc change, do not edit it.
- Update runs must be surgical. Preserve useful existing structure and wording when it remains accurate. Prefer replacing one stale sentence over adding new paragraphs.
- Only edit pages whose current content is inaccurate, incomplete, or misleading because of the recent changes. Do not refresh every page.
- Keep each concept in one canonical page. If the same detail appears in multiple pages, keep the detailed explanation in the canonical page and make other mentions brief or link-only.
- Do not make formatting-only edits. Do not reformat Markdown tables, normalize blank lines, reorder source lists, or polish wording unless the surrounding content is already being changed for accuracy.
- Do not update Source Map sections, git evidence lists, or generic "things to watch" sections during an update unless they are materially wrong because of the source changes.
- Do not include or refresh persistent commit hash lists unless a specific commit explains an important historical decision.
- Use a soft diff budget: if fewer than about 5 source files changed, update at most 1-2 wiki pages. Avoid touching quickstart unless the top-level product behavior, setup, or navigation changed. If you believe more than 3 wiki pages need edits, think very deeply on why before making broad changes.
- Update stale pages, add missing pages, remove obsolete claims, and keep quickstart links accurate only when needed by the docs impact plan.
- Updates may be a no-op. If there are no relevant source, workflow, product, or existing-doc changes since the previous successful run, and the current wiki is already accurate, do not edit files. Say that the wiki is already current.
- **[adapted]** Record successful run metadata in openwiki/.last-update.json yourself, only if content changed, per Step 4.

## Step 4 — Persist metadata (after the work; ported from upstream `writeLastUpdateMetadata` and the content-snapshot check)

Recompute the Step 2 hash with the same command.

- Hash unchanged → **no-op**: do not write `openwiki/.last-update.json`; tell the user the wiki is already current.
- Hash changed → write `openwiki/.last-update.json` with exactly these fields:

```json
{
  "updatedAt": "<UTC ISO-8601, from: date -u +%Y-%m-%dT%H:%M:%S.000Z>",
  "command": "init | update",
  "gitHead": "<from: git rev-parse HEAD; omit the key if not a git repo>",
  "model": "<your model id if known, else claude-code or codex>"
}
```

Run the `date` and `git` commands — never guess the timestamp or the head.

## The user prompt to act on

**init:**

> Initialize OpenWiki documentation for this repository.
>
> Inspect the project thoroughly, identify the major technical and business domains, and write the initial documentation under openwiki/.
>
> Start with openwiki/quickstart.md as the entrypoint. Then create section directories and pages that explain the repository in a way that is useful to both humans and future agents.
>
> Git context: *(the Step 1 output)*

**update:**

> Update the existing OpenWiki documentation for this repository.
>
> Inspect openwiki/, identify recent source changes, and refresh only the documentation pages directly affected by those changes. Use the git evidence below when available. Keep edits surgical: do not rewrite accurate sections, do not update source maps or git evidence just to refresh them, and do not make formatting-only changes. If the wiki is already current, do not edit files. Update openwiki/.last-update.json only when OpenWiki content changes.
>
> Last update metadata: *(contents of openwiki/.last-update.json, or "No previous OpenWiki update metadata was found.")*
>
> Git change summary: *(the Step 1 output)*

If the user gave an additional instruction, append:

> Additional user instruction: *(that text)*

## Final response

Summarize the completed documentation changes and important caveats — or state that the wiki is already current. Do not paste subagent reports.

## Automation

Asked to set up scheduled/recurring updates or CI? Read `references/automation.md` in this skill's directory.
`````

- [ ] **Step 2: Verify frontmatter parses and fields are correct**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && awk 'NR>1 && /^---$/{exit} NR>1{print}' skills/openwiki/SKILL.md | python3 -c "
import sys, yaml
d = yaml.safe_load(sys.stdin)
assert d['name'] == 'openwiki', d
assert isinstance(d['description'], str) and 'openwiki/' in d['description'], d
print('frontmatter OK')"
```

Expected: `frontmatter OK`.

- [ ] **Step 3: Verify the AGENTS.md template is byte-identical to upstream**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && python3 - <<'EOF'
import re
skill = open('skills/openwiki/SKILL.md').read()
up = open('/Users/a78256/Projects/oss/openwiki/src/agent/prompt.ts').read()
m1 = re.search(r'```markdown\n(.*?)\n```', skill, re.S)
m2 = re.search(r'\\`\\`\\`markdown\n(.*?)\n\\`\\`\\`', up, re.S)
t1 = m1.group(1)
t2 = m2.group(1).replace('\\`', '`')
assert t1.strip() == t2.strip(), 'TEMPLATE MISMATCH:\n---skill---\n%s\n---upstream---\n%s' % (t1, t2)
print('template verbatim OK')
EOF
```

Expected: `template verbatim OK`. On mismatch, fix the skill file (upstream is canonical), never the upstream clone.

- [ ] **Step 4: Verify every upstream prompt section survived the port**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && python3 - <<'EOF'
sections = [
    "You are OpenWiki, an expert technical writer",
    "Run discipline:", "Subagent discipline:", "Planning discipline:",
    "Git discipline:", "Existing documentation discipline:",
    "Root agent instruction files:", "Security and privacy rules:",
    "Documentation goals:", "Section quality rules:",
    "Required documentation structure:",
    "This is an initial documentation run.",
    "This is a maintenance update run.",
    "Initialize OpenWiki documentation for this repository.",
    "Update the existing OpenWiki documentation for this repository.",
    "Additional user instruction",
]
skill = open('skills/openwiki/SKILL.md').read()
missing = [s for s in sections if s not in skill]
assert not missing, 'MISSING SECTIONS: %s' % missing
print('all %d upstream sections present' % len(sections))
EOF
```

Expected: `all 16 upstream sections present`.

- [ ] **Step 5: Commit**

```bash
git add skills/openwiki/SKILL.md
git commit -m "feat: add openwiki skill (verbatim port of upstream agent prompt + runtime steps)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: skills/openwiki/references/automation.md

**Files:**
- Create: `skills/openwiki/references/automation.md`

**Interfaces:**
- Consumes: nothing from earlier tasks (standalone leaf; SKILL.md routes to it by relative path)
- Produces: `skills/openwiki/references/automation.md`, linked from `README.md` (Task 5)

- [ ] **Step 1: Write the file**

Create `skills/openwiki/references/automation.md` with exactly this content:

`````markdown
# Automation — scheduled wiki updates

Ways to keep `openwiki/` fresh automatically. Keyless (subscription-auth) options first — they match this port's "no API key" premise; CI needs a credential.

## 1. Keyless — subscription auth (recommended)

Headless Claude Code (`claude -p`) reuses your local subscription login; no API key is needed on your own machine.

**Local schedule (cron / OS scheduler)** — run from the target repo:

```bash
cd /path/to/repo && claude -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
```

Add the scoped permissions below to the target repo so the run never blocks on prompts. Example crontab entry (weekdays 09:03 local; off-minute cron times collide less):

```
3 9 * * 1-5 cd /path/to/repo && claude -p "Use the openwiki skill to update this repository's wiki." >> "$HOME/.openwiki-update.log" 2>&1
```

**Claude Code cloud routine (`/schedule`)** — runs on Anthropic infrastructure under your plan; your machine can be off. A routine starts from a fresh clone, so its prompt must also commit and push the changed `openwiki/` (or open a PR).

**In-session recurring (`/loop`, session cron)** — session-scoped; fine for a trial, not set-and-forget.

## 2. Scoped permissions for headless runs

Safer than `--dangerously-skip-permissions` on your own machine. Add to the **target repo's** `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git --no-pager status:*)",
      "Bash(git --no-pager rev-parse:*)",
      "Bash(git --no-pager log:*)",
      "Bash(git --no-pager diff:*)",
      "Bash(git --no-pager show:*)",
      "Bash(git --no-pager blame:*)",
      "Bash(rg:*)",
      "Bash(find:*)",
      "Bash(shasum:*)",
      "Bash(sha256sum:*)",
      "Bash(date:*)",
      "Bash(rm -f openwiki/_plan.md)",
      "Write(openwiki/**)",
      "Edit(openwiki/**)",
      "Write(AGENTS.md)",
      "Edit(AGENTS.md)",
      "Write(CLAUDE.md)",
      "Edit(CLAUDE.md)"
    ]
  }
}
```

Read/Glob/Grep are already read-only, the git commands above are read-only, and the skill itself forbids reading `.env`/secrets. Writes stay scoped to `openwiki/**` plus the two root instruction files.

## 3. CI with an API credential

CI runners have no subscription login. Set `ANTHROPIC_API_KEY` (API billing) or `CLAUDE_CODE_OAUTH_TOKEN` (subscription; generate with `claude setup-token`) as a CI secret. The skill is cloned into the workspace at `.claude/skills/openwiki/`; the PR/MR steps only add `openwiki/`, so the clone is never committed.

### GitHub Actions

Save as `.github/workflows/openwiki-update.yml`:

```yaml
name: OpenWiki Update

on:
  workflow_dispatch:
  schedule:
    # GitHub schedules use UTC; 08:00 UTC is midnight PST.
    - cron: "0 8 * * *"

permissions:
  contents: write
  pull-requests: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: true

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install Claude Code and the openwiki skill
        run: |
          npm install --global @anthropic-ai/claude-code
          git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill /tmp/openwiki-skill
          mkdir -p .claude/skills
          cp -R /tmp/openwiki-skill/skills/openwiki .claude/skills/openwiki

      - name: Run openwiki update
        # --dangerously-skip-permissions is acceptable here: the job runs in an
        # isolated, throwaway CI container.
        run: claude --dangerously-skip-permissions -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          # Subscription alternative (instead of ANTHROPIC_API_KEY):
          # CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

      - name: Create OpenWiki update pull request
        uses: peter-evans/create-pull-request@v7
        with:
          add-paths: openwiki
          branch: openwiki/update
          commit-message: "docs: update OpenWiki"
          title: "docs: update OpenWiki"
          body: |
            Automated OpenWiki documentation update.

            This PR was generated by the scheduled OpenWiki workflow.
```

### GitLab CI

Include in `.gitlab-ci.yml` (set `ANTHROPIC_API_KEY` and `OPENWIKI_GITLAB_TOKEN` as CI/CD variables):

```yaml
openwiki_update:
  image: node:22
  stage: deploy
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_PIPELINE_SOURCE == "web"'
  before_script:
    - apt-get update && apt-get install -y git curl
    - npm install --global @anthropic-ai/claude-code
    - git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill /tmp/openwiki-skill
    - mkdir -p .claude/skills
    - cp -R /tmp/openwiki-skill/skills/openwiki .claude/skills/openwiki
    - git config user.name "${GITLAB_USER_NAME:-OpenWiki Bot}"
    - git config user.email "${GITLAB_USER_EMAIL:-openwiki@example.com}"
  script:
    - claude --dangerously-skip-permissions -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
    - |
      if [ -z "$(git status --porcelain -- openwiki)" ]; then
        echo "OpenWiki is already up to date."
        exit 0
      fi
    - export OPENWIKI_BRANCH="openwiki/update-${CI_PIPELINE_ID}"
    - git checkout -b "$OPENWIKI_BRANCH"
    - git add openwiki
    - git commit -m "docs: update OpenWiki"
    - git push "https://oauth2:${OPENWIKI_GITLAB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" "$OPENWIKI_BRANCH"
    - |
      curl --fail --request POST \
        --header "PRIVATE-TOKEN: ${OPENWIKI_GITLAB_TOKEN}" \
        --form "source_branch=${OPENWIKI_BRANCH}" \
        --form "target_branch=${CI_DEFAULT_BRANCH}" \
        --form "title=docs: update OpenWiki" \
        --form "description=Automated OpenWiki documentation update generated by the scheduled GitLab pipeline." \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests"
```

## 4. Codex headless

Codex runs the same update through its non-interactive mode:

```bash
codex exec "Use the openwiki skill to update this repository's wiki."
```

Check `codex exec --help` for your version's sandbox/approval flags; the run only needs read-only git plus writes under `openwiki/`, `AGENTS.md`, and `CLAUDE.md`.
`````

- [ ] **Step 2: Verify the YAML and JSON blocks parse**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && python3 - <<'EOF'
import re, yaml
src = open('skills/openwiki/references/automation.md').read()
yblocks = re.findall(r'```yaml\n(.*?)\n```', src, re.S)
assert len(yblocks) == 2, 'expected 2 yaml blocks, got %d' % len(yblocks)
for i, b in enumerate(yblocks):
    yaml.safe_load(b)
    print('yaml block %d OK' % (i + 1))
import json
jblocks = re.findall(r'```json\n(.*?)\n```', src, re.S)
assert len(jblocks) == 1, 'expected 1 json block, got %d' % len(jblocks)
json.loads(jblocks[0])
print('permissions json OK')
EOF
```

Expected: `yaml block 1 OK`, `yaml block 2 OK`, `permissions json OK`.

- [ ] **Step 3: If Codex is installed locally, sanity-check the `codex exec` wording**

Run:

```bash
command -v codex >/dev/null && codex exec --help | head -20 || echo "codex not installed - keep generic wording"
```

If the help output contradicts the automation.md Codex section (e.g. `exec` subcommand doesn't exist), fix the section to match reality before committing.

- [ ] **Step 4: Commit**

```bash
git add skills/openwiki/references/automation.md
git commit -m "feat: add automation guide (keyless scheduling, scoped permissions, CI templates)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: skills/openwiki-ask/SKILL.md

**Files:**
- Create: `skills/openwiki-ask/SKILL.md`

**Interfaces:**
- Consumes: the `openwiki/` on-disk format produced by the `openwiki` skill (Task 1)
- Produces: `skills/openwiki-ask/SKILL.md`, exercised by E2E scenario 4 (Task 6)

- [ ] **Step 1: Write the file**

Create `skills/openwiki-ask/SKILL.md` with exactly this content:

````markdown
---
name: openwiki-ask
description: "Answer questions about this repository using its openwiki/ wiki as the primary source. Use for 'how does X work / where is Y / explain Z' questions when an openwiki/ directory exists."
---

# OpenWiki ask — answer from the wiki

Answer the user's repository question using the maintained `openwiki/` wiki as the primary source, instead of re-deriving everything from raw source.

## Steps

1. **Check the wiki exists.** No `openwiki/` → say so and offer to run the `openwiki` skill. You may still answer from source, but note that the wiki is missing.
2. **Orient at the entrypoint.** Read `openwiki/quickstart.md`, follow its links toward the question's area, and grep `openwiki/` for the question's key terms to find the canonical page.
3. **Answer from the wiki, citing pages.** Name the wiki page(s) the answer comes from, and surface their inline source references (e.g. `src/...`) so the user can jump to the code.
4. **Verify when the wiki falls short.** If the wiki looks stale or does not cover the question, say so, confirm against the actual source before answering, and suggest running the `openwiki` skill's update mode.

Keep answers grounded: prefer the wiki's canonical explanation plus a source pointer over guessing.
````

- [ ] **Step 2: Verify frontmatter**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && awk 'NR>1 && /^---$/{exit} NR>1{print}' skills/openwiki-ask/SKILL.md | python3 -c "
import sys, yaml
d = yaml.safe_load(sys.stdin)
assert d['name'] == 'openwiki-ask', d
assert isinstance(d['description'], str) and d['description'], d
print('frontmatter OK')"
```

Expected: `frontmatter OK`.

- [ ] **Step 3: Commit**

```bash
git add skills/openwiki-ask/SKILL.md
git commit -m "feat: add openwiki-ask skill (wiki-first Q&A)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: UPSTREAM.md + CHANGELOG.md

**Files:**
- Create: `UPSTREAM.md`
- Create: `CHANGELOG.md`

**Interfaces:**
- Consumes: the pinned commit from Global Constraints; mapping mirrors Tasks 1-2 outputs
- Produces: `UPSTREAM.md` (linked from `README.md`, Task 5); `CHANGELOG.md` (ship workflow appends here)

- [ ] **Step 1: Write UPSTREAM.md**

Create `UPSTREAM.md` with exactly this content:

````markdown
# Upstream tracking

This skill is derived from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT).

- Pinned commit: `7d355379423172049308ba166ec2eff02c1c2e7d` (2026-07-06)

## File mapping

Only these upstream paths carry skill-relevant behavior. Changes anywhere else (CLI/UI, provider plumbing, credentials onboarding, tests) are intentionally out of scope.

| Upstream | This repo |
|---|---|
| `src/agent/prompt.ts` (system prompt, user prompts, AGENTS.md template) | `skills/openwiki/SKILL.md` — Step 3 and "The user prompt to act on" are line-mapped verbatim ports |
| `src/agent/utils.ts` (git evidence, content snapshot, no-op detection, metadata) | `skills/openwiki/SKILL.md` — Steps 1, 2, 4 |
| `src/constants.ts` (path conventions) | `skills/openwiki/SKILL.md` |
| `examples/*.yml` | `skills/openwiki/references/automation.md` |

`skills/openwiki-ask/SKILL.md` is original to this repo (no upstream counterpart).

## Sync procedure

1. Refresh a local upstream clone (this machine: `~/Projects/oss/openwiki`; if it is shallow, run `git fetch --unshallow` first, then `git pull`).
2. List relevant changes: `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/constants.ts examples/`
3. No output → report that the skill is current with upstream, and stop. Output → review each commit's diff and port it. Because Step 3 of `skills/openwiki/SKILL.md` is a verbatim, line-mapped copy of the prompt, prompt changes apply as near-direct diffs; keep `[adapted]`/`[omitted]` markers intact.
4. Update the pinned commit above and add a CHANGELOG entry describing what was ported.
````

- [ ] **Step 2: Write CHANGELOG.md**

Create `CHANGELOG.md` with exactly this content:

````markdown
# Changelog

## Unreleased

- Initial release: `openwiki` skill (verbatim port of the langchain-ai/openwiki agent prompt @ `7d35537`, wrapped in the CLI's runtime steps — git evidence with early no-op, content-snapshot hash, metadata write) and `openwiki-ask` skill (wiki-first Q&A), plus a keyless automation guide with scoped permissions and CI templates.
````

- [ ] **Step 3: Verify the pin matches the actual upstream clone**

Run:

```bash
grep -c "7d355379423172049308ba166ec2eff02c1c2e7d" /Users/a78256/Projects/openwiki-skill/UPSTREAM.md && git -C /Users/a78256/Projects/oss/openwiki rev-parse HEAD
```

Expected: `1` followed by `7d355379423172049308ba166ec2eff02c1c2e7d` (same hash).

- [ ] **Step 4: Commit**

```bash
git add UPSTREAM.md CHANGELOG.md
git commit -m "feat: add upstream tracking and changelog

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: README.md

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: `UPSTREAM.md` (Task 4), `skills/openwiki/references/automation.md` (Task 2), `LICENSE` (already committed)
- Produces: `README.md`, the repo landing page

- [ ] **Step 1: Write the file**

Create `README.md` with exactly this content:

`````markdown
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
`````

- [ ] **Step 2: Verify every relative link target exists**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && grep -o '](\([^)h][^)]*\))' README.md | sed 's/](//;s/)//' | while read -r f; do test -e "$f" && echo "$f OK" || echo "MISSING $f"; done
```

Expected output (5 lines, no MISSING):

```
skills/openwiki/SKILL.md OK
skills/openwiki-ask/SKILL.md OK
skills/openwiki/references/automation.md OK
UPSTREAM.md OK
LICENSE OK
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: End-to-end verification (spec scenarios 1-4)

**Files:**
- Create: fixture repo under `/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e/` (throwaway)
- Modify: any skill file that the scenarios prove wrong (fix + commit)

**Interfaces:**
- Consumes: the complete skills (Tasks 1-5)
- Produces: verified skills; fixes committed if scenarios expose defects

- [ ] **Step 1: Create the fixture repo**

```bash
FIX=/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e
rm -rf "$FIX" && mkdir -p "$FIX/src" && cd "$FIX" && git init -b main -q
cat > package.json <<'EOF'
{
  "name": "greeter-cli",
  "version": "1.0.0",
  "bin": { "greet": "./src/index.js" },
  "description": "Tiny CLI that greets a user by name"
}
EOF
cat > src/index.js <<'EOF'
#!/usr/bin/env node
const { greet } = require("./greet");
const name = process.argv[2] || "world";
console.log(greet(name));
EOF
cat > src/greet.js <<'EOF'
function greet(name) {
  return `Hello, ${name}!`;
}
module.exports = { greet };
EOF
cat > README.md <<'EOF'
# greeter-cli
Tiny CLI that greets a user by name. Run: `node src/index.js Alice`
EOF
git add -A && git commit -q -m "initial greeter" && git log --oneline
```

Expected: one commit line ending in `initial greeter`.

- [ ] **Step 2: Scenario 1 — run init**

Act as the skill runtime: read `/Users/a78256/Projects/openwiki-skill/skills/openwiki/SKILL.md` and follow it against the fixture repo (`cd "$FIX"` for all repo commands). No `openwiki/quickstart.md` exists, so mode resolution must select init. Execute Steps 1-4 exactly as written — this is a test of the skill text; if any instruction is ambiguous or contradictory while following it, note it for Step 6.

- [ ] **Step 3: Verify scenario 1 outcomes**

```bash
FIX=/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e
cd "$FIX"
test -f openwiki/quickstart.md && echo "quickstart OK"
test ! -f openwiki/_plan.md && echo "plan deleted OK"
PAGES=$(find openwiki -name '*.md' | wc -l | tr -d ' '); [ "$PAGES" -le 3 ] && echo "page budget OK ($PAGES)" || echo "PAGE BUDGET EXCEEDED ($PAGES)"
grep -q "^## OpenWiki$" AGENTS.md && echo "agents-md section OK"
python3 -c "
import json
d = json.load(open('openwiki/.last-update.json'))
assert d['command'] == 'init', d
assert d['updatedAt'] and d['gitHead'] and d['model'], d
print('metadata OK')"
```

Expected: `quickstart OK`, `plan deleted OK`, `page budget OK (...)` (small repo → ≤3 including quickstart), `agents-md section OK`, `metadata OK`.

- [ ] **Step 4: Scenario 2 — commit the wiki, change source, run update**

```bash
FIX=/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e
cd "$FIX" && git add -A && git commit -q -m "docs: init wiki"
cat > src/greet.js <<'EOF'
function greet(name, { shout = false } = {}) {
  const message = `Hello, ${name}!`;
  return shout ? message.toUpperCase() : message;
}
module.exports = { greet };
EOF
git add -A && git commit -q -m "feat: add shout option to greet"
```

Then act as the skill runtime again (same as Step 2). `openwiki/quickstart.md` now exists → update mode. The early no-op must NOT fire (a non-openwiki commit exists). Verify afterwards:

```bash
FIX=/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e
cd "$FIX"
CHANGED=$(git status --porcelain -- openwiki | wc -l | tr -d ' '); [ "$CHANGED" -ge 1 ] && [ "$CHANGED" -le 3 ] && echo "surgical edit OK ($CHANGED files)" || echo "UNEXPECTED CHANGE COUNT ($CHANGED)"
python3 -c "
import json
d = json.load(open('openwiki/.last-update.json'))
assert d['command'] == 'update', d
print('metadata updated OK')"
git add -A && git commit -q -m "docs: update wiki"
```

Expected: `surgical edit OK (...)` (1 source change → at most 1-2 pages + metadata), `metadata updated OK`.

- [ ] **Step 5: Scenario 3 — run update again with no changes (no-op)**

Act as the skill runtime once more, with no new commits. The Step 1 early no-op must fire (clean worktree, no commits since gitHead) and the run must stop before any discovery work, reporting the wiki is already current. Verify nothing was written:

```bash
FIX=/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e
cd "$FIX" && [ -z "$(git status --porcelain)" ] && echo "no-op OK" || { echo "NO-OP VIOLATED:"; git status --porcelain; }
```

Expected: `no-op OK`.

- [ ] **Step 6: Scenario 4 — openwiki-ask smoke test**

Act as the `openwiki-ask` runtime: read `/Users/a78256/Projects/openwiki-skill/skills/openwiki-ask/SKILL.md` and answer, against the fixture repo, the question: "How does the shout option work?" The answer must cite at least one wiki page by name (e.g. `openwiki/quickstart.md`) and surface an inline source reference to `src/greet.js`. If the flow can't produce that from the skill text as written, note the defect for Step 7.

- [ ] **Step 7: Fix any defects the scenarios exposed**

If any check printed a failure, or following the skill text hit an ambiguity/contradiction: fix the responsible file(s) in `/Users/a78256/Projects/openwiki-skill`, re-run the failed scenario, then commit:

```bash
cd /Users/a78256/Projects/openwiki-skill && git add -A
git commit -m "fix: correct skill rules found by e2e scenarios

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

If everything passed on the first try, skip this step (no empty commits).

---

### Task 7: Publish to GitHub

**Files:**
- No file changes; remote creation + push only

**Interfaces:**
- Consumes: the complete, verified repo (Tasks 1-6)
- Produces: `https://github.com/JHSeo-git/openwiki-skill` (public), install path for `npx skills add JHSeo-git/openwiki-skill`

- [ ] **Step 1: Confirm clean tree**

```bash
cd /Users/a78256/Projects/openwiki-skill && git status --porcelain && git log --oneline
```

Expected: empty status; commit list from Tasks 1-6 plus the pre-existing docs/LICENSE commits.

- [ ] **Step 2: Create the public repo and push**

```bash
cd /Users/a78256/Projects/openwiki-skill && gh repo create JHSeo-git/openwiki-skill --public --source . --remote origin --push --description "Agent skills: generate/maintain repo wiki docs in openwiki/ (verbatim port of langchain-ai/openwiki for Claude Code & Codex)"
```

Expected: repo created, `main` pushed.

- [ ] **Step 3: Verify**

```bash
gh repo view JHSeo-git/openwiki-skill --json url,visibility --jq '.url + " " + .visibility'
```

Expected: `https://github.com/JHSeo-git/openwiki-skill PUBLIC`.

Then tell the user the repo is live and that they can smoke-test installation with `npx skills add JHSeo-git/openwiki-skill` (interactive; not run automatically).
