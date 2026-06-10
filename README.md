# git-skill

> Because `apt install git` is just too easy.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Skill](https://img.shields.io/badge/claude-skill-orange.svg)](SKILL.md)
[![Cost per session](https://img.shields.io/badge/cost%20per%20session-%240.80-red.svg)](#cost)

A Claude Code skill that performs complete git repository operations — `init`, `log`, `branch`, `commit`, `push`, `pull`, `patch`, `apply`, `revert`, `remote` — by reading and writing `.git/` internals directly and using the GitHub REST API for remote operations. The `git` binary is never called. There is no good reason for this.

## Why Would Anyone Do This?

When you ask Claude to `commit these changes`, instead of running `git commit -m "..."`, Claude will:

1. Walk your working directory and read every file
2. Serialize each file into a zlib-compressed, SHA1-addressed blob object
3. Construct a Merkle tree representing your filesystem hierarchy
4. Assemble a commit object with author metadata and a pointer to that tree
5. Write all of it into `.git/objects/` using Python's `hashlib`, `zlib`, and `struct`
6. Update `.git/refs/heads/<branch>` and `.git/HEAD`
7. For a push: serialize everything into a PACK file and upload it via the GitHub REST API

Each of these steps is implemented as inline Python passed to `bash -c`. It works correctly. It is a complete reimplementation of a version control system that has existed since 2005.

> [!NOTE]
> This project is also a gentle critique of the Claude Code skill pattern. Some skills are genuinely powerful agent workflows that benefit from LLM reasoning — dynamic decision trees, multi-step orchestration, context-aware judgment calls. Others are a git client. The right tool for "git commit" is `git commit`. This skill exists to make that point as expensively as possible.

## How It Works

The skill implements git's object model from scratch using Python's standard library:

- **Objects**: Blobs, trees, and commits are zlib-compressed and SHA1-addressed, stored under `.git/objects/` in the standard two-level directory layout
- **Refs**: HEAD, branches, and tags are plain text files under `.git/refs/` and `.git/HEAD`
- **Config**: Remote URLs and user identity are read from `.git/config` (standard INI format)
- **Remote transport (HTTPS)**: Uses the GitHub REST API with a bearer token — no raw git protocol required
- **Remote transport (SSH)**: Implements the git smart protocol over an SSH subprocess, including pkt-line framing and PACK file construction

The [`references/git-internals.md`](references/git-internals.md) file contains reusable Python building blocks (`read_obj`, `write_obj`, `build_tree`, `resolve_head`, etc.) shared across all agent files. It is genuinely well-written reference material for anyone who wants to understand git's internals. The rest of this project is a $0.80 proof of concept.

## Operations

| Operation | Description |
|-----------|-------------|
| `init`    | Create `.git/` directory structure and bootstrap core files |
| `log`     | Walk the commit chain and display history |
| `branch`  | List, create, switch, or delete branches |
| `commit`  | Build blob and tree objects, write a commit, update branch ref |
| `push`    | Upload objects to GitHub via REST API or git smart protocol over SSH |
| `pull`    | Fetch remote commits, reconstruct objects, update working directory |
| `patch`   | Generate a unified diff between commits or working tree vs HEAD |
| `apply`   | Apply a unified diff patch to the working directory |
| `revert`  | Create a new commit that undoes a previous commit |
| `remote`  | Add, remove, rename, or list remotes in `.git/config` |

## Requirements

- **Claude Code** with this skill installed
- **Python 3** in your shell (used for all git object computation)
- **`GITHUB_TOKEN`** environment variable — a GitHub personal access token with `repo` scope, for push and pull operations
- **`curl`** for GitHub API calls
- **SSH access to GitHub** if using SSH remotes

> [!WARNING]
> This skill deliberately never calls the `git` binary. If `git` happens to not be installed on your system, this skill has you covered. If `git` is installed — and it almost certainly is — it is right there, in your PATH, ready to commit your files for free.

## Usage

Install the skill in Claude Code, then ask Claude to perform git operations using natural language:

```
init a new repo here
commit these changes with message "fix the thing"
push to origin main
show me the last 10 commits
create and switch to a new branch called feature/my-thing
show a diff between HEAD and the previous commit
```

Claude will identify the intent and delegate to the appropriate agent file in [`agents/`](agents/).

## Installation

Copy this repository into your Claude Code skills directory, or reference the [`SKILL.md`](SKILL.md) manifest directly. Consult the [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code) for skill installation details.

## Cost

A representative session consisting of `git init` + one `git commit` + `git push` costs approximately **$0.80** in Claude Sonnet API tokens.

`git` — the binary — is free. It has been free since April 7, 2005. It is available on every package manager ever created. `brew install git`. `apt install git`. `winget install Git.Git`. It takes about four seconds.

> [!CAUTION]
> Calling `log` on a repository with hundreds of commits will require Claude to read and parse every commit object individually. This will cost real money. Consider whether you need a git log bad enough to spend $3 on it. `git log` is also free.

## Acknowledgments

The git internals reference in [`references/git-internals.md`](references/git-internals.md) is a genuinely useful document for anyone curious about how git stores data under the hood.

The rest of this project is a monument to the idea that just because you *can* make an LLM do something doesn't mean you *should*. It is also, somehow, fully functional.

> [!IMPORTANT]
> This repository — including its initial commit, every subsequent commit, and every push to GitHub — was managed exclusively using this skill. The `git` binary was never invoked. Not even once. You are reading a README that was written, committed, and pushed entirely by an LLM reimplementing git from scratch in inline Python. We have the API bills to prove it.
