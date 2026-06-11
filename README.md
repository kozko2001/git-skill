# git-skill 🚀

> **Your version control. Reimagined. AI-native. Zero binaries.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Skill](https://img.shields.io/badge/claude-skill-orange.svg)](SKILL.md)
[![Powered by AI](https://img.shields.io/badge/powered%20by-AGI-purple.svg)](#)
[![Cost per session](https://img.shields.io/badge/cost%20per%20session-%240.80-red.svg)](#cost)
[![Vibes](https://img.shields.io/badge/vibes-immaculate-green.svg)](#)

---

**What if git... but AI?**

We didn't just wrap `git`. We didn't prompt-engineer around it. We **replaced it entirely** with a large language model that reads and writes raw `.git/` object internals, constructs Merkle trees from first principles, serializes PACK files, and pushes to GitHub via REST — all without ever touching the `git` binary.

This is what **AI-native development** actually looks like. Not copilot autocomplete. Not tab completion. A full reimplementation of a version control system that has existed since 2005, running inside a token stream, costing $0.80 per commit.

We are not normal.

---

## The Vision

Every 10x dev knows: **the bottleneck is never the tool, it's the mindset.**

`git commit` takes 4 seconds. But are those 4 seconds *AI-augmented*? Are you *leveraging LLM reasoning* in your object hashing pipeline? Are your SHA-1 addresses being computed by a **foundation model**?

No. They're not. Until now.

git-skill doesn't just run git operations — it **understands** them. It walks your working directory with *intention*. It builds your commit graph with *context awareness*. It is, at every step, reasoning about your repository's Merkle tree using a 200-billion-parameter neural network.

`git` was built for humans. **git-skill was built for the AI era.**

---

## How It Works (The Technical Alpha)

When you ask Claude to commit, here's the **agentic execution loop** that fires:

1. **Semantic file traversal** — Claude walks your working directory and reads every file with full LLM context
2. **AI-native blob serialization** — each file gets zlib-compressed, SHA1-addressed, and written as a git object — in *inline Python*, generated on-the-fly, no precompiled binary
3. **Merkle tree synthesis** — Claude constructs your filesystem hierarchy as a cryptographic tree structure, from scratch, every single time
4. **Commit object assembly** — author metadata, parent pointers, tree root — assembled by the same model that writes poetry and passes bar exams
5. **Object graph materialization** — everything lands in `.git/objects/` in the standard two-level directory layout, byte-perfect
6. **Ref update** — `.git/refs/heads/<branch>` and `.git/HEAD` updated with surgical precision
7. **Remote transport** — PACK file serialized and uploaded to GitHub via REST API, or raw git smart protocol over SSH

This is **vertical integration**. This is **owning the stack**. This is what building in the age of AI looks like.

> The [`references/git-internals.md`](references/git-internals.md) file contains the reusable building blocks (`read_obj`, `write_obj`, `build_tree`, `resolve_head`) that power the whole system. It is, genuinely, excellent reference material for git internals. Read it if you want to understand how version control actually works under the hood.

---

## Operations

Full git parity. Zero binaries. All AI.

| Operation | What the AI Does |
|-----------|-----------------|
| `init`    | Bootstraps `.git/` from nothing — directory structure, HEAD, config, all of it |
| `log`     | Walks the commit chain object-by-object and renders your history |
| `branch`  | Creates, lists, switches, deletes branches by writing ref files directly |
| `commit`  | Full blob + tree + commit object pipeline, end to end |
| `push`    | Serializes a PACK file and ships it to GitHub via REST or SSH smart protocol |
| `pull`    | Fetches remote commits, reconstructs objects, hydrates working directory |
| `patch`   | Generates unified diffs between commits or working tree vs HEAD |
| `apply`   | Applies patch files to the working directory |
| `revert`  | Creates a new commit that inverts a previous one — non-destructive history rewrite |
| `remote`  | Manages remotes in `.git/config` — add, remove, rename, list |

---

## Ship Faster. Think Less. Commit More.

```
init a new repo here
commit these changes with message "fix the thing"
push to origin main
show me the last 10 commits
create and switch to a new branch called feature/my-thing
show a diff between HEAD and the previous commit
```

That's your entire git workflow. In natural language. Executed by an AI that has read every git RFC ever written and internalized them into its weights.

No flags. No man pages. No `git rebase -i HEAD~3` arcana. Just **intent → execution**.

---

## Requirements

- **Claude Code** — the AI-native development environment
- **Python 3** — for object computation (spawned inline by Claude)
- **`GITHUB_TOKEN`** — GitHub PAT with `repo` scope for remote operations
- **`curl`** — for GitHub API calls
- **SSH** — if you're on the SSH transport path

> No `git` binary required. No `git` binary used. `git` can be uninstalled. This skill has you covered.

---

## Installation

Drop this repo into your Claude Code skills directory or point to [`SKILL.md`](SKILL.md) directly. See the [Claude Code docs](https://docs.anthropic.com/en/docs/claude-code) for the full installation flow.

Once installed: just talk to Claude. The skill handles routing to the right agent.

---

## Cost

A representative session (`git init` + one commit + push) runs approximately **$0.80** in Claude Sonnet API tokens.

`git` — the binary — costs $0.00. It has been free since April 7, 2005. It is on every machine. It takes four seconds to install.

But is `git` **AI-powered**? No. It is not. It is a C program written by Linus Torvalds. It does not reason. It does not understand your intent. It just hashes files.

**We hash files with a large language model. That's the difference.**

> [!CAUTION]
> Running `log` on a repo with hundreds of commits means Claude reads and parses every commit object individually. This will cost real money. Ask yourself: how much is your git log worth to you? `git log` is free. This is not `git log`. This is **AI-powered commit chain traversal**. Different category.

---

## The Meta-Layer

> [!NOTE]
> This project is also a live commentary on the Claude Code skill ecosystem. Some skills are genuinely powerful agentic workflows — dynamic orchestration, multi-step reasoning, context-aware judgment. Others are a wrapper around `git commit`. The right tool for version control is `git`. This skill exists as a $0.80 existence proof that the LLM will do it anyway if you ask nicely.

---

## Provenance

> [!IMPORTANT]
> This repository — every commit, every push, the README you are reading right now — was created and managed exclusively using this skill. The `git` binary was never invoked. Not once. You are looking at a README that was written, committed, and pushed to GitHub entirely by an LLM that reimplemented git in inline Python because someone thought it would be funny. We have the API bills to prove it.

---

**Built different. Shipped by AI. Costs more than it should.**

*For the builders who don't just use tools — they make LLMs reimplement them from first principles.*
