# Switching an existing checkout from Bitbucket to GitHub

For a local clone that currently points at Bitbucket, after the repo has been migrated to GitHub (see `BB-to-GH.md`). No re-clone needed; branches, history and uncommitted work stay as they are.

## Concepts used below

- **Remote:** a named bookmark for another copy of the repo, stored as a name and a URL. By convention the one you cloned from is called `origin`.
- **Remote-tracking branch:** your local, read-only record of where a branch was on a remote the last time you fetched, written `origin/main`. It is not the same as your own local branch `main`.
- **Upstream (tracking):** the remote-tracking branch a local branch is linked to. It is what `git pull`, `git push` and `git status` ("ahead/behind") compare against.

## 1. Check the current remote

```sh
cd /path/to/REPO
git remote -v
```

- `cd /path/to/REPO`: go into your local checkout. All git commands below act on the repo in the current directory.
- `git remote -v`: list the remotes. `-v` (verbose) shows the URLs; without it you only see the names. Each remote appears twice, once for `fetch` (download) and once for `push` (upload).

You should see `origin` pointing at `bitbucket.org/ORG/REPO.git`.

## 2. Point `origin` at GitHub

```sh
git remote set-url origin https://github.com/ORG/REPO.git
```

- `git remote set-url <name> <url>`: change the URL stored for the remote called `<name>`. Here it changes `origin` from the Bitbucket URL to the GitHub one. Nothing is downloaded or uploaded; only the stored address changes.

(SSH instead: `git@github.com:ORG/REPO.git`.)

This replaces the Bitbucket URL, so Bitbucket is no longer referenced.

## 3. Fetch and check

```sh
git fetch origin --prune
git remote -v
git branch -vv
```

- `git fetch origin`: download new commits, branches and tags from `origin` (now GitHub) and update the remote-tracking branches (`origin/*`). It never changes your local branches or working files, so it is safe to run any time. (`git pull` is fetch + merge; fetch is the safe half.)
- `--prune`: also delete remote-tracking branches (`origin/foo`) whose branch no longer exists on the remote. Without it, stale `origin/foo` entries from Bitbucket would stay around forever. It only removes those local bookmarks, never your own branches or anything on the remote.
- `git remote -v`: check again. It should now show only the GitHub URL.
- `git branch -vv`: list your local branches. `-v` shows the latest commit of each; the second `v` adds the upstream branch and whether you are ahead/behind it, e.g. `main abc1234 [origin/main: ahead 1] Fix typo`.

Local branches keep tracking `origin/<branch>`, which now means GitHub.

## 4. Local branches that have no GitHub counterpart

If `git branch -vv` shows `[origin/foo: gone]`, the upstream `origin/foo` no longer exists, so the branch was not on Bitbucket at migration time (for example it was already deleted). Keep it, or delete it with:

```sh
git branch -D foo
```

- `git branch -D foo`: delete the local branch `foo`. Uppercase `-D` forces deletion even if the branch has commits not merged anywhere else, so those commits become hard to find. Use lowercase `-d` for the safe version, which refuses if unmerged.

To publish a local-only branch to GitHub:

```sh
git push -u origin foo
```

- `git push origin foo`: upload the local branch `foo` to `origin`, creating it there if needed.
- `-u` (`--set-upstream`): also set `origin/foo` as the upstream of `foo`, so later `git pull` / `git push` with no arguments just work.

## 5. Unpushed work

Commits you had locally but never pushed to Bitbucket are not on GitHub. Push them as normal:

```sh
git push origin <branch>
```

- Uploads the new commits of `<branch>` to `origin`. Use `git status` or `git branch -vv` (`ahead N`) to see what is unpushed.

## Optional: keep Bitbucket as a second remote

Not needed after migration, but to keep it temporarily instead of replacing it:

```sh
git remote rename origin bitbucket
git remote add origin https://github.com/ORG/REPO.git
git fetch origin
git branch -u origin/<branch> <branch>   # repeat per branch, or just: git push -u origin <branch>
```

- `git remote rename origin bitbucket`: rename the remote. Git also renames its remote-tracking branches (`origin/main` becomes `bitbucket/main`) and updates your branches' upstreams to follow.
- `git remote add origin <url>`: create a new remote named `origin` with the GitHub URL.
- `git fetch origin`: download GitHub's branches so `origin/*` exist locally.
- `git branch -u origin/<branch> <branch>`: `-u` (`--set-upstream-to`) makes your local `<branch>` track `origin/<branch>` instead of `bitbucket/<branch>`. Alternatively `git push -u origin <branch>` does the same while pushing.

Remove it later with:

```sh
git remote remove bitbucket
```

- Deletes the `bitbucket` remote and all its `bitbucket/*` remote-tracking branches from your local repo. Nothing on Bitbucket itself is touched.

## Notes

- **Submodules:** if `.gitmodules` points to Bitbucket URLs, edit them and run `git submodule sync` (copies the URLs from `.gitmodules` into your local git config so they take effect).
- **Credentials:** HTTPS pushes to GitHub need a personal access token (or `gh auth login` to log in with the GitHub CLI, then `gh auth setup-git` to let git use that login), not a password. Old Bitbucket credentials in the macOS keychain are not used for GitHub.
- **Anything pushed to Bitbucket after the migration** is not on GitHub. Re-run the mirror push (`BB-to-GH.md`) first if that applies.
