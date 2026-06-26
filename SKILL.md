---
name: git-workflow
description: GitHub repo creation, branch protection, committing, WIP PRs, and merging. Use when creating a GitHub repo, starting a feature branch, managing PRs, or merging into main.
---

## Core Policy

Default to feature branches and PRs. Work directly on `main` only when the
user explicitly says so or the repo instructions specify it (e.g.
`bypass_private_pr: always`). Otherwise, create a feature branch.

For public upstream work that should stay private, use a private working repo
or mirror branch and ask before exposing it publicly.

Push rules depend on the branch:

- **Feature branch**: commit and push after each logical piece of work.
- **Main**: commit after each logical piece of work, but never push.
- **Merge**: only with explicit user approval. Never self-merge or bypass branch
  protection.

Before changing branches, rebasing, stashing, or doing any operation that could
affect unrelated work, inspect the worktree. If there are unrelated uncommitted
changes, ask before proceeding.

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
backup branch at the current branch tip or PR head (see Backup Branches below).

### Backup Branches

A backup branch is cheap and keeps commits reachable if a later command rewrites
or removes the visible ref. Create one before any operation that could lose work:

```sh
git branch backup/<name>-<YYYYMMDD-HHMMSS>
```

The merge checklist in step 2 shows the full pattern for PR heads. Do not delete
a backup branch in the same session that created it.

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

Use Conventional Commits (`type(scope): description`). See
conventionalcommits.org for the full spec. Common types: `feat`, `fix`,
`docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`.

Stage specific files. Never use `git add -A`:

```sh
git add path/to/file another/file
git commit -m "$(cat <<'EOF'
feat: add login endpoint
EOF
)"
```

Commit as you go — after each logical piece of work, while the changes are
still in context. This is more token-efficient than coming back later and
re-reading files to reconstruct what changed. Use concise commit messages that
match the repo style.

On feature branches, push after each commit. On `main`, never push.

## Feature Branches and WIP PRs

```sh
git checkout -b feature/my-thing
# work, commit
```

Push the branch and open a PR:

```sh
git push -u origin feature/my-thing
```

Draft PRs are fine to create at any point. The branch will continue to evolve
with more commits and pushes. When the user asks to merge, follow the pre-merge
checklist below. Preserve the branch tip before rebasing or merging so the work
can be recovered if GitHub, a merge command, or branch cleanup behaves
unexpectedly.

Open a draft PR after pushing the branch. Write the body to a file and use
`--body-file` to avoid quoting issues (heredocs inside `--body` clash with
single quotes in the content):

```sh
mkdir -p ./tmp
cat > ./tmp/pr-body.md <<'EOF'
## What

Brief description.

## Status

- [x] Initial scaffolding
- [ ] Tests
EOF
gh pr create \
  --title "WIP: my thing" \
  --body-file ./tmp/pr-body.md \
  --draft
```

Update the PR description as work progresses using the same pattern:

```sh
cat > ./tmp/pr-body.md <<'EOF'
updated body...
EOF
gh pr edit <number> --body-file ./tmp/pr-body.md
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

## History: Squash by Default, Rebase Optional, Never Merge

Avoid merge commits. `gh pr merge --merge` produces a "Merge pull request #N"
commit, which is generally undesirable. Prefer squash or rebase. The merge step
takes an explicit strategy flag every time; don't rely on the GitHub default.

Default to squash (`gh pr merge --squash`): the PR collapses to one commit on
`main`, so the PR title must be a semantic commit message (`feat: ...`,
`fix: ...`, `docs: ...`) because it becomes that commit's subject.

Rebase (`gh pr merge --rebase`) is the alternative, used when the branch's
individual commits are each clean and worth preserving; then every commit message
must be semantic, since rebase keeps them verbatim on `main`. Follow a repo's
AGENTS.md if it specifies a strategy.

Clean up a feature branch before merging either way:

```sh
# Rebase interactively to squash/fixup noise commits
git rebase -i main

# Or just rebase to keep all commits in order
git rebase main
git push --force-with-lease origin feature/my-thing  # requires user confirmation
```

When a rebase hits conflicts:

```sh
# Resolve the conflicted files, then:
git add <resolved-files>
git rebase --continue
# If the result looks wrong, abort and try a different approach:
git rebase --abort
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
   PR head SHA (see Backup Branches above for rationale).

   ```sh
   git fetch origin pull/<number>/head:backup/pr-<number>-<YYYYMMDD-HHMMSS>
   git rev-parse backup/pr-<number>-<YYYYMMDD-HHMMSS>
   git log --oneline --decorate -5 backup/pr-<number>-<YYYYMMDD-HHMMSS>
   ```

   Compare the backup branch SHA to `headRefOid` from `gh pr view`. Do not
   delete this backup branch during the merge session.

3. **Update the PR title.** Remove any "WIP" prefix. Use a Conventional Commits
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

7. **Merge without deleting refs.** Default to `--squash`. Use `--rebase` when
   the user requests it (or a repo's AGENTS.md specifies it). Avoid `--merge`
   unless the user explicitly asks for it. Always pass the strategy flag
   explicitly.

   ```sh
   gh pr merge <number> --squash
   # if the user asked for rebase:
   gh pr merge <number> --rebase
   # rarely, if the user asked for merge:
   gh pr merge <number> --merge
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
PR head with a backup branch (see Backup Branches above). Rebase rewrites history
and any force push still requires explicit user confirmation.

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

**Create issues using `--body-file`.** Write the body to `./tmp/` and pass
`--body-file` to `gh issue create`. This avoids quoting problems when the
body contains single quotes (common in YAML, code blocks, or file paths):

```sh
mkdir -p ./tmp
cat > ./tmp/issue-body.md <<'EOF'
## Description

The queries path `../db/sql/queries/` resolves incorrectly...
EOF
gh issue create \
  --title "fix: description" \
  --body-file ./tmp/issue-body.md
```

## Repo Temp Directory

Every repo has `./tmp/` gitignored. Write temp files to `./tmp/` (not `/tmp/`).
This keeps temp files repo-scoped, avoids system temp collisions, and makes
file paths relative and predictable.

Ensure `.gitignore` contains:

```
tmp/
```
