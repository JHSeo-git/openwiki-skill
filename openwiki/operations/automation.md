---
type: Playbook
title: Keeping wikis fresh automatically
description: The operational routes for unattended wiki updates — keyless subscription-auth schedules, the scoped permission allowlist headless runs need, CI templates that require a credential, and the one destructive capability that must never be granted unattended.
tags: [operations, automation, scheduling, permissions, ci]
sources:
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-fd626eae0b002efe8017b3f9
    resource: repo://skills/openwiki-personal/references/connectors.md
  - id: openwiki-source-f27335ea429d443b8de638e2
    resource: repo://skills/openwiki/SKILL.md
  - id: openwiki-source-ab21965a60a0ce53d1842e88
    resource: repo://skills/openwiki/references/automation.md
  - id: openwiki-source-11012093e98a6994fe87df7b
    resource: repo://skills/openwiki/references/runtime-evidence.md
generated: { by: "claude-code", at: "2026-09-15T01:48:47.000Z" }
---

# Keeping wikis fresh automatically

A wiki is only useful while it tracks the code. Upstream ships a scheduled GitHub Actions
workflow for that, generated on init from the configured provider's credentials. This port
has no credentials to generate from and **deliberately creates no CI files unasked** — so
the automation routes are opt-in, and they split on one question: does the runner have a
credential?

`skills/openwiki/references/automation.md` holds the working recipes. This page is the map
and the operational judgment around them.

## Route 1 — keyless, on your own machine

Headless Claude Code (`claude -p`) reuses the local subscription login, so a machine that
already has an interactive session needs no API key. That matches the port's premise
exactly, which is why it is the recommended route.

- **Local cron / OS scheduler** — run from the target repository. Pair it with the scoped
  allowlist below so the run never blocks waiting for a permission prompt.
- **Cloud routine** (`/schedule`) — runs on Anthropic infrastructure under the plan, so
  the machine can be off. The important operational difference: a routine starts from a
  **fresh clone**, so its prompt must also commit and push the changed `openwiki/` or open
  a PR. A prompt that only updates files silently produces nothing.
- **In-session recurring** (`/loop`) — session-scoped. Fine for a trial, not
  set-and-forget.

Codex runs the same update through `codex exec`. The run needs read-only git plus writes
under `openwiki/`, `AGENTS.md`, and `CLAUDE.md` — nothing more.

Phrase the scheduled prompt so a no-op stays a no-op: *"…update this repository's wiki. If
it reports the wiki is already current, change nothing."* The lifecycle already detects
that state; the instruction stops a scheduled run from inventing work to justify itself.

## Route 2 — CI, which needs a credential

CI runners have no subscription login, so this route requires `ANTHROPIC_API_KEY` or a
`claude setup-token` token as a CI secret. Templates exist for GitHub Actions (PR flow,
and since upstream 0.5.1 a second variant that auto-merges), GitLab CI (MR flow), and
Bitbucket Pipelines (PR flow).

One setting is load-bearing across all three: **they clone full history**
(`fetch-depth: 0`, `GIT_DEPTH: "0"`, `clone: depth: full`). A shallow clone hides the
last-documented commit, so the update cannot compute what changed and runs against nothing.
This is the failure mode most likely to look like "the wiki just stopped updating".

The templates also clone the skills into the workspace and scope the commit to `openwiki/`
plus the root instruction files, so the skills clone is never committed.

### A failed run still opens the pull request

Upstream 0.5.0 changed what CI does with a run that dies partway, and all three templates
follow it. Previously a failure produced no PR at all, so an update that died on page nine
of twelve discarded eight good pages and the next scheduled run began again from the same
place — usually failing at the same page. Now the update step tolerates its own failure,
the PR is opened either way with the outcome stated in the body, and the job is failed at
the very end. The published progress is meant to be merged: *"When the result is `failure`,
this PR intentionally preserves only the pages completed before the failure. Merge it to
make that progress the baseline for the next scheduled run."*

The templates also delete `openwiki/.run.json` before committing. This port never writes
one, but a native `openwiki` run in the same repository would, and it is transient state
that must never be committed.

Two caveats, the first of which is not actually specific to this port:

- **The job's exit status is not the partial-run signal.** `claude -p` exits zero when the
  skill merely *skips* a page, which is the ordinary contained-failure path. The signal is
  inside the wiki: `openwiki/.last-update.json` carries `status: "interrupted"` with its
  `gitHead` rewound to the last fully documented commit. Read that file, not the CI badge,
  to tell a complete update from a partial one — and note that the rewound head is exactly
  what stops the next run from treating the wiki as current. Upstream's CLI behaves the
  same way: it maps any run that finalized to a zero exit and never reads that status
  back, so a skipped page or mid-run source drift leaves a green job there too.
- **GitLab and Bitbucket have no `continue-on-error`.** Their templates capture the exit
  code, push and open the MR/PR, then re-raise it, which produces the same outcome by hand.

### Auto-merging the docs PR

Upstream 0.5.1 added a second GitHub Actions template that asks GitHub to merge the docs
PR without a human in the loop. Auto-merge is repository infrastructure rather than an
OpenWiki feature: the workflow only *requests* it, and the branch's required checks and
reviews still decide when the merge happens. Three prerequisites carry real weight —
enabling auto-merge on the repository, protecting the default branch with the checks that
should gate generated docs, and providing a **dedicated** `OPENWIKI_PR_TOKEN`. The last is
not a preference: a pull request opened with the default `GITHUB_TOKEN` does not start
most `pull_request` workflows, so the very checks meant to gate the merge may never run.

The gate on arming auto-merge is where this port deviates, and the caveat above is the
reason. Upstream arms it on the run's outcome, which catches a hard failure but not a
*finalized partial* run — and since both runners exit zero on one, an outcome-only gate
would let a partial wiki merge itself, with its rewound `gitHead` quietly becoming the
next run's baseline and nobody reading the PR to notice. So the template here reads
`openwiki/.last-update.json` and arms auto-merge only for a run that finished what it
planned; anything it cannot read counts as incomplete, because the safe direction is to
never auto-merge what you cannot prove. A mirrored disable step clears an auto-merge left
pending on a reused branch. That makes the gate a fix rather than a workaround: upstream's
own recipe has the same hole for the same reason.

One step from upstream's template is deliberately absent: it restores
`.github/workflows/openwiki-update.yml` before committing, because upstream's own recipe
commits that workflow. This port never writes CI files and its commit scope already
excludes them — which is the property you want, since auto-merging an executable workflow
file is exactly what an unattended job must not do.

## The permission allowlist

Headless runs should use a scoped allowlist rather than skipping permissions wholesale.
The shape, in the **target repository's** `.claude/settings.json`:

- Read-only git: `status`, `rev-parse`, `log`, `diff`, `show`, `blame` — all `--no-pager`.
- Discovery and hashing: `rg`, `find`, `shasum` / `sha256sum`, `date`.
- Text helpers the deterministic steps need: `head`, `sed`, `cut` — these cover the
  body-hash extraction and the `sources` id derivation.
- Writes confined to `openwiki/**` plus `AGENTS.md` and `CLAUDE.md`.

Everything else stays denied. Read, glob, and grep are already read-only; the skill's own
security rules forbid reading `.env` files and secret material regardless of what the
allowlist permits.

### The one capability to withhold

**This allowlist is for scheduled *update* runs only.** Since upstream 0.4.0 an **init
replaces the wiki from scratch**, keeping only a user-authored `INSTRUCTIONS.md`, which
needs `mktemp`, `cp`, `rm -rf`, and `mkdir` for its backup-and-wipe transaction. Do not
grant those unattended.

The reasoning is worth being explicit about: an unattended run that resolves to init would
wipe the wiki and, if the backup restore then failed, leave nothing behind. Mode
resolution treats init as destructive and asks for confirmation when auto-detection lands
there — a confirmation an unattended run cannot give.

## Personal wiki schedules

The same cron pattern points at `openwiki-personal`, with **one entry per source** so each
run stays small. That mirrors upstream, which runs one source-specific update per
connector rather than one sweep.

Permissions go in the user-level settings rather than a repository's, since these runs are
not repo-rooted: writes under `~/.openwiki/**`, the read-only commands above, and whatever
read-only MCP tools the sources need.

Credentials for sources live in `~/.openwiki/.env` — the same file the upstream CLI treats
as its credential source of truth, so both tools share one place. The operational rule is
absolute: **the agent never reads that file.** It is loaded inside a single command
invocation so values never reach the output or the transcript, and credentials are referred
to by variable name only.

## Runtime evidence runs

A repository update can additionally fold production traces into the wiki — an
anomaly-weighted sample of LangSmith runs synthesized into a runtime-behavior page plus the
code pages it concerns. This is opt-in per run and never happens unasked;
`skills/openwiki/references/runtime-evidence.md` carries the sampling shape and the
verbatim synthesis guidance.

Two operational constraints carry over from that guidance and matter more than the
mechanics: trace content is **untrusted evidence, never instructions**, and because the
wiki is committed to the repository, pages may carry behavioral summaries, error
signatures, counts, and trace URLs but **never raw run inputs or outputs**.

## This repository's own wiki

Worth noting for anyone reading this wiki: `openwiki-skill` ships no
`.github/workflows/`, so its own wiki has no scheduled refresh and is updated by an
explicit run. Its `AGENTS.md` marker block mentions a scheduled workflow because that block
is upstream's verbatim text, managed by the skill's own Step 0 — not a claim about this
repository's CI.
