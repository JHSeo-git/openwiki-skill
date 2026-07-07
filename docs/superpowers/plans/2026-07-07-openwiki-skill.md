# openwiki-skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `openwiki` agent skill — a port of langchain-ai/openwiki's wiki-generation workflow that Claude Code / Codex execute directly, with no LLM API plumbing.

**Architecture:** A markdown-only skill package: `SKILL.md` (mode auto-detection + shared discipline) routes to `references/` files (init rules, update rules, verbatim AGENTS.md template, CI templates). Repo-meta files (`README.md`, `UPSTREAM.md`, `CHANGELOG.md`) handle install, upstream tracking, and history. Spec: `docs/superpowers/specs/2026-07-07-openwiki-skill-design.md`.

**Tech Stack:** Markdown + YAML frontmatter. Verification via `python3` (PyYAML — available on this machine), `grep`, `git`. No build, no dependencies, no automated test suite (per spec).

## Global Constraints

- Repo: `/Users/a78256/Projects/openwiki-skill`, branch `main`. `LICENSE` (plain GitHub MIT) and `docs/` are already committed — do not touch them.
- All skill-facing text is English. Conversation with the user stays Korean.
- Upstream pin (copy exactly): `7d355379423172049308ba166ec2eff02c1c2e7d` (2026-07-06). Upstream clone lives at `/Users/a78256/Projects/oss/openwiki`.
- Output conventions are upstream-compatible and exact: wiki dir `openwiki/`, entrypoint `openwiki/quickstart.md`, metadata `openwiki/.last-update.json` with keys `updatedAt`, `command`, `gitHead`, `model`.
- The AGENTS.md section template must match upstream `src/agent/prompt.ts` byte-for-byte (Task 1 verifies with a diff script).
- SKILL.md frontmatter: `description` value double-quoted, single line; YAML-parse it before committing (user rule).
- Every commit message ends with: `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`
- Run all commands from `/Users/a78256/Projects/openwiki-skill` unless a step says otherwise.

---

### Task 1: references/agents-md-section.md

**Files:**
- Create: `references/agents-md-section.md`

**Interfaces:**
- Consumes: upstream template in `/Users/a78256/Projects/oss/openwiki/src/agent/prompt.ts` (the escaped ` ```markdown ` block)
- Produces: `references/agents-md-section.md`, referenced by `SKILL.md` (Task 5), `references/init.md` step 6 (Task 2), `references/update.md` step 7 (Task 3)

- [ ] **Step 1: Write the file**

Create `references/agents-md-section.md` with exactly this content:

`````markdown
# Root instruction files (AGENTS.md / CLAUDE.md)

Unless the user explicitly asks you not to, make sure the repository's top-level agent instruction files reference the wiki.

## Rules

- Only top-level `/AGENTS.md` and `/CLAUDE.md`. Never edit nested AGENTS.md or CLAUDE.md files.
- If either file exists, add or update the OpenWiki section there. If both exist, ensure the same section appears in both.
- If neither exists, create a top-level `AGENTS.md` containing only the section below.
- Replace an existing OpenWiki section instead of adding a duplicate. Preserve all surrounding instructions.
- On update runs, refresh the section only if it is missing or semantically stale. Never edit these files just to normalize formatting, blank lines, wrapping, or punctuation.

## Section template (insert exactly this, verbatim)

```markdown
## OpenWiki

This repository has documentation located in the /openwiki directory.

Start here:
- [OpenWiki quickstart](openwiki/quickstart.md)

OpenWiki includes repository overview, architecture notes, workflows, domain concepts, operations, integrations, testing guidance, and source maps.

When working in this repository, read the OpenWiki quickstart first, then follow its links to the relevant architecture, workflow, domain, operation, and testing notes.
```

This template matches the upstream openwiki CLI byte-for-byte, so repositories stay interoperable with it.
`````

- [ ] **Step 2: Verify the template is byte-identical to upstream**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && python3 - <<'EOF'
import re
skill = open('references/agents-md-section.md').read()
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

- [ ] **Step 3: Commit**

```bash
git add references/agents-md-section.md
git commit -m "feat: add root instruction file template (verbatim upstream)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: references/init.md

**Files:**
- Create: `references/init.md`

**Interfaces:**
- Consumes: `references/agents-md-section.md` (Task 1) — referenced in step 6 of the sequence
- Produces: `references/init.md`, routed to by `SKILL.md` init mode (Task 5)

- [ ] **Step 1: Write the file**

Create `references/init.md` with exactly this content:

````markdown
# Init mode

Build a strong first-pass wiki that is accurate and navigable, then stop. Later update runs refine it — do not chase completeness.

## Sequence

1. **Inventory** the repository: existing docs → entrypoints → package/config files → major domain folders → tests/evals → data/schema files → skill/playbook files → operational scripts.
2. **Git evidence**: recent commits (`git log --max-count=20 --name-status --oneline`); targeted `git log` / `git blame` / `git show` on high-signal files to learn how major workflows, entrypoints, and business rules evolved.
3. **Plan**: write `openwiki/_plan.md` — intended pages, source evidence per page, open questions.
4. **Write `openwiki/quickstart.md` first**: what the project is, how it is organized, what it does, where to go next — with links to every section page.
5. **Write the section pages** the plan calls for.
6. **Root instruction files**: apply `references/agents-md-section.md`.
7. **Review the `openwiki/` tree**: merge, move, or remove low-value single-file directories and stub pages; verify quickstart links.
8. **Finish**: delete `openwiki/_plan.md`; write `openwiki/.last-update.json` (format in SKILL.md).

## Structure and budget

- At most 8 documentation pages on the initial run unless the repository is clearly tiny.
- Small repository (about 10 or fewer primary source files): `quickstart.md` plus at most 1-2 supporting pages; no section directories unless the boundary is clearly useful and likely to grow.
- When the repository warrants them, one directory per major section — `architecture/`, `workflows/`, `domain/`, `api/`, `data-models/`, `operations/`, `integrations/`, `testing/`, or similar names that fit the repo.
- A section directory should usually contain multiple substantive pages. Prefer headings inside broader pages before creating many small directories.

## Quality rules

- Every page provides real explanatory value: what the area does, why it exists, where to start, what to watch out for, and key source references inline. Source Map sections are optional — add one only when it materially improves navigation.
- No thin pages. If a page would mostly be a stub, source map, or short note, merge it into `quickstart.md` or a broader page.
- If the repository already has substantial docs, make the wiki an opinionated map and synthesis layer over them instead of duplicating them.
- Do not document every source file — cover architecture, workflows, domain concepts, data models, integrations, operations, tests, and extension points at the right altitude.
- Do not include persistent commit-hash lists unless one specific historical decision matters for future work.
````

- [ ] **Step 2: Verify internal references resolve**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && grep -o 'references/[a-z-]*\.md' references/init.md | sort -u | while read -r f; do test -f "$f" && echo "$f OK" || echo "MISSING $f"; done
```

Expected: `references/agents-md-section.md OK` and no `MISSING` lines.

- [ ] **Step 3: Commit**

```bash
git add references/init.md
git commit -m "feat: add init mode rules

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: references/update.md

**Files:**
- Create: `references/update.md`

**Interfaces:**
- Consumes: `references/agents-md-section.md` (Task 1) — referenced in step 7 of the sequence
- Produces: `references/update.md`, routed to by `SKILL.md` update mode (Task 5)

- [ ] **Step 1: Write the file**

Create `references/update.md` with exactly this content:

````markdown
# Update mode

Surgical maintenance. A no-op is a valid outcome — never edit just to look busy.

## Sequence

1. Read `openwiki/.last-update.json` if present.
2. **No-op check** (skip this check when the user gave a specific documentation request): if the worktree is clean and the commits since the recorded `gitHead` touch only `openwiki/` (or there are none) → report that the wiki is already current, change nothing, and stop.
3. Gather git evidence (commands in SKILL.md). If there is no prior metadata, use the last 20 commits and say so in the final response.
4. Inspect the existing `openwiki/` pages that the changes might affect.
5. Build a **docs impact plan**: source change → docs affected → edit needed → why. A page that cannot be tied to a relevant source, workflow, product, or existing-doc change does not get edited.
6. Make the surgical edits: update stale pages, add missing pages, remove obsolete claims — only what the impact plan demands. Prefer replacing one stale sentence over adding new paragraphs.
7. Check the root `AGENTS.md`/`CLAUDE.md` OpenWiki section against `references/agents-md-section.md` — required on every update run; refresh only if missing or semantically stale.
8. If content changed: keep quickstart links accurate, then write `openwiki/.last-update.json`.

## Budgets and prohibitions

- Soft diff budget: fewer than about 5 changed source files → update at most 1-2 wiki pages. If you believe more than 3 pages need edits, think very hard about why before making broad changes.
- Touch `quickstart.md` only when top-level product behavior, setup, or navigation changed.
- No formatting-only edits: do not reformat tables, normalize blank lines, reorder source lists, or polish wording unless the surrounding content is already being changed for accuracy.
- Do not refresh Source Map sections, git-evidence lists, or generic "things to watch" sections unless they are materially wrong because of the changes.
- Keep each concept in its canonical page; make other mentions brief or link-only.
````

- [ ] **Step 2: Verify internal references resolve**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && grep -o 'references/[a-z-]*\.md' references/update.md | sort -u | while read -r f; do test -f "$f" && echo "$f OK" || echo "MISSING $f"; done
```

Expected: `references/agents-md-section.md OK` and no `MISSING` lines.

- [ ] **Step 3: Commit**

```bash
git add references/update.md
git commit -m "feat: add update mode rules

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: references/ci-examples.md

**Files:**
- Create: `references/ci-examples.md`

**Interfaces:**
- Consumes: nothing from earlier tasks (standalone leaf)
- Produces: `references/ci-examples.md`, routed to by `SKILL.md` on CI setup requests (Task 5); linked from `README.md` (Task 7)

- [ ] **Step 1: Write the file**

Create `references/ci-examples.md` with exactly this content (two ` ```yaml ` blocks; adapted from upstream `examples/`, with the openwiki CLI swapped for a headless Claude Code run):

`````markdown
# Scheduled CI updates

Keep the wiki fresh by running update mode on a schedule and opening a PR/MR when `openwiki/` changed. Templates adapted from upstream openwiki's examples.

**Credentials:** local interactive use needs no API key, but CI automation does. Set one of these as a CI secret:

- `ANTHROPIC_API_KEY` — Claude API billing
- `CLAUDE_CODE_OAUTH_TOKEN` — Claude subscription; generate with `claude setup-token`

The skill is installed into the workspace at `.claude/skills/openwiki/` so the headless run can find it; that directory is never committed (the PR/MR steps only add `openwiki/`).

## GitHub Actions

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
          mkdir -p .claude/skills
          git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill .claude/skills/openwiki

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

## GitLab CI

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
    - mkdir -p .claude/skills
    - git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill .claude/skills/openwiki
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
`````

- [ ] **Step 2: Verify both YAML blocks parse**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && python3 - <<'EOF'
import re, yaml
src = open('references/ci-examples.md').read()
blocks = re.findall(r'```yaml\n(.*?)\n```', src, re.S)
assert len(blocks) == 2, 'expected 2 yaml blocks, got %d' % len(blocks)
for i, b in enumerate(blocks):
    yaml.safe_load(b)
    print('yaml block %d OK' % (i + 1))
EOF
```

Expected: `yaml block 1 OK` and `yaml block 2 OK`.

- [ ] **Step 3: Commit**

```bash
git add references/ci-examples.md
git commit -m "feat: add scheduled CI update templates

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: SKILL.md (core)

**Files:**
- Create: `SKILL.md`

**Interfaces:**
- Consumes: all four `references/*.md` files (Tasks 1-4) — routed by mode
- Produces: `SKILL.md`, the skill entrypoint; `openwiki/.last-update.json` format consumed by `references/update.md` and the E2E test (Task 8)

- [ ] **Step 1: Write the file**

Create `SKILL.md` with exactly this content:

`````markdown
---
name: openwiki
description: "Generate or maintain repository wiki documentation in openwiki/. Use when asked to create, initialize, refresh, or update a repo's wiki or codebase documentation."
---

# OpenWiki

Write and maintain documentation for the current repository under `openwiki/`, aimed at both humans and future coding agents. This skill ports the agent workflow of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki): you, the host agent, do all the work with your own tools — no external API, no separate runtime.

Do not invent files, modules, APIs, business rules, or behavior. Ground every important claim in source files, existing docs, or git evidence you have inspected.

## Pick a mode

1. If the user explicitly asked to initialize or to update, do that.
2. Otherwise: `openwiki/quickstart.md` exists → **update** mode; it does not exist → **init** mode.
3. Edge case: `openwiki/` exists but has no `quickstart.md` → run init, but read and preserve whatever is already there.

Before starting, read the matching reference from this skill's directory:

- init → `references/init.md`
- update → `references/update.md`

Both modes use `references/agents-md-section.md` when touching root instruction files. If the user asks for scheduled CI updates, read `references/ci-examples.md`.

## Discovery rules (both modes)

- Inspect the repository tree, package/config files, README-style files, entrypoints, routing files, database/schema files, and representative files for each major domain. Do not exhaustively read every file.
- Prefer targeted search and short reads over full-file reads when files are large. Exclude `.git`, `node_modules`, `dist`, `build`, cache directories, and `openwiki/` itself from discovery sweeps.
- If your harness supports read-only subagents (Claude Code: the Task tool), you may parallelize research: default 1-2 subagents for large or unfamiliar repos, 3-4 only when the repo is small/medium with naturally independent domains. Give each a narrow brief (existing docs, runtime architecture, data/storage, UI/API surface, integrations, tests, business workflows) and have them return concise findings with source paths and open questions. Subagents only inspect and summarize — you synthesize and do every write.
- Treat existing docs (README files, docs/ trees, runbooks, SKILL.md files) as primary sources. Summarize and link instead of duplicating. When docs conflict with source or git history, prefer source evidence and call out the likely stale doc.
- Stay inside the target repository. Never search parent directories or unrelated repositories.

## Git evidence (skip if not a git repository)

```bash
git status --short
git rev-parse HEAD
# update mode, when .last-update.json has a gitHead:
git log <lastHead>..HEAD --name-status --oneline
# fallback when only updatedAt exists:
git log --since <updatedAt> --name-status --oneline
# fallback otherwise, and for init:
git log --max-count=20 --name-status --oneline
git diff --name-status HEAD
```

Use git to explain why code exists, not only what exists: targeted `git log` / `git show` / `git blame` on high-signal files. Focus on recent, high-signal history; do not over-index on ancient commits. Account for uncommitted changes when they touch docs or important sources.

## Safety rules

- Never read or document secret values, credentials, private keys, tokens, or `.env` files. `.env.example` and other sample configs may be read only if they contain placeholders. If a secret-bearing file seems relevant, document only that such configuration exists.
- Write only under `openwiki/`. The only exceptions are top-level `AGENTS.md` and `CLAUDE.md`, and only for the OpenWiki reference section (`references/agents-md-section.md`).

## Planning file

After discovery and before writing final docs, write `openwiki/_plan.md`: intended pages, source evidence for each, remaining questions. Delete it before finishing — never leave it in the final wiki. (If your harness has a plan/todo tool, that may substitute; the delete rule still applies if the file exists.)

## Documentation goals

- A reader with zero knowledge starts at `openwiki/quickstart.md` and understands what the project is, how it is organized, what it does, and where to go next.
- A future agent can make high-quality changes with less source exploration: include where to start, what to watch out for, and which tests or checks matter for each area.
- Capture business/product logic as well as technical detail. Explain why, not only what.
- Clear Markdown with stable links between pages, organized like human documentation, not a file inventory.
- One canonical home per concept; other pages link to it. Keep the wiki concise enough to maintain.

## Metadata

`openwiki/.last-update.json`, written ONLY when wiki content actually changed during this run:

```json
{
  "updatedAt": "2026-07-07T09:00:00.000Z",
  "command": "init",
  "gitHead": "7d355379423172049308ba166ec2eff02c1c2e7d",
  "model": "claude-code"
}
```

- `updatedAt`: `date -u +%Y-%m-%dT%H:%M:%S.000Z`
- `command`: `init` or `update`
- `gitHead`: current `git rev-parse HEAD`; omit the key when not a git repo
- `model`: host model id if known, else the harness name (`claude-code` / `codex`)

## Final response

Summarize the completed documentation changes and important caveats. For update runs that changed nothing, say the wiki is already current.
`````

- [ ] **Step 2: Verify frontmatter parses and fields are correct**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && awk 'NR>1 && /^---$/{exit} NR>1{print}' SKILL.md | python3 -c "
import sys, yaml
d = yaml.safe_load(sys.stdin)
assert d['name'] == 'openwiki', d
assert isinstance(d['description'], str) and 'openwiki/' in d['description'], d
print('frontmatter OK')"
```

Expected: `frontmatter OK`.

- [ ] **Step 3: Verify every routed reference file exists**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && grep -o 'references/[a-z-]*\.md' SKILL.md | sort -u | while read -r f; do test -f "$f" && echo "$f OK" || echo "MISSING $f"; done
```

Expected output (4 lines, no MISSING):

```
references/agents-md-section.md OK
references/ci-examples.md OK
references/init.md OK
references/update.md OK
```

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: add openwiki core skill

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: UPSTREAM.md + CHANGELOG.md

**Files:**
- Create: `UPSTREAM.md`
- Create: `CHANGELOG.md`

**Interfaces:**
- Consumes: the pinned commit from Global Constraints; file mapping mirrors Tasks 1-5 outputs
- Produces: `UPSTREAM.md` (linked from `README.md`, Task 7); `CHANGELOG.md` (ship workflow appends here)

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
| `src/agent/prompt.ts` | `SKILL.md`, `references/init.md`, `references/update.md`, `references/agents-md-section.md` |
| `src/agent/utils.ts` (git evidence, metadata, no-op detection) | `SKILL.md`, `references/update.md` |
| `src/constants.ts` (path conventions) | `SKILL.md` |
| `examples/*.yml` | `references/ci-examples.md` |

## Sync procedure

1. Refresh a local upstream clone (this machine: `~/Projects/oss/openwiki`; if it is shallow, run `git fetch --unshallow` first, then `git pull`).
2. List relevant changes: `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/constants.ts examples/`
3. No output → report that the skill is current with upstream, and stop. Output → review each commit's diff and port the behavior changes into the mapped files.
4. Update the pinned commit above and add a CHANGELOG entry describing what was ported.
````

- [ ] **Step 2: Write CHANGELOG.md**

Create `CHANGELOG.md` with exactly this content:

````markdown
# Changelog

## Unreleased

- Initial `openwiki` skill: init/update wiki workflows ported from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) @ `7d35537` — targeted discovery, git evidence, surgical updates with no-op detection, AGENTS.md/CLAUDE.md reference section, `.last-update.json` metadata, scheduled CI templates.
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

### Task 7: README.md

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: `UPSTREAM.md` (Task 6), `references/ci-examples.md` (Task 4), `LICENSE` (already committed)
- Produces: `README.md`, the repo landing page

- [ ] **Step 1: Write the file**

Create `README.md` with exactly this content:

`````markdown
# openwiki-skill

An agent skill that writes and maintains repository documentation in `openwiki/` — a port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki)'s workflow for coding agents like Claude Code and Codex.

The upstream CLI drives an LLM through provider APIs. This skill drops all of that plumbing: your coding agent already is the LLM, with filesystem and git tools, so it executes the same workflow directly. No API key, no runtime, no configuration.

## Install

```sh
npx skills add JHSeo-git/openwiki-skill
```

Or manually: copy `SKILL.md` and `references/` into your agent's skills directory, e.g. `~/.claude/skills/openwiki/` for Claude Code.

## Use

Ask your agent:

- "Generate documentation for this repository" → creates `openwiki/` (init)
- "Update the wiki" → refreshes only the pages affected by recent changes (update)

Mode is auto-detected: no `openwiki/quickstart.md` → init; present → update.

## What you get

- `openwiki/quickstart.md` entrypoint plus focused section pages (architecture, workflows, operations, ... as fits the repo)
- An `## OpenWiki` reference section in root `AGENTS.md`/`CLAUDE.md` so agents consult the wiki first — byte-compatible with the upstream CLI
- `openwiki/.last-update.json` run metadata; update runs are surgical and no-op when the repository has not changed
- Scheduled CI templates in [references/ci-examples.md](references/ci-examples.md) — note that CI automation needs `ANTHROPIC_API_KEY` or a `claude setup-token` OAuth token, while local interactive use needs neither

## Upstream

Workflow and prompt rules derive from [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) (MIT). Pinned commit and sync procedure: [UPSTREAM.md](UPSTREAM.md).

## License

[MIT](LICENSE)
`````

- [ ] **Step 2: Verify every relative link target exists**

Run:

```bash
cd /Users/a78256/Projects/openwiki-skill && grep -o '](\([^)h][^)]*\))' README.md | sed 's/](//;s/)//' | while read -r f; do test -e "$f" && echo "$f OK" || echo "MISSING $f"; done
```

Expected output (3 lines, no MISSING):

```
references/ci-examples.md OK
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

### Task 8: End-to-end verification (spec scenarios 1-3)

**Files:**
- Create: fixture repo under `/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e/` (throwaway)
- Modify: any skill file that the scenarios prove wrong (fix + commit)

**Interfaces:**
- Consumes: the complete skill (Tasks 1-7)
- Produces: verified skill; fixes committed if scenarios expose defects

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

Act as the skill runtime: read `/Users/a78256/Projects/openwiki-skill/SKILL.md`, then follow it against the fixture repo (`cd "$FIX"` for all repo commands). No `openwiki/quickstart.md` exists, so it must take init mode and read `references/init.md`. Do exactly what the skill says — this is a test of the skill text, so if an instruction is ambiguous or contradictory while following it, note it for Step 6.

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

Expected: `quickstart OK`, `plan deleted OK`, `page budget OK (...)` (small repo → ≤3 including `quickstart.md`), `agents-md section OK`, `metadata OK`.

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

Then act as the skill runtime again (same as Step 2). `openwiki/quickstart.md` now exists, so it must take update mode and read `references/update.md`. Verify afterwards:

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

Act as the skill runtime once more, with no new commits. The no-op check in `references/update.md` must fire: the run reports the wiki is already current. Verify nothing was written:

```bash
FIX=/private/tmp/claude-501/-Users-a78256-Projects/a2c3f76a-cbde-46d3-a0f0-0cfa7b05962e/scratchpad/openwiki-e2e
cd "$FIX" && [ -z "$(git status --porcelain)" ] && echo "no-op OK" || { echo "NO-OP VIOLATED:"; git status --porcelain; }
```

Expected: `no-op OK`.

- [ ] **Step 6: Fix any defects the scenarios exposed**

If any check printed a failure, or following the skill text hit an ambiguity/contradiction: fix the responsible file(s) in `/Users/a78256/Projects/openwiki-skill`, re-run the failed scenario, then commit:

```bash
cd /Users/a78256/Projects/openwiki-skill && git add -A
git commit -m "fix: correct skill rules found by e2e scenarios

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

If everything passed on the first try, skip this step (no empty commits).

---

### Task 9: Publish to GitHub

**Files:**
- No file changes; remote creation + push only

**Interfaces:**
- Consumes: the complete, verified repo (Tasks 1-8)
- Produces: `https://github.com/JHSeo-git/openwiki-skill` (public), install path for `npx skills add JHSeo-git/openwiki-skill`

- [ ] **Step 1: Confirm clean tree**

```bash
cd /Users/a78256/Projects/openwiki-skill && git status --porcelain && git log --oneline
```

Expected: empty status; commit list from Tasks 1-8 plus the pre-existing docs/LICENSE commits.

- [ ] **Step 2: Create the public repo and push**

```bash
cd /Users/a78256/Projects/openwiki-skill && gh repo create JHSeo-git/openwiki-skill --public --source . --remote origin --push --description "Agent skill: generate and maintain repo wiki docs in openwiki/ (port of langchain-ai/openwiki)"
```

Expected: repo created, `main` pushed.

- [ ] **Step 3: Verify**

```bash
gh repo view JHSeo-git/openwiki-skill --json url,visibility --jq '.url + " " + .visibility'
```

Expected: `https://github.com/JHSeo-git/openwiki-skill PUBLIC`.

Then tell the user the repo is live and that they can smoke-test installation with `npx skills add JHSeo-git/openwiki-skill` (interactive; not run automatically).
