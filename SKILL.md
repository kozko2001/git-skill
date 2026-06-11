---
name: git-skill
description: "Perform git operations (init, log, branch, commit, push, pull, patch, apply, revert, remote) without using the git binary or python3. Claude reads and writes .git/ internals directly using only POSIX tools (gzip, sha1sum, dd, od, awk, sed, diff, patch, curl, jq, ssh). Use when the user asks to: commit changes, push or pull, checkout or create a branch, show git log, init a repo, create or apply a patch, revert a commit, or manage remotes."
---

# git-skill — LLM-native Git Operations

Perform complete git repository operations using only POSIX shell tools — no `git` binary, no `python3`.

## CRITICAL RULES

1. **Never run `git`** as a shell command.
2. **Never run `python3`** or any other scripting runtime.
3. All object compression/decompression uses the **gzip header-swap technique**.
4. All SHA-1 hashing uses `sha1sum`.
5. Always read `references/git-internals.md` before performing any object operation — it contains all the core shell functions.

## Allowed tools

`gzip`, `sha1sum`, `dd`, `od`, `printf`, `awk`, `sed`, `grep`, `find`, `diff`, `patch`, `curl`, `jq`, `ssh`, `base64`, `wc`, `tr`, `cut`, `sort`, `mktemp`, `mkdir`, `cat`, `rm` — and bash builtins.

## When to use

- "commit these changes", "git commit", "make a commit"
- "push to github", "push my changes"
- "pull from origin", "fetch latest changes"
- "checkout branch", "switch to branch X", "create a branch"
- "show git log", "show history", "list commits"
- "init a repo", "initialize git here", "git init"
- "create a patch", "make a patch file"
- "apply this patch", "apply patch file"
- "revert commit", "undo a commit"
- "add remote", "list remotes", "remove remote"

## Operations

| User intent | Agent file |
|---|---|
| init / initialize repo | `agents/init.md` |
| log / show history | `agents/log.md` |
| checkout / create / list branches | `agents/branch.md` |
| commit changes | `agents/commit.md` |
| push to GitHub | `agents/push.md` |
| pull / fetch from GitHub | `agents/pull.md` |
| manage remotes | `agents/remote.md` |
| create patch file | `agents/patch.md` |
| apply patch file | `agents/apply.md` |
| revert a commit | `agents/revert.md` |

## Workflow

1. Identify which operation the user wants
2. Read `references/git-internals.md` — this defines all reusable shell functions
3. Read the corresponding `agents/<operation>.md` file
4. Execute step-by-step using the Bash tool (with the functions from references)
5. Report: SHA (short), files affected, branch name

## GitHub Authentication

Remote operations (push, pull) need a GitHub token (`repo` scope) **or** SSH key access.

- Token: check `$GITHUB_TOKEN` env var; if absent, ask the user
- SSH: remote URL starts with `git@github.com:` — use `ssh` binary directly

## Quick repo state check

```bash
cat .git/HEAD
cat .git/refs/heads/$(sed 's|ref: refs/heads/||' .git/HEAD) 2>/dev/null || echo "(no commits)"
cat .git/config
```
