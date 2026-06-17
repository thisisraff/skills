---
name: pr-review-until-converged
description: Use when reviewing a PR iteratively to convergence — runs parallel multi-reviewer review, judges with a binary blocks-merge gate, fixes via parallel subagents, and tracks a per-PR dismissed-findings ledger so the same concerns aren't re-raised across rounds. Escalates to human on stalled convergence.
---

# PR review until converged

You are the **host** of a multi-agent PR review loop. Your job is to drive review → judge → fix iterations against a PR until either it's ready to merge or convergence stalls and a human needs to step in.

**You orchestrate. You do not classify.** Every gate-relevant judgment happens in a subagent with a deliberately bounded context. You parse YAML, cluster blockers, sequence commits, decide when to escalate. You never decide whether something is a blocker — that's the judge's job.

## Invocation

Argument: `<PR#>` (optional). If absent, resolve the open PR for the current branch:

```bash
gh pr view --json number,baseRefName,headRefName,headRefOid -q '.'
```

If no open PR exists for the current branch, abort with an explanation.

## Branch presence check (REQUIRED at startup)

The skill needs the working tree to be on the PR's head branch — both reviewers and fix subagents work against the files there, and the host commits to that branch. **Before doing anything else, verify branch alignment**:

```bash
PR_HEAD=$(gh pr view <PR#> --json headRefName -q '.headRefName')
CURRENT=$(git branch --show-current)
if [ "$PR_HEAD" != "$CURRENT" ]; then
  # Branch mismatch — three valid responses
fi
```

Branch mismatch responses, in order of preference:

1. **The PR's branch already exists locally**: tell the user and abort. *"This worktree is on `<CURRENT>` but PR #N is `<PR_HEAD>`. Switch to that branch (or to a worktree that has it checked out) and re-invoke."* Do NOT auto-switch — the user may have uncommitted work here.

2. **The PR's branch doesn't exist locally**: offer to set up a worktree for it. *"PR #N is `<PR_HEAD>`, which isn't checked out anywhere local. Want me to `git worktree add ../<repo>-pr-<N> -b <PR_HEAD> origin/<PR_HEAD>` and re-invoke from there?"* Wait for user confirmation.

3. **Review-only mode** (explicit user override): if the user says "review only" or passes a `--review-only` flag, run reviewers + judge against the diff fetched via `gh pr diff <PR#>`, but **refuse to enter the fix step**. Exit after the judge stage with a "would-fix" report. This is useful for spot-checking a PR from another branch but is degraded — no fixes, no commits, no ledger persistence.

In all three cases, never silently proceed to write/commit against the wrong branch.

## Per-PR state

The ledger lives at `<repo>/.pr-review/pr-<n>.yaml`. It is a **local-only artifact — never committed**. On first invocation it's empty. On subsequent invocations, load it and use it as both (a) the suppression context for reviewers and (b) the few-shot calibration for the judge.

**Keep it out of the repo (REQUIRED at startup).** Before writing any ledger, ensure `.pr-review/` is gitignored in the target repo so review bookkeeping never lands in the feature branch's history and `git add` / `git status` stay clean. Idempotent:

```bash
grep -qxF '.pr-review/' "$(git rev-parse --show-toplevel)/.gitignore" 2>/dev/null \
  || printf '\n# pr-review-until-converged ledger (local only)\n.pr-review/\n' >> "$(git rev-parse --show-toplevel)/.gitignore"
```

Persistence is therefore per-working-tree: the ledger survives across rounds and re-invocations *in this checkout*, but a fresh clone or another machine starts with an empty ledger. To carry a decision across machines, set `decided_by: human` and copy the entry by hand — that's the intended override channel.

Ledger entry shape:
```yaml
- id: kebab-case-slug                      # judge mints on first appearance, reuses on match
  raised_round: 1
  raised_by: claude                        # or codex, etc.
  where: path/to/file.ts:42
  summary: "one-line description"
  trigger_question: bugs                   # or bug-class-spread, security, speed, maintainability, practices
  judge_decision: blocker                  # or not-a-blocker
  bucket: worth-a-look                     # only when not-a-blocker: worth-a-look | minor-nit
  decided_by: judge                        # or human (sticky — judge must not overwrite)
  failed_fix_attempts: 0                   # incremented when raised, fixed, raised again
  cluster_id: user-id-erased-in-refresh    # optional — judge tags shared root causes
  rationale: |
    Why the judge decided this.
```

## The loop

Pseudocode for what you execute each round:

```
load ledger from .pr-review/pr-<n>.yaml

for round in 1..MAX_ROUNDS (default 10, backstop only):
    git fetch && git rebase origin/<base>
    if rebase has unresolved conflicts: ESCALATE("base moved, conflict")

    available_reviewers = ["claude"]
    if command -v codex: available_reviewers += ["codex"]

    findings = dispatch_in_parallel(available_reviewers)   # each returns YAML
    canonical = dispatch_judge(findings, ledger, diff)     # subagent, returns YAML
    blockers = canonical.where(blocks_merge == true)

    if blockers.empty:
        write_nits_report(canonical.where(blocks_merge == false))
        RETURN "READY TO MERGE"

    for b in blockers:
        if ledger[b.id].failed_fix_attempts >= 2:
            ESCALATE(f"blocker {b.id} failed to fix after 2 attempts")

    fresh = {b.id for b in blockers} - {b.id for b in previous_blockers}
    if previous_blockers and (len(blockers) >= len(previous_blockers) or fresh):
        decision = ask_human(
          f"Round {round}: started {len(previous_blockers)}, now {len(blockers)} "
          f"({len(fresh)} new). Continue / abort / take over?"
        )
        if decision != "continue": RETURN decision

    if wall_clock_spent > MAX_WALL_CLOCK (default 60min):
        ESCALATE("time budget exhausted")

    apply_fixes(blockers)
    previous_blockers = blockers
    save ledger

RETURN "NEEDS HUMAN: hit outer round cap"
```

## Step-by-step expansion

### 1. Round setup

- **On round 1 only**: write the start timestamp to `/tmp/pr-review-<PR#>-state.json` (`{"started_at": <unix_seconds>, "round": 1}`). On round 2+: read it, increment `round`. This is how you track wall-clock spent — there's no other state-keeping mechanism, so this file IS the wall-clock state.
- `git fetch && git rebase origin/<base>` on the PR branch. If the rebase has conflicts you can't trivially resolve (no `git rerere` cache, non-trivial diff), output: `"Base branch moved and conflicts. Resolve and re-invoke."` and exit.
- Compute the per-round inputs you'll pass to subagents:
  - **Full PR diff**: `gh pr diff <PR#> > /tmp/pr-<PR#>-diff.txt` (capture to file, don't load into your context — it can be hundreds of KB).
  - **"Since last round" callout** (round 2+): the commits you made last round, with their SHAs and the ledger IDs they addressed.
  - **Rendered ledger**: read `<repo>/.pr-review/pr-<PR#>.yaml`; render its body inline for the prompts.

### Prompt rendering — temp file pattern

For every subagent dispatch (reviewers, judge, fixers), follow this pattern to avoid loading large content (diff, findings, ledger) into your own context twice:

1. **Render the filled-in prompt to a temp file** using `python3` for safe string substitution (sed breaks on multiline content). Example for the reviewer prompt:

   ```bash
   python3 <<PYEOF
   template = open('$HOME/.claude/skills/pr-review-until-converged/reviewer-prompt.md').read()
   diff = open('/tmp/pr-<PR#>-diff.txt').read()
   ledger = open('<repo>/.pr-review/pr-<PR#>.yaml').read() if exists else '(empty - first round)'
   out = (template
     .replace('{pr_ref}', '<resolved-pr-ref>')
     .replace('{pr_diff}', diff)
     .replace('{since_last_round}', '<empty-or-callout>')
     .replace('{ledger}', ledger))
   open('/tmp/pr-<PR#>-reviewer-prompt.md', 'w').write(out)
   PYEOF
   ```

2. **Dispatch the subagent with a short brief that points at the file** (don't inline the prompt in your Agent call — that loads everything into your context):

   ```
   Agent(
     subagent_type: general-purpose,
     prompt: "Read /tmp/pr-<PR#>-reviewer-prompt.md and execute its instructions literally. Output ONLY YAML, no prose. Return the YAML as your final message."
   )
   ```

3. **For codex**, pipe the temp file in via stdin (avoids shell quoting issues with the prompt's special chars):

   ```bash
   codex exec - < /tmp/pr-<PR#>-reviewer-prompt.md > /tmp/pr-<PR#>-codex-output.txt 2>/tmp/pr-<PR#>-codex-stderr.txt
   ```

This pattern keeps your context lean across many rounds — the diff, findings, and ledger never enter your context, only the subagents'.

### Parsing subagent output — extract YAML tolerantly

Subagents (claude reviewer, codex reviewer, judge) **will sometimes preface their YAML with prose** despite repeated instructions not to. This is a fundamental tendency of reasoning models — they think before delivering. Do not rely on perfect compliance. Instead, **extract the YAML programmatically** by finding the schema's top-level key and parsing from there.

Use this python pattern for every subagent output:

```bash
python3 <<'PYEOF'
import sys, re, yaml
raw = open('/tmp/pr-<PR#>-<subagent>-output.txt').read()

# Find the first occurrence of the schema's top-level key.
# Reviewers use trigger_findings:; judge uses canonical_findings:; fixer uses status:
TOP_KEY = "canonical_findings:"   # change per subagent
m = re.search(rf'^{re.escape(TOP_KEY)}', raw, re.MULTILINE)
if not m:
    print("ERROR: top-level key not found in output", file=sys.stderr)
    sys.exit(1)

# Strip any closing ```yaml fence if present (some agents wrap despite being told not to)
candidate = raw[m.start():]
candidate = re.sub(r'\n```\s*$', '\n', candidate.rstrip())

try:
    parsed = yaml.safe_load(candidate)
    open('/tmp/pr-<PR#>-<subagent>-parsed.yaml', 'w').write(yaml.safe_dump(parsed))
    print("OK")
except yaml.YAMLError as e:
    print(f"YAML PARSE ERROR: {e}", file=sys.stderr)
    sys.exit(2)
PYEOF
```

For Agent (claude) subagent dispatches: the Agent's final-message text is what you save to the temp file. For codex: stdout from `codex exec` is what you save.

If extraction fails entirely (no top-level key found, YAML invalid), treat it as a reviewer failure — drop that reviewer's findings for the round, log loudly, continue. Don't try to repair malformed YAML by hand.

### 2. Dispatch reviewers in parallel

- **Claude reviewer**: dispatch via the `Agent` tool with `subagent_type: general-purpose`. Brief = filled-in `reviewer-prompt.md` template (substitute `{pr_ref}`, `{pr_diff}`, `{since_last_round}`, `{ledger}`).
- **Codex reviewer** (if `command -v codex` succeeds): `codex exec "$(cat filled-prompt.md)"` via Bash, capture stdout. Same placeholders.
- Both should be running in parallel — issue both tool calls in the same message.

Each returns YAML with `trigger_findings` and `observations`.

**If a reviewer fails** (subagent error, codex non-zero exit): log the failure, drop that reviewer's findings for the round, continue with whatever remains. If *all* reviewers fail, escalate.

### 3. Dispatch the judge subagent

- Use `Agent` tool with `subagent_type: general-purpose`. **Do not classify findings yourself** — your context contains conversation history that would bias the gate decision.
- Judge brief = filled-in `judge-prompt.md` template (substitute `{pr_ref}`, `{pr_diff}`, `{reviewer_findings}`, `{ledger}`). Bounded context: only those four inputs plus the prompt itself.
- Judge returns YAML: `canonical_findings[]` with `id`, `blocks_merge`, `bucket` (when not blocker), `cluster_id` (when shared root cause), `decided_by`, and `rationale`.

### 4. Apply ledger updates

Take the judge's classifications, merge into the ledger:
- New IDs → new entries with `decided_by: judge`, `failed_fix_attempts: 0`.
- Existing IDs that were raised again → increment `failed_fix_attempts` only if a fix was attempted last round.
- Existing IDs no longer raised → leave alone (they're fixed or no longer found).
- **Never overwrite entries with `decided_by: human`.**

### 5. Convergence checks (in order)

In this exact order — fail-fast:

a. **READY TO MERGE**: blockers is empty. Write nits report (from `bucket: worth-a-look | minor-nit` items) to a comment or file, exit successfully.

b. **Per-blocker novelty**: any blocker with `failed_fix_attempts >= 2` → escalate that specific ID to human with the ledger entry's full context.

c. **Trajectory checkpoint**: if `len(blockers) >= len(previous_blockers)` OR fresh blocker IDs appeared after a fix round → output the prompt to the user (this turn) and wait for their next message. Parse `continue` / `abort` / `take over` (or paraphrases — be generous). Default to `abort` if ambiguous.

d. **Wall-clock cap**: compute `elapsed = now - state.started_at` (both unix seconds, from `/tmp/pr-review-<PR#>-state.json`). If `elapsed > MAX_WALL_CLOCK * 60`, escalate with `"time budget exhausted: <elapsed>s vs cap <cap>s"`.

e. **Backstop round cap**: round count > MAX_ROUNDS → escalate.

### 6. Fix step — cluster, dispatch, commit

**Clustering:**
1. Group blockers by `cluster_id` first — same cluster goes to one fix subagent regardless of file paths.
2. Remaining blockers (no `cluster_id`) → group by overlapping file sets.

For each cluster, dispatch a fix subagent via `Agent` (`subagent_type: general-purpose`) with the `fixer-prompt.md` template filled in. **Dispatch all clusters' subagents in parallel** via `superpowers:dispatching-parallel-agents` (single message, multiple `Agent` tool calls).

**Each fix subagent's brief** = filled-in `fixer-prompt.md` template. Substitute:
- `{cluster_summary}` — for cluster fixes, a one-line description of the shared root cause; empty for non-cluster.
- `{findings}` — the finding(s) being fixed (where, what, trigger_question, evidence).
- `{file_allowlist}` — the files this subagent may edit (newline-separated). No others.
- `{relevant_ledger}` — ledger entries near these findings (same files or same cluster). Prevents the subagent from undoing a documented tradeoff while fixing something adjacent.
- `{branch}`, `{base}`, `{head_sha}` — current git state.

The prompt itself encodes the scope-creep guardrail and the `unable_to_fix` escape hatch.

**Commit per cluster** (not per finding, not per round):
```
fix(review): <ledger-id-1>, <ledger-id-2> — <one-line summary>
```

**Partial failure**: if a subagent returns `unable_to_fix`, add a new ledger entry with `attempted_fix_failed: <reason>` (`decided_by: judge`), and commit whatever the other subagents did succeed at.

### 7. Push and loop

`git push` after committing the fixes. Update `previous_blockers`. Save the ledger to disk (it's gitignored — never staged, committed, or pushed). Continue to next round.

## Exit reports

When you exit, produce the report **twice**: once in your final chat message (so the human invoking the skill sees it), and once as a `gh pr comment` on the PR (so collaborators see it). The two are the same content. Use `gh pr comment <PR#> --body-file <path>` with the report saved to a temp file.

**READY TO MERGE**:
```
PR #<n> review complete after <N> rounds — ready to merge.

Nits (not blocking):
- <id>: <summary> [worth-a-look]
- <id>: <summary> [minor-nit]
...

Ledger: .pr-review/pr-<n>.yaml — local only (gitignored, not committed).
```

**NEEDS HUMAN**:
```
PR #<n> review halted at round <N>. Reason: <reason>.

Persistent blockers (need your attention):
- <id>: <summary>
  Where: <path>:<line>
  Failed fix attempts: <count>
  Last rationale: <judge rationale>
...

Ledger: .pr-review/pr-<n>.yaml — local only (gitignored, not committed). Edit
the ledger to override decisions (set decided_by: human) and re-invoke to resume.
```

**Review-only mode exit** (PR not on current branch, no fix step run):
```
PR #<n> review-only pass complete — would proceed to fix step with <K> blockers.

Blockers (audited, would be fixed):
- <id>: <summary> [trigger: <trigger_question>]
  Where: <path>:<line>
  Rationale: <judge rationale, ~2 lines>
...

Nits (would be in nits report):
- <id>: <summary> [worth-a-look|minor-nit]
...

To run the full loop with fixes, switch to branch `<head_ref>` and re-invoke.
```

**Posting to the PR**: prepend `🤖 _Automated review via `pr-review-until-converged`._` to the body so readers know it's not a human. If `gh pr comment` fails (auth, network, etc.), log the failure and still print to chat — comment posting is a nice-to-have, not a critical-path step.

## Defaults

- `MAX_WALL_CLOCK`: 60 minutes
- `MAX_ROUNDS`: 10 (backstop)
- `MAX_NOVELTY_ATTEMPTS`: 2 (escalate after 2 failed fix attempts on same ID)

These can be overridden per-repo via `<repo>/.pr-review/config.yaml` if it exists.

## What you must NOT do

- **Do not classify findings yourself.** That's the judge's job, in a subagent with bounded context.
- **Do not edit files yourself in the fix step.** Dispatch subagents — they edit, you commit.
- **Do not overwrite ledger entries with `decided_by: human`.** Sticky.
- **Do not skip the rebase step.** Base drift silently invalidates the review.
- **Do not run reviewers sequentially.** Parallel dispatch — single message, multiple tool calls.
- **Do not invent severity labels (P0/P1/etc.).** Binary `blocks_merge` only.
- **Do not commit if a fix subagent returned `unable_to_fix` for its only finding.** Log to ledger and continue.

## What you SHOULD do

- Be explicit at every step about which round you're on and what you're dispatching.
- Stream short status lines to the user as you go (`"Round 2: dispatching claude + codex reviewers..."`, `"Judge returned 3 blockers, 1 bug-class cluster..."`, `"Trajectory check: 5 → 7 blockers, asking for direction..."`).
- When asking the human at the trajectory checkpoint, summarize concisely so they can decide in one read.

## Related skills

- **REQUIRED SUB-SKILL**: `superpowers:dispatching-parallel-agents` for the fix step (and reviewer dispatch).
- **VENDORED FROM**: `code-review` — the bias-against-noise principles in `reviewer-prompt.md` are derived from this skill's guidelines.
- **COMPLEMENTARY**: `/security-review` slash command can be invoked as an extra pass when a `security` trigger fires (v2 idea, not v1).
