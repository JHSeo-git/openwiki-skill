# Automation — scheduled wiki updates

Ways to keep `openwiki/` (and the personal wiki, §5) fresh automatically. Keyless (subscription-auth) options first — they match this port's "no API key" premise; CI needs a credential.

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
      "Bash(head:*)",
      "Bash(sed:*)",
      "Bash(cut:*)",
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

Read/Glob/Grep are already read-only, the git commands above are read-only, and the skill itself forbids reading `.env`/secrets. Writes stay scoped to `openwiki/**` plus the two root instruction files. `head`/`sed`/`cut` cover Step 2's body-hash helper and Step 5's `sources` id derivation. This allowlist is for scheduled **update** runs only: since upstream 0.4.0 an init replaces the wiki, which needs `mktemp`/`cp`/`rm -rf`/`mkdir` for the backup-and-wipe transaction — do not grant those unattended.

## 3. CI with an API credential

CI runners have no subscription login. Set `ANTHROPIC_API_KEY` (API billing) or `CLAUDE_CODE_OAUTH_TOKEN` (subscription; generate with `claude setup-token`) as a CI secret. The skills are cloned into the workspace at `.claude/skills/` (`openwiki` plus its `mermaid-diagrams` companion); the PR/MR steps only add `openwiki/` and the root instruction files, so the clone is never committed.

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
    # Scheduled runs only fire in the origin repo by default, so contributor
    # forks do not silently arm a daily job that requires a provider secret
    # (upstream #370). Replace OWNER/REPO with this repository. Fork owners can
    # opt in without editing this file by setting the
    # OPENWIKI_ENABLE_SCHEDULED_UPDATE repository variable to "true". Manual
    # dispatch remains available for intentional runs and testing.
    if: github.event_name == 'workflow_dispatch' || github.repository == 'OWNER/REPO' || vars.OPENWIKI_ENABLE_SCHEDULED_UPDATE == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
        with:
          # Full history so the update can diff HEAD against the commit it last
          # documented; a shallow clone hides that commit and the update runs
          # against an empty change summary.
          fetch-depth: 0
          persist-credentials: true

      - name: Set up Node.js
        uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4
        with:
          node-version: "22"

      - name: Install Claude Code and the openwiki skills
        run: |
          npm install --global @anthropic-ai/claude-code
          git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill /tmp/openwiki-skill
          mkdir -p .claude/skills
          cp -R /tmp/openwiki-skill/skills/openwiki .claude/skills/openwiki
          cp -R /tmp/openwiki-skill/skills/mermaid-diagrams .claude/skills/mermaid-diagrams

      - name: Run openwiki update
        id: openwiki
        # Publish whatever the run finished even when it fails; the PR step
        # below runs regardless and the failure is propagated at the end.
        continue-on-error: true
        # --dangerously-skip-permissions is acceptable here: the job runs in an
        # isolated, throwaway CI container.
        run: claude --dangerously-skip-permissions -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          # Subscription alternative (instead of ANTHROPIC_API_KEY):
          # CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

      - name: Remove transient OpenWiki run state
        # This skill never writes .run.json, but a native `openwiki` run in the
        # same repository would; it is transient and must never be committed.
        if: ${{ !cancelled() }}
        run: rm -f -- openwiki/.run.json

      - name: Create OpenWiki update pull request
        if: ${{ !cancelled() }}
        uses: peter-evans/create-pull-request@22a9089034f40e5a961c8808d113e2c98fb63676 # v7
        with:
          add-paths: |
            openwiki
            AGENTS.md
            CLAUDE.md
          branch: openwiki/update
          commit-message: "docs: update OpenWiki"
          title: "docs: update OpenWiki"
          body: |
            Automated OpenWiki documentation update.

            OpenWiki result: ${{ steps.openwiki.outcome }}

            When the result is `failure`, this PR intentionally preserves only the
            pages completed before the failure. Merge it to make that progress the
            baseline for the next scheduled run.

      - name: Propagate OpenWiki failure
        if: ${{ steps.openwiki.outcome == 'failure' }}
        run: exit 1
```

**Why the workflow publishes a failed run's output** (ported from upstream 0.5.0, #720). Until 0.4.3 a failed run produced no PR, so an update that died on page nine of twelve threw away eight good pages and the next scheduled run started over from the same place — usually failing at the same page. Now the PR is opened either way, the body says which it was, and merging it makes the finished pages the next run's baseline. `continue-on-error` plus a final `exit 1` keeps the job's status honest: the PR exists *and* the run is still red.

Two caveats. The first is not really specific to this port: `claude -p` exits non-zero only on a hard failure (crash, rate limit, timeout) — a page this skill *skips* per SKILL.md Step 4 is a normal exit, so `steps.openwiki.outcome` will say `success`. Upstream's CLI does the same, exiting 0 for any run that finalized, skipped pages included. The signal for that case is inside the wiki: SKILL.md Step 6 writes `status: "interrupted"` with a rewound `gitHead` in `openwiki/.last-update.json`, and the next run reads it and does not skip. Check that file, not the job status, to tell a complete update from a partial one. Second, upstream's per-page baseline lives in `openwiki/.page-manifest.json`, which this port reads but never writes (SKILL.md Step 1) — the `add-paths: openwiki` above still commits one if a native run created it, which is what you want.

### GitHub Actions with auto-merge

Upstream 0.5.1 (#842) added a second GitHub recipe that lands the docs PR without a human in the loop. Auto-merge is repository infrastructure rather than an OpenWiki feature: the workflow only *requests* it, and the branch's required checks and reviews still decide when GitHub merges.

Set the repository up first — the workflow is inert otherwise:

1. Enable **Allow auto-merge** in the repository's pull request settings.
2. Add branch protection or a ruleset for the default branch. Require the checks that should gate generated docs, and decide whether OpenWiki PRs still need a human review.
3. Create a fine-grained personal access token or GitHub App token scoped to this repository alone, with **Contents: read and write** and **Pull requests: read and write**, and save it as the `OPENWIKI_PR_TOKEN` secret. The dedicated token is not a nicety: pull requests created with the default `GITHUB_TOKEN` do not start most `pull_request` workflows, so the very checks meant to gate the merge may never run. Organization policy may require an App token rather than a PAT.

Then save as `.github/workflows/openwiki-update.yml`. It is §3's recipe plus a `concurrency` group, a completeness check, a dedicated PR token on an id'd PR step, and the two auto-merge steps after it:

```yaml
name: OpenWiki Update and Auto-merge

on:
  workflow_dispatch:
  schedule:
    # GitHub schedules use UTC; 08:00 UTC is midnight PST.
    - cron: "0 8 * * *"

permissions:
  contents: write
  pull-requests: write

# One update at a time: a slow run and the next schedule must not both push
# the openwiki/update branch, and must not race to arm auto-merge on it.
concurrency:
  group: openwiki-update
  cancel-in-progress: false

jobs:
  update:
    # Same fork guard as §3, and it matters more here: a fork must not silently
    # arm a daily job that auto-merges into its default branch.
    if: github.event_name == 'workflow_dispatch' || github.repository == 'OWNER/REPO' || vars.OPENWIKI_ENABLE_SCHEDULED_UPDATE == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
        with:
          fetch-depth: 0
          persist-credentials: true

      - name: Set up Node.js
        uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4
        with:
          node-version: "22"

      - name: Install Claude Code and the openwiki skills
        run: |
          npm install --global @anthropic-ai/claude-code
          git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill /tmp/openwiki-skill
          mkdir -p .claude/skills
          cp -R /tmp/openwiki-skill/skills/openwiki .claude/skills/openwiki
          cp -R /tmp/openwiki-skill/skills/mermaid-diagrams .claude/skills/mermaid-diagrams

      - name: Run openwiki update
        id: openwiki
        continue-on-error: true
        run: claude --dangerously-skip-permissions -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          # Subscription alternative (instead of ANTHROPIC_API_KEY):
          # CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

      - name: Remove transient OpenWiki run state
        if: ${{ !cancelled() }}
        run: rm -f -- openwiki/.run.json

      - name: Check whether the update completed
        id: completeness
        if: ${{ !cancelled() }}
        # `claude -p` exits 0 even when the skill skips a page (SKILL.md Step 4),
        # so the job's outcome cannot gate an unattended merge on its own. The
        # real signal is the metadata SKILL.md Step 6 writes. Anything this step
        # cannot read counts as not-complete: never auto-merge what you cannot
        # prove. (jq is preinstalled on GitHub-hosted runners.)
        run: |
          wiki_status=missing
          if [ -f openwiki/.last-update.json ]; then
            wiki_status=$(jq -r 'if .status == "interrupted" then "interrupted" else "complete" end' openwiki/.last-update.json 2>/dev/null || echo unreadable)
          fi
          echo "status=${wiki_status}" >> "$GITHUB_OUTPUT"

      - name: Create OpenWiki update pull request
        id: create-pr
        if: ${{ !cancelled() }}
        uses: peter-evans/create-pull-request@5f6978faf089d4d20b00c7766989d076bb2fc7f1 # v8.1.1
        with:
          token: ${{ secrets.OPENWIKI_PR_TOKEN }}
          add-paths: |
            openwiki
            AGENTS.md
            CLAUDE.md
          branch: openwiki/update
          commit-message: "docs: update OpenWiki"
          title: "docs: update OpenWiki"
          body: |
            Automated OpenWiki documentation update.

            OpenWiki result: ${{ steps.openwiki.outcome }} (wiki status: ${{ steps.completeness.outputs.status }})

            A `failure` result, or any wiki status other than `complete`, means this
            PR preserves only the pages the run finished, and auto-merge was left
            off. A later successful run can update it.

      - name: Enable auto-merge after a complete update
        if: ${{ steps.openwiki.outcome == 'success' && steps.completeness.outputs.status == 'complete' && steps.create-pr.outputs.pull-request-number != '' }}
        run: |
          [[ "$PR_NUMBER" =~ ^[0-9]+$ ]]
          gh pr merge --auto --squash "$PR_NUMBER"
        env:
          GH_TOKEN: ${{ secrets.OPENWIKI_PR_TOKEN }}
          PR_NUMBER: ${{ steps.create-pr.outputs.pull-request-number }}

      - name: Disable auto-merge after a failed or partial update
        if: ${{ !cancelled() && (steps.openwiki.outcome == 'failure' || steps.completeness.outputs.status != 'complete') && steps.create-pr.outputs.pull-request-number != '' }}
        run: |
          [[ "$PR_NUMBER" =~ ^[0-9]+$ ]]
          gh pr merge --disable-auto "$PR_NUMBER"
        env:
          GH_TOKEN: ${{ secrets.OPENWIKI_PR_TOKEN }}
          PR_NUMBER: ${{ steps.create-pr.outputs.pull-request-number }}

      - name: Propagate OpenWiki failure
        if: ${{ steps.openwiki.outcome == 'failure' }}
        run: exit 1
```

**[adapted]** The completeness gate is this port's, and it is the one step not to drop — it closes a hole upstream's own recipe still has. Upstream arms auto-merge on `steps.openwiki.outcome == 'success'`, which catches a hard failure, since that exits 1. It does not catch a **finalized partial** run: a skipped page (#732) or mid-run source drift (#740) still finishes the run, so `openwiki code --update` exits 0 while writing `status: "interrupted"` into its own metadata — upstream's CLI maps a successful run straight to exit 0 and never reads that status back (`src/cli/app/app.tsx`). `claude -p` behaves the same way for the same reason, which is §3's caveat. Unattended, that is the difference between "the docs PR merged itself" and "the docs PR merged itself while a rewound `gitHead` quietly became the baseline". So both arms here read the signal both runners *do* write, `openwiki/.last-update.json`'s `status`: auto-merge is armed only for a run that finished what it planned, anything the step cannot read counts as not-complete, and the disable arm is widened to clear an auto-merge left pending on a reused branch. Keep the gate even if you swap `claude -p` for the native CLI.

Two smaller differences from upstream's file. It has an `Exclude workflow changes` step (`git checkout -- .github/workflows/openwiki-update.yml`) because upstream's own recipe commits that workflow; this port never writes CI files (SKILL.md Step 0's `[omitted]`) and `add-paths` already excludes it, so nothing needs excluding — keep it that way, since auto-merging an executable workflow file is exactly the change no unattended job should make. And the PR action is pinned a major ahead of §3's (`v8.1.1` vs `v7`), mirroring upstream's two files; both expose `outputs.pull-request-number`, so pin whichever you already trust and keep it pinned by SHA.

### GitLab CI

Include in `.gitlab-ci.yml` (set `ANTHROPIC_API_KEY` and `OPENWIKI_GITLAB_TOKEN` as CI/CD variables):

```yaml
openwiki_update:
  image: node:22
  stage: deploy
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_PIPELINE_SOURCE == "web"'
  variables:
    # Full history so the update can diff HEAD against the commit it last
    # documented; GitLab's default shallow clone hides that commit and the
    # update runs against an empty change summary.
    GIT_DEPTH: "0"
  before_script:
    - apt-get update && apt-get install -y git curl
    - npm install --global @anthropic-ai/claude-code
    - git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill /tmp/openwiki-skill
    - mkdir -p .claude/skills
    - cp -R /tmp/openwiki-skill/skills/openwiki .claude/skills/openwiki
    - cp -R /tmp/openwiki-skill/skills/mermaid-diagrams .claude/skills/mermaid-diagrams
    - git config user.name "${GITLAB_USER_NAME:-OpenWiki Bot}"
    - git config user.email "${GITLAB_USER_EMAIL:-openwiki@example.com}"
  script:
    # One block so the captured exit code survives; GitLab may inject its own
    # commands between separate script entries.
    - |
      set +e
      claude --dangerously-skip-permissions -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
      OPENWIKI_EXIT=$?
      set -e
      # Transient native-CLI run state must never be committed.
      rm -f -- openwiki/.run.json
      if [ -z "$(git status --porcelain -- openwiki AGENTS.md CLAUDE.md)" ]; then
        echo "OpenWiki produced no changes."
        exit "$OPENWIKI_EXIT"
      fi
      OPENWIKI_BRANCH="openwiki/update-${CI_PIPELINE_ID}"
      git checkout -b "$OPENWIKI_BRANCH"
      git add openwiki AGENTS.md CLAUDE.md
      git commit -m "docs: update OpenWiki"
      git push "https://oauth2:${OPENWIKI_GITLAB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" "$OPENWIKI_BRANCH"
      curl --fail --request POST \
        --header "PRIVATE-TOKEN: ${OPENWIKI_GITLAB_TOKEN}" \
        --form "source_branch=${OPENWIKI_BRANCH}" \
        --form "target_branch=${CI_DEFAULT_BRANCH}" \
        --form "title=docs: update OpenWiki" \
        --form "description=Automated OpenWiki documentation update. OpenWiki exit code: ${OPENWIKI_EXIT}. A non-zero code means this MR preserves only the pages completed before the failure; merge it to make that progress the next run's baseline." \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests"
      # Publish progress first, then keep the pipeline red.
      exit "$OPENWIKI_EXIT"
```

### Bitbucket Pipelines

Save as `bitbucket-pipelines.yml` (set `ANTHROPIC_API_KEY` as a secured repository variable — Bitbucket exports it to the build environment; `OPENWIKI_BITBUCKET_TOKEN` must be a repository access token with write access to repositories and pull requests). Configure a scheduled pipeline in Repository settings > Pipelines > Schedules that runs the `openwiki-update` custom pipeline (e.g. daily):

```yaml
image: node:22

pipelines:
  custom:
    openwiki-update:
      - step:
          name: Run OpenWiki update
          # Full history so the update can diff HEAD against the commit it last
          # documented; Bitbucket's default shallow clone hides that commit and
          # the update runs against an empty change summary.
          clone:
            depth: full
          script:
            - apt-get update && apt-get install -y git curl
            - npm install --global @anthropic-ai/claude-code
            - git clone --depth 1 https://github.com/JHSeo-git/openwiki-skill /tmp/openwiki-skill
            - mkdir -p .claude/skills
            - cp -R /tmp/openwiki-skill/skills/openwiki .claude/skills/openwiki
            - cp -R /tmp/openwiki-skill/skills/mermaid-diagrams .claude/skills/mermaid-diagrams
            - git config user.name "${BITBUCKET_USER_NAME:-OpenWiki Bot}"
            - git config user.email "${BITBUCKET_USER_EMAIL:-openwiki@example.com}"
            # One block so the captured exit code survives.
            - |
              set +e
              claude --dangerously-skip-permissions -p "Use the openwiki skill to update this repository's wiki. If it reports the wiki is already current, change nothing."
              OPENWIKI_EXIT=$?
              set -e
              # Transient native-CLI run state must never be committed.
              rm -f -- openwiki/.run.json
              if [ -z "$(git status --porcelain -- openwiki AGENTS.md CLAUDE.md)" ]; then
                echo "OpenWiki produced no changes."
                exit "$OPENWIKI_EXIT"
              fi
              OPENWIKI_BRANCH="openwiki/update-${BITBUCKET_BUILD_NUMBER}"
              git checkout -b "$OPENWIKI_BRANCH"
              git add openwiki AGENTS.md CLAUDE.md
              git commit -m "docs: update OpenWiki"
              git push "https://x-token-auth:${OPENWIKI_BITBUCKET_TOKEN}@bitbucket.org/${BITBUCKET_WORKSPACE}/${BITBUCKET_REPO_SLUG}.git" "$OPENWIKI_BRANCH"
              curl --fail --request POST \
                --url "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_WORKSPACE}/${BITBUCKET_REPO_SLUG}/pullrequests" \
                --header "Authorization: Bearer ${OPENWIKI_BITBUCKET_TOKEN}" \
                --header "Content-Type: application/json" \
                --data "{\"title\":\"docs: update OpenWiki\",\"description\":\"Automated OpenWiki documentation update. OpenWiki exit code: ${OPENWIKI_EXIT}. A non-zero code means this PR preserves only the pages completed before the failure; merge it to make that progress the next run's baseline.\",\"source\":{\"branch\":{\"name\":\"${OPENWIKI_BRANCH}\"}},\"destination\":{\"branch\":{\"name\":\"${BITBUCKET_BRANCH}\"}}}"
              # Publish progress first, then keep the pipeline red.
              exit "$OPENWIKI_EXIT"
```

## 4. Codex headless

Codex runs the same update through its non-interactive mode:

```bash
codex exec "Use the openwiki skill to update this repository's wiki."
```

Check `codex exec --help` for your version's sandbox/approval flags; the run only needs read-only git plus writes under `openwiki/`, `AGENTS.md`, and `CLAUDE.md`.

## 5. Personal wiki schedules (openwiki-personal)

Upstream schedules personal ingestion with a macOS launchd job running `openwiki ingest all`. The keyless equivalent here is the same cron pattern as §1, pointed at the `openwiki-personal` skill — one entry per source keeps runs small (upstream also runs one source-specific update per connector):

```
7 8 * * 1-5 claude -p "Use the openwiki-personal skill to run a source update for slack." >> "$HOME/.openwiki/logs/ingestion.schedule.log" 2>&1
17 8 * * 1-5 claude -p "Use the openwiki-personal skill to run a source update for gmail." >> "$HOME/.openwiki/logs/ingestion.schedule.log" 2>&1
```

Scoped permissions (user-level `~/.claude/settings.json`, since runs are not repo-rooted): allow `Write(~/.openwiki/**)` and `Edit(~/.openwiki/**)` plus the read-only commands from §2 (`find`, `shasum`/`sha256sum`, `date`, `rg`), and whatever read-only MCP tools the sources need. The log directory matches upstream's (`~/.openwiki/logs/`); create it once with `mkdir -p ~/.openwiki/logs`.
