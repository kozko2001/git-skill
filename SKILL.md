---
name: git-skill
description: "Perform git operations (init, log, branch, commit, push, pull, patch, apply, revert, remote) without using the git binary. Claude reads and writes .git/ internals directly and uses the GitHub REST API for remote operations. Use when the user asks to: commit changes, push or pull, checkout or create a branch, show git log, init a repo, create or apply a patch, revert a commit, or manage remotes."
---

# git-skill — LLM-native Git Operations

Perform complete git repository operations without ever calling the `git` binary. Claude directly reads and writes the `.git/` directory's object store and uses the GitHub REST API for all remote operations.

## CRITICAL RULE

**Never run `git` as a shell command.** All operations must be performed by:
- Reading/writing files directly (Read, Write, Edit tools)
- Running Python inline via Bash for object hashing, zlib, and diff logic
- Calling the GitHub REST API via curl for remote operations

## When to use

- "commit these changes", "git commit", "make a commit"
- "push to github", "push my changes", "push branch"
- "pull from origin", "fetch latest", "sync with remote"
- "checkout branch", "switch to branch X", "create a branch"
- "show git log", "show history", "list commits"
- "init a repo", "initialize git here", "git init"
- "create a patch", "make a patch file", "export diff"
- "apply this patch", "apply patch file"
- "revert commit", "undo a commit", "revert HEAD"
- "add remote", "list remotes", "remove remote", "change remote URL"

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

1. Identify which operation the user wants from the table above
2. Read the corresponding agent file from the `agents/` directory
3. Follow the instructions in that file step-by-step using Read, Write, Edit, and Bash tools
4. For git object manipulation, use the Python patterns in `references/git-internals.md`
5. Report results clearly: commit SHA (short), files affected, branch name, etc.

## GitHub Authentication

Remote operations (push, pull) require a GitHub personal access token with `repo` scope. Check `$GITHUB_TOKEN` env var first. If not set, ask the user to provide one before proceeding.

```bash
# Check if token is available
echo $GITHUB_TOKEN | head -c 10
```

## Identifying current repo state

Before any operation, quickly orient yourself:

```bash
# Current branch and HEAD
cat .git/HEAD

# Latest commit SHA (if branch exists)
cat .git/refs/heads/$(cat .git/HEAD | sed 's|ref: refs/heads/||') 2>/dev/null || echo "(no commits yet)"

# Remote config
cat .git/config
```
