# GitHub commands (`gh`)

Commands for repos on `github.com`. The policy that governs when to run any of
these is in `SKILL.md` — read that first. Placeholders: `<owner>`, `<name>`,
`<number>`.

## Create a repo

```sh
gh repo create <owner>/<name> --public --source . --remote origin --push
```

Use `--private` instead of `--public` when the repo should not be public.

## Branch protection

After the first push to `main`:

```sh
gh api repos/<owner>/<name>/branches/main/protection \
  --method PUT \
  --field enforce_admins=true \
  --field required_linear_history=true \
  --field allow_force_pushes=false \
  --field allow_deletions=false \
  --field 'required_pull_request_reviews[required_approving_review_count]=0' \
  --field 'required_pull_request_reviews[dismiss_stale_reviews]=false' \
  --field 'required_status_checks=null' \
  --field 'restrictions=null'
```

`required_status_checks` MUST be included, even as `null`, or the API returns
422.

This enforces the four intents listed in `SKILL.md`: PR required, linear history,
admins included, no force pushes or deletions.

## Bodies through a file

```sh
gh pr create --body-file ./tmp/pr-body.md
gh pr edit <number> --body-file ./tmp/pr-body.md
gh issue create --body-file ./tmp/issue-body.md
gh pr comment <number> --body-file ./tmp/comment.md
gh issue comment <number> --body-file ./tmp/comment.md
```

## Draft PRs

GitHub has a real draft flag:

```sh
gh pr create \
  --title "WIP: my thing" \
  --body-file ./tmp/pr-body.md \
  --draft
```

## Pre-merge checklist commands

Numbered to match the checklist in `SKILL.md`.

**1. Inspect PR state.**

```sh
gh pr view <number> --json number,title,state,isDraft,baseRefName,headRefName,headRepositoryOwner,headRefOid,mergeStateStatus
```

**2. Preserve the PR head.** Compare the backup branch SHA to `headRefOid` from
the command above.

**3. Update the PR title.**

```sh
gh pr edit <number> --title "feat: descriptive summary"
```

**4. Update the PR body.**

```sh
gh pr edit <number> --body-file ./tmp/pr-body.md
```

**5. Mark ready.**

```sh
gh pr ready <number>
```

**6. Verify the diff.**

```sh
gh pr view <number> --json headRefOid,commits,files \
  --jq '{head: .headRefOid, files: [.files[] | "\(.path) +\(.additions) -\(.deletions)"]}'
```

`gh pr diff` has no `--stat`. For a diffstat, pipe the patch through git:

```sh
gh pr diff <number> --patch | git apply --stat
```

**7. Merge.**

```sh
gh pr merge <number> --squash
# if the user asked for rebase:
gh pr merge <number> --rebase
# rarely, if the user explicitly asked for a merge commit:
gh pr merge <number> --merge
```

Do not use `--delete-branch`. Never use `--admin` to bypass branch protection.

**8. Verify after merge.**

```sh
gh pr view <number> --json state,mergedAt,mergeCommit
```

## Issues

Read the whole thread before starting work:

```sh
gh issue view <number> --comments
```

Or via the API, which is easier to read in bulk:

```sh
gh api repos/<owner>/<name>/issues/<number>/comments \
  --jq '.[] | "=== \(.user.login) on \(.created_at) ===\n\(.body)\n"'
```

Create an issue:

```sh
gh issue create \
  --title "fix: description" \
  --body-file ./tmp/issue-body.md
```
