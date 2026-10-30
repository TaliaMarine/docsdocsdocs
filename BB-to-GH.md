# Migrating a repo from Bitbucket to GitHub (code + full git history)

Moves all commits, all active branches and all tags. Does **not** move pull requests, issues, wikis, CI config in the hosting UI, or permissions.

"Lost smashed branches" (branches already deleted, or squash/force-pushed away on Bitbucket) are not recoverable and are intentionally ignored: only what exists on the Bitbucket remote right now is migrated.

You need: `git`, read access to the Bitbucket repo, and an **empty** GitHub repo you can push to.

## 1. Create the empty GitHub repo

On GitHub: **New repository**. Leave it empty: no README, `.gitignore` or license, otherwise the first push will conflict.

## 2. Mirror-clone from Bitbucket

```sh
git clone --mirror git@bitbucket.org:WORKSPACE/REPO.git
cd REPO.git
```

`--mirror` copies every branch, tag and note exactly as it is on the remote (bare repo, no working tree).

(HTTPS works too: `https://bitbucket.org/WORKSPACE/REPO.git`.)

## 3. Push the mirror to GitHub

```sh
git push --mirror git@github.com:ORG_OR_USER/REPO.git
```

That's the whole migration. Branches, tags and history are now on GitHub.

### If the push is rejected

GitHub refuses refs under `refs/pull/*` (Bitbucket can expose similar refs). Push only branches and tags instead:

```sh
git push git@github.com:ORG_OR_USER/REPO.git --all
git push git@github.com:ORG_OR_USER/REPO.git --tags
```

## 4. Verify

```sh
# same branch/tag tips on both sides? (no diff output = identical)
diff <(git ls-remote git@bitbucket.org:WORKSPACE/REPO.git | grep -E 'refs/(heads|tags)/' | sort) \
     <(git ls-remote git@github.com:ORG_OR_USER/REPO.git   | grep -E 'refs/(heads|tags)/' | sort)
```

(`diff <(...)` needs bash/zsh.)

## 5. Switch your working clones

In each existing local clone:

```sh
git remote set-url origin git@github.com:ORG_OR_USER/REPO.git
git fetch origin
```

Or just re-clone.

## 6. Set the default branch and wrap up

- GitHub → Settings → Branches: set the default branch if it isn't what you expect.
- Make Bitbucket read-only or archive it so nobody keeps pushing there. Anything pushed to Bitbucket after step 2 needs re-running steps 2–3 (it's safe to repeat; `git push --mirror` will update GitHub).

## Notes

- **Large files:** GitHub rejects files > 100 MB. If Bitbucket used Git LFS, also run `git lfs fetch --all` before and `git lfs push --all git@github.com:ORG_OR_USER/REPO.git` after the push.
- **Submodules:** their URLs in `.gitmodules` still point wherever they pointed; update them separately if they were on Bitbucket.
- **Commit authors:** emails are preserved in history; GitHub links them to accounts if the email is verified on that account.
- **Cleanup:** delete the `REPO.git` mirror directory when done.
