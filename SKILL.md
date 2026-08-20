---
name: git-workflow
description: Branch, commit, PR, and merge policy for GitHub and Forgejo repos. Use when creating a repo, starting a feature branch, managing PRs, or merging into main.
---

## Core Policy

Default to feature branches and PRs. Work directly on `main` only when the
user explicitly says so or the repo instructions specify it (e.g.
`bypass_private_pr: always`). Otherwise, create a feature branch.

For public upstream work that should stay private, use a private working repo
or mirror branch and ask before exposing it publicly.

Push rules depend on the branch:

- **Feature branch**: commit and push after each logical piece of work.
- **Main**: commit after each logical piece of work. Do not push unless the
  user asks. "Commit and push", "push to main", and "push this" while on
  `main` all count. Then push. Do not open a PR instead, and do not ask
  again. If branch protection rejects the push, report the error; do not
  disable protection.
- **Merge**: only with explicit user approval. Never self-merge or bypass branch
  protection.

`bypass_private_pr` means commit on `main`. It is not permission to push.

Before changing branches, rebasing, stashing, or doing any operation that could
affect unrelated work, inspect the worktree. If there are unrelated uncommitted
changes, ask before proceeding.

## Pick the Forge First

Everything in this file applies to any forge. The CLI does not. Before running
any repo, PR, or issue command, check where the repo lives:

```sh
git remote get-url origin
```

- `github.com` → read `references/gh.md` for the commands
- a Forgejo or Gitea host → read `references/fj.md`

Read only the one that matches. If your global agent instructions name specific
hosts or say where new repos belong, they take precedence over any guess made
from the remote.

For a new repo with no `origin` yet, those same instructions decide the forge.
If nothing covers it, ask.

## Repo Exposure

Treat repo visibility and work exposure as separate concerns.

If the upstream project is public and the work should stay private, do not push
work branches to a public fork or upstream by default. Use a private working
clone or private mirror repo until the user deliberately chooses to expose the
work.

A fork is usually as public as its upstream — on GitHub, everything in a fork
network is visible. If the user asks for a "private fork" of a public repo,
clarify whether they mean a private mirror or a detached private working repo.

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

## Creating a New Repo

Only create and push a new repo when the user explicitly asks.

After the first push to `main`, apply branch protection — as part of creation,
not as later cleanup. Every forge should end up enforcing:

- PR required before any merge to `main`
- Linear history: no merge commits; rebase or squash only
- Rules apply to admins too
- No force pushes, no branch deletion

## PR, Issue, and Comment Bodies Go Through a File

Anything posted to a forge goes into a file in `./tmp/` and gets passed by path.
These bodies are multi-paragraph Markdown carrying the quotes, backticks, and
`$` found in YAML, code, and paths, and inline `--body` mangles them — don't try
inline first.

Every CLI covered here takes `--body-file` on PR create, PR edit, issue create,
and comment commands. See the reference file for exact syntax.

Build the file with a quoted heredoc (`<<'EOF'`), which passes backticks and `$`
through untouched. An unquoted heredoc expands them and corrupts the body.

### Commit Messages Are Usually Inline

A commit message is one semantic line most of the time, so `-m` is the default:

```sh
git commit -m "feat: add login endpoint"
```

A short body fits in a second `-m`; git separates the two with a blank line:

```sh
git commit -m "fix: resolve queries path from module root" \
  -m "The relative path resolved against the caller's cwd, so tests passed but the installed binary did not."
```

Switch to a file when the message gets long or cumbersome to quote — several
paragraphs, a bullet list, or text containing a single quote, a backtick, or `$`:

```sh
mkdir -p ./tmp
cat > ./tmp/commit-msg.txt <<'EOF'
fix: resolve queries path from module root

The relative path `../db/sql/queries/` resolved against the caller's cwd.
EOF
git commit -F ./tmp/commit-msg.txt
```

### Never Hard-Wrap PR or Issue Bodies

Forge Markdown renders a newline inside a paragraph as a literal line break, so
an 80-column-wrapped body becomes a ragged column of short lines. This is true of
GitHub and Forgejo alike. Write each paragraph as one unwrapped line; blank lines
still separate paragraphs, and list items still get their own line. This covers
everything posted to a forge: PR and issue bodies, comments, release notes.

Commit messages are the opposite — plain text, not Markdown. Subject under ~50
characters, body wrapped at 72.

## Committing

Use Conventional Commits (`type(scope): description`). See
conventionalcommits.org for the full spec. Common types: `feat`, `fix`,
`docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`.

Stage specific files. Never use `git add -A`:

```sh
git add path/to/file another/file
git commit -m "feat: add login endpoint"
```

For a long or awkward-to-quote message, write it to `./tmp/commit-msg.txt` and
commit with `-F` (see above).

Commit as you go — after each logical piece of work, while the changes are
still in context. This is more token-efficient than coming back later and
re-reading files to reconstruct what changed. Use concise commit messages that
match the repo style.

On feature branches, push after each commit. On `main`, push only when the
user asked to push this work.

## Feature Branches and WIP PRs

```sh
git checkout -b feature/my-thing
# work, commit
git push -u origin feature/my-thing
```

Draft PRs are fine to create at any point. The branch will continue to evolve
with more commits and pushes. When the user asks to merge, follow the pre-merge
checklist below. Preserve the branch tip before rebasing or merging so the work
can be recovered if the forge, a merge command, or branch cleanup behaves
unexpectedly.

Open a draft PR after pushing the branch, with the body in a file (see the file
rule above):

```sh
mkdir -p ./tmp
cat > ./tmp/pr-body.md <<'EOF'
## What

Brief description.

## Status

- [x] Initial scaffolding
- [ ] Tests
EOF
```

Then create the PR pointing at `./tmp/pr-body.md`. Update the description as work
progresses by rewriting that file and re-running the PR-edit command.

"Draft" is not implemented the same way everywhere — one forge has a draft flag,
another keys off a `WIP:` title prefix.

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

Avoid merge commits. A merge-strategy merge produces a "Merge pull request #N"
commit, which is generally undesirable. Prefer squash or rebase. The merge step
takes an explicit strategy flag every time; don't rely on the forge's default.

Default to squash: the PR collapses to one commit on `main`, so the PR title must
be a semantic commit message (`feat: ...`, `fix: ...`, `docs: ...`) because it
becomes that commit's subject.

Rebase is the alternative, used when the branch's individual commits are each
clean and worth preserving; then every commit message must be semantic, since
rebase keeps them verbatim on `main`. Follow a repo's AGENTS.md if it specifies a
strategy.

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
create or update the PR, not merge it. "Commit and push to main" is approval
to push `main`. It is not approval to merge a PR or disable branch protection.

**For public repos:** Follow the PR workflow unless the user asked to commit
and push to `main`. Merge a PR only after user approval.

**For private repos:** Feature branch and PR is still the default. Direct commits
to `main` require explicit user approval.

**Never self-merge. Never bypass branch protection**, including any admin
override flag the CLI offers.

### The safe way to merge a PR

Always merge through the forge CLI. Never do local merges and push, and never
disable branch protection to force a push to main.

**Pre-merge checklist.** Run these steps in order before merging. The git
commands below work anywhere; the forge commands for each numbered step are in
`references/gh.md` or `references/fj.md` under the same numbers.

1. **Inspect local and PR state.** Confirm the current branch, worktree, PR head,
   and base before changing anything.

   ```sh
   git status
   git branch --show-current
   git remote -v
   git log --oneline --decorate -10
   ```

   Then view the PR through the forge CLI.

2. **Preserve the PR head.** Create a local backup branch pointing at the exact
   PR head SHA (see Backup Branches above for rationale). Both forges serve the
   PR head under `refs/pull/<number>/head`:

   ```sh
   git fetch origin pull/<number>/head:backup/pr-<number>-<YYYYMMDD-HHMMSS>
   git rev-parse backup/pr-<number>-<YYYYMMDD-HHMMSS>
   git log --oneline --decorate -5 backup/pr-<number>-<YYYYMMDD-HHMMSS>
   ```

   Compare the backup branch SHA to the PR head SHA reported by the forge. Do
   not delete this backup branch during the merge session.

3. **Update the PR title.** Remove any "WIP" prefix. Use a Conventional Commits
   message (e.g. `feat: ...`, `fix: ...`, `docs: ...`). The PR title becomes
   the squash commit message.

4. **Update the PR body.** Make sure the description reflects the final state
   of the work. Edit it through a file, same as any other forge body.

5. **Mark the PR as ready** if it is still a draft. On some forges this is the
   same action as step 3.

6. **Verify the forge sees the full diff.** A squash or rebase merge against the
   wrong PR head can drop work from the merge result. Verify both the head SHA
   and the diff that will be merged.

   Compare the PR head SHA to the preserved backup branch and compare the file
   count and line counts against what you expect. If the diff looks wrong or
   incomplete, stop and diagnose. Do not merge a surprising diff.

   If the forge needs a new push to refresh the PR, ask before pushing. After
   approval, prefer an empty commit on the PR branch over rewriting history:

   ```sh
   git commit --allow-empty -m "chore: refresh PR"
   git push
   # wait a moment, then verify the diff again
   ```

   Only proceed once the diff matches expectations.

7. **Merge without deleting refs.** Default to squash. Use rebase when the user
   requests it (or a repo's AGENTS.md specifies it). Avoid a merge-commit merge
   unless the user explicitly asks for it. Always pass the strategy flag
   explicitly.

   Do not pass a delete-branch flag in the merge command. Branch deletion is
   cleanup, not part of merging.

8. **Verify after merge.** Fetch main and verify the merge result before any
   branch cleanup.

   ```sh
   git fetch origin main
   git log --oneline --decorate -10 origin/main
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

If the user asks to merge and the merge fails due to conflicts, resolve them by
rebasing the feature branch onto main only after preserving the current PR head
with a backup branch (see Backup Branches above). Rebase rewrites history and any
force push still requires explicit user confirmation.

**Never disable branch protection.** Toggling protection off and back on around a
direct push is a hack that breaks the PR workflow and leaves ghost PRs in draft
state.

## Working with Issues

**Always read the entire issue thread before starting work.** Use the reference
file's command to pull the issue with all its comments.

Key reminders:
- Comments may contain clarifications or scope changes not in the original issue
- Read ALL comments before making any code changes
- Ask questions if the requirements are unclear

**Close issues from the PR body.** This skill squash-merges by default, so `main`
gets one commit built from the PR title and body — not the feature branch's
individual commit messages. A `Closes #6` line buried in a commit body is
unreliable under squash; put the closing keyword in the PR body
(`./tmp/pr-body.md`) so the forge closes the issue when the PR merges:

```
Closes #6
```

When a branch is instead merged with rebase (commits kept verbatim on `main`), a
`Closes #6` line in the relevant commit body works too. Either way, the keyword
must land on `main` for the forge to close the issue.

**Create issues with `--body-file`**, same as any other body (see the file rule
above):

```sh
mkdir -p ./tmp
cat > ./tmp/issue-body.md <<'EOF'
## Description

The queries path `../db/sql/queries/` resolves incorrectly...
EOF
```

## Repo Temp Directory

Every repo has `./tmp/` gitignored. Write temp files to `./tmp/` (not `/tmp/`).
This keeps temp files repo-scoped, avoids system temp collisions, and makes
file paths relative and predictable.

Ensure `.gitignore` contains:

```
tmp/
```
