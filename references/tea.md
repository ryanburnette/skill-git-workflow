# Gitea commands (`tea`)

Commands for repos on a Gitea host, using the `tea` CLI. The policy that governs
when to run any of these is in `SKILL.md` — read that first. Placeholders:
`<owner>`, `<name>`, `<number>`, `<gitea-host>`.

Verified against tea v0.15.1. Check `tea <command> --help` if something here does
not match your version.

Install with `go install gitea.dev/tea@latest`. The module moved: the old
`code.gitea.io/tea` path still resolves, but its highest semver tag is a stale
`v1.3.3` that outranks every real release, so `code.gitea.io/tea@latest` silently
installs a 2019 build that reports `0.1.0-dev` and has no `pulls create`. Use
`gitea.dev/tea`.

## Where `tea` differs from `gh`

Extrapolating from `gh` gets each of these wrong:

- **No `--body-file` anywhere.** Every body flag is `--description`/`-d` and
  takes a string, on `pulls create`, `pulls edit`, `issues create`, and
  `comments`. See Bodies through a file below for how to keep the file rule.
- **`--description` is the body**, not the repo description, on issues and pulls.
  On `repos create` the body-ish flag is `--desc`.
- **Merge defaults to a merge commit.** `tea pulls merge` with no `--style` makes
  a merge commit. Always pass `-s squash` (or the strategy the user asked for).
- **Nouns are plural** (`pulls`, `issues`, `comments`), though `pr`, `issue`, and
  `comment` are accepted aliases.
- **Viewing is a bare index**, not a `view` subcommand: `tea pulls <number>`.
- **`tea repos create` does not add a remote or push.** There is no equivalent of
  `gh repo create --source . --push`; wire up `origin` yourself.
- **No branch protection command.** Use `tea api` (see below).

Unlike `fj`, `tea` does have `--output json` and a real `tea api` escape hatch,
so prefer those over scraping human-readable output.

## Create a repo

`tea repos create` only creates the remote repo. Add the remote and push
separately:

```sh
tea repos create --name <name> --owner <owner>
git remote add origin git@<gitea-host>:<owner>/<name>.git
git push -u origin main
```

Add `--private` to make it private.

## Branch protection

`tea` has no protection command, but `tea api` authenticates for you. Gitea has
no equivalent of GitHub's `required_linear_history`, so it takes two calls: force
everything through a PR, then ban merge commits at the repo level.

`{owner}` and `{repo}` are filled in from the current repo; pass `-r
<owner>/<name>` to target another.

```sh
# 1. Protect main: no direct pushes, admins included
tea api -X POST /repos/{owner}/{repo}/branch_protections -d '{
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
tea api -X PATCH /repos/{owner}/{repo} -d '{
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
tea api /repos/{owner}/{repo}/branch_protections | jq
```

Never disable protection to push — same rule as anywhere else.

## Calling the API directly

`tea api <endpoint>` signs the request with the stored token, so there is no
token to read out, echo, or leak. The path is prefixed with `/api/v1/`
automatically.

```sh
tea api /user/repos
tea api /repos/{owner}/{repo}/pulls/<number>
tea api -X PATCH /repos/{owner}/{repo}/issues/<number> -F body=@./tmp/body.md
```

`-d` sends a raw JSON body (`-d @file` reads it from a file). `-f key=value` adds
a string field, `-F key=value` a typed one, and `-F key=@file` reads that single
field's value from a file — which is how you put Markdown into the API without
quoting it through the shell. Supplying a body defaults the method to POST, so
pass `-X` explicitly for PATCH or PUT. Quote endpoints containing `?` or `&`.

Config, including the token, lives in `$XDG_CONFIG_HOME/tea` — on this setup
`~/.config/tea/config.yml`. It is machine-local: never copy it into a repo.

Each instance publishes its own API reference at `https://<gitea-host>/api/swagger`,
machine-readable at `https://<gitea-host>/swagger.v1.json`. Check field names
there against your version rather than trusting the bodies above blindly.

## Bodies through a file

`tea` has no `--body-file`, so read the file into the flag. Command substitution
does not re-expand the file's contents, so backticks and `$` survive intact and
the quoted-heredoc rule in `SKILL.md` still applies:

```sh
tea pulls create --title "title" -d "$(cat ./tmp/pr-body.md)"
tea pulls edit <number> -d "$(cat ./tmp/pr-body.md)"
tea issues create --title "title" -d "$(cat ./tmp/issue-body.md)"
tea comments add <number> -d "$(cat ./tmp/comment.md)"
```

For a very long body, or to avoid the shell entirely, go through `tea api` with
`-F body=@<file>`:

```sh
tea api -X PATCH /repos/{owner}/{repo}/pulls/<number> -F body=@./tmp/pr-body.md
```

## Draft PRs

Gitea marks drafts with a literal `WIP: ` title prefix, but `tea` wraps that in
real flags — unlike `fj`, you do not manage the prefix by hand:

```sh
tea pulls create --title "my thing" -d "$(cat ./tmp/pr-body.md)" --draft
```

`--draft` prepends the prefix and is idempotent. `tea pulls edit <number> --ready`
strips it. Useful flags on create: `--base <branch>`, `--head <branch>`.

## Pre-merge checklist commands

Numbered to match the checklist in `SKILL.md`.

**1. Inspect PR state.**

```sh
tea pulls <number>
tea pulls <number> --fields index,title,state,mergeable,base,head,url
```

**2. Preserve the PR head.** Gitea serves the same `refs/pull/<number>/head`
namespace as GitHub:

```sh
git ls-remote origin refs/pull/<number>/head
```

Cross-check against what the forge reports:

```sh
tea api /repos/{owner}/{repo}/pulls/<number> | jq -r .head.sha
```

Both must match the backup branch created in `SKILL.md` step 2. Confirm the
namespace exists the first time you use it on a new host:

```sh
git ls-remote origin 'refs/pull/*'
```

**3. Update the PR title.** Unlike `fj`, this does not clear draft status — step
5 is a separate action.

```sh
tea pulls edit <number> --title "feat: descriptive summary"
```

**4. Update the PR body.**

```sh
tea pulls edit <number> -d "$(cat ./tmp/pr-body.md)"
```

**5. Mark ready.** Strips any `WIP: ` prefix; safe to run when not a draft.

```sh
tea pulls edit <number> --ready
```

**6. Verify the diff.**

```sh
tea pulls <number> --fields diff
tea pulls <number> --fields patch
tea api /repos/{owner}/{repo}/pulls/<number>/files | jq -r '.[].filename'
```

**7. Merge.** The default style is a merge commit, so the flag is not optional:

```sh
tea pulls merge <number> -s squash
# if the user asked for rebase:
tea pulls merge <number> -s rebase
# rarely, if the user explicitly asked for a merge commit:
tea pulls merge <number> -s merge
```

Other `-s` values this CLI accepts: `rebase-merge`. `--title` and `--message` set
the resulting commit's subject and body. Delete the feature branch in step 9
after verification — do not let `tea pulls clean` do it here.

**8. Verify after merge.**

```sh
tea pulls <number> --fields index,title,state
```

**9. Clean up to main.** Keep the local backup. Gitea has no Restore branch on
the PR.

```sh
git checkout main
git merge --ff-only origin/main
git push origin --delete <feature-branch>
git branch -D <feature-branch>
```

`tea pulls clean <number>` deletes the local and remote feature branches for a
closed PR in one step. It matches the branch by commit hash, so it is safe, but
the explicit git commands above are preferred: they fail loudly and touch only
the branch you name.

## Issues

Read the whole thread before starting work:

```sh
tea issues <number>              # title and body
tea comments list <number>       # every comment
```

Create an issue:

```sh
tea issues create --title "fix: description" -d "$(cat ./tmp/issue-body.md)"
```
