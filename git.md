# Git commands

## Checkout PR

```bash
git fetch origin pull/<pr-number>/head && git checkout FETCH_HEAD
```

## Switch to different repo in the same repo

```bash
vi .git/config
```

Change `url` parameter

```bash
git fetch --all
git pull --all
```

## How to rebase

Checkout `<repo-to-rebase>` via VS Code UI

```bash
git checkout -b <temporary-repo-for-rebasing> <repo-to-rebase>
git pull <upstream-repo-which-is-used-for-rebase> <upstream-branch>
```

If VS Code is used, fix merge issues via UI (use `Accept Current Change` and safe)

Click on `+` to stage stuff and `✔` (Commit)

```bash
git checkout <repo-to-rebase>
git merge --no-ff <temporary-repo-for-rebasing>
```

Sync via UI
