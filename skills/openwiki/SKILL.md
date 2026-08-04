---
name: openwiki
description: "Generate or maintain repository wiki documentation in openwiki/. Auto-detects init (no wiki yet) vs update (wiki exists). Use when asked to create, initialize, refresh, or update a repo's wiki or codebase documentation."
---

# OpenWiki — repository wiki agent (code mode)

Port of [langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) v0.3.0, repository ("code") mode: the upstream per-command system prompts reproduced verbatim (Step 3 routes to `references/prompt-init.md` / `references/prompt-update.md` — upstream `src/agent/prompts/code.ts` rendered by `src/agent/prompt.ts`, plus the init subagents from `src/agent/skeleton_critic.ts` + `src/agent/wiki_qa_subagents.ts`), wrapped in the runtime steps the upstream CLI performs around them (Step 0 from `src/code-mode.ts`; Steps 1, 2, 5 from `src/agent/utils.ts` + `src/language.ts` + `src/agent/openwiki-ignore.ts`; Step 2's translation and normalization passes and Step 4 from `src/agent/translation-middleware.ts` + `src/okf/frontmatter.ts` + `src/okf/index-sync.ts` + `src/okf/index-labels.ts` + `src/mermaid/wiki.ts` + `src/agent/wiki-link-validator.ts`, wired by `src/agent/okf-middleware.ts`). You are the agent; the current repository is the target. No CLI, no API key — you do the work with your own tools.

Harness adaptations are marked **[adapted]**; upstream content with no equivalent here is marked **[omitted]**. Everything else is upstream text — keep it that way so upstream syncs stay line-mappable (see `UPSTREAM.md` in this skill's source repo). Upstream's personal knowledge wiki ("local-wiki" mode at `~/.openwiki/wiki`) is ported as the separate `openwiki-personal` skill; wiki Q&A as `openwiki-ask`.

## Mode resolution

- The user explicitly asks to initialize / build from scratch → **init**.
- The user explicitly asks to update / refresh → **update**.
- Otherwise auto-detect: `openwiki/quickstart.md` exists → **update**; it does not → **init**. (If `openwiki/` exists without `quickstart.md`, run init but read and preserve what is already there.)
- The user asks to migrate the wiki to OKF / fix wiki front matter → run **update**: Step 2's normalization pass migrates every non-compliant page deterministically (upstream 0.2.1 replaced the bundled migrate-wiki-to-okf skill with this code pass). The request counts as an additional user instruction, so Step 1's early no-op exit does not apply.
- The user asks for the wiki in a specific language ("write the wiki in Korean", "switch the wiki to zh-CN") → that is the run's **requested output language**, resolved in Step 1 (upstream: the `--language` flag; since 0.2.4 the wiki's language is persisted state, and an update that switches it retranslates every page in Step 2). The request counts as an additional user instruction, so Step 1's early no-op exit does not apply.
- The user asks to pull LangSmith runtime traces / production run evidence into the repo wiki → **runtime evidence update run**: read `references/runtime-evidence.md` in this skill's directory and follow it (Steps 0, 1, 2, 4, 5 here still apply).
- Any other instruction in the user's request (e.g. "focus on the API routes") is an **additional user instruction** — append it to the user prompt as shown in the Step 3 reference file for the resolved mode.

## Model tier

Upstream defaults to frontier coding models. Documentation quality depends on it — run this skill on a frontier tier, not a small/fast model.

## Step 0 — Code setup (ported from upstream `code-mode.ts`)

Upstream performs this repository setup on every code-mode invocation, outside the agent. Do it at the start of every init/update run, before Step 1:

1. Ensure `/AGENTS.md` and `/CLAUDE.md` each carry the snippet below (upstream `CODE_MODE_AGENT_FILES` — each file is created when missing and refreshed in place when already present):
   - Markers `<!-- OPENWIKI:START -->` / `<!-- OPENWIKI:END -->` present → the file must contain exactly one `START` marker followed by exactly one `END` marker (since 0.3.0, #547). Then replace everything between and including the markers with the snippet — skip the write when the existing block is already identical (**[adapted]** upstream rewrites unconditionally; the byte outcome is identical). Malformed or duplicated markers → fail the setup with upstream's error (`Cannot update <file> because its OpenWiki managed markers are malformed or duplicated. Expected either no markers or exactly one <!-- OPENWIKI:START --> marker followed by one <!-- OPENWIKI:END --> marker. Repair or remove the markers and retry; the file was left unchanged.`) and write NEITHER file — upstream validates both files before writing either, so a malformed sibling leaves both untouched.
   - **[adapted]** A legacy `## OpenWiki` section without markers (written by pre-0.1.0 versions of this skill) present → replace that section with the snippet instead of appending a duplicate (upstream never sees this state; this port migrates it).
   - Neither present → append the snippet to the end of the file, separated by one blank line; if the file does not exist, create it containing only the snippet.
2. **[adapted]** Exception for `/CLAUDE.md`: if it imports AGENTS.md (an `@AGENTS.md` line) and `/AGENTS.md` carries the snippet, it counts as covered — do not append a duplicate (Claude Code loads AGENTS.md through the import; upstream has no import concept).
3. **[omitted]** Upstream also creates `.github/workflows/openwiki-update.yml` (a scheduled `openwiki code --update --print` workflow that needs a provider API key) — since 0.2.3 only `openwiki code --init` creates it, and an existing file is never overwritten, so operator customizations survive. This keyless port does not create CI files unasked — for scheduled updates read `references/automation.md`.

The snippet — keep byte-identical to upstream `createCodeModeAgentsSnippet()`:

```markdown
<!-- OPENWIKI:START -->

## OpenWiki

This repository has a generated `openwiki/` evidence index. It is optional just-in-time context, not required startup reading.

- Treat source code and tests as authoritative. A brief's unknowns and review items are verification gaps, not automatic requirements.
- Prefer the narrowest quiet validation that proves the changed behavior. Preserve complete failure output.

The scheduled OpenWiki GitHub Actions workflow refreshes the repository wiki. Do not hand-edit generated OpenWiki pages unless explicitly asked; prefer updating source code/docs and letting OpenWiki regenerate.

<!-- OPENWIKI:END -->
```

(The body text changed at upstream 0.3.0 — "optional just-in-time context" — so refreshing the block in a repo set up by an older version is a real, expected write.)

Only Step 0 may touch `/AGENTS.md` and `/CLAUDE.md`. The documentation run itself never does — see "Root agent instruction files" in Step 3.

## Step 1 — Run context and early no-op check (before any write; all git read-only; use `git --no-pager`)

Read `openwiki/.last-update.json` if it exists to recover `gitHead`, `updatedAt`, `status`, and `language` (upstream `readLastUpdate`: an unreadable or structurally invalid file counts as no metadata; a `status` other than `"interrupted"`, including the field being absent from pre-0.2.4 metadata, counts as `"complete"`). Read `openwiki/INSTRUCTIONS.md` if it exists — the user-authored OpenWiki brief for this repository, injected as "Wiki brief" in the user prompt (ported from upstream `readRepositoryWikiInstructions`; absent or empty → "(not provided)").

**Resolve the wiki output language** (ported from upstream `resolveLanguage` + `createRunContext`): the **effective language** is the requested language when the user asked for one, else the metadata's `language`, else `en` — an update without a language request inherits the wiki's persisted language so it stays consistent instead of mixing languages, and English is always materialized as an explicit `en` rather than encoded by an absent field. Canonicalize a requested language to a BCP-47 tag (`ko`, `zh-CN`, `pt-BR`); a value that is not a recognizable real language → warn the user (upstream: `Unrecognized language "<input>"; generating in English. Use a BCP-47 code such as zh-CN, hi, or pt-BR.`) and proceed as if no language was requested.

**Load `.openwikiignore`** (ported from upstream `OpenWikiIgnore.load`/`.parse` in `src/agent/openwiki-ignore.ts`, since 0.2.5 — loaded for repository runs only): read `.openwikiignore` from the repo root; a missing file means no rules. Drop blank lines and `#` comments; each remaining line is a gitignore-style pattern — `*` matches within one path segment, `?` one non-slash character, `**` spans directories; a leading `/` (or any embedded slash) anchors the pattern to the repo root, otherwise it matches at any path segment; a trailing `/` scopes it to directories (still excluding everything nested beneath); a leading `!` re-includes, and the last matching rule wins. Matching is case-insensitive (deliberate and security-relevant: on case-insensitive filesystems an alternate-cased spelling would otherwise slip past an exclusion), and paths are canonicalized before matching (backslashes → `/`, `.`/`..` segments collapsed without escaping the repo root) so spellings like `./secrets/x` or `secrets/../secrets/x` cannot dodge an anchored rule. No usable patterns → the rules are **inactive** and every `.openwikiignore` provision in this skill is a no-op.

**Then run the early no-op check** (update mode, only when the user gave no additional instruction — ported from upstream `getUpdateNoopStatus`). Since 0.3.0 this check is the only git the runtime itself runs: upstream deleted the injected git summary (`createGitSummary` and its `.openwikiignore` filtering are gone), and the update prompt instead has you inspect git history yourself during the Step 3 documentation work.

- No recorded metadata or no recorded `gitHead`, or not a git repository → not a no-op; proceed. (Without git, infer changes during Step 3 from filesystem timestamps, source inspection, and existing docs.)
- Recorded `status: "interrupted"` → do not skip; the previous run may have left a partial wiki, so it must be retried (#365).
- Otherwise run:

```bash
git --no-pager status --short --untracked-files=all
git --no-pager rev-parse HEAD
```

- The worktree must be clean, ignoring the `openwiki/.last-update.json` status line itself and — when ignore rules are active — status lines whose paths are excluded by `.openwikiignore` (a rename line counts when either its old or new path matches). Any other line → not a no-op.
- HEAD equals the recorded `gitHead` → **no-op**. HEAD moved → run `git --no-pager diff --name-only <gitHead>..HEAD`; every changed path lies under `openwiki/` or is excluded by `.openwikiignore` (at least one such path — if git reports no changed paths, do NOT treat it as a no-op) → **no-op**: report that the wiki is already current and stop here (an ignored path changing on its own never forces a rebuild). Anything else → proceed.
- (Step 0 still runs before this exit — upstream refreshes the repo setup even on no-op runs.)

## Step 2 — Snapshot, translate on a language switch, then normalize the wiki (before the work; ported from upstream `createOpenWikiContentSnapshot` + `translation-middleware.ts` + `migrateWikiToOkf`)

```bash
find openwiki -type f ! -name '.last-update.json' ! -name '_plan.md' -print0 2>/dev/null | LC_ALL=C sort -z | xargs -0 shasum -a 256 2>/dev/null | shasum -a 256
```

Record the hash. (`shasum -a 256` covers macOS and most Linux; on minimal Linux images substitute `sha256sum` in both places. Upstream's snapshot ignores `.last-update.json` and `_plan.md` — since 0.2.1 the plan file never counts as content.) You will recompute it in Step 5. **[adapted]** The hash is compared only within this run — upstream never persists it. Upstream's snapshot additionally hashes directory entries and scopes the metadata exclusions to the wiki root; this one-liner's changed/unchanged verdict differs only on states documentation runs don't produce (empty directories, nested metadata files).

**Then, update runs only: bring existing pages into the wiki language** (ported from upstream `src/agent/translation-middleware.ts` — a before-agent pass mounted on every update run, never init or chat; its writes land after the snapshot, so a pure translation run still counts as changed content in Step 5). Resolve the plan from Step 1: **target** = the effective language; **source** = the metadata's `language`, else `en` (a hint only — detection below decides); **translate-all** = the user requested a language whose primary subtag differs from the source's (a region-only change such as `en` → `en-GB` does not warrant retranslation). Then, for every `.md` file under `openwiki/` except `index.md`, `log.md`, `_plan.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories:

- Not translate-all → skip every page whose front matter has no `openwiki_translation_pending` field; a wiki with none is left untouched, so plain updates cost nothing here. Translate-all → every page.
- Translate each remaining page into the target language. **[adapted]** Upstream makes one un-streamed model call per page ("Translating wiki docs..."); here you are that model — apply its exact rules yourself, and keep the translated bodies out of your user-facing output:
  - The source language is a hint, not a guarantee: detect the page's actual language and translate any content not already in the target language; a page already entirely in the target language is returned unchanged.
  - Translate prose, headings, list items, blockquotes, and table cell text.
  - In the YAML front matter, fully translate the human-readable "title", "description", and "type" values, even when they are dense with product names, feature names, or technical terminology; within those values keep unchanged only literal code identifiers, file paths, commands, and URLs. Leave the "tags" values in English so they stay stable across pages as cross-cutting aggregation keys. Keep every front matter key as written, and copy all other values (URLs, file paths, identifiers, timestamps) byte-for-byte.
  - Do NOT translate code identifiers, file paths, commands, API names, URLs, or anything inside inline code spans or fenced code blocks.
  - Preserve all Markdown syntax, link targets, mermaid fences, and the document's whitespace and structure.
- On success, deterministically remove any `openwiki_translation_pending` front matter field, and write the page back only when the content changed.
- A page that cannot be brought into the target language never aborts the run: leave it in its previous language, set `openwiki_translation_pending: "<target tag>"` in its front matter (preserving every other line), continue with the next page, and report the failed pages once — the next update retries them via the marker sweep above.

**Then normalize the wiki** (ported from upstream `migrateWikiToOkf` in `src/okf/index-sync.ts` — `okf-middleware.ts` runs it before the agent starts, so the run operates over an already-conformant wiki and can enrich flagged pages as it works). For every `.md` file under `openwiki/` except `index.md`, `log.md`, `_plan.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories:

- Its front matter parses as YAML and `type` is a non-empty string → leave the file untouched, even when optional fields are junk. An author's `type` and custom fields are never overwritten.
- Otherwise (no front matter, unparseable YAML, or missing/empty `type`) → replace the front matter (or add one) with exactly this minimal block — one blank line after the closing `---`, then the body with its leading whitespace trimmed:

```markdown
---
type: "Reference"
title: "<first ATX H1 in the body; fallback: filename without .md, -/_ runs → spaces, first character upper-cased>"
openwiki_generated: true
---
```

- Values are JSON-double-quoted. `openwiki_generated: true` flags code-derived metadata; the documentation run should replace it with accurate metadata per "Front matter requirements (OKF)" when it touches the page.
- Non-English wiki language → the derived `type` is that language's localized label from Step 4's table instead of `"Reference"` (upstream `resolveConceptTypeLabel`: full tag → primary subtag → English fallback).
- The rebuild would drop the page's extension fields, so carry an existing `openwiki_translation_pending` field over into the replacement block (upstream `PRESERVED_EXTENSION_FIELDS`) — a page that is both non-conformant and pending translation must not lose its control marker.

## Step 3 — System prompt (act as this agent)

> Since upstream 0.3.0 (#528) each command has its own rendered system prompt (`src/agent/prompts/code.ts`, rendered by `src/agent/prompt.ts`), so this port keeps them as separate line-mapped files. Read the resolved mode's file in this skill's directory and act on it exactly:
>
> - **init** → `references/prompt-init.md` — the init workflow (repository mapping into a temporary `openwiki/_skeleton.md`, evidence gates, and the QA verification loop), its three read-only subagents (`skeleton_critic`, `wiki_question_finder`, `wiki_answer_verifier`), and the init user prompt.
> - **update** → `references/prompt-update.md` — the update system prompt (impact-plan-driven surgical edits with no preset page limit) and the update user prompt.

Shared adaptation conventions (both files assume these): (a) DeepAgents virtual-filesystem tools and `/`-rooted virtual paths become your native file tools on real repo-relative paths — `/openwiki/quickstart.md` means `openwiki/quickstart.md` — and upstream's raw doubled-article renders ("the the target repository's ... tree") are shortened rather than reproduced; (b) the DeepAgents task tool becomes your harness's read-only subagents (Claude Code: the Task tool) — if your harness has none (e.g. Codex), perform the subagent roles yourself sequentially with fresh eyes, keeping their procedures and output formats; (c) metadata recording moves from the CLI to Step 5; (d) upstream enforces the write boundary and the `.openwikiignore` exclusions in code (`src/agent/docs-only-backend.ts`: reads/edits of ignored paths hard-denied, ls/glob/grep results silently filtered, shell execute allowlisted) — here they are hard rules you follow; (e) upstream keeps the wiki OKF-conformant and render-safe in code (`src/agent/okf-middleware.ts`: a before-run normalization pass, a per-write front matter warning, and after-run Mermaid validation, index regeneration, and internal-link validation — `src/okf/frontmatter.ts` / `src/mermaid/wiki.ts` / `src/okf/index-sync.ts` / `src/agent/wiki-link-validator.ts`) — here Step 2's normalization, the self-check bullet under "Front matter requirements (OKF)", and Step 4 stand in; (f) template placeholders are rendered in place: `{OUTPUT_LANGUAGE_INSTRUCTIONS}` carries Step 1's effective language, and the `.openwikiignore`-conditional passages (`{GIT_HISTORY_HINT}`, `{DISCOVERY_INSTRUCTION}`, `{OPENWIKIIGNORE_INSTRUCTIONS}`) show both variants marked "*`.openwikiignore` active/inactive*" — with no active rules the inactive variant applies and the prompt renders without any ignore section. **[omitted]** Upstream's chat-mode prompt (wiki-first question answering + the OpenWiki CLI reference) is the `openwiki-ask` skill's domain; connector-fed personal wikis are the `openwiki-personal` skill's.

The init run's temporary `openwiki/_skeleton.md` is the agent's own working file: the init workflow tells you to delete it yourself once every page is populated (unlike `openwiki/_plan.md`, which Step 4 removes deterministically) — do not leave it behind, and do not link to it.

## Step 4 — Validate diagrams, synchronize directory indexes, then validate links (after the work; ported from upstream `src/mermaid/wiki.ts` + `src/okf/index-sync.ts` + `src/okf/index-labels.ts` + `src/agent/wiki-link-validator.ts`)

Upstream runs three deterministic after-run passes on every init/update run, not chat (`okf-middleware.ts`: `validateWikiMermaid`, then `synchronizeWikiIndexes`, then `validateWikiInternalLinks` — the third since 0.3.0, #371). Here, do all three yourself in that order after the documentation work, before Step 5, so their writes land in the Step 5 content hash. Skip them only when Step 1 already exited at the early no-op (upstream never reaches the passes on that path).

First: delete `openwiki/_plan.md` if it still exists (upstream `removeTemporaryPlanFile` runs on every non-chat run — since 0.2.5 the prompt no longer tells the agent to delete the plan, so this pass is the removal, not a backstop), and if any concept page still lacks a usable `type`, repair it per Step 2's normalization rule — upstream re-normalizes every concept file while collecting index entries, so index generation never fails on a non-compliant page.

**Validate Mermaid diagrams** (ported from `validateWikiMermaid`): for every `.md` file under `openwiki/` except `index.md`, `log.md`, `_plan.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories (upstream `EXCLUDED_FILES`), check that every fenced ```mermaid block parses (a ```mermaid example nested inside a longer outer fence does not count):

- **[adapted]** Upstream parses each fence with the real Mermaid parser when its optional `mermaid` + `jsdom` peers are installed, and otherwise falls back to a conservative heuristic that only flags near-certain breakages (a `flowchart`/`graph` node id named `end`; a semicolon inside a `[]`/`()`/`{}` label; an unescaped angle bracket inside a label). Here, run the check yourself: apply that heuristic plus the `mermaid-diagrams` skill's syntax-safety rules — or a locally installed Mermaid parser when one is available.
- **[adapted]** A broken fence you can confidently repair (you usually wrote it this run) → fix it in place; that matches what upstream's write-time prompt guidance would have produced. Otherwise degrade it exactly as upstream does: replace the ```mermaid fence with a ```text fence holding the same body, preceded — at the fence's indentation — by a one-line HTML comment: `<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: <one-line reason> -->`. A later update run repairs it per the Diagram discipline.
- Files whose fences all parse are left byte-for-byte unchanged, so this pass creates no diff noise.

Then, for every directory under `openwiki/` (recursively, skipping dot-directories — the wiki root itself included), regenerate its `index.md`:

1. Collect the directory's direct children:
   - Files: every `.md` file directly in it except `index.md`, `log.md`, `_plan.md`, `INSTRUCTIONS.md`, and dot-files. For each, read its front matter — link label = `title` when it is a non-empty string (fallback: the filename without `.md`), and keep `description` when it is a non-empty string (unusable optional fields are ignored, not errors).
   - Directories: every subdirectory whose name does not start with `.`.
2. Render exactly this shape — **no front matter** (`index.md` is a reserved OKF document), except the wiki root's index, which starts with exactly the three-line `okf_version` block shown; one blank line between sections; a section with no entries omitted entirely; when both sections are empty the sections part is just `# Files` (the root still keeps its okf_version block above it); trailing newline:

```markdown
---
okf_version: "0.1"
---

# Files

- [<label>](<URL-encoded filename>) - <description, only when the page has one>

# Directories

- [<name>](<URL-encoded name>/)
```

   - Non-root directories: the same content without the `okf_version` block.
   - Sort each list alphabetically by link target (upstream: `localeCompare`). Escape `\`, `[`, and `]` in labels.
   - The `Files` / `Directories` headings (including the empty-directory `# Files`) are the wiki language's labels from the table below (upstream `resolveIndexLabels`: full tag → primary subtag → English fallback, so `pt-BR` uses the `pt` row while `pt-PT` has its own row; an unlisted language keeps English). These two words are curated structural chrome, not translated prose — use the table verbatim, never your own translation. The Derived type column is the localized `type` that Step 2's normalization (re-applied here) stamps on repaired pages (upstream `resolveConceptTypeLabel`, same resolution; a language upstream leaves out of that map falls back to English `Reference`).

| Language | Files | Directories | Derived type |
|---|---|---|---|
| en | Files | Directories | Reference |
| ar | ملفات | مجلدات | مرجع |
| bg | Файлове | Директории | Reference |
| ca | Fitxers | Directoris | Referència |
| cs | Soubory | Adresáře | Reference |
| da | Filer | Mapper | Reference |
| de | Dateien | Verzeichnisse | Referenz |
| el | Αρχεία | Κατάλογοι | Αναφορά |
| es | Archivos | Directorios | Referencia |
| fi | Tiedostot | Hakemistot | Reference |
| fr | Fichiers | Répertoires | Référence |
| he | קבצים | תיקיות | Reference |
| hi | फ़ाइलें | निर्देशिकाएँ | संदर्भ |
| hr | Datoteke | Direktoriji | Referenca |
| hu | Fájlok | Könyvtárak | Reference |
| id | Berkas | Direktori | Referensi |
| it | File | Cartelle | Riferimento |
| ja | ファイル | ディレクトリ | リファレンス |
| ko | 파일 | 디렉터리 | 참조 |
| ms | Fail | Direktori | Rujukan |
| nb | Filer | Mapper | Referanse |
| nl | Bestanden | Mappen | Referentie |
| no | Filer | Mapper | Referanse |
| pl | Pliki | Katalogi | Reference |
| pt | Arquivos | Diretórios | Referência |
| pt-PT | Ficheiros | Diretórios | Referência |
| ro | Fișiere | Directoare | Referință |
| ru | Файлы | Каталоги | Справочник |
| sk | Súbory | Adresáre | Referencia |
| sl | Datoteke | Mape | Reference |
| sr | Датотеке | Директоријуми | Референца |
| sv | Filer | Kataloger | Referens |
| th | ไฟล์ | ไดเรกทอรี | อ้างอิง |
| tr | Dosyalar | Dizinler | Referans |
| uk | Файли | Каталоги | Довідник |
| vi | Tập tin | Thư mục | Tham khảo |
| zh | 文件 | 目录 | 参考 |
| zh-TW | 檔案 | 目錄 | 參考 |

3. Compare with the existing `index.md` and write only when the content differs — byte-identical output is skipped, so no-op runs stay no-ops.

**Then validate internal links** (ported from upstream `validateWikiInternalLinks` in `src/agent/wiki-link-validator.ts`, since 0.3.0 — runs after index sync; it stamps broken links in place and never fails the run). For every `.md` file under `openwiki/` except `index.md`, `log.md`, `_plan.md`, `INSTRUCTIONS.md`, and dot-files/dot-directories:

1. Strip any previously inserted stamp lines — full-line HTML comments matching `<!-- openwiki: broken internal link ... -->` — so revalidation starts clean and a fixed link leaves no residual comment.
2. Collect every inline Markdown link `[text](dest)` with its line number, skipping image links (`![...]`). Ignore external destinations (any URI scheme, or protocol-relative `//…`) and empty ones. Drop a trailing Markdown link title (`path "Title"`), then split an optional `#anchor` off the path (URL-decode the anchor before comparing).
3. Validate each remaining link:
   - Anchor-only (`#foo`) → the source file's own headings must expose the anchor. Anchors are GitHub-style slugs of ATX heading titles: trim, lowercase, strip everything except Unicode letters/numbers/spaces/`_`/`-`, spaces → hyphens; duplicate slugs get `-1`, `-2`, … suffixes. Missing → message `heading anchor "<anchor>" does not exist in <source path>`.
   - Path → resolve relative to the source file's directory (or the wiki root when it starts with `/`), normalized; a path escaping the wiki root is broken (`link "<path>" is outside the wiki root`). A trailing `/` makes it a directory link — the directory must exist (`directory "<path>" does not exist`); otherwise the target file must exist (`file "<path>" does not exist`).
   - Path + anchor (non-directory) → the target file's headings must also expose the anchor (same slug rules) → else `heading anchor "<anchor>" does not exist in "<path>"`.
4. Insert one stamp line directly above each broken link's line (insert bottom-up so line numbers stay valid; multiple broken links on one line get one stamp each), in upstream's exact format: `<!-- openwiki: broken internal link [<href>] <message>. Fix the href or restore the target, then delete this comment. -->`
5. Write a file back only when its content changed. A later update run repairs stamped links per the prompt's "Link integrity" section.

## Step 5 — Persist metadata (after the work; ported from upstream `persistRunMetadataIfChanged`)

Recompute the Step 2 hash with the same command.

- Hash unchanged → **no-op**: do not write `openwiki/.last-update.json`; tell the user the wiki is already current. One exception (#365): if the previous metadata recorded `status: "interrupted"` and this run completed, rewrite the metadata anyway (with `status: "complete"`) even though content is unchanged, so the update no-op check can skip again instead of re-running forever.
- Hash changed → write `openwiki/.last-update.json` with exactly these fields:

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

Run the `date` and `git` commands — never guess the timestamp or the head.

Run this step even when the run fails after generating content (upstream invokes `persistRunMetadataIfChanged` on the error path too): if the hash changed, write the metadata before reporting the failure — with `status: "interrupted"` instead of `"complete"`, so the already-generated content stays diffable and the next update knows the wiki may be partial and does not skip (#365).

## Final response

Summarize the completed documentation changes and important caveats — or state that the wiki is already current. Do not paste subagent reports.

## Automation

Asked to set up scheduled/recurring updates or CI? Read `references/automation.md` in this skill's directory.

## Runtime evidence (LangSmith)

Asked to fold LangSmith runtime traces into the repository wiki? Read `references/runtime-evidence.md` in this skill's directory.
