---
type: Concept
title: OKF output contract
description: The artifact both wiki skills produce — OKF v0.2 concept front matter with its author-owned and code-owned fields, reserved files, deterministic directory indexes, the normalization pass that repairs non-compliant pages, and the render-safety passes for diagrams and internal links.
tags: [okf, frontmatter, provenance, indexes, determinism]
sources:
  - id: openwiki-source-0b5fb88ef5740ec3069d4d5d
    resource: repo://skills/mermaid-diagrams/SKILL.md
  - id: openwiki-source-a086a22600cde1e5b60be90f
    resource: repo://skills/openwiki-personal/SKILL.md
  - id: openwiki-source-f27335ea429d443b8de638e2
    resource: repo://skills/openwiki/SKILL.md
  - id: openwiki-source-88378f6a3ac54313171799db
    resource: repo://skills/openwiki/references/prompt-page.md
generated: {by: "claude-code", at: "2026-08-26T00:12:15.000Z"}
---

# OKF output contract

Both wiki skills emit the same artifact: a directory of Markdown concept pages, each
opening with Open Knowledge Format front matter, cross-linked as a concept graph, with a
deterministically generated `index.md` per directory. Upstream 0.4.0 moved this output
from OKF v0.1 to **v0.2**, whose defining change is that some front-matter fields became
**code-owned**.

The artifact is the interoperability surface: because this port emits byte-compatible
output, the upstream CLI can continue a wiki this port started and vice versa. That is why
the field shapes below are exact rather than approximate.

## Author-owned front matter

A page worker writes exactly these fields:

```yaml
---
type: <short descriptive concept type>
title: <human-readable page title>
description: <one or two sentence retrieval-oriented summary>
tags: [<stable English tag>, ...]
---
```

- **`type` is the only required field.** It is a free-form human concept kind — `API
  Endpoint`, `Playbook`, `Metric`, `Reference` — deliberately not drawn from a registry.
- `description` is the retrieval surface. It is what a reader or an agent greps and what
  a directory index quotes, so it is written for search rather than for prose flow.
- `tags` stay in **English even in a non-English wiki**, so they remain stable
  cross-cutting aggregation keys. `type`, `title`, and `description` are written in the
  wiki's language.
- Unknown producer-defined fields are valid OKF and **must survive round trips**. An
  update preserves them unless they are factually wrong.

## Code-owned front matter

These fields are never authored by a page worker. Writing them by hand is a boundary
violation, not a shortcut:

| Field | Owner | Meaning |
|---|---|---|
| `generated` | finalize step | `{by, at}` — the producing actor and the time the page's **body** last changed |
| `sources` | finalize step | Provenance resources projected from the page's Claims |
| `verified` | upstream only | Trust events; **omitted by this port** |
| `timestamp` | legacy | Superseded by `generated.at`; dropped from any page whose body changes |

### `generated` — provenance on body change

The rule is precise and matters for diff hygiene:

- A **new page, or a page whose body hash changed**, gets the run's shared timestamp:
  `generated: {by: "claude-code", at: "<run timestamp>"}`, and any legacy `timestamp` is
  dropped. Whitespace counts as a body change.
- A page whose **body is unchanged** has its prior event restored exactly, even if the
  run's own edits dropped or altered it.

The body is the content after the leading front-matter block, so a front-matter-only edit
never advances the stamp. This is why the provenance pass runs **last**, after every other
finalize pass: the index, link, and `sources` passes all edit front matter, and none of
those edits should look like a content change.

The actor is the producing host — `claude-code`, `codex`, `opencode` — where upstream
stamps its own version. See [the port contract](../architecture/port-contract.md) for why.

### `sources` — Claim evidence as provenance

Rendered shape:

```yaml
sources:
  - id: openwiki-source-a953060a04ccefcf777de48e
    resource: repo://src/agent/index.ts
```

The id suffix is the first 24 hex characters of the SHA-256 of the resource string, which
makes it deterministic: a later run replaces or removes only its own projection and leaves
entries another producer authored alone. Line ranges are reduced to whole-file form here —
precise spans stay in the Claim, while OKF provenance exposes source *files*.
[Claims and grounding](claims-and-grounding.md) covers where the resources come from and
how the field is read back as a staleness signal.

## Reserved files

`index.md`, `log.md`, and `INSTRUCTIONS.md` are reserved. They are never given concept
front matter, never validated as concepts, never planned as pages, and never carry Claims.
`INSTRUCTIONS.md` in particular is **user-authored control metadata**, not generated
documentation: a run reads it as scope and priority input and never rewrites it.

Nothing in this port writes a `log.md`. Reserved-file handling would tolerate one, but the
port deliberately does not generate what upstream does not.

## Normalization repairs non-compliance

Before any authoring, every concept page is checked:

- Front matter parses as YAML **and** `type` is a non-empty string → left byte-for-byte
  untouched, even if optional fields are junk. An author's `type` and custom fields are
  never overwritten.
- Otherwise → front matter is replaced with a minimal block carrying `type`, a `title`
  derived from the first H1 (falling back to the filename), and `openwiki_generated: true`
  as a flag that the metadata was code-derived.

A page worker that later touches a flagged page replaces that placeholder with accurate
metadata grounded in the body and removes the flag.

The rebuild would otherwise drop fields it does not know about, so it carries across
verbatim: the `openwiki_translation_pending` control marker, and the structured v0.2
families `generated`, `verified`, and `sources`. A page that is both non-compliant and
stamped must not lose its provenance.

## Deterministic directory indexes

Every directory gets an `index.md` regenerated from its direct children — files by their
`title` and `description`, subdirectories by name — sorted by link target, with `\`, `[`,
and `]` escaped in labels. Only the wiki-root index carries front matter, exactly:

```yaml
---
okf_version: "0.2"
---
```

Headings are `Files` / `Directories` in English and come from a curated per-language table
otherwise — structural chrome, never translated ad hoc. An empty directory renders just
the files heading. Output is compared before writing, so identical content is skipped and
a no-op run produces no diff.

Index pages are generated, so a page worker never creates or edits one, and a plan never
includes one.

## Render safety

Two finalize passes keep a page from breaking where it is read:

- **Mermaid.** Every fence is checked; one that fails is either repaired in place or
  degraded to a `text` fence preceded by an `openwiki: mermaid parse failed` comment
  carrying the parser error. The page renders either way. Files whose fences all parse are
  left byte-for-byte unchanged.
- **Internal links.** Previous stamps are cleared first so a fixed link leaves no
  residue, then every relative link and heading anchor is validated **repo-wide** — a wiki
  page may legitimately link out to any repository file. Broken ones are stamped in place
  with `openwiki: broken internal link` and the run still succeeds.

Both passes are self-correcting by design: they never fail a run, and the stamped comment
is the instruction a later run repairs from. Anchors are GitHub-style slugs and are checked
only on Markdown targets, so a `#L10` line anchor on a source file is never flagged.

## Where this is executed

The exact commands, orderings, and exclusion lists live in the two lifecycles:
[repository wiki run](../workflows/repository-wiki-run.md) and
[personal wiki run](../workflows/personal-wiki-run.md). The orderings are part of the
contract, not an implementation detail — provenance must be stamped after the passes that
edit front matter, and indexes must be synchronized before links are validated so a
freshly generated index is what gets checked.
