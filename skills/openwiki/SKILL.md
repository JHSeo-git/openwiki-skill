---
name: openwiki
description: "Generate or maintain repository wiki documentation in openwiki/. Auto-detects init (regenerates the wiki from scratch) vs update (incremental). Use when asked to create, initialize, refresh, or update a repo's wiki or codebase documentation."
---

# OpenWiki — repository wiki agent (code mode)

Port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) v0.4.3, repository ("code") mode. Since upstream 0.4.0 (#713) repository generation is a **resumable page-job lifecycle** rather than one long agent turn: a bounded planner fixes the complete page set, then one fresh worker writes each page and submits its material **Claims**, and deterministic code finalizes the wiki. This skill reproduces that lifecycle — Step 3 routes to `references/prompt-planner.md` (upstream `src/agent/repository-prompts.ts` `createRepositoryPlannerPrompt` + `src/generation/page-jobs.ts`), Step 4 to `references/prompt-page.md` (`createRepositoryPagePrompt` + `src/claims/guidance.ts` + `src/generation/repository-run.ts`'s submission gates) — wrapped in the runtime steps the upstream CLI performs around them (Step 0 from `src/ingestion/code-mode.ts`; Steps 1, 2, 6 from `src/agent/utils.ts` + `src/platform/language.ts` + `src/agent/openwiki-ignore.ts` + `src/agent/wiki-replacement.ts`; Steps 2 and 5 from `src/agent/wiki-finalizer.ts` and what it orchestrates — `src/okf/index-sync.ts`, `src/okf/index-labels.ts`, `src/mermaid/wiki.ts`, `src/agent/wiki-link-validator.ts`, `src/okf/claim-sources.ts`, `src/okf/generated-provenance.ts`). You are the agent; the current repository is the target. No CLI, no API key — you do the work with your own tools.

Harness adaptations are marked **[adapted]**; upstream content with no equivalent here is marked **[omitted]**. Everything else is upstream text — keep it that way so upstream syncs stay line-mappable (see `UPSTREAM.md` in this skill's source repo). Upstream's personal knowledge wiki ("local-wiki" mode at `~/.openwiki/wiki`) is ported as the separate `openwiki-personal` skill; wiki Q&A as `openwiki-ask`.

## Mode resolution

- The user explicitly asks to initialize / build from scratch → **init**.
- The user explicitly asks to update / refresh → **update**.
- Otherwise auto-detect: `openwiki/quickstart.md` exists → **update**; it does not → **init**.
- **Init is destructive since upstream 0.4.0 (#699):** it regenerates the wiki from scratch, keeping only the user-authored `openwiki/INSTRUCTIONS.md`. Step 2 performs that replacement. If auto-detection lands on init because `openwiki/` exists without a `quickstart.md`, say so and confirm before wiping — the user may have meant update.
- The user asks to migrate the wiki to OKF / fix wiki front matter → run **update**: Step 2's normalization pass migrates every non-compliant page deterministically. The request counts as an additional user instruction, so Step 1's early no-op exit does not apply.
- The user asks for the wiki in a specific language ("write the wiki in Korean", "switch the wiki to zh-CN") → that is the run's **requested output language**, resolved in Step 1 (upstream: the `--language` flag). Since 0.4.0 a language switch on an update is no longer a separate translation pass: it forces the run (#548) and Step 3 injects a rewrite job for every existing page, so the page workers rewrite them in the new language. The request counts as an additional user instruction, so Step 1's early no-op exit does not apply.
- Step 1 finds **stale or unresolved page evidence** → run **update** even on a clean tree: upstream runs Claims validation before no-op detection precisely so a clean `git status` cannot hide stale grounding.
- The user asks to pull LangSmith runtime traces / production run evidence into the repo wiki → **runtime evidence update run**: read `references/runtime-evidence.md` in this skill's directory and follow it (every step here still applies; the file's guidance block becomes Step 3's planning context).
- Any other instruction in the user's request (e.g. "focus on the API routes") is an **additional user instruction**. Upstream passes it as the run's `planningContext` and it *forces* the run (`force: Boolean(userMessage)`), skipping the no-op check — carry it into Step 3's planner prompt as the "User and connector planning context" block, and let the planner copy the relevant parts into the affected pages' `instructions`.

## Model tier

Upstream defaults to frontier coding models. Documentation quality depends on it — run this skill on a frontier tier, not a small/fast model.

## Step 0 — Code setup (ported from upstream `code-mode.ts`)

Upstream performs this repository setup on every code-mode invocation, outside the agent. Do it at the start of every init/update run, before Step 1:

1. Ensure `/AGENTS.md` and `/CLAUDE.md` each carry **its** snippet below (upstream `CODE_MODE_AGENT_FILES` — each file is created when missing and refreshed in place when already present; since 0.3.3, #640, the two files get different snippets: AGENTS.md the full block, CLAUDE.md a minimal pointer to it, so one file stays the canonical source of agent instructions while Claude Code still has a file it reads at startup):
   - Markers `<!-- OPENWIKI:START -->` / `<!-- OPENWIKI:END -->` present → the file must contain exactly one `START` marker followed by exactly one `END` marker (since 0.3.0, #547). Then replace everything between and including the markers with the file's snippet — skip the write when the existing block is already identical (**[adapted]** upstream rewrites unconditionally; the byte outcome is identical). Malformed or duplicated markers → fail the setup with upstream's error (`Cannot update <file> because its OpenWiki managed markers are malformed or duplicated. Expected either no markers or exactly one <!-- OPENWIKI:START --> marker followed by one <!-- OPENWIKI:END --> marker. Repair or remove the markers and retry; the file was left unchanged.`) and write NEITHER file — upstream validates both files before writing either, so a malformed sibling leaves both untouched.
   - **[adapted]** A legacy `## OpenWiki` section without markers (written by pre-0.1.0 versions of this skill) present → replace that section with the file's snippet instead of appending a duplicate (upstream never sees this state; this port migrates it).
   - Neither present → append the file's snippet to the end of the file, separated by one blank line; if the file does not exist, create it containing only the snippet.
2. **[adapted]** Exception for `/CLAUDE.md`: if it imports AGENTS.md (an `@AGENTS.md` line) and `/AGENTS.md` carries the snippet, it counts as covered — do not add the pointer block (Claude Code loads AGENTS.md through the import, which already does the pointer's job; upstream has no import concept).
3. **[omitted]** Upstream also creates `.github/workflows/openwiki-update.yml` (a scheduled `openwiki code --update --print` workflow that needs a provider API key) — since 0.2.3 only `openwiki code --init` creates it, and an existing file is never overwritten, so operator customizations survive. This keyless port does not create CI files unasked — for scheduled updates read `references/automation.md`.

The AGENTS.md snippet — keep byte-identical to upstream `createCodeModeAgentsSnippet()`:

```markdown
<!-- OPENWIKI:START -->

## OpenWiki

This repository has a generated `openwiki/` evidence index. It is optional just-in-time context, not required startup reading.

- Treat source code and tests as authoritative. A brief's unknowns and review items are verification gaps, not automatic requirements.
- Prefer the narrowest quiet validation that proves the changed behavior. Preserve complete failure output.

The scheduled OpenWiki GitHub Actions workflow refreshes the repository wiki. Do not hand-edit generated OpenWiki pages unless explicitly asked; prefer updating source code/docs and letting OpenWiki regenerate.

<!-- OPENWIKI:END -->
```

The CLAUDE.md snippet — keep byte-identical to upstream `createCodeModeClaudeSnippet()` (since 0.3.3):

```markdown
<!-- OPENWIKI:START -->

## OpenWiki

See [AGENTS.md](AGENTS.md) for OpenWiki agent instructions.

<!-- OPENWIKI:END -->
```

Only Step 0 may touch `/AGENTS.md` and `/CLAUDE.md`. The documentation run itself never does.

**[omitted]** Upstream 0.4.0 also ships `openwiki install <host>` (#685, #711), which installs a bundled `openwiki` skill plus an MCP server into Codex, Claude Code, or OpenCode so those hosts drive the same lifecycle through `openwiki_begin` / `openwiki_submit_plan` / `openwiki_next_page` / `openwiki_submit_page` / `openwiki_finish` tools. This port *is* that integration without the runtime: the behavioral contract from upstream's bundled `integrations/openwiki/SKILL.md` is folded into Steps 3–5 and `references/prompt-page.md`, and the lifecycle's code-owned bookkeeping is this skill's deterministic steps.

## Step 1 — Run context, evidence preflight, and early no-op check (before any write; all git read-only; use `git --no-pager`)

Read `openwiki/.last-update.json` if it exists to recover `gitHead`, `updatedAt`, `status`, `language`, and `model` (upstream `readLastUpdate`: an unreadable or structurally invalid file counts as no metadata; a `status` other than `"interrupted"`, including the field being absent from pre-0.2.4 metadata, counts as `"complete"`). Read `openwiki/INSTRUCTIONS.md` if it exists — the user-authored OpenWiki brief for this repository, rendered into Step 3's planner prompt as "Repository OpenWiki instructions" (upstream `readRepositoryWikiInstructions`; absent or empty → the block is omitted entirely).

**Resolve the wiki output language** (ported from upstream `resolveLanguage` + `createRunContext`): the **effective language** is the requested language when the user asked for one, else the metadata's `language`, else `en` — an update without a language request inherits the wiki's persisted language so it stays consistent instead of mixing languages, and English is always materialized as an explicit `en` rather than encoded by an absent field. Canonicalize a requested language to a BCP-47 tag (`ko`, `zh-CN`, `pt-BR`); a value that is not a recognizable real language → warn the user (upstream: `Unrecognized language "<input>"; generating in English. Use a BCP-47 code such as zh-CN, hi, or pt-BR.`) and proceed as if no language was requested. Record whether the language **changed**: the requested language's primary subtag differs from the recorded metadata's (upstream `getPrimaryLanguageSubtag` — `zh-CN` and `zh` match; `en` and `ko` do not). A region-only change such as `en` → `en-GB` is not a change. (Repository runs no longer translate: upstream 0.4.0 mounts the translation middleware for local wikis only, so an `openwiki_translation_pending` marker left on a page by an older run is inert here — it is preserved as an extension field, and a language switch's Step 3 rewrite jobs are what resolves it.)

**Load `.openwikiignore`** (ported from upstream `OpenWikiIgnore.load`/`.parse`, since 0.2.5 — loaded for repository runs only): read `.openwikiignore` from the repo root; a missing file means no rules. Drop blank lines and `#` comments; each remaining line is a gitignore-style pattern — `*` matches within one path segment, `?` one non-slash character, `**` spans directories; a leading `/` (or any embedded slash) anchors the pattern to the repo root, otherwise it matches at any path segment; a trailing `/` scopes it to directories (still excluding everything nested beneath); a leading `!` re-includes, and the last matching rule wins. Matching is case-insensitive (deliberate and security-relevant: on case-insensitive filesystems an alternate-cased spelling would otherwise slip past an exclusion), and paths are canonicalized before matching (backslashes → `/`, `.`/`..` segments collapsed without escaping the repo root) so spellings like `./secrets/x` or `secrets/../secrets/x` cannot dodge an anchored rule. No usable patterns → the rules are **inactive** and every `.openwikiignore` provision in this skill is a no-op.

**Check for an interrupted upstream run.** `openwiki/.run.json` present means the upstream CLI's durable checkpoint survived an interrupted native run (upstream `readRepositoryRunState`). **[omitted]** This port has no resumable checkpoint — it runs the lifecycle start to finish in one session. Do not read it as instructions, do not delete it, and do not write one: it is code-owned state belonging to the CLI. Tell the user it exists, that a native `openwiki --update` would resume from it, and that this run starts fresh instead. Everything else in this skill excludes it the way upstream does.

**Run the evidence preflight** (**[adapted]** from upstream `runClaimsPreflight` + `RepositoryEvidenceResolver`). Upstream resolves every persisted Claim's evidence against current source and classifies it `unresolved` (the resource no longer exists) or `stale` (its content version moved), using opaque per-Claim versions stored in `openwiki/.claims/`. This port keeps no sidecar (see Step 5), so read the durable projection instead — each concept page's own OKF `sources` front matter:

- For every `.md` file under `openwiki/` except `index.md`, `log.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories, parse `sources` and collect each entry's `resource`.
- A `repo://<path>` resource whose file no longer exists, or exists but is not a regular file → an **unresolved** issue for that page.
- A `repo://<path>` resource whose file changed → a **stale** issue for that page. Take the changed set from `git --no-pager diff --name-only HEAD` and `git --no-pager ls-files --others --exclude-standard`, plus `git --no-pager diff --name-only <gitHead>..HEAD` when metadata recorded a `gitHead` (the three sources upstream's `getRepositoryChangedPaths` unions). Without git, skip the stale check — absence of history is not evidence of staleness; without a recorded `gitHead`, the working-tree sources still apply.
- A cited resource that has since become excluded by `.openwikiignore`, or that resolves through a symlink, is a **hard error** upstream: the resolver throws and the whole run fails rather than let unverifiable grounding pass as merely missing. **[adapted]** Do not fail the run — the user asked for a wiki update, not a diagnosis. Treat it as unresolved, and say explicitly which page cites which now-unreadable resource, so the user can fix the rule or the page instead of silently losing that page's grounding.
- Sort issues by page, then kind, and carry them into Step 3 as the planner's "Claims requiring attention" block. Fidelity gap worth knowing: this is per-page and per-file, where upstream is per-Claim and content-versioned against opaque resolver tokens, so it cannot distinguish a line range that merely moved from one that was rewritten, it never reports the "range can no longer be located" flavour of unresolved, and a page whose cited file changed anywhere is flagged.
- **Init runs skip the preflight entirely** (upstream `freshInit`): Step 2 replaces the wiki, so there is nothing to reconcile.

**Then run the early no-op check** (update mode only — ported from upstream `getUpdateNoopStatus` as called from `beginRepositoryRun`). Upstream deliberately orders it *after* the evidence preflight so a clean tree cannot hide stale grounding.

- The user gave an additional instruction → not a no-op (upstream forces the run).
- No recorded metadata or no recorded `gitHead`, or not a git repository → not a no-op; proceed. (Without git, infer changes during Step 3 from filesystem timestamps, source inspection, and existing docs.)
- Recorded `status: "interrupted"` → do not skip; the previous run may have left a partial wiki, so it must be retried (#365).
- The output language changed (above) → not a no-op (#548): the pages have to be rewritten in the new language.
- The preflight found any issue → not a no-op.
- Otherwise run:

```bash
git --no-pager status --short --untracked-files=all
git --no-pager rev-parse HEAD
```

- The worktree must be clean, ignoring the `openwiki/.last-update.json` status line itself and — when ignore rules are active — status lines whose paths are excluded by `.openwikiignore` (a rename line counts when either its old or new path matches). Any other line → not a no-op.
- HEAD equals the recorded `gitHead` → **no-op**. HEAD moved → run `git --no-pager diff --name-only <gitHead>..HEAD`; every changed path lies under `openwiki/` or is excluded by `.openwikiignore` (at least one such path — if git reports no changed paths, do NOT treat it as a no-op) → **no-op**. Anything else → proceed.
- **On a no-op, still refresh the metadata** (#647, new in 0.4.0): rewrite `openwiki/.last-update.json` so freshness checks reflect the actual last run, carrying the previous run's `model` and `language` forward unchanged, with a fresh `updatedAt`, the current `gitHead`, `command: "update"`, and `status: "complete"` — the last of which also clears a previous `interrupted` status. Then report that the wiki is already current and stop: Steps 2 through 6 do not run, and this refresh is the run's only write.
- (Step 0 still runs before this exit — upstream refreshes the repo setup even on no-op runs.)

## Step 2 — Prepare the wiki (before the work)

**Init runs only: replace the existing wiki** (ported from upstream `beginRepositoryWikiReplacement` in `src/agent/wiki-replacement.ts`, new in 0.4.0 / #699). `openwiki/INSTRUCTIONS.md` is user-owned control metadata; everything else under `openwiki/` is generated state and is removed before the run sees the repository:

- No `openwiki/` directory → nothing to do.
- `openwiki` exists but is not a real directory, or is a symlink → refuse, with upstream's error: `Refusing to replace openwiki: expected a real directory below the repository root.` Same refusal for an `openwiki/INSTRUCTIONS.md` that is not a regular file: `Refusing to preserve openwiki/INSTRUCTIONS.md: expected a regular file.`
- Otherwise: copy `openwiki/` to a private temp directory as a recovery backup, delete the whole tree, recreate `openwiki/` empty, and copy `INSTRUCTIONS.md` back if the backup has one. Tell the user where the backup is. If the run fails or is cancelled before it finishes, restore the backup over `openwiki/` — upstream rolls back on SIGINT/SIGTERM for exactly this reason. Discard the backup once the run completes.
- **[adapted]** Upstream drops the backup as soon as its `.run.json` checkpoint is durable, because partial pages then become the recovery mechanism. This port has no checkpoint, so keep the backup for the whole run — it is the only recovery path.

**Then snapshot the wiki content** (ported from upstream `createOpenWikiContentSnapshot`):

```bash
find openwiki -type f ! -name '.last-update.json' ! -name '.run.json' -print0 2>/dev/null | LC_ALL=C sort -z | xargs -0 shasum -a 256 2>/dev/null | shasum -a 256
```

Record the hash. (`shasum -a 256` covers macOS and most Linux; on minimal Linux images substitute `sha256sum` in both places. Upstream's snapshot ignores only run metadata — `.last-update.json` and, since 0.4.0, the `.run.json` checkpoint; `_plan.md` is gone from the exclusion list because the plan file itself is gone.) You will recompute it in Step 6. **[adapted]** The hash is compared only within this run — upstream never persists it. Upstream's snapshot additionally hashes directory entries and scopes the metadata exclusions to the wiki root; this one-liner's changed/unchanged verdict differs only on states documentation runs don't produce (empty directories, nested metadata files).

**[adapted]** Upstream also fingerprints every model-visible repository source file here (`createRepositorySourceFingerprint`). Until 0.4.2 a mid-run change invalidated the whole plan and forced a replan; since 0.4.3 (#740) it does not — the run **finalizes once** and records that a later update is due instead, on the same reasoning as a skipped page: a partially refreshed wiki plus a forced follow-up beats discarding the work. Do the cheap equivalent here: record the current `git rev-parse HEAD` and worktree state now, re-check it **twice** — once before Step 5's passes and once at Step 6, since a change that starts mid-finalize counts too — and if either check differs, treat the run as drifted. Step 6 says what that costs.

**Then repair the wiki's front matter** (ported from upstream `repairOkfFrontmatter`, which `migrateWikiToOkf` delegates to, run by `prepareWikiForAuthoring` so the run operates over an already-conformant wiki). Since 0.4.1 (#728) this is a **field-by-field repair, not a rebuild**: invalid *optional* metadata used to abort a run, and now it is repaired or removed deterministically instead. Every path below ends in a page that passes OKF validation. For every `.md` file under `openwiki/` except `index.md`, `log.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories:

1. **The page already passes OKF validation → leave it byte-for-byte unchanged.** (This is a stronger test than 0.4.0's "parses and has a non-empty `type`": junk optional fields are no longer tolerated, they are repaired.)
2. **Otherwise, if the YAML block still parses to a mapping → repair in place**, keeping every other line and every producer extension field as written:
   - `type` missing or not a non-empty string → set the fallback `type` **and** set `openwiki_generated: true`.
   - `title` → set the derived title when either the `type` was just derived and no `title` key is present, or a `title` key is present but is not a non-empty string. A valid page missing only `title` is left alone — `title` is optional.
   - `description`, `resource`, `timestamp` present but not a non-empty string → **remove the field**.
   - `tags` present → keep only the non-empty string entries; nothing left → remove the field.
   - `generated` present but not a valid `{by, at?}` actor event → **remove it**. A trust assertion that cannot be proven conformant is removed rather than rewritten into a false one.
   - `verified` present → keep only the conformant events (re-rendered as a list).
   - `sources` present → keep only the entries with a non-empty string `resource`.
   - `status` not one of `draft`/`stable`/`deprecated`, or `stale_after` not an ISO 8601 datetime with an explicit offset → remove the field.
3. **If the YAML block cannot be parsed, or the repair still does not validate → replace the front matter with exactly this minimal derived block** (one blank line after the closing `---`, then the body **as-is** — since 0.4.1 its leading whitespace is no longer trimmed, so a page that had an unusable block may keep a blank line the old rule removed):

```markdown
---
type: "Reference"
title: "<first ATX H1 in the body; fallback: filename without .md, -/_ runs → spaces, first character upper-cased>"
openwiki_generated: true
---
```

- Values are JSON-double-quoted. `openwiki_generated: true` flags code-derived metadata; the page's Step 4 worker should replace it with accurate metadata grounded in the page body and then remove the field.
- Non-English wiki language → the derived `type` is that language's localized label from `references/index-labels.md` instead of `"Reference"` (upstream `resolveConceptTypeLabel`: full tag → primary subtag → English fallback).
- Upstream's 0.4.0 carry-across lists (`PRESERVED_EXTENSION_FIELDS`, `PRESERVED_STRUCTURED_FIELDS`) are **gone in 0.4.1** and this port drops them too: path 2 preserves the original block outright, so nothing needs carrying. The consequence is worth knowing — on path 3, where the YAML is unusable, `openwiki_translation_pending` and the code-owned `generated` / `verified` / `sources` families are **lost**, because they cannot be read back from a block that does not parse.

**Then capture the generated-provenance baseline** (ported from upstream `snapshotGeneratedProvenance`, new in 0.4.0 / #581, #684). For every concept page (same exclusions), record two things — Step 5 needs both:

1. the SHA-256 of its **body**, i.e. the content after the leading front-matter block, whitespace included:

```bash
# Set `page` per file and re-run. Deliberately a variable, not a shell
# function taking a positional parameter: a dollar sign followed by a digit
# is substituted with this skill's invocation arguments before the agent
# ever reads the file, which would silently corrupt the command.
page=openwiki/quickstart.md
if [ "$(head -n1 "$page")" = "---" ]; then sed '1,/^---$/d' "$page"; else cat "$page"; fi | shasum -a 256
```

2. its existing `generated` event, when it has a valid one (a mapping with a non-empty string `by`, and an `at` that is a non-empty string when present).

## Step 3 — Plan the run (planning phase)

Read `references/prompt-planner.md` in this skill's directory and act on it exactly. It carries the planner system prompt, the plan payload schema, and the validation rules the plan must satisfy — including the required jobs an update run has to inject for Step 1's preflight issues and for a language change.

Shared adaptation conventions (this and Step 4's file assume them): (a) upstream's `/`-rooted virtual paths stay the wiki's canonical page identifiers in the plan and the queue, and resolve to real repo-relative files when you touch the filesystem — `/openwiki/quickstart.md` is `openwiki/quickstart.md`; (b) upstream enforces every boundary in code (`src/agent/docs-only-backend.ts`: writes confined to `openwiki/`, and since 0.4.0 confined further to the *one* page a worker owns; `.claims` state hidden from every generic tool and from shell; reads/edits of `.openwikiignore` paths hard-denied and `ls`/`glob`/`grep` results filtered; unbounded root globs and `.git` targets rejected) — here they are hard rules you follow; (c) upstream's lifecycle tools (`submit_plan`, `submit_page`) do not exist for you, so their payloads become records you validate yourself against the same gates; (d) template placeholders are rendered in place. **[omitted]** Upstream's chat-mode prompt (wiki-first question answering + the OpenWiki CLI reference) is the `openwiki-ask` skill's domain; connector-fed personal wikis are the `openwiki-personal` skill's.

An update whose plan has no pages and no deletions is legitimate: skip Step 4 and go straight to Step 5.

## Step 4 — Work the page queue (generating phase)

Read `references/prompt-page.md` in this skill's directory and act on it once per planned page, in the queue order Step 3 fixed (all other pages first, `/openwiki/quickstart.md` last). It carries the page-worker system prompt, the Claims substance and reconciliation standards, the submission gates each page must pass before you move on, and upstream's non-negotiable boundaries.

Two boundaries deserve repeating here because they replace behavior earlier versions of this skill had:

- **No subagents.** Upstream 0.4.0 deleted `skeleton_critic`, `wiki_question_finder`, and `wiki_answer_verifier`, strips the delegation tool from every worker, and its bundled host skill says plainly not to spawn planning, page, reviewer, critic, or QA subagents. Work the queue yourself, sequentially.
- **No working files.** `openwiki/_plan.md` and `openwiki/_skeleton.md` no longer exist; any `_`-prefixed page path is rejected. The plan lives in your working notes, not in the wiki.

### A page you cannot finish is skipped, not fatal (since 0.4.1, #732)

Before 0.4.1 a page worker that could not complete aborted the whole update, throwing away every page already written. Now the failure is contained to its own page. Follow the same protocol:

1. **Snapshot the page before you touch it** (upstream `captureRepositoryPageSnapshot`): record its exact current Markdown, or that it does not exist yet. **An absent page is a valid snapshot, never a failure** — it is the normal case on init and for every newly planned page, and upstream had to fix exactly this in 0.4.2 (#737). **[adapted]** Upstream also snapshots the page's `.claims` sidecar; this port has none (Step 5), so the Markdown is the whole snapshot.
2. **If you cannot complete the page** — you cannot ground it, its gates keep failing, or the work is not converging — **restore the snapshot exactly**: write the recorded Markdown back, or delete the page if it did not exist before. Then mark the job **skipped**, tell the user which page and why, and move to the next job. Do not leave a half-written page behind.
3. **Keep going.** Skipped jobs do not block the rest of the queue, and they do not block Step 5.
4. **One exception stays fatal**: a submission failure that is *not* a correctable input problem — a page you cannot persist at all — aborts the run. A page that merely fails validation is correctable, so fix it and re-check; a wiki you cannot write to is not.

Carry the list of skipped pages and their snapshots into Steps 5 and 6; both need it. **[adapted]** Upstream also resets a skipped job to pending in its durable checkpoint so the next run retries it. This port re-plans from scratch every run, so the retry is automatic — but Step 6's metadata is what makes the next run *look*, so do not skip it.

## Step 5 — Finalize (after the work; ported from upstream `finishRepositoryRun` + `src/agent/wiki-finalizer.ts`)

Upstream runs these deterministic passes in exactly this order on every init/update run. Do the same, after the page work and before Step 6, so their writes land in the Step 6 content hash. Skip the whole step only when Step 1 exited at the early no-op.

**First, restore every skipped page.** For each page Step 4 skipped, write its snapshot back one more time before anything else runs (upstream `finishRepositoryRun` re-restores them at this point, because a later pass may have touched them). A skipped page must reach Step 6 byte-identical to how this run found it. Then treat those pages as **excluded** for the rest of this step: they take no `sources` projection and no Claims reconciliation, since this run produced no Claims for them and projecting an empty set would strip the grounding the previous run recorded.

**Then apply deletions.** If a mid-run replan abandoned pages this run created that are in neither the final plan nor the pre-run page inventory, delete those (upstream `applyAbandonedGeneratedPageDeletions` — it never touches a page that existed before the run). Then delete each page in the plan's `deletePages`; a page that is already gone is not an error. Also re-run Step 2's front-matter repair over every concept page that no longer passes OKF validation, so index generation never fails on a non-compliant page.

**Then validate Mermaid diagrams** (ported from `validateWikiMermaid`): for every `.md` file under `openwiki/` except `index.md`, `log.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories (upstream `EXCLUDED_FILES`), check that every fenced ```mermaid block parses (a ```mermaid example nested inside a longer outer fence does not count):

- **[adapted]** Upstream parses each fence with the real Mermaid parser when its optional `mermaid` + `jsdom` peers are installed, and otherwise falls back to a conservative heuristic that only flags near-certain breakages (a `flowchart`/`graph` node id named `end`; a semicolon inside a `[]`/`()`/`{}` label; an unescaped angle bracket inside a label). Here, run the check yourself: apply that heuristic plus the `mermaid-diagrams` skill's syntax-safety rules — or a locally installed Mermaid parser when one is available.
- **[adapted]** A broken fence you can confidently repair (you usually wrote it this run) → fix it in place. Otherwise degrade it exactly as upstream does: replace the ```mermaid fence with a ```text fence holding the same body, preceded — at the fence's indentation — by a one-line HTML comment: `<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: <one-line reason> -->`. A later update run repairs it.
- Files whose fences all parse are left byte-for-byte unchanged, so this pass creates no diff noise.

**Then synchronize directory indexes.** For every directory under `openwiki/` (recursively, skipping dot-directories — the wiki root itself included), regenerate its `index.md`:

1. Collect the directory's direct children:
   - Files: every `.md` file directly in it except `index.md`, `log.md`, `INSTRUCTIONS.md`, and dot-files. For each, read its front matter — link label = `title` when it is a non-empty string (fallback: the filename without `.md`), and keep `description` when it is a non-empty string (unusable optional fields are ignored, not errors).
   - Directories: every subdirectory whose name does not start with `.`.
2. Render exactly this shape — **no front matter** (`index.md` is a reserved OKF document), except the wiki root's index, which starts with exactly the three-line `okf_version` block shown; one blank line between sections; a section with no entries omitted entirely; when both sections are empty the sections part is just `# Files` (the root still keeps its okf_version block above it); trailing newline:

```markdown
---
okf_version: "0.2"
---

# Files

- [<label>](<URL-encoded filename>) - <description, only when the page has one>

# Directories

- [<name>](<URL-encoded name>/)
```

   - Non-root directories: the same content without the `okf_version` block.
   - Sort each list alphabetically by link target (upstream: `localeCompare`). Escape `\`, `[`, and `]` in labels.
   - The `Files` / `Directories` headings (including the empty-directory `# Files`) and the derived `type` that Step 2's normalization (re-applied above) stamps on repaired pages are the wiki language's labels from upstream's curated table (`resolveIndexLabels` / `resolveConceptTypeLabel`: full tag → primary subtag → English fallback). English (`en`, the default) uses `Files` / `Directories` / `Reference` — any other effective language → read `references/index-labels.md` in this skill's directory and use its row verbatim (curated structural chrome, never your own translation).
3. Compare with the existing `index.md` and write only when the content differs — byte-identical output is skipped, so no-op runs stay no-ops.

**Then validate internal links** (ported from upstream `validateWikiInternalLinks`; it stamps broken links in place and never fails the run). For every `.md` file under `openwiki/` except `index.md`, `log.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories:

1. Strip any previously inserted stamp lines — full-line HTML comments matching `<!-- openwiki: broken internal link ... -->` — so revalidation starts clean and a fixed link leaves no residual comment.
2. Collect every inline Markdown link `[text](dest)` with its line number, skipping image links (`![...]`). Ignore external destinations (any URI scheme, or protocol-relative `//…`) and empty ones. Drop a trailing Markdown link title (`path "Title"`), then split an optional `#anchor` off the path (URL-decode the anchor before comparing).
3. Validate each remaining link (since 0.3.1, #585, targets resolve **repo-wide**, not just within the wiki subtree — a wiki page may legitimately link out to any repository file, which renders correctly on GitHub; a link is broken only when its target genuinely does not exist):
   - Anchor-only (`#foo`) → the source file's own headings must expose the anchor. Anchors are GitHub-style slugs of ATX heading titles, matching `github-slugger` exactly: trim, lowercase, strip everything except Unicode letters/numbers/combining marks/spaces/`_`/`-` (combining marks kept so decomposed accents slug correctly), then replace each whitespace character with a hyphen — per character, not collapsed, so stripping the `&` from "A & B" leaves two spaces and the valid anchor is `a--b`; duplicate slugs get `-1`, `-2`, … suffixes. Missing → message `heading anchor "<anchor>" does not exist in <source path>`.
   - Path → resolve relative to the source file's directory (or the repository root when it starts with `/`), normalized — normalization clamps `..` at the repository root, so nothing escapes it; a path that cannot be resolved to an absolute one → `link "<path>" cannot be resolved`. A trailing `/` makes it a directory link — the directory must exist (`directory "<path>" does not exist`); otherwise the target file must exist (`file "<path>" does not exist`).
   - Path + anchor → the anchor is validated only when the target is a `.md` file (anchors on directories and on non-Markdown targets — e.g. GitHub `#L10` line anchors on source files — are never flagged): the target's headings must expose the anchor (same slug rules) → else `heading anchor "<anchor>" does not exist in "<path>"`.
4. Insert one stamp line directly above each broken link's line (insert bottom-up so line numbers stay valid; multiple broken links on one line get one stamp each), in upstream's exact format: `<!-- openwiki: broken internal link [<href>] <message>. Fix the href or restore the target, then delete this comment. -->`
5. Write a file back only when its content changed. A later update run repairs stamped links.

**Then project this run's Claims evidence into OKF `sources`** (ported from upstream `synchronizeClaimSources` in `src/okf/claim-sources.ts`, new in 0.4.0 / #692; skipped pages excluded, per above). Upstream re-projects every page its session holds Claim state for — which on an update is every page with a sidecar, not only the pages this run revisited. **[adapted]** Without a sidecar this port only knows the Claims its own Step 4 workers submitted, and that is sufficient: an unrevisited page's `sources` on disk already *is* its persisted projection, so re-deriving it would be a no-op write. So: for every concept page whose Step 4 worker submitted Claims (skipping any page the run deleted):

1. Collect that page's complete evidence-resource set and reduce each resource to its **whole-file** form — drop any `#Lx-Ly` fragment, so `repo://src/agent/index.ts#L40-L82` becomes `repo://src/agent/index.ts`. Precise ranges stay in the Claim; OKF provenance exposes source files.
2. Read the page's current `sources`. Keep every entry that is *not* OpenWiki-owned — that is, every entry whose `id` does not start with `openwiki-source-`. A malformed `sources` value (not a list, or entries without a non-empty string `resource`) counts as empty and gets repaired by this projection.
3. Deduplicate the projected resources, sort them (`localeCompare`), drop any already carried by a retained entry, and append one mapping per remaining resource with a deterministic id derived from the resource itself:

```bash
printf '%s' 'repo://src/agent/index.ts' | shasum -a 256 | cut -c1-24   # → the id suffix
```

   giving `id: openwiki-source-<that 24-hex-character digest>` — the example resource above yields `openwiki-source-a953060a04ccefcf777de48e`. Stable ids let a later run replace or remove only its own projection.
4. Since 0.4.1 (#728) the pass is self-healing and byte-stable: run Step 2's front-matter repair over the page **before** reading `sources`, apply the projection to the repaired content, run the repair once more over the result, and then compare the **final bytes against the page as it was read**. Write only when they differ — so a repair alone is enough to justify a write, and a projection that changes nothing is not. Replace just the `sources` field and leave every other front-matter line byte-for-byte. Rendered shape:

```yaml
sources:
  - id: openwiki-source-a953060a04ccefcf777de48e
    resource: repo://src/agent/index.ts
```

   An empty list removes the field. A page this run produced no Claims for is left untouched — this pass never strips another producer's or an earlier run's projection just because this run did not revisit the page.

**[omitted]** Upstream also persists each page's reconciled Claim set to `openwiki/.claims/<page>.json` and stamps an OKF `verified: [{by, at}]` event once a page's whole Claim set reconciles durably (`ClaimsStore`, `synchronizeClaimsVerification`). Both are skipped here: the sidecar's evidence versions are opaque resolver tokens (`repo-file-v1:sha256:…`, and for line ranges `repo-lines-v1:sha256:<content hash>:<base64url relocation anchors>`) that only the resolver can produce and interpret, and a `verified` stamp with no durable Claim state behind it would assert a machine verification that never happened. The `sources` projection above is this port's durable grounding record, and Step 1's preflight reads it back. Never hand-write `.claims` or `verified`.

**Finally, reconcile generated provenance** (ported from upstream `finalizeGeneratedProvenance`, new in 0.4.0 / #581, #684 — it runs last, after every other pass, so front-matter-only edits by those passes do not count as body changes). Pick **one** ISO 8601 timestamp for the whole run — upstream uses the moment the run began:

```bash
date -u +%Y-%m-%dT%H:%M:%S.000Z
```

For every concept page (same exclusions), recompute the body hash with Step 2's body-extraction command and compare it to the Step 2 baseline:

- **New page, or body hash changed** → set `generated: { by: "<actor>", at: "<run timestamp>" }` and remove any `timestamp` field, which OKF v0.2 supersedes. Whitespace counts: any body change advances the stamp. Then, since 0.4.1 (#730), **canonicalize the page's trailing line endings to exactly one LF** — and only those. Prose wrapping, indentation, tables, and every other authored choice stay untouched, and a page whose body did not change never passes through this normalization at all, so its bytes are preserved exactly.
- **Body unchanged** → restore the baseline: re-set the `generated` event the page had before the run (a Step 4 rewrite may have dropped or altered it), or remove `generated` entirely if it had none. A front-matter-only change never advances the stamp. Since 0.4.1, **compare by meaning first**: if the event already on the page has the same `by` and `at` as the baseline, leave the page completely alone rather than re-rendering the field. That is what keeps an older `{by: …}` spelling from being rewritten into the new `{ by: … }` one on a page nobody touched.
- The actor is the producing host, matching upstream's host registry: `claude-code`, `codex`, or `opencode` — **[adapted]** upstream's own runs stamp `openwiki/<version>`, which would misattribute this port's output.
- Render it as a single-line flow mapping with JSON-quoted members **and a space inside each brace** (the spacing changed in 0.4.1, #728), replacing an existing `generated:` line in place and leaving every other front-matter line untouched: `generated: { by: "claude-code", at: "2026-08-26T00:00:00.000Z" }`.
- Run Step 2's front-matter repair over the reconciled page before writing it, then write only when the content changed.
- **This pass never fails the run** (since 0.4.1, #728): provenance is optional trust metadata, so a page that cannot be read is skipped and a write that fails is skipped, leaving the already-persisted page as the deterministic fallback rather than discarding the rest of a finalized wiki. Report which pages were skipped.

Never author, edit, or remove `generated`, `verified`, `sources`, or `timestamp` during Step 4 — they are code-owned, which here means owned by this step.

## Step 6 — Persist metadata (ported from upstream `persistRunMetadataIfChanged`)

Recompute the Step 2 content hash with the same command, then write `openwiki/.last-update.json` with exactly these fields. Since 0.4.0 (#647) the metadata is written on **every** completed init/update run, whether or not content changed — a no-op update still means OpenWiki ran, and freshness checks should reflect that:

```json
{
  "updatedAt": "<UTC ISO-8601, from: date -u +%Y-%m-%dT%H:%M:%S.000Z>",
  "command": "init | update",
  "gitHead": "<from: git rev-parse HEAD; omit the key if not a git repo>",
  "model": "<your model id if known, else claude-code or codex>",
  "status": "complete",
  "language": "<the effective language tag from Step 1, e.g. en>"
}
```

Run the `date` and `git` commands — never guess the timestamp or the head. Report the recomputed hash's verdict to the user (changed → what changed; unchanged → the wiki was already accurate) even though it no longer gates the write.

**If Step 4 skipped any page** (since 0.4.1, #732) **or the repository source changed during the run** (since 0.4.3, #740), write the metadata differently: `status: "interrupted"`, and `gitHead` set to the **previous run's recorded head** rather than the current one — or the key **omitted entirely** when there was no previous head (upstream 0.4.3 made that explicit: the override accepts `null` for "no successful baseline exists", where before it silently fell back to the current head). Both matter and for the same reason — the next update must not treat this wiki as complete and current. An `interrupted` status defeats the early no-op exit, and rewinding `gitHead` keeps the changed-path and staleness evidence in view. Recording the current head here would make the skipped page, or the source change, invisible forever.

On drift, also tell the user plainly, in upstream's terms: the wiki was finalized without advancing its source checkpoint, and a follow-up update run is needed to reconcile the change. The metadata is what forces that run; the message is what explains it.

Run this step even when the run fails after generating content (upstream persists metadata on the error path too): write the metadata before reporting the failure — with `status: "interrupted"` instead of `"complete"`, so the already-generated content stays diffable and the next update knows the wiki may be partial and does not skip (#365).

Two exceptions on the failure path:

- **A failed init that had a Step 2 backup**: restore the backup first and write no metadata. The backup contains the previous `openwiki/.last-update.json`, so restoring it puts the recorded state back in agreement with the restored wiki — writing an `interrupted` record on top would describe a partial wiki that no longer exists. Report the rollback. (**[adapted]** Upstream only rolls back when the failure precedes its checkpoint becoming durable; after that, partial pages *are* its recovery mechanism. This port always rolls back, because it has no checkpoint to resume from.)
- **A failed first init** — no prior `openwiki/`, so no backup: keep the partial content and write `status: "interrupted"`, exactly as upstream's no-op replacement path does.

## Final response

Summarize the completed documentation changes and important caveats — the planned page set, what each page covers, deletions, any pages left with stamped Mermaid or link comments, and (on init) that the previous wiki was replaced. State plainly whether the source moved mid-run.

## Automation

Asked to set up scheduled/recurring updates or CI? Read `references/automation.md` in this skill's directory.

## Runtime evidence (LangSmith)

Asked to fold LangSmith runtime traces into the repository wiki? Read `references/runtime-evidence.md` in this skill's directory.
