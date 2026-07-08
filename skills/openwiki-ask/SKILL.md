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
