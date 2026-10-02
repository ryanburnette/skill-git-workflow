# skill-git-workflow

Branch, commit, PR, and merge policy for GitHub and Gitea repos. Use when creating a repo, starting a feature branch, managing PRs, or merging into main.

## Usage

This is an [Agent Skills](https://agentskills.io/) compatible skill. Load it with your agent harness and invoke via `skill:git-workflow`.

The policy applies to any forge; only the CLI differs. `SKILL.md` checks `git remote get-url origin` and pulls in exactly one reference file, so a GitHub repo never loads Gitea commands into context and vice versa.

Forgejo is a fork of Gitea, so a remote URL cannot distinguish the two. `SKILL.md` defers to your global agent instructions for the host-to-forge mapping and covers no CLI for a Forgejo host.

## Structure

- `SKILL.md` — Policy and frontmatter. Forge-agnostic
- `references/gh.md` — GitHub commands (`gh`)
- `references/tea.md` — Gitea commands (`tea`)
