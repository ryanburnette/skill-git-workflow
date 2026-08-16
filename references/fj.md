# Forgejo commands (`fj`)

Commands for repos on a Forgejo or Gitea host, using the `fj` CLI
(forgejo-cli). The policy that governs when to run any of these is in `SKILL.md`
— read that first. Placeholders: `<owner>`, `<name>`, `<number>`,
`<forgejo-host>`.

Verified against fj v0.6.0. Check `fj <command> --help` if something here does
not match your version.

## Where `fj` differs from `gh`

Extrapolating from `gh` gets each of these wrong:

- **Drafts are a title prefix**, not a flag. Forgejo marks a PR as a draft by a
  literal `WIP: ` at the front of the title. There is no `fj pr ready`.
- **No `--json` on anything.** Use the `fj pr view` subcommands for detail, and
  git itself for SHAs.
- **No `fj api`.** There is no generic escape hatch; anything the subcommands
  don't cover needs `curl` (see below).
- **No branch protection command.** `fj repo edit` has no protection flags.
- **`fj whoami` can fail with `410 Gone`.** Use `fj auth list` to confirm
  authentication.

## Create a repo

```sh
fj repo create <name> -r origin -p -S
```

`-r origin` adds the remote, `-p` pushes the current branch, `-S` uses SSH.
Add `-P` to make it private.

## Branch protection

`fj` cannot do this — use the API. Forgejo has no equivalent of GitHub's
`required_linear_history`, so it takes two calls: force everything through a PR,
then ban merge commits at the repo level.

```sh
FJ_HOST=<forgejo-host>
FJ_TOKEN=$(jq -r --arg h "$FJ_HOST" '.hosts[$h].token' \
  "$HOME/Library/Application Support/forgejo-cli.forgejo-cli/keys.json")
REPO=<owner>/<name>

# 1. Protect main: no direct pushes, admins included
curl -s -X POST \
  -H "Authorization: token $FJ_TOKEN" \
  -H "Content-Type: application/json" \
  "https://$FJ_HOST/api/v1/repos/$REPO/branch_protections" \
  -d '{
    "rule_name": "main",
    "enable_push": false,
    "apply_to_admins": true,
    "required_approvals": 0,
    "enable_status_check": false,
    "block_on_rejected_reviews": true,
    "block_on_outdated_branch": false,
    "require_signed_commits": false
  }'

# 2. Linear history: squash and rebase only, no merge commits
curl -s -X PATCH \
  -H "Authorization: token $FJ_TOKEN" \
  -H "Content-Type: application/json" \
  "https://$FJ_HOST/api/v1/repos/$REPO" \
  -d '{
    "allow_merge_commits": false,
    "allow_squash_merge": true,
    "allow_rebase": true,
    "allow_rebase_explicit": false,
    "allow_manual_merge": false,
    "default_merge_style": "squash"
  }'
```

`enable_push: false` is what makes a PR mandatory. A protected branch also cannot
be deleted, covering GitHub's `allow_deletions=false`. Verify:

```sh
curl -s -H "Authorization: token $FJ_TOKEN" \
  "https://$FJ_HOST/api/v1/repos/$REPO/branch_protections" | jq
```

Never disable protection to push — same rule as anywhere else.

## Calling the API directly

The token lives in `fj`'s own config. Read it into a variable; never echo it,
never write it to `./tmp/`, never put it in a URL.

```sh
FJ_HOST=<forgejo-host>
FJ_TOKEN=$(jq -r --arg h "$FJ_HOST" '.hosts[$h].token' \
  "$HOME/Library/Application Support/forgejo-cli.forgejo-cli/keys.json")
curl -s -H "Authorization: token $FJ_TOKEN" "https://$FJ_HOST/api/v1/user/repos"
```

That config path is macOS. On Linux it is `~/.config/forgejo-cli/keys.json`.

Each instance publishes its own API reference at `https://<forgejo-host>/api/swagger`,
machine-readable at `https://<forgejo-host>/swagger.v1.json`. Check field names
there against your version rather than trusting the bodies above blindly.

## Bodies through a file

```sh
fj pr create "title" --body-file ./tmp/pr-body.md
fj pr edit <number> body --body-file ./tmp/pr-body.md
fj issue create "title" --body-file ./tmp/issue-body.md
fj pr comment <number> --body-file ./tmp/comment.md
fj issue comment <number> --body-file ./tmp/comment.md
```

## Draft PRs

The `WIP: ` prefix is the draft mechanism:

```sh
fj pr create "WIP: my thing" --body-file ./tmp/pr-body.md
```

Useful flags: `--base <branch>`, `--head <branch>`, `-A` to autofill title and
body from the branch's commits.

## Pre-merge checklist commands

Numbered to match the checklist in `SKILL.md`.

**1. Inspect PR state.**

```sh
fj pr view <number>
fj pr status <number>     # mergeability and CI; --wait to block on checks
```

**2. Preserve the PR head.** There is no `--json` to read the head SHA from, so
get it from git. Forgejo serves the same `refs/pull/<number>/head` namespace as
GitHub:

```sh
git ls-remote origin refs/pull/<number>/head
```

Compare that SHA to the backup branch created in `SKILL.md` step 2. Confirm the
namespace exists the first time you use it on a new host:

```sh
git ls-remote origin 'refs/pull/*'
```

**3. Update the PR title.** This also clears draft status — dropping the `WIP: `
prefix is what marks the PR ready, so steps 3 and 5 are one action here.

```sh
fj pr edit <number> title "feat: descriptive summary"
```

**4. Update the PR body.**

```sh
fj pr edit <number> body --body-file ./tmp/pr-body.md
```

**5. Mark ready.** Done by step 3. Verify with `fj pr view <number>` that the
title no longer starts with `WIP: `.

**6. Verify the diff.**

```sh
fj pr view <number> diff
fj pr view <number> files
fj pr view <number> commits
```

**7. Merge.**

```sh
fj pr merge <number> -M squash
# if the user asked for rebase:
fj pr merge <number> -M rebase
# rarely, if the user explicitly asked for a merge commit:
fj pr merge <number> -M merge
```

Other `-M` values this CLI accepts: `rebase-merge`, `manual`. Do not pass `-d`,
which deletes the branch after merging.

**8. Verify after merge.**

```sh
fj pr view <number>
```

## Issues

Read the whole thread before starting work:

```sh
fj issue view <number>            # title and body
fj issue view <number> comments   # every comment
```

Create an issue:

```sh
fj issue create "fix: description" --body-file ./tmp/issue-body.md
```

If the repo disables blank issues, `--template <name>` is required; list them
with `fj issue templates`.
