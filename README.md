# skill-git-workflow

Branch, commit, PR, and merge policy for GitHub and Forgejo repos. Use when creating a repo, starting a feature branch, managing PRs, or merging into main.

## Usage

This is an [Agent Skills](https://agentskills.io/) compatible skill. Load it with your agent harness and invoke via `skill:git-workflow`.

The policy applies to any forge; only the CLI differs. `SKILL.md` checks `git remote get-url origin` and pulls in exactly one reference file, so a GitHub repo never loads Forgejo commands into context and vice versa.

## Structure

- `SKILL.md` — Policy and frontmatter. Forge-agnostic
- `references/gh.md` — GitHub commands (`gh`)
- `references/fj.md` — Forgejo commands (`fj`)
