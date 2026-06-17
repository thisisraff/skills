# pr-review-until-converged

An iterative, multi-agent PR review loop. It drives a pull request to one of two
terminal states — **ready to merge** or **needs a human** — by repeating:

```
rebase → review (parallel) → judge → fix (parallel) → repeat
```

Each round runs independent reviewers, a separate judge that makes the
binary *does-this-block-merge?* call, and parallel fix agents. A per-PR ledger
remembers what was already decided so the same nit isn't re-litigated round after
round.

## Why it's structured this way

The single idea behind the design is **separation of judgment from
orchestration**:

- The **host** (the agent running the skill) orchestrates — it rebases, renders
  prompts, parses YAML, clusters findings, sequences commits, and decides when to
  escalate. It **never decides whether a finding blocks merge**.
- That call is made by a **judge** subagent with a deliberately *bounded* context
  (only the diff, the reviewers' findings, and the ledger) — no conversation
  history that could bias it toward "looks good, ship it."

This keeps the gate honest: the thing that wrote the code (or shepherded it) is
not the thing that decides whether it's mergeable.

## The loop, in detail

Each round:

1. **Rebase** the PR branch onto its base. If the base moved and conflicts, stop
   and ask a human — stale reviews are worse than no review.
2. **Review in parallel.** Two reviewers by default — one via the host's own
   subagent runtime ("claude"), one via the `codex` CLI if installed. Each reads
   the full diff and the ledger and returns YAML: `trigger_findings` (things it
   believes block merge) and `observations` (everything else). A reviewer that
   errors is dropped for the round; if all fail, escalate.
3. **Judge.** A single judge subagent takes both reviewers' findings, the diff,
   and the ledger, and returns one canonical list with a binary `blocks_merge`
   per finding. It mints stable kebab-case ids, reuses them on recurrence, and
   tags shared root causes so they can be fixed together.
4. **Converge or fix.** If there are zero blockers → **ready to merge**.
   Otherwise the host clusters the blockers and dispatches parallel fix agents,
   each with a strict file allowlist, commits one commit per cluster, pushes, and
   loops.

It stops and hands back to you when: there are no blockers, a single finding
survives two fix attempts, the blocker count isn't going down (trajectory
checkpoint), a wall-clock budget is hit, or a hard round cap is reached.

## The ledger

`<repo>/.pr-review/pr-<n>.yaml` — created on first run, **local-only and
gitignored** (the skill adds `.pr-review/` to the repo's `.gitignore` on
startup). It is *not* committed to the PR branch.

It serves two purposes across rounds:

- **Suppression** — reviewers see previously-dismissed findings and don't
  re-raise them.
- **Calibration** — the judge sees prior decisions as few-shot examples.

Because it's per-working-tree, a fresh clone starts empty. To carry a decision
across machines, set `decided_by: human` on the entry and copy it by hand — the
judge will never overwrite a human decision.

## Requirements

- **An agent runtime that provides this skill + a subagent/`Agent` tool** (e.g.
  [Claude Code](https://claude.com/claude-code)). The host dispatches reviewers,
  the judge, and fixers as subagents.
- **`git`** and **`gh`** (GitHub CLI, authenticated) — PR resolution, diffs, and
  posting the result comment.
- **`python3`** with `pyyaml` — used to render prompts and parse subagent YAML
  without bloating the host's context.
- **`codex` CLI** *(optional)* — adds a second, independent reviewer. The loop
  works with just the built-in reviewer if `codex` isn't installed.

## Usage

From a checkout that is **on the PR's head branch**:

```
/pr-review-until-converged           # reviews the open PR for the current branch
/pr-review-until-converged 219       # reviews PR #219 explicitly
```

If the working tree isn't on the PR's branch, the skill won't write to the wrong
branch — it either tells you to switch, offers to set up a worktree, or (if you
say "review only") runs a read-only pass with no fixes or commits.

## Tuning

Defaults: 60-minute wall-clock, 10-round backstop, escalate after 2 failed fix
attempts on the same finding. Override per-repo with
`<repo>/.pr-review/config.yaml`.

## Output

On exit the skill posts the same report to the PR (as a comment, clearly marked
automated) and to your session: either a ready-to-merge summary with the
non-blocking nits, or a needs-human summary naming the persistent blockers and
where they are.

## Files

| File | Role |
|------|------|
| `SKILL.md` | The host's operating instructions (the loop, escalation, safety rules). |
| `reviewer-prompt.md` | Reviewer subagent brief — find findings, bias against noise. |
| `judge-prompt.md` | Judge subagent brief — binary blocks-merge gate + ledger ids. |
| `fixer-prompt.md` | Fix subagent brief — scoped fix within a file allowlist, or `unable_to_fix`. |
