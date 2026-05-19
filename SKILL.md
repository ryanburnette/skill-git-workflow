---
name: git-workflow
description: GitHub repo creation, branch protection, committing, WIP PRs, and merging. Use when creating a GitHub repo, starting a feature branch, managing PRs, or merging into main.
---

## Core Policy

Default to feature branches and PRs. Work directly on `main` only when the user
explicitly asks for it, or after asking and receiving clear approval.

Keep local commits separate from publishing. The agent may create local commits
when the task calls for it, but the user handles `git push` unless pushing was
explicitly discussed and approved for this task or session.

Never merge, push to `main`, bypass branch protection, or self-merge without the
user explicitly asking.

## Repo Exposure

Treat repo visibility and work exposure as separate concerns.

If the upstream project is public and the work should stay private, do not push
work branches to a public fork or upstream by default. Use a private working
clone or private mirror repo until the user deliberately chooses to expose the
work.

GitHub public forks are usually public within the fork network. If the user asks
for a "private fork" of a public repo, clarify whether they mean a private mirror
or detached private working repo.

If repo visibility or desired exposure is unclear, ask before pushing or opening
a PR.

## Branching Defaults

Use this decision order:

1. If the user explicitly says to work on `main`, work on `main`.
2. If repo instructions allow direct commits to `main`, that means allowed after
   user intent is clear, not automatic.
3. For normal implementation sessions, create a feature branch.
4. For public upstream work that should stay private, use a private working repo
   or mirror branch and ask before exposing it publicly.

Before changing branches, rebasing, stashing, or doing any operation that could
affect unrelated work, inspect the worktree. If there are unrelated uncommitted
changes, ask before proceeding.

## Before Git Mutations

Before commits, branch changes, pushes, PRs, rebases, or merges, inspect state:

```sh
git status
git branch --show-current
git remote -v
git log --oneline -10
```

Before committing, inspect the full intended diff:

```sh
git diff
git diff --staged
```

Do not use destructive commands such as `git reset --hard`, `git checkout --`,
or deleting branches unless the user explicitly requests or approves them.

Before rebasing, force pushing, merging, or deleting a branch, create a local
backup branch at the current branch tip or PR head. A backup branch is cheap and
keeps commits reachable if a later command rewrites or removes the visible ref.

## Create a New GitHub Repo

Only create and push a new GitHub repo when the user explicitly asks.

```sh
gh repo create ryanburnette/<name> --public --source . --remote origin --push
```

After the first push to `main`, apply branch protection:

```sh
gh api repos/ryanburnette/<name>/branches/main/protection \
  --method PUT \
  --field enforce_admins=true \
  --field required_linear_history=true \
  --field allow_force_pushes=false \
  --field allow_deletions=false \
  --field 'required_pull_request_reviews[required_approving_review_count]=0' \
  --field 'required_pull_request_reviews[dismiss_stale_reviews]=false' \
  --field 'required_status_checks=null' \
  --field 'restrictions=null'
# Note: required_status_checks MUST be included (even as null) or the API returns 422.
```

This enforces:

- PR required before any merge to `main`
- Linear history: no merge commits; rebase or squash only
- Rules apply to admins too (`enforce_admins=true`)
- No force pushes, no branch deletion

## Committing

Stage specific files. Never use `git add -A`:

```sh
git add path/to/file another/file
git commit -m "$(cat <<'EOF'
type: short summary of what and why
EOF
)"
```

Commit when it creates a useful checkpoint. Use concise commit messages that
match the repo style.

The user handles `git push` manually unless pushing was explicitly discussed and
approved for this task or session.

## Feature Branches and WIP PRs

```sh
git checkout -b feature/my-thing
# work, commit
```

Push only after user approval:

```sh
git push -u origin feature/my-thing
```

Draft PRs are fine to create at any point. The branch will continue to evolve
with more commits and pushes. When the user asks to merge, follow the pre-merge
checklist below. Preserve the branch tip before rebasing or merging so the work
can be recovered if GitHub, a merge command, or branch cleanup behaves
unexpectedly.

Open a draft PR when the user asks for a PR workflow or approves publishing the
branch:

```sh
gh pr create \
  --title "WIP: my thing" \
  --body "$(cat <<'EOF'
## What

Brief description.

## Status

- [x] Initial scaffolding
- [ ] Tests
EOF
)" \
  --draft
```

Update the PR description as work progresses:

```sh
gh pr edit <number> --body "$(cat <<'EOF'
updated body...
EOF
)"
```

## Force Pushes Require User Confirmation

Always ask before any force push. Prefer `--force-with-lease`. Do not use plain
`--force` unless there is an exceptional reason and the user explicitly approves
that exact command.

Before asking, double-check and report the branch, remote, and expected
overwrite:

```sh
git status
git branch --show-current
git remote -v
git log --oneline --decorate -10
```

Force pushes are destructive. Even with safeguards, they can cause data loss.

## History: Rebase or Squash

Never merge commits. Every branch must rebase onto `main` before merging.

Clean up a feature branch before merge:

```sh
# Rebase interactively to squash/fixup noise commits
git rebase -i main

# Or just rebase to keep all commits in order
git rebase main
git push --force-with-lease origin feature/my-thing  # requires user confirmation
```

## Merging into Main Only After Explicit Approval

Never merge, push to `main`, or bypass branch protection without the user
explicitly asking. This includes private repos. "git-workflow everything" means
create or update the PR, not merge it.

**For public repos:** Follow the PR workflow. Merge only after user approval.

**For private repos:** Feature branch and PR is still the default. Direct commits
to `main` require explicit user approval.

**Never self-merge. Never use `--admin` to bypass branch protection.**

### The safe way to merge a PR

Always use `gh pr merge` -- never do local merges and push, and never disable
branch protection to force a push to main.

**Pre-merge checklist.** Run these steps in order before merging:

1. **Inspect local and PR state.** Confirm the current branch, worktree, PR head,
   and base before changing anything.

   ```sh
   git status
   git branch --show-current
   git remote -v
   git log --oneline --decorate -10
   gh pr view <number> --json number,title,state,isDraft,baseRefName,headRefName,headRepositoryOwner,headRefOid,mergeStateStatus
   ```

2. **Preserve the PR head.** Create a local backup branch pointing at the exact
   PR head SHA before rebasing, refreshing, merging, or deleting anything.

   ```sh
   git fetch origin pull/<number>/head:backup/pr-<number>-<YYYYMMDD-HHMMSS>
   git rev-parse backup/pr-<number>-<YYYYMMDD-HHMMSS>
   git log --oneline --decorate -5 backup/pr-<number>-<YYYYMMDD-HHMMSS>
   ```

   Compare the backup branch SHA to `headRefOid` from `gh pr view`. Do not
   delete this backup branch during the merge session.

3. **Update the PR title.** Remove any "WIP" prefix. Use a semantic commit
   message (e.g. `feat: ...`, `fix: ...`, `docs: ...`). The PR title becomes
   the squash commit message.

   ```sh
   gh pr edit <number> --title "feat: descriptive summary"
   ```

4. **Update the PR body.** Make sure the description reflects the final state
   of the work.

5. **Mark the PR as ready.** If the PR is still a draft:

   ```sh
   gh pr ready <number>
   ```

6. **Verify GitHub sees the full diff.** A squash or rebase merge against the
   wrong PR head can drop work from the merge result. Verify both the head SHA
   and the diff GitHub will merge.

   ```sh
   gh pr view <number> --json headRefOid,commits,files
   gh pr diff <number> --stat
   ```

   Compare the PR head SHA to the preserved backup branch and compare the file
   count and line counts against what you expect. If the diff looks wrong or
   incomplete, stop and diagnose. Do not merge a surprising diff.

   If GitHub needs a new push to refresh the PR, ask before pushing. After
   approval, prefer an empty commit on the PR branch over rewriting history:

   ```sh
   git commit --allow-empty -m "chore: refresh PR"
   git push
   # wait a moment, then verify again
   gh pr diff <number> --stat
   ```

   Only proceed once the diff matches expectations.

7. **Merge without deleting refs.** The merge strategy (squash vs rebase) is
   project-specific. Check AGENTS.md or ask the user. Default to `--squash` if
   unspecified.

   ```sh
   gh pr merge <number> --squash
   # or
   gh pr merge <number> --rebase
   ```

   Do not use `--delete-branch` in the merge command. Branch deletion is cleanup,
   not part of merging.

8. **Verify after merge.** Fetch main and verify the merge result before any
   branch cleanup.

   ```sh
   git fetch origin main
   git log --oneline --decorate -10 origin/main
   gh pr view <number> --json state,mergedAt,mergeCommit
   ```

   For squash merges, compare the final tree from the preserved backup branch to
   `origin/main`. If no other work landed on `main` during the merge, this diff
   should be empty:

   ```sh
   git diff --stat backup/pr-<number>-<YYYYMMDD-HHMMSS> origin/main
   ```

   If other work landed at the same time, review the diff carefully instead of
   assuming missing files are safe. For rebase merges, confirm the expected
   commits or patch are present on `origin/main`.

9. **Clean up only after verification.** Delete remote or local feature branches
   only if the user asked for cleanup and the preserved backup branch is still
   available. Never delete the backup branch in the same session that created it.

If the user asks to merge and `gh pr merge` fails due to merge conflicts, resolve
them by rebasing the feature branch onto main only after preserving the current
PR head with a backup branch. Rebase rewrites history and any force push still
requires explicit user confirmation.

**Never disable branch protection.** The old pattern of toggling `enforce_admins`
off/on around a direct push is a hack that breaks the PR workflow and leaves
ghost PRs in draft state.

## Working with GitHub Issues

**Always read the entire issue thread before starting work.**

```sh
gh issue view <number> --comments
```

Or use the API for full thread:

```sh
gh api repos/<owner>/<repo>/issues/<number>/comments --jq '.[] | "=== \(.user.login) on \(.created_at) ===\n\(.body)\n"'
```

Key reminders:
- Comments may contain clarifications or scope changes not in the original issue
- Read ALL comments before making any code changes
- Ask questions if the requirements are unclear

**Close issues in commit messages:**

```sh
git commit -m "fix: description of fix

Closes #6"
```

This automatically closes the issue when pushed to main.
