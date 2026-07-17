---
name: openwiki-ask
description: "Answer questions from an OpenWiki wiki — the repository's openwiki/ or the personal wiki at ~/.openwiki/wiki. Use for 'how does X work / where is Y / explain Z' repo questions, or personal-knowledge questions when a personal wiki exists."
---

# OpenWiki ask — answer from the wiki

Answer the user's question using the maintained OpenWiki wiki as the primary source, instead of re-deriving everything from raw sources.

## Which wiki

- A question about a repository/codebase → that repo's `openwiki/` directory.
- A question about the user's own knowledge, commitments, themes, or connected sources → the personal wiki at `~/.openwiki/wiki`.
- Ambiguous → prefer the repo wiki when working inside a repository that has one; otherwise ask the user.

## Steps

1. **Check the wiki exists.** Missing → say so and offer the matching skill (`openwiki` for a repo wiki, `openwiki-personal` for the personal wiki). You may still answer from source, but note that the wiki is missing.
2. **Orient at the entrypoint.** Read the wiki's `quickstart.md`, follow its links toward the question's area, and grep the wiki for the question's key terms to find the canonical page. Wikis written since upstream v0.2.0 also carry OKF YAML front matter whose `description` is written for retrieval, plus a generated `index.md` per directory — both are good grep/navigation targets.
3. **Answer from the wiki, citing pages.** Name the wiki page(s) the answer comes from, and surface their inline source references so the user can jump to the evidence.
4. **Verify when the wiki falls short.** If the wiki looks stale or does not cover the question, say so, confirm against the actual source before answering, and suggest the matching update run (`openwiki` update / `openwiki-personal` update).

## Wiki-first rules (ported from upstream "Wiki-first question answering")

- For ordinary questions, inspect the generated wiki first. Use quickstart/index pages, section pages, and targeted grep/glob over the wiki before looking at raw evidence.
- If the user asks you to "look at the wiki", answer "based on the wiki", report "what the wiki says", or otherwise frames the request around the wiki, use only wiki pages unless the wiki cannot support the answer.
- Assume the synthesized wiki contains the answer most of the time. Do not inspect raw evidence just because it exists.
- **[adapted]** Route between wikis as in "Which wiki" above. (Upstream: "Never treat a repository-local openwiki/ directory as the canonical generated wiki unless the user explicitly asks about that repository documentation directory" — in this port the repo wiki is first-class for repository questions; the personal wiki is canonical for personal-knowledge questions.)
- Use raw evidence only when the wiki is missing the needed detail, clearly stale, ambiguous, contradicted, the user explicitly asks for source-level evidence, or the question is specifically about the latest uncompiled data since the last wiki update.
- If a wiki-framed question cannot be answered from the wiki, say what important context is missing before deciding whether raw evidence is necessary. When appropriate, suggest or run a targeted source update instead of browsing broad raw dumps.
- When the wiki answers the question, do not inspect or mention raw evidence.
- When you do inspect raw evidence, keep reads narrow: open only the specific files needed, and summarize only the minimum evidence required to answer or update the wiki.

## Ground rules

Ported from upstream's `chat` mode and its shared security rules (loose port — not line-mapped):

- Answering is read-only: do not create or update OpenWiki documentation unless the user explicitly asks you to modify documentation. If the user asks to initialize or update a wiki, run the `openwiki` or `openwiki-personal` skill instead.
- Do not read or quote secret values, credentials, private keys, tokens, .env files, or other sensitive material. .env.example and other sample configuration files may be read only if they contain placeholders, not live secrets.

Keep answers grounded: prefer the wiki's canonical explanation plus a source pointer over guessing.
