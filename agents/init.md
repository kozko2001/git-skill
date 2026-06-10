# Agent: git init

Initialize a new git repository in the target directory.

## Steps

### 1. Determine target directory

- Default: current working directory (`.`)
- If the user specified a path, use that (create the directory if it doesn't exist)

```bash
TARGET_DIR="."   # or user-specified path
mkdir -p "$TARGET_DIR"
```

### 2. Check if already initialized

```bash
ls "$TARGET_DIR/.git" 2>/dev/null && echo "EXISTS" || echo "CLEAN"
```

If `.git/` already exists, warn the user and ask before reinitializing.

### 3. Create directory structure

```bash
mkdir -p \
  "$TARGET_DIR/.git/objects/pack" \
  "$TARGET_DIR/.git/objects/info" \
  "$TARGET_DIR/.git/refs/heads" \
  "$TARGET_DIR/.git/refs/tags" \
  "$TARGET_DIR/.git/info"
```

### 4. Write core files

Write `.git/HEAD`:
```
ref: refs/heads/main
```

Write `.git/config`:
```ini
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
```

Write `.git/description`:
```
Unnamed repository; edit this file 'description' to name the repository.
```

Write `.git/info/exclude`:
```
# git ls-files --others --exclude-from=.git/info/exclude
# Lines that start with '#' are comments.
.DS_Store
Thumbs.db
*.swp
*~
```

### 5. Optionally configure user identity

If the user provided a name and email, append to `.git/config`:

```ini
[user]
	name = <provided name>
	email = <provided email>
```

If not provided, skip — the commit agent will ask when needed.

### 6. Report success

```
Initialized empty Git repository in <absolute-path>/.git/
```

Resolve to absolute path with:
```bash
realpath "$TARGET_DIR"
```
