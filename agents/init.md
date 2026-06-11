# Agent: git init

Initialize a new git repository. No `git` binary. Pure `mkdir` + `printf`.

## Steps

### 1. Determine target directory

Default: current directory (`.`). Use a user-specified path if provided.

```bash
TARGET="${1:-.}"
mkdir -p "$TARGET"
cd "$TARGET"
```

### 2. Check if already initialized

```bash
[ -d .git ] && echo "Already a git repository" && exit 0
```

Warn the user and stop. If they want to reinitialize, they must confirm.

### 3. Create directory structure

```bash
mkdir -p \
  .git/objects/pack \
  .git/objects/info \
  .git/refs/heads \
  .git/refs/tags \
  .git/info
```

### 4. Write core files

Write `.git/HEAD`:
```bash
printf 'ref: refs/heads/main\n' > .git/HEAD
```

Write `.git/config`:
```bash
printf '[core]\n\trepositoryformatversion = 0\n\tfilemode = true\n\tbare = false\n\tlogallrefupdates = true\n' > .git/config
```

Write `.git/description`:
```bash
printf "Unnamed repository; edit this file 'description' to name the repository.\n" > .git/description
```

Write `.git/info/exclude`:
```bash
printf '# git ls-files --others --exclude-from=.git/info/exclude\n.DS_Store\nThumbs.db\n*.swp\n*~\n' > .git/info/exclude
```

### 5. Optionally set user identity

If the user provided name and/or email, append to `.git/config`:

```bash
# Only if name provided
printf '[user]\n\tname = %s\n\temail = %s\n' "$USER_NAME" "$USER_EMAIL" >> .git/config
```

### 6. Report success

```bash
printf 'Initialized empty Git repository in %s/.git/\n' "$(pwd)"
```
