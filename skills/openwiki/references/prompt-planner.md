# Planning phase — planner prompt, submission schema, and plan validation

> Reproduced from upstream `src/agent/repository-prompts.ts` `createRepositoryPlannerPrompt` (v0.5.2), with its interpolated context blocks rendered in place as SKILL.md Step 3 describes. The submission schema is upstream `src/agent/repository-runner.ts` `PlanSchema`; the validation rules are upstream `src/generation/page-jobs.ts` `createRepositoryPlan`. **[adapted]** upstream's `/`-rooted virtual paths (`/openwiki/quickstart.md`) are the wiki's canonical page identifiers throughout the lifecycle — keep writing them that way in the plan, and read `/openwiki/x.md` as the real repo-relative file `openwiki/x.md` when you touch the filesystem.
>
> Since 0.4.0 (#713) repository generation is a two-role lifecycle: **one bounded planner** decides the complete page set, then **one fresh worker per page** writes it (`references/prompt-page.md`). Upstream gives the planner read-only filesystem tools (`read_file`, `ls`, `glob`, `grep`), no shell, no write access to any wiki page, and strips the delegation tool — **[adapted]** hold yourself to the same boundary while planning: research read-only, write nothing, and do not hand the planning out to a subagent.

## System prompt

You are planning an OpenWiki code wiki for this repository.

Your only output action is submit_plan. Do not write documentation, do not delegate work, and do not emit narrative or conversational text. Invoke submit_plan directly.

Design the smallest complete repository-specific information architecture that helps a coding agent understand and safely change the system. Organize around owned systems, runtime domains, and cross-system workflows rather than mirroring the source tree. Use hierarchical paths for meaningful groups such as /openwiki/architecture/, /openwiki/concepts/, /openwiki/workflows/, /openwiki/operations/, /openwiki/integrations/, and /openwiki/testing/ when the repository has enough coverage to warrant them. Do not emit a flat dump of unrelated top-level pages. Include /openwiki/quickstart.md for init.

Explore before submitting the plan. First map manifests, major directories, entrypoints, and public surfaces. Then trace representative end-to-end control and data flows across callers, state/persistence, failure handling, configuration, operations, and integrations. Finally inspect focused tests and neighboring implementations to verify boundaries, invariants, and non-obvious connections. Do not stop at directory names or one representative file. Explore only until the major systems, behaviors, and relationships are supported by repository evidence; avoid exhaustive file-by-file inventory. Page paths are final once submitted.

Populate relatedPages with the most useful conceptual and workflow neighbors so the resulting wiki is navigable across system boundaries. The quickstart must route readers through the hierarchy; generated index pages will provide folder navigation and must not be included in the plan.

Init MUST include /openwiki/quickstart.md. Update MUST NOT delete quickstart. If an update adds, deletes, moves, or materially regroups documentation pages, include /openwiki/quickstart.md in the plan so its task-routing map is refreshed. An update with no required page edits and no deletions may submit pages: [].

For every page provide a concise purpose and useful seedPaths. seedPaths are starting points, not research boundaries. Copy only relevant global constraints from the user/connector context into that page's instructions array; do not copy unrelated context into every job.

User and connector planning context (*rendered only when the run has one — here: the user's additional instruction from SKILL.md's mode resolution; absent otherwise, along with this heading*):

> *(that text)*

*(Update runs only — the whole block below, through the windows, replaces the flat "Changed repository paths" list upstream rendered until 0.4.3. New in 0.5.0, #720.)*

For update, evaluate each existing page inside its own committed update window. A page already advanced by a merged partial update must not be regenerated for changes at or before its baseline. Schedule it only when changes after that baseline, current Claims issues, language rewriting, navigation changes, or cross-page consistency require work.

Committed per-page update windows (*upstream `getRepositoryPageUpdateWindows`: existing pages grouped by the `gitHead` recorded for them in the committed `openwiki/.page-manifest.json` ledger, each cohort carrying its own changed-path list from `getRepositoryChangedPaths` — the union of `git diff --name-only <baseline>..HEAD`, `git diff --name-only HEAD`, and `git ls-files --others --exclude-standard`, dropping everything under `openwiki/` and everything excluded by `.openwikiignore`, sorted; history lookup failures deliberately yield an empty list rather than failing the run. A page with no recorded baseline goes into the `fullReview` window. Rendered as one cohort per baseline, or `- (none)`*):

> ```
> - Baseline <gitHead, or the literal text: unknown (full review required)>:
>   - Pages: <comma-separated pages, or (none)>
>   - Changed paths: <comma-separated paths, or (none)>
> ```

**[adapted]** SKILL.md Step 1 fills this block in from whatever baselines it can actually prove. When `openwiki/.page-manifest.json` exists (a native or CI run wrote it), read the per-page `gitHead` values out of it and render the real cohorts — this is the whole point of #720, and it is why merging a partially-failed CI PR is worth doing. When it does not exist, this port has exactly **one** provable baseline, the `gitHead` in `openwiki/.last-update.json`, so the block degrades to a single window listing every existing page, or to `fullReview` when no baseline was recorded. Read a window whose `Changed paths` is `(none)` as upstream's fast-forward case: nothing visible moved since that cohort's baseline, so those pages need no source-driven work — only Claims issues, a language switch, or navigation changes can still schedule them.

Claims requiring attention (*update runs only — the deterministic staleness preflight from SKILL.md Step 1, rendered as one bullet per issue or `- (none)`. Upstream's line is `- <page>: <claimId> (stale|unresolved) -> <resource>, <resource>`;* **[adapted]** *this port has no per-Claim sidecar, so the bullet carries the page, the issue kind, and the changed or missing `repo://` evidence resources it read from the page's own OKF `sources` front matter*):

> `- <page>: (stale|unresolved) -> <resource>, <resource>`

Repository OpenWiki instructions (*rendered only when `openwiki/INSTRUCTIONS.md` exists and is non-empty after trimming; absent otherwise, along with this heading*):

> *(contents of openwiki/INSTRUCTIONS.md)*

## The planning message to act on

> Plan this repository wiki now.

## submit_plan — the only completion action

Upstream exposes exactly one completion tool to the planner:

> **submit_plan** — Submit the final canonical OpenWiki page plan. This is the only completion action for planning.

**[adapted]** You have no such tool. Produce the same payload as your own plan record and validate it against the rules below before starting the page loop; you may keep it in your working notes, but do **not** write it into the wiki — `_plan.md` and `_skeleton.md` no longer exist (upstream 0.4.0 replaced both with the code-owned `openwiki/.run.json` checkpoint, and any `_`-prefixed page path is rejected outright). 0.5.2 (#872) added "do not emit narrative or conversational text. Invoke submit_plan directly" to the prompt above because planners were answering in prose instead of calling the tool; having no tool does not exempt you from the rule it encodes — go from research straight to the payload, without a conversational preamble about what you are about to plan.

Since 0.5.0 (#789) submitting again is **not** an error, but replacing the plan is: upstream dropped the "already called" guard and now re-validates the second payload and compares it against the persisted plan ignoring job ids, accepting an identical semantic plan and rejecting a different one (`This OpenWiki run already has a different persisted plan.`). The rule that matters for you: **the page set is fixed once you start writing pages.** Restating the plan you already fixed is harmless; quietly revising it mid-queue is not — the pages already written were planned against the old set, and page paths are final once submitted.

Payload shape (upstream `PlanSchema` — no other keys are accepted):

```json
{
  "pages": [
    {
      "path": "/openwiki/architecture/overview.md",
      "title": "Architecture overview",
      "purpose": "Explain how the runtime is decomposed into owned systems and how a request flows across them.",
      "seedPaths": ["src/agent/index.ts", "src/generation/repository-run.ts"],
      "relatedPages": ["/openwiki/workflows/generation.md"],
      "instructions": ["Only document the public HTTP surface."]
    }
  ],
  "deletePages": ["/openwiki/legacy/old-topic.md"]
}
```

- `path`, `title`, `purpose` are required and must be non-empty after trimming. `seedPaths`, `relatedPages`, `instructions` are optional.
- `seedPaths` are repository-relative (backslashes → `/`, leading `/` stripped); a `..` segment is invalid.
- `deletePages` is the update run's explicit removal set; omit it on init.

## Plan validation (upstream `createRepositoryPlan` — apply these before you start writing pages)

Upstream normalizes and hard-validates the submitted plan, and rejects it back to the planner for correction. Check the same things yourself:

1. **Canonicalize every page path.** Accept a canonical (`/openwiki/x.md`), repository-relative (`openwiki/x.md`), or wiki-relative (`x.md`) spelling and normalize it to `/openwiki/…`; traversal segments (`.`, `..`) are invalid. The path must end in `.md`, must not sit under `openwiki/.claims`, and its basename must not start with `_` (reserved working page). `index.md`, `log.md`, and `INSTRUCTIONS.md` are reserved and can never be planned pages.
2. **Init plans cannot delete pages** — a non-empty `deletePages` on init is invalid.
3. **`/openwiki/quickstart.md` can never be deleted**, in either mode.
4. **No duplicate planned page**, and **no page may be both generated and deleted**.
5. **Init must include `/openwiki/quickstart.md`.**
6. **Update runs get required jobs injected** for work the planner omitted — add them yourself:
   - one job per page named by a Step 1 staleness issue that the plan neither generates nor deletes, with purpose *"Reconcile stale or unresolved Claims and update this page from current repository evidence while preserving unaffected accurate content."* and its seed paths taken from the issue's evidence resources (the `repo://` prefix and any `#Lx-Ly` fragment stripped);
   - one job per existing concept page — every non-reserved `.md` page present under `openwiki/` before the run, so `index.md` / `log.md` / `INSTRUCTIONS.md` are excluded — when the run changes the wiki's output language (SKILL.md Step 1's language switch), with purpose *"Rewrite this existing page in the run's target language while preserving every accurate repository-supported fact and reconciling its complete Claim set."* and no seed paths.
   - A title is derived from the path for these injected jobs: the basename without `.md`, split on `-`/`_`, each part's first character upper-cased, joined with spaces.
7. **Order the queue deterministically:** all non-quickstart pages first in code-unit order by path, then `/openwiki/quickstart.md` last — quickstart is the synthesis and navigation page, so it is written after the domain pages exist. `seedPaths`, `relatedPages`, and `instructions` are each deduplicated and sorted the same way.

Deletions are not applied during the page loop; SKILL.md Step 5 applies them deterministically after every page job completes.
