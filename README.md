# skills

Personal [Claude Code](https://claude.com/claude-code) / agent skills. This repo is the source of truth; install locations (`~/.claude/skills/`, `~/.agents/skills/`) symlink back here.

## Skills

- **[pr-review-until-converged](pr-review-until-converged/)** — iterative multi-agent PR review loop: parallel reviewers → binary blocks-merge judge → parallel fixers, converging until ready-to-merge or escalation. Tracks a per-PR dismissed-findings ledger (`.pr-review/pr-<n>.yaml`, committed to the PR branch) so the same concerns aren't re-raised across rounds.

## Installing a skill

Symlink it into your skills dir:

```bash
ln -s "$PWD/<skill-name>" ~/.claude/skills/<skill-name>
```
