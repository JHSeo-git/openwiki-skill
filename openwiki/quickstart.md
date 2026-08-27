---
type: Quickstart Guide
title: openwiki-skill quickstart
description: Entry point for the openwiki-skill repository wiki — what this prompt-only port of langchain-ai/openwiki is, a task-routing map from intent to the page and files that own it, and the invariants any change must preserve.
tags: [quickstart, openwiki, skills, port, routing]
sources:
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
  - id: openwiki-source-ff18555ad70362de0f452788
    resource: repo://UPSTREAM.md
  - id: openwiki-source-d781a3f9949210939d21e676
    resource: repo://docs/superpowers/plans/2026-07-10-openwiki-0.1.0-port.md
  - id: openwiki-source-dbd023210529f29ad3d88c02
    resource: repo://docs/superpowers/specs/2026-07-07-openwiki-skill-design.md
  - id: openwiki-source-a086a22600cde1e5b60be90f
    resource: repo://skills/openwiki-personal/SKILL.md
  - id: openwiki-source-f27335ea429d443b8de638e2
    resource: repo://skills/openwiki/SKILL.md
generated: { by: "claude-code", at: "2026-08-27T00:28:56.000Z" }
---

# openwiki-skill quickstart

Agent skills that write, maintain, and answer from OpenWiki wikis — a port of
[langchain-ai/openwiki](https://github.com/langchain-ai/openwiki) **v0.4.1** for coding
agents such as Claude Code and Codex.

Upstream is a CLI that drives a model through provider APIs. This repository drops that
plumbing: your coding agent already *is* the model, with filesystem and git tools attached,
so it executes the same workflow directly. Upstream's prompts are reproduced verbatim inside
the skills; harness differences are marked `[adapted]` and missing behavior `[omitted]`. No
API key, no runtime, no configuration.

There is no build, no dependency manifest, and no test suite. **The deliverable is Markdown
that an agent reads**, which makes fidelity to upstream — not compilation — the correctness
criterion.

## Task routing

| If you want to… | Read | Files that own it |
|---|---|---|
| Understand what "port" means and what may be edited | [The port contract](architecture/port-contract.md) | `UPSTREAM.md`, every `skills/*/SKILL.md` |
| Find which skill does what, or add a skill | [Skill catalog and boundaries](architecture/skill-catalog.md) | `skills/` |
| Change how a repository wiki is generated | [Repository wiki run](workflows/repository-wiki-run.md) | `skills/openwiki/SKILL.md`, `skills/openwiki/references/prompt-planner.md`, `prompt-page.md` |
| Change how the personal wiki is built or ingested | [Personal wiki run](workflows/personal-wiki-run.md) | `skills/openwiki-personal/SKILL.md`, `references/sources.md`, `references/connectors.md` |
| Port a new upstream release | [Upstream sync](workflows/upstream-sync.md) | `UPSTREAM.md`, `CHANGELOG.md` |
| Change front matter, indexes, diagrams, or link validation | [OKF output contract](concepts/okf-output.md) | both `SKILL.md` finalize steps, `skills/mermaid-diagrams/SKILL.md` |
| Work on evidence, Claims, or staleness detection | [Claims and grounding](concepts/claims-and-grounding.md) | `skills/openwiki/references/prompt-page.md`, `skills/openwiki/SKILL.md` Steps 1 and 5 |
| Schedule updates, set up CI, or scope permissions | [Automation](operations/automation.md) | `skills/openwiki/references/automation.md` |
| Change wording users see before installing | — | `README.md` |

## The two lifecycles at a glance

**Repository wiki** (`openwiki`) — seven steps: marker setup, context with evidence
preflight and no-op check, prepare, plan, per-page queue, finalize, metadata. A bounded
planner fixes the page set; one worker writes each page and records Claims against
`repo://` evidence. **Init regenerates the wiki from scratch**, keeping only a user-authored
`openwiki/INSTRUCTIONS.md`.

**Personal wiki** (`openwiki-personal`) — five steps: context, prepare, one monolithic
authoring prompt, finalize, metadata. This is the only mode still on upstream's shared
agent, and the only one that still runs a whole-wiki translation pass. Its init is not
destructive.

Both produce the same artifact shape: OKF v0.2 concept pages, deterministic per-directory
indexes, code-owned `generated` provenance, self-correcting Mermaid and link validation.

## Invariants

Four rules govern every change here. Each is expanded in
[the port contract](architecture/port-contract.md).

1. **Unmarked text is upstream's — never paraphrase it.** Paraphrase breaks the next sync's
   diff.
2. **A behavior change needs a marker.** New behavior upstream lacks, or dropped behavior it
   has, without an `[adapted]` or `[omitted]` note is indistinguishable from drift.
3. **Deterministic passes stay deterministic.** Write only when content differs. That is what
   keeps a no-op a no-op and lets either tool continue the other's wiki.
4. **Version stamps move together.** The pin, the skill intros, and the reference-file
   provenance headers name one upstream version.

Two mechanical traps worth knowing before you edit a skill:

- A **dollar sign followed by a digit** inside a `SKILL.md` is substituted with the skill's
  invocation arguments before the agent reads the file. Shell snippets use named variables.
- The two `references/index-labels.md` files are **byte-identical duplicates on purpose** —
  skills load per directory, so they cannot share one file. Change both.

## Getting oriented in the source

`skills/openwiki/SKILL.md` is the best single entry point: it is the largest skill, it names
its upstream source for every step, and its `[adapted]` notes are where the port's reasoning
is most visible. Read it beside `UPSTREAM.md`'s mapping table, which tells you what each step
was ported from.

`docs/superpowers/` holds the original design specs and implementation plans. They are
historical — how the repository came to exist, not how it behaves now.
