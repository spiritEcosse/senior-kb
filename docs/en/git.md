## 12. Git

### Core concepts

**Working tree → Index (staging) → Repository:**

```
working tree  →  git add  →  index (staging)  →  git commit  →  .git/objects
```

- `git status` — show what's staged, unstaged, untracked
- `git diff` — unstaged changes; `git diff --staged` — staged vs last commit
- `git log --oneline --graph --all` — visual branch history

---

### Branching and merging

```bash
git checkout -b feature/my-feature   # create + switch
git merge feature/my-feature         # merge into current branch (creates merge commit)
git rebase main                      # replay commits on top of main (linear history)
git cherry-pick abc1234              # apply a single commit to current branch
```

**merge vs rebase:**

| | `merge` | `rebase` |
|---|---|---|
| History | Preserves — merge commit shows the join | Rewrites — linear, cleaner |
| Safe on shared branch | ✅ Yes | ❌ No — rewrites shared history |
| Use when | `main`, `develop`, shared branches | Local feature branches before PR |

!!! warning
    Never rebase a branch that others have checked out — it rewrites commit SHAs and forces everyone else to reset.

---

### Reset, revert, restore

```bash
# git reset — move HEAD (and optionally staging/working tree)
git reset --soft HEAD~1    # undo last commit, keep changes staged
git reset --mixed HEAD~1   # undo last commit, keep changes unstaged (default)
git reset --hard HEAD~1    # undo last commit, discard all changes ⚠️

# git revert — create a new commit that undoes a previous one (safe for shared branches)
git revert abc1234

# git restore — discard working tree changes (does not touch commits)
git restore file.py        # discard unstaged changes in file.py
git restore --staged file.py  # unstage a file
```

---

### Stash

```bash
git stash                      # save dirty working tree, revert to HEAD
git stash push -m "wip: login" # with a description
git stash list                 # show all stashes
git stash pop                  # apply latest stash + delete it
git stash apply stash@{2}      # apply specific stash without deleting
git stash drop stash@{0}       # delete a stash
```

---

### Useful inspection commands

```bash
git blame file.py              # who changed each line and when
git bisect start               # binary search for the commit that introduced a bug
git bisect bad                 # current commit is bad
git bisect good v1.0           # v1.0 was good
# git bisect automatically checks out the midpoint — test, then:
git bisect good / git bisect bad

git reflog                     # history of where HEAD has been — recover lost commits
git shortlog -sn               # commit count per author
```

---

### git flow

```
main        — always production-ready
develop     — integration branch
feature/*   — branched from develop, merged back to develop
release/*   — branched from develop, merged to main + develop
hotfix/*    — branched from main, merged to main + develop
```

Simpler alternative: **GitHub Flow** — branch from `main`, PR, merge to `main`, deploy.

---

### Fixing mistakes

```bash
# Amend the last commit message or add forgotten files
git add forgotten.py
git commit --amend --no-edit

# Undo a pushed commit safely
git revert HEAD
git push

# Squash last 3 commits into one (interactive rebase)
git rebase -i HEAD~3
# In editor: keep first as 'pick', change others to 'squash'

# Find a deleted file
git log --all --full-history -- path/to/file.py
git checkout <sha>^ -- path/to/file.py
```

---

### .gitignore patterns

```gitignore
*.pyc           # all .pyc files
__pycache__/    # directories named __pycache__
.env            # secrets file
dist/           # build output
!dist/.gitkeep  # except this one file
```

`git rm --cached file` — stop tracking a file already committed without deleting it locally.

---
