# Switching an existing checkout from Bitbucket to GitHub

For a local clone that currently points at Bitbucket, after the repo has been migrated to GitHub (see `BB-to-GH.md`). No re-clone needed; branches, history and uncommitted work stay as they are.

## 1. Check the current remote

```sh
cd /path/to/REPO
git remote -v
```

You should see `origin` pointing at `bitbucket.org/avertis/REPO.git`.

## 2. Point `origin` at GitHub

```sh
git remote set-url origin https://github.com/AvertisSRL/REPO.git
```

(SSH instead: `git@github.com:AvertisSRL/REPO.git`.)

This replaces the Bitbucket URL, so Bitbucket is no longer referenced.

## 3. Fetch and check

```sh
git fetch origin --prune
git remote -v
git branch -vv
```

`git remote -v` should show only the GitHub URL. Local branches keep tracking `origin/<branch>`, which now means GitHub.

## 4. Local branches that have no GitHub counterpart

If `git branch -vv` shows `[origin/foo: gone]`, that branch did not exist on Bitbucket at migration time (for example it was already deleted). Keep it, or delete it with `git branch -D foo`. To publish a local-only branch to GitHub:

```sh
git push -u origin foo
```

## 5. Unpushed work

Commits you had locally but never pushed to Bitbucket are not on GitHub. Push them as normal:

```sh
git push origin <branch>
```

## Optional: keep Bitbucket as a second remote

Not needed after migration, but to keep it temporarily instead of replacing it:

```sh
git remote rename origin bitbucket
git remote add origin https://github.com/AvertisSRL/REPO.git
git fetch origin
git branch -u origin/<branch> <branch>   # repeat per branch, or just: git push -u origin <branch>
```

Remove it later with `git remote remove bitbucket`.

## Notes

- **Submodules:** if `.gitmodules` points to Bitbucket URLs, edit them and run `git submodule sync`.
- **Credentials:** HTTPS pushes to GitHub need a personal access token (or `gh auth login` / `gh auth setup-git`), not a password. Old Bitbucket credentials in the macOS keychain are not used for GitHub.
- **Anything pushed to Bitbucket after the migration** is not on GitHub. Re-run the mirror push (`BB-to-GH.md`) first if that applies.
