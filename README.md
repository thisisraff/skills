# skills

Personal [Claude Code](https://claude.com/claude-code) / agent skills. This repo
is the source of truth; install locations (`~/.claude/skills/`,
`~/.agents/skills/`) symlink back here.

## Skills

- **[pr-review-until-converged](pr-review-until-converged/)** — iterative
  multi-agent PR review loop: parallel reviewers → binary blocks-merge judge →
  parallel fixers, converging until ready-to-merge or escalation. Tracks a per-PR
  dismissed-findings ledger (`.pr-review/pr-<n>.yaml`, **local-only and
  gitignored** — never committed) so the same concerns aren't re-raised across
  rounds. See its [README](pr-review-until-converged/README.md) for how it works
  and what it needs.

## Installing a skill

Each skill is a directory. "Installing" is symlinking that directory into your
agent's skills folder so the skill loads on session start.

```bash
git clone https://github.com/<you>/skills.git ~/workspace/skills
cd ~/workspace/skills

# Claude Code:
ln -s "$PWD/pr-review-until-converged" ~/.claude/skills/pr-review-until-converged
# (and/or other runtimes that read ~/.agents/skills/, etc.)
ln -s "$PWD/pr-review-until-converged" ~/.agents/skills/pr-review-until-converged
```

Symlinking (rather than copying) keeps every install pointing at this repo, so a
`git pull` here updates them all. Restart your agent session to pick up a newly
linked skill.

Per-skill requirements (CLIs, optional tools) are listed in each skill's own
README — e.g. `pr-review-until-converged` needs `gh`, `git`, and `python3`, with
`codex` as an optional second reviewer.
