# 🌿 Git — Medium / Hard Interview Questions

![Git](https://img.shields.io/badge/Git-2.x-F05032?style=for-the-badge&logo=git&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-Advanced-181717?style=for-the-badge&logo=github&logoColor=white)
![Level](https://img.shields.io/badge/Level-Medium%20%2F%20Hard-red?style=for-the-badge)
![Topic](https://img.shields.io/badge/Topic-Advanced%20Git%20Internals%20%26%20Workflows-blueviolet?style=for-the-badge)

> **Self-generated comprehensive reference** covering advanced Git concepts that appear in mid-level to senior DevOps/DevSecOps/SRE interviews. Goes beyond basic `git add`, `git commit`, `git push`.

---

## 📑 Table of Contents

| # | Topic | Level |
|---|---|---|
| 1 | Git Branching Strategies — GitFlow, GitHub Flow, Trunk-Based | Medium |
| 2 | Merge vs Rebase vs Squash — When to Use Each | Medium/Hard |
| 3 | Git Reset vs Revert vs Restore | Medium/Hard |
| 4 | Git Cherry-Pick | Medium |
| 5 | Git Stash — Advanced Usage | Medium |
| 6 | Git Tags and Semantic Versioning | Medium |
| 7 | Resolving Complex Merge Conflicts | Medium/Hard |
| 8 | Git Reflog — Recovering Lost Work | Hard |
| 9 | Git Bisect — Binary Search Debugging | Medium/Hard |
| 10 | Git Hooks — Automation and Enforcement | Medium/Hard |
| 11 | Git Internals — Objects, Refs, Packfiles | Hard |
| 12 | Git Submodules and Subtrees | Hard |
| 13 | Git LFS — Large File Storage | Medium |
| 14 | Git Security — Secrets, Signing, History Rewriting | Hard |
| 15 | Monorepo Git Strategies — Sparse Checkout, Worktrees | Hard |
| 16 | Troubleshooting Scenarios | Medium/Hard |

---

---

## 1. Git Branching Strategies

### ❓ Interview Question
> **"What branching strategies have you used in production? Compare GitFlow, GitHub Flow, and trunk-based development."**

### 📖 GitFlow

GitFlow uses long-lived branches for structured release management. Best for versioned software with scheduled releases.

```
main         ──────●──────────────────────────●──── (production releases, tagged)
                   │                          │
develop      ──────●──────────●──────●────────●──── (integration branch)
                   │          │      │
feature/A    ──────●──────────●      │              (short-lived feature)
                              │      │
feature/B                     ──────●               (short-lived feature)
                                    │
release/1.2                         ──────────●     (stabilization — only bugfixes)
                                              │
hotfix/1.1.1                                  ●     (emergency fix off main)
```

```bash
# GitFlow branch naming conventions
git checkout -b feature/JIRA-1234-add-oauth develop
git checkout -b release/1.2.0 develop
git checkout -b hotfix/1.1.1 main
```

**When to use**: Projects with versioned releases (mobile apps, on-prem software, libraries). Not ideal for continuous deployment — the long-lived `develop` branch delays feedback.

---

### 📖 GitHub Flow

Simple model — only `main` is long-lived. Every change goes through a short-lived feature branch and PR.

```
main    ──●──────────────────────────●──────────────●──── (always deployable)
          │                          │              │
feat/A    ──────────●──────────────PR→              │
                                     │              │
feat/B                               ──────────PR→
```

```bash
git checkout -b feat/add-rate-limiting main
# ... develop ...
git push origin feat/add-rate-limiting
# Open PR → CI passes → 2 approvals → Merge → Deploy
git branch -d feat/add-rate-limiting
```

**When to use**: Web applications with continuous deployment. `main` is always production-ready. Simple, fast feedback loop.

---

### 📖 Trunk-Based Development (TBD)

All developers commit directly to `main` (trunk) — or to very short-lived branches (< 1 day). Feature flags gate incomplete work.

```mermaid
flowchart LR
    Dev1["Developer 1\ncommit every few hours"]
    Dev2["Developer 2\ncommit every few hours"]
    Dev3["Developer 3\nshort branch < 1 day"]

    Main["main / trunk\n(CI runs on every commit)"]

    Dev1 -->|direct push or tiny PR| Main
    Dev2 -->|direct push or tiny PR| Main
    Dev3 -->|1-day branch → merge| Main

    Main -->|deploy continuously| Prod["Production"]
```

```bash
# Feature flag wraps incomplete code — merge to trunk safely
if feature_flags.is_enabled("new-checkout-flow", user):
    return new_checkout_flow()
else:
    return legacy_checkout_flow()
```

**When to use**: High-performance engineering teams (Google, Meta, Netflix). Requires strong CI, feature flags, and discipline. Maximum deployment frequency.

---

### Comparison Table

| Dimension | GitFlow | GitHub Flow | Trunk-Based |
|---|---|---|---|
| **Long-lived branches** | main + develop + release | main only | main only |
| **Complexity** | High | Low | Low (but requires discipline) |
| **Release cadence** | Scheduled/versioned | Continuous | Continuous |
| **PR size** | Large (feature complete) | Medium | Small (daily) |
| **Feature flags needed** | No | No | Yes |
| **CI feedback speed** | Slow (develop branch) | Fast | Fastest |
| **Best for** | Libraries, mobile, on-prem | Web apps/APIs | Large eng teams, SaaS |

---

## 2. Merge vs Rebase vs Squash

### ❓ Interview Question
> **"Explain the difference between `git merge`, `git rebase`, and squash merging. When would you use each, and what are the risks of rebasing?"**

### 📖 `git merge` — Preserves Full History

Creates a merge commit joining two branch histories. History is non-linear but complete.

```
Before:
main    A──B──C
              \
feat           D──E──F

After git merge:
main    A──B──C──────G   (G = merge commit)
              \     /
feat           D──E──F
```

```bash
git checkout main
git merge feat/add-oauth
# Creates merge commit G: "Merge branch 'feat/add-oauth'"

# Merge with explicit message
git merge feat/add-oauth -m "feat: merge OAuth2 login feature"

# Abort a merge mid-conflict
git merge --abort
```

**Use when**: Long-lived branches (release merges back to main), preserving feature branch context is important, working in teams where history archaeology matters.

---

### 📖 `git rebase` — Linear History

Moves commits from a branch on top of another branch. History becomes linear — as if the feature was developed after the target branch's latest commit.

```
Before:
main    A──B──C
              \
feat           D──E──F

After git rebase main:
main    A──B──C
                \
feat             D'──E'──F'   (D', E', F' = new commits with new hashes)
```

```bash
git checkout feat/add-oauth
git rebase main              # replay feat commits on top of latest main

# Interactive rebase — squash, edit, reorder commits
git rebase -i HEAD~4         # interactive rebase of last 4 commits

# Rebase with autostash (stash uncommitted changes automatically)
git rebase --autostash main

# Abort rebase
git rebase --abort

# Continue after resolving conflict
git rebase --continue
```

**⚠️ Golden Rule of Rebasing**: **Never rebase commits that have been pushed to a shared remote branch.** Rebase rewrites commit hashes — if teammates have the old hashes, their history will diverge and create chaos.

```bash
# Safe: rebase your local feature branch before opening a PR
git fetch origin
git rebase origin/main           # bring feature branch up to date cleanly

# DANGEROUS: rebasing a branch others are working on
git push --force origin shared-branch   # ← this breaks everyone's local copies
# Use --force-with-lease instead (only force if no one else pushed)
git push --force-with-lease origin feat/my-branch
```

---

### 📖 Squash Merge — One Clean Commit

Takes all commits from a feature branch and squashes them into a single commit on the target branch. The feature branch's individual commits are lost from main's history.

```
Before:
main    A──B──C
              \
feat           D──E──F──G──H  (5 WIP commits)

After squash merge:
main    A──B──C──S   (S = single squashed commit with all changes)
```

```bash
# Squash merge via CLI
git checkout main
git merge --squash feat/add-oauth
git commit -m "feat: add OAuth2 login with GitHub

Squashed 5 commits:
- Add GitHubStrategy to passport config
- Add /auth/github callback route
- Add user session persistence
- Add logout route
- Add tests for OAuth flow"

# Squash during interactive rebase
git rebase -i HEAD~5
# Change "pick" to "squash" or "s" for commits 2-5
```

**Use when**: Feature branches have messy WIP commits ("fix typo", "fix again", "actually fix"). Keep `main` history clean with one semantic commit per feature.

---

### 📖 Interactive Rebase — The Power Tool

```bash
# Rewrite last 4 commits interactively
git rebase -i HEAD~4

# Editor opens with:
pick a1b2c3 feat: add oauth strategy
pick d4e5f6 fix typo
pick g7h8i9 wip
pick j0k1l2 actually working now

# Rewrite as:
pick a1b2c3 feat: add oauth strategy    ← keep as-is
squash d4e5f6 fix typo                  ← squash into previous
squash g7h8i9 wip                       ← squash into previous
reword j0k1l2 actually working now      ← keep but edit message

# Commands: pick | squash (s) | fixup (f) | reword (r) | edit (e) | drop (d)
# fixup = squash but discard that commit's message
# drop = remove commit entirely from history
```

---

### Decision Guide

```mermaid
flowchart TD
    Q1{"Is the branch\nshared with others?"}
    Q1 -->|Yes| MERGE["Use git merge\nNever rebase shared branches"]
    Q1 -->|No, local only| Q2

    Q2{"Do you want\nlinear history?"}
    Q2 -->|Yes| Q3
    Q2 -->|No — preserve context| MERGE2["Use git merge\nMerge commit shows branch context"]

    Q3{"Many messy\nWIP commits?"}
    Q3 -->|Yes| SQUASH["Use squash merge\nor rebase -i to clean up\nbefore merging"]
    Q3 -->|No — clean commits| REBASE["Use git rebase\nfor clean linear history"]

    style MERGE fill:#cce5ff,stroke:#0066cc
    style MERGE2 fill:#cce5ff,stroke:#0066cc
    style SQUASH fill:#fff3cd,stroke:#856404
    style REBASE fill:#d4edda,stroke:#155724
```

---

## 3. Git Reset vs Revert vs Restore

### ❓ Interview Question
> **"Explain the difference between `git reset`, `git revert`, and `git restore`. When is each appropriate and what are the dangers?"**

### 📖 Quick Comparison

| Command | Modifies History | Safe for Shared Branches | Use Case |
|---|---|---|---|
| `git reset` | ✅ Yes (rewrites) | ❌ No | Undo local commits not yet pushed |
| `git revert` | ❌ No (adds new commit) | ✅ Yes | Undo commits already on shared branch |
| `git restore` | ❌ No (working tree only) | ✅ Yes | Discard uncommitted file changes |

---

### `git reset` — Move HEAD Backwards

```bash
# --soft: move HEAD, keep changes staged
git reset --soft HEAD~1
# → Last commit undone. Changes are staged (ready to recommit differently)
# Use: rework the last commit message or combine with next commit

# --mixed (default): move HEAD, unstage changes but keep in working tree
git reset HEAD~2
# → Last 2 commits undone. Changes are unstaged in working tree.
# Use: split one commit into multiple, or completely rework

# --hard: move HEAD, DESTROY all changes (dangerous!)
git reset --hard HEAD~1
# → Last commit undone. Changes are GONE from staging and working tree.
# Use: throw away last commit and all its changes completely

# Reset to a specific commit
git reset --hard a1b2c3d4

# Undo a reset (within reflog window — 90 days default)
git reflog                  # find the hash before reset
git reset --hard <old-hash>
```

```
Timeline visualization:

Before:  A──B──C──D  (HEAD at D)

git reset --soft HEAD~2:
After:   A──B         (HEAD at B, C+D changes staged)

git reset --hard HEAD~2:
After:   A──B         (HEAD at B, C+D changes GONE)
```

---

### `git revert` — Safe Undo for Shared History

```bash
# Create a new commit that undoes the changes of a specific commit
git revert a1b2c3d4          # opens editor for revert commit message
git revert a1b2c3d4 --no-edit  # use default message

# Revert a range of commits (oldest first)
git revert a1b2c3..d4e5f6   # reverts each commit individually

# Revert multiple commits as a single revert commit
git revert -n a1b2c3 d4e5f6 # -n = --no-commit, stage but don't commit
git commit -m "revert: undo OAuth changes due to provider outage"

# Revert a merge commit (must specify which parent to keep)
git revert -m 1 <merge-commit-hash>   # -m 1 = keep mainline parent
```

```
Timeline:
Before:  A──B──C──D  (D is broken, already on main)

git revert D:
After:   A──B──C──D──D'   (D' undoes D's changes, but D still in history)
# Safe for shared branches — history is additive, not rewritten
```

---

### `git restore` — Working Tree and Stage Operations

```bash
# Discard unstaged changes in a file (restore from last commit)
git restore main.tf               # working tree changes discarded
git restore .                     # discard ALL unstaged changes

# Unstage a file (restore index from HEAD)
git restore --staged main.tf      # moves file from staged to unstaged

# Restore a file from a specific commit
git restore --source=a1b2c3 main.tf

# Restore to a specific branch's version
git restore --source=origin/main main.tf
```

> **`git restore` vs old commands**: `git restore` (Git 2.23+) replaced parts of `git checkout` and `git reset HEAD` for clarity. `git restore --staged` = old `git reset HEAD <file>`. `git restore <file>` = old `git checkout -- <file>`.

---

## 4. Git Cherry-Pick

### ❓ Interview Question
> **"What is `git cherry-pick` and when would you use it? What are the risks of cherry-picking in a long-running project?"**

### 📖 Core Concept

Cherry-pick applies the changes introduced by specific commits onto the current branch — without merging the entire branch.

```
main     A──B──C──D──E
              │
hotfix        X──Y──Z   (Z = critical security fix)

# Need Z on main NOW without merging all of hotfix
git checkout main
git cherry-pick Z

main     A──B──C──D──E──Z'  (Z' = same changes as Z, new hash)
```

```bash
# Cherry-pick a single commit
git cherry-pick a1b2c3d4

# Cherry-pick multiple commits
git cherry-pick a1b2c3 d4e5f6 g7h8i9

# Cherry-pick a range (inclusive)
git cherry-pick a1b2c3^..g7h8i9

# Cherry-pick without committing (stage only)
git cherry-pick --no-commit a1b2c3

# Cherry-pick and edit the commit message
git cherry-pick -e a1b2c3

# If conflict during cherry-pick
git status                    # see conflicted files
# resolve conflicts
git add <resolved-files>
git cherry-pick --continue    # or --abort to cancel
```

### Common Use Cases

```bash
# 1. Backport a bug fix from main to a release branch
git checkout release/1.5
git cherry-pick <bugfix-commit-from-main>

# 2. Apply a hotfix from hotfix branch to both main and develop
git checkout main
git cherry-pick <hotfix-hash>
git checkout develop
git cherry-pick <hotfix-hash>

# 3. Rescue work from a deleted or diverged branch
git reflog                    # find commits from the deleted branch
git cherry-pick <hash1> <hash2> <hash3>
```

### ⚠️ Risks of Cherry-Picking

- **Duplicate commits**: Same logical change exists in two branches with different hashes — can cause confusing conflicts when branches eventually merge.
- **Missing context**: A commit may depend on other commits not cherry-picked — causes subtle bugs.
- **History divergence**: Heavy cherry-picking between long-lived branches signals a broken branching strategy — fix the strategy instead.

---

## 5. Git Stash — Advanced Usage

### ❓ Interview Question
> **"You're mid-way through a feature and need to switch branches urgently. Walk me through using `git stash` correctly, including named stashes and recovering a dropped stash."**

### 📖 Core Stash Commands

```bash
# Stash current changes (tracked files only)
git stash

# Stash with a descriptive name
git stash push -m "WIP: OAuth2 callback logic — half-done"

# Stash including untracked files
git stash push --include-untracked -m "WIP: with new files"

# Stash including ignored files too
git stash push --all -m "full state including .env"

# List all stashes
git stash list
# stash@{0}: On feat/oauth: WIP: OAuth2 callback logic — half-done
# stash@{1}: On main: quick debug stash

# Apply most recent stash (keeps stash in list)
git stash apply

# Apply a specific stash
git stash apply stash@{1}

# Pop most recent stash (apply + remove from list)
git stash pop

# Pop a specific stash
git stash pop stash@{1}

# Show what's in a stash
git stash show -p stash@{0}   # -p for full diff

# Drop a specific stash
git stash drop stash@{1}

# Drop all stashes
git stash clear
```

### Creating a Branch from a Stash

```bash
# Useful when the stash conflicts with current branch state
git stash branch feat/recovered-work stash@{0}
# Creates new branch at the commit where stash was created, applies stash
```

### Recovering a Dropped Stash

```bash
# Stashes are commits — they stay in object database until GC
# Find lost stash objects
git fsck --no-reflogs | grep "dangling commit"
# dangling commit a1b2c3d4

# Inspect each dangling commit
git show a1b2c3d4   # check if this is your stash

# Recover it
git stash apply a1b2c3d4
# or
git checkout -b recovered-stash a1b2c3d4
```

---

## 6. Git Tags and Semantic Versioning

### ❓ Interview Question
> **"What is the difference between a lightweight and annotated tag? How do you use tags for semantic versioning in a CI/CD pipeline?"**

### 📖 Lightweight vs Annotated Tags

```bash
# Lightweight tag — just a pointer to a commit (no metadata)
git tag v1.2.0

# Annotated tag — full object with tagger, date, message, GPG signature
git tag -a v1.2.0 -m "Release 1.2.0 — adds OAuth2 login, rate limiting"

# Signed annotated tag (GPG)
git tag -s v1.2.0 -m "Release 1.2.0"

# Tag a specific commit (not HEAD)
git tag -a v1.1.1 a1b2c3d4 -m "Hotfix 1.1.1 — patch XSS vulnerability"

# List tags
git tag                          # all tags
git tag -l "v1.*"                # filter by pattern
git tag -n                       # with tag messages

# Show tag details
git show v1.2.0

# Push a single tag
git push origin v1.2.0

# Push all tags
git push origin --tags

# Delete a local tag
git tag -d v1.2.0

# Delete a remote tag
git push origin --delete v1.2.0
```

### CI/CD Pipeline Triggered by Tags

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'    # trigger on semver tags only

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0            # fetch all history + tags

      - name: Extract version from tag
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV

      - name: Build Docker image
        run: |
          docker build -t myapp:${{ env.VERSION }} .
          docker tag myapp:${{ env.VERSION }} myapp:latest

      - name: Push to registry
        run: |
          docker push myapp:${{ env.VERSION }}
          docker push myapp:latest

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true    # auto-generate from Conventional Commits
```

### Automated Semantic Versioning

```bash
# Using conventional-changelog + semantic-release
# semantic-release reads Conventional Commits and auto-tags:
# feat → MINOR bump (1.2.0 → 1.3.0)
# fix  → PATCH bump (1.2.0 → 1.2.1)
# feat! or BREAKING CHANGE → MAJOR bump (1.2.0 → 2.0.0)

# Manual semver tagging workflow
CURRENT=$(git describe --tags --abbrev=0)   # get latest tag: v1.2.0
# After a fix commit:
git tag -a v1.2.1 -m "Release v1.2.1"
git push origin v1.2.1
```

---

## 7. Resolving Complex Merge Conflicts

### ❓ Interview Question
> **"Walk me through how you resolve a complex merge conflict. What tools and strategies do you use? What is the difference between `ours` and `theirs` in a merge vs a rebase?"**

### 📖 Understanding Conflict Markers

```python
# Python file during merge conflict
def get_user(user_id):
<<<<<<< HEAD                          ← your branch (current)
    return db.query(User).filter_by(id=user_id).first()
||||||| merged common ancestor        ← original (only with diff3 style)
    return User.query.get(user_id)
=======
    user = cache.get(f"user:{user_id}")   ← incoming branch (theirs)
    if not user:
        user = User.query.get(user_id)
        cache.set(f"user:{user_id}", user)
    return user
>>>>>>> feat/add-redis-cache
```

```bash
# Show conflicts with 3-way diff (shows original/ancestor too — highly recommended)
git config --global merge.conflictstyle diff3

# After resolving:
git add <resolved-file>
git merge --continue   # or: git commit
```

### Merge Tools

```bash
# Set up a visual merge tool
git config --global merge.tool vimdiff
git config --global merge.tool vscode

# VS Code as merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Open merge tool for all conflicted files
git mergetool

# Check for remaining conflict markers (before committing)
git diff --check
grep -r "<<<<<<" .    # find any remaining conflict markers
```

### `ours` vs `theirs` — Context Matters!

```bash
# During MERGE:
# "ours"   = the branch you're ON (git checkout main → merge feat)
# "theirs" = the branch being MERGED IN (feat)

# During REBASE:
# "ours"   = the UPSTREAM branch (main — what you're rebasing onto)
# "theirs" = YOUR commits (the branch being replayed)
# ⚠️ This is REVERSED from merge — a common interview trap!

# Accept all "ours" for a file during merge
git checkout --ours path/to/file.py
git add path/to/file.py

# Accept all "theirs" for a file during merge
git checkout --theirs path/to/file.py
git add path/to/file.py

# Strategy: accept all "ours" for entire merge
git merge -s ours feat/branch     # entire merge uses our files

# Recursive strategy with theirs preference per-file
git merge -X theirs feat/branch
```

### Large Conflict — Binary File Strategy

```bash
# Binary files can't be diff'd — must choose one version entirely
# During merge conflict on a binary:
git checkout --ours   image.png   # keep our version
git checkout --theirs image.png   # keep their version
git add image.png
```

---

## 8. Git Reflog — Recovering Lost Work

### ❓ Interview Question
> **"You accidentally ran `git reset --hard` and lost commits. How do you recover them using `git reflog`?"**

### 📖 What Reflog Is

The reflog is a local log of every movement of HEAD and branch refs — including resets, rebases, checkouts, merges, and commits. It retains entries for 90 days by default (configurable). It is your safety net for any destructive operation.

```bash
# View full reflog
git reflog

# Example output:
# a1b2c3d (HEAD -> main) HEAD@{0}: reset: moving to HEAD~3      ← the bad reset
# d4e5f6g HEAD@{1}: commit: feat: add rate limiting
# g7h8i9a HEAD@{2}: commit: feat: add Redis cache
# j0k1l2m HEAD@{3}: commit: fix: resolve null pointer
# ...

# The commits d4e5f6g, g7h8i9a, j0k1l2m are now "lost" from main
# but still exist in the object database

# Recovery option 1: reset back to before the bad reset
git reset --hard HEAD@{1}    # restore to state before the reset

# Recovery option 2: cherry-pick specific lost commits
git cherry-pick d4e5f6g g7h8i9a j0k1l2m

# Recovery option 3: create a branch at the lost commit
git checkout -b recovered HEAD@{1}

# View reflog for a specific branch
git reflog show feat/oauth

# View reflog with dates
git reflog --date=iso
```

### Recovering from Bad Rebase

```bash
# You rebased and the result looks wrong
git reflog
# HEAD@{0}: rebase (finish): returning to refs/heads/feat/oauth
# HEAD@{1}: rebase (pick): fix: typo
# HEAD@{7}: rebase (start): checkout main
# HEAD@{8}: commit: feat: add oauth callback   ← pre-rebase state

# Recover pre-rebase state
git reset --hard HEAD@{8}
# or
git checkout -b pre-rebase-recovery ORIG_HEAD
# ORIG_HEAD is automatically set before any merge/rebase to the previous HEAD
```

### Recovering a Deleted Branch

```bash
# Branch 'feat/payment' was accidentally deleted
git reflog | grep "feat/payment"
# d4e5f6g HEAD@{12}: checkout: moving from feat/payment to main

# The last commit on that branch was d4e5f6g
git checkout -b feat/payment d4e5f6g
```

---

## 9. Git Bisect — Binary Search Debugging

### ❓ Interview Question
> **"A performance regression was introduced somewhere in the last 200 commits. How would you use `git bisect` to find it efficiently?"**

### 📖 Manual Bisect

```bash
# Start bisect session
git bisect start

# Mark current commit as bad (regression present)
git bisect bad

# Mark a known good commit (no regression)
git bisect good v1.0.0       # tag
git bisect good a1b2c3d4     # specific hash
git bisect good "3 months ago"   # relative date

# Git checks out midpoint — you test, then mark:
git bisect good   # or:
git bisect bad

# After ~7-8 iterations through 200 commits:
# "d4e5f6g is the first bad commit"
# → git show d4e5f6g tells you exactly what changed

# End bisect session (returns to original HEAD)
git bisect reset
```

### Automated Bisect with a Test Script

```bash
# Write a test that exits 0 for good, non-zero for bad
cat > /tmp/test-performance.sh << 'EOF'
#!/bin/bash
# Build and run performance test
go build -o /tmp/myapp ./cmd/server
LATENCY=$(curl -o /dev/null -s -w "%{time_total}" http://localhost:8080/api/users)
# Fail if latency > 200ms
python3 -c "exit(0 if $LATENCY < 0.200 else 1)"
EOF
chmod +x /tmp/test-performance.sh

# Run fully automated bisect
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
git bisect run /tmp/test-performance.sh
# Git automatically finds the bad commit with no manual input

git bisect reset
```

### Bisect with Unit Test

```bash
# Use existing test suite — test must be deterministic
git bisect start
git bisect bad
git bisect good v1.5.0
git bisect run pytest tests/test_payment.py::test_checkout_total -x
# -x = stop on first failure
git bisect reset
```

---

## 10. Git Hooks — Automation and Enforcement

### ❓ Interview Question
> **"What are Git hooks? Describe the difference between client-side and server-side hooks and give examples of how you've used them."**

### 📖 Client-Side Hooks

Hooks live in `.git/hooks/` — scripts that run at specific Git events. They are local to each developer's machine (not committed to the repo by default).

```bash
# Install Husky to share hooks via npm (committed to repo)
npm install --save-dev husky
npx husky init

# Hooks are now in .husky/ directory (committed to repo)
```

#### `pre-commit` — Validate before committing

```bash
# .husky/pre-commit
#!/bin/sh
echo "Running pre-commit checks..."

# 1. Run linter on staged files only
npx lint-staged

# 2. Check for debug statements
if git diff --cached | grep -E "console\.log|debugger|pdb\.set_trace" > /dev/null; then
  echo "❌ Debug statements found in staged changes"
  exit 1
fi

# 3. Prevent committing to main directly
BRANCH=$(git symbolic-ref --short HEAD)
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "master" ]; then
  echo "❌ Direct commits to $BRANCH are not allowed"
  echo "   Create a feature branch: git checkout -b feat/your-feature"
  exit 1
fi
```

#### `commit-msg` — Enforce Conventional Commits

```bash
# .husky/commit-msg
#!/bin/sh
npx --no -- commitlint --edit "$1"
```

#### `pre-push` — Run tests before push

```bash
# .husky/pre-push
#!/bin/sh
echo "Running tests before push..."
npm test || exit 1

# Prevent force push to main
BRANCH=$(git symbolic-ref --short HEAD)
if [ "$BRANCH" = "main" ]; then
  echo "❌ Force push to main is not allowed"
  exit 1
fi
```

---

### 📖 Server-Side Hooks

Server-side hooks run on the remote (GitHub, GitLab, Bitbucket). They enforce rules centrally — developers cannot bypass them.

| Hook | Trigger | Use Case |
|---|---|---|
| `pre-receive` | Before any refs are updated | Reject commits with secrets, enforce commit format |
| `update` | Once per pushed branch | Enforce per-branch policies |
| `post-receive` | After refs updated | Trigger CI/CD, send notifications, update issue tracker |

```bash
# Server-side pre-receive hook (on self-hosted Git)
#!/bin/bash
# Reject commits containing potential secrets
while read oldrev newrev refname; do
  if git diff $oldrev $newrev | \
     grep -E "(password|secret|api_key|AWS_SECRET)\s*=\s*['\"][^'\"]+['\"]"; then
    echo "❌ Potential secret detected in push. Push rejected."
    exit 1
  fi
done
```

> **Note**: On GitHub/GitLab, server-side enforcement is handled by branch protection rules, required CI status checks, and platform-level push rules — not custom hook scripts.

### lint-staged — Only Lint Staged Files

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts}": ["eslint --fix", "prettier --write"],
    "*.py": ["black", "flake8"],
    "*.tf": ["terraform fmt"],
    "*.{yaml,yml}": ["yamllint"]
  }
}
```

---

## 11. Git Internals — Objects, Refs, Packfiles

### ❓ Interview Question
> **"Explain Git's internal object model. What are the four object types? What is the difference between HEAD and ORIG_HEAD?"**

### 📖 The Four Object Types

Git is a content-addressed object store. Every object is identified by its SHA-1 hash of its content.

```
.git/objects/
├── a1/b2c3d4e5...   ← blob
├── f6/g7h8i9...     ← tree
├── j0/k1l2m3...     ← commit
└── n4/o5p6q7...     ← tag
```

| Object | Contains | Created By |
|---|---|---|
| **blob** | Raw file contents — no filename, no path | `git add` |
| **tree** | Directory listing — maps filenames to blob/tree hashes | Each commit |
| **commit** | Author, committer, message, pointer to tree, parent commit hash(es) | `git commit` |
| **tag** | Annotated tag — tagger, date, message, pointer to commit | `git tag -a` |

```bash
# Inspect Git objects
git cat-file -t a1b2c3d4     # show type: blob/tree/commit/tag
git cat-file -p a1b2c3d4     # show content

# See the tree object of HEAD
git cat-file -p HEAD^{tree}

# See the blob for a specific file
git ls-tree HEAD              # list tree at HEAD
git cat-file -p HEAD:main.tf  # contents of main.tf at HEAD
```

### 📖 Refs — Pointers to Commits

```
.git/refs/
├── heads/
│   ├── main          ← content: a1b2c3d4 (hash of latest commit)
│   └── feat/oauth    ← content: d4e5f6g7
├── remotes/
│   └── origin/
│       └── main      ← content: j0k1l2m3
└── tags/
    └── v1.2.0        ← content: n4o5p6q7 (annotated tag object hash)
```

```bash
# Special refs
cat .git/HEAD            # → ref: refs/heads/main  (or a commit hash in detached HEAD)
cat .git/ORIG_HEAD       # → set before merge/rebase to the previous HEAD
cat .git/MERGE_HEAD      # → exists during a merge conflict
cat .git/CHERRY_PICK_HEAD # → exists during a cherry-pick conflict

# Detached HEAD — HEAD points to a commit hash directly (not a branch)
git checkout a1b2c3d4    # → HEAD is now a1b2c3d4 (detached)
git checkout -b recovery-branch  # → create branch to escape detached HEAD
```

### 📖 Packfiles — Efficient Storage

```bash
# Loose objects → packed objects
git gc                   # run garbage collection + pack
git gc --aggressive      # more thorough repacking

# Count loose objects
git count-objects -v

# The .git/objects/pack/ directory after packing:
# pack-abc123.pack   ← compressed delta-encoded object data
# pack-abc123.idx    ← index for fast lookup
```

---

## 12. Git Submodules and Subtrees

### ❓ Interview Question
> **"What is the difference between Git submodules and Git subtrees? When would you use each for managing shared libraries?"**

### 📖 Git Submodules — Pointer to External Repo

A submodule is a Git repository embedded inside another. The parent repo stores a `.gitmodules` file and a pointer (specific commit hash) to the submodule repo.

```bash
# Add a submodule
git submodule add https://github.com/myorg/shared-lib.git libs/shared

# Clone a repo with submodules
git clone --recurse-submodules https://github.com/myorg/myapp.git

# Initialize submodules after a regular clone
git submodule init
git submodule update

# Update all submodules to their latest tracked commit
git submodule update --remote

# Run a command in every submodule
git submodule foreach 'git pull origin main'
```

```ini
# .gitmodules
[submodule "libs/shared"]
    path = libs/shared
    url = https://github.com/myorg/shared-lib.git
    branch = main
```

**Submodule pros**: Clear separation, submodule has its own history, can pin to exact commit.
**Submodule cons**: Complex for developers (`--recurse-submodules` required), easy to forget to update, nested clone complexity.

---

### 📖 Git Subtrees — Copy External Repo into Subdirectory

Subtrees merge the external repo's history directly into the parent repo. Simpler for contributors — no special clone steps.

```bash
# Add a subtree
git subtree add \
  --prefix=libs/shared \
  https://github.com/myorg/shared-lib.git \
  main \
  --squash          # squash external history into one commit

# Pull updates from the subtree remote
git subtree pull \
  --prefix=libs/shared \
  https://github.com/myorg/shared-lib.git \
  main \
  --squash

# Push changes back to the subtree remote
git subtree push \
  --prefix=libs/shared \
  https://github.com/myorg/shared-lib.git \
  main
```

### Comparison

| Dimension | Submodules | Subtrees |
|---|---|---|
| **External repo linked** | Yes — pointer only | Yes — history merged in |
| **Clone complexity** | `--recurse-submodules` needed | Regular `git clone` works |
| **Contributor experience** | Complex — easy to forget update | Simple — feels like normal directory |
| **Pushing changes upstream** | Easy — commit in submodule, push | Possible but complex (`git subtree push`) |
| **Pinned to specific commit** | Yes — explicit in `.gitmodules` | Yes — squash commit or tracked |
| **History clarity** | Clean (separate repo) | Can clutter parent history |
| **Best for** | Libraries updated independently by another team | Shared code owned by same team, rarely updated |

---

## 13. Git LFS — Large File Storage

### ❓ Interview Question
> **"What is Git LFS and when do you need it? What happens to repository performance without it when binary files are committed?"**

### 📖 The Problem

Git stores every version of every file as a blob object. For text files (code), this is efficient because diffs are small. For binary files (images, ML models, compiled artifacts, videos), every version is stored in full — a 500MB ML model changed 10 times = 5GB in `.git/objects`. Repository becomes unusably slow to clone.

### 📖 How LFS Works

LFS replaces large files in the repo with small text pointer files. The actual binary content lives on an LFS server (GitHub, GitLab, S3-backed).

```
# What's in the repo (pointer file — tiny):
version https://git-lfs.github.com/spec/v1
oid sha256:a1b2c3d4e5f6...
size 524288000

# Actual file lives on LFS server — fetched on checkout
```

```bash
# Install Git LFS
git lfs install

# Track file types with LFS
git lfs track "*.psd"
git lfs track "*.pkl"       # ML model files
git lfs track "*.pt"        # PyTorch models
git lfs track "datasets/**"

# .gitattributes (committed to repo — defines LFS tracking)
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.pkl filter=lfs diff=lfs merge=lfs -text

git add .gitattributes
git add large-model.pkl
git commit -m "chore: add ML model via LFS"
git push                    # pushes pointer to Git, binary to LFS server

# Check LFS status
git lfs status
git lfs ls-files            # list LFS-tracked files

# Migrate existing repo to LFS (rewrite history)
git lfs migrate import --include="*.pkl,*.pt" --everything
git push --force --all      # push rewritten history
```

### LFS in CI/CD

```yaml
# GitHub Actions — LFS files available in checkout
- uses: actions/checkout@v4
  with:
    lfs: true              # fetch LFS files during checkout
```

---

## 14. Git Security — Secrets, Signing, History Rewriting

### ❓ Interview Question
> **"A developer accidentally committed AWS access keys to a public repository. What do you do immediately and how do you permanently remove the secret from Git history?"**

### 📖 Immediate Response (First 60 seconds)

```bash
# 1. IMMEDIATELY revoke the credentials in AWS IAM
#    Do NOT wait — bots scan GitHub in seconds
aws iam delete-access-key --access-key-id AKIAIOSFODNN7EXAMPLE

# 2. Check if it was pushed to remote
git log --all --oneline | head -20
git remote -v

# 3. Assess blast radius
git log --all -S "AKIAIOSFODNN7EXAMPLE" --oneline
```

### 📖 Removing Secrets from History

**Method 1: `git filter-repo` (modern, recommended)**

```bash
# Install
pip install git-filter-repo

# Remove a specific file entirely from all history
git filter-repo --path secrets.env --invert-paths

# Replace a specific string throughout all history
git filter-repo --replace-text <(echo 'AKIAIOSFODNN7EXAMPLE==>***REMOVED***')

# After rewriting history — force push all branches and tags
git push origin --force --all
git push origin --force --tags

# All collaborators must re-clone — their local copies have the old history
```

**Method 2: BFG Repo Cleaner (simpler for common cases)**

```bash
# Install BFG
brew install bfg

# Remove all files named credentials.json from history
bfg --delete-files credentials.json

# Replace all occurrences of a secret string
echo 'AKIAIOSFODNN7EXAMPLE' > secrets.txt
bfg --replace-text secrets.txt

# Clean up and push
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force --all
```

### 📖 Commit Signing — Proving Identity

```bash
# GPG signing setup
gpg --gen-key
gpg --list-secret-keys --keyid-format=long

# Configure Git to sign all commits
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Sign a single commit explicitly
git commit -S -m "feat: add payment processor"

# Verify a commit signature
git verify-commit HEAD
git log --show-signature

# SSH signing (Git 2.34+, simpler — no GPG needed)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

### 📖 `.gitignore` Best Practices for Security

```gitignore
# .gitignore — prevent accidental secret commits

# Environment files
.env
.env.*
!.env.example        # keep the template
*.env

# Credentials and keys
*.pem
*.key
*.p12
*.pfx
id_rsa
id_ed25519
credentials.json
service-account*.json

# Cloud provider credentials
.aws/credentials
.azure/
gcloud-service-key.json

# Terraform state (may contain secrets)
*.tfstate
*.tfstate.*
.terraform/

# IDE and OS files (not secrets but noise)
.DS_Store
.idea/
*.swp
```

```bash
# Check if a secret is tracked before it can be committed
git check-ignore -v secrets.env

# Scan entire repo history for secrets (CI integration)
trufflehog git file://. --since-commit HEAD~50 --only-verified

# Pre-commit secret scanning
pip install detect-secrets
detect-secrets scan > .secrets.baseline
```

---

## 15. Monorepo Git Strategies

### ❓ Interview Question
> **"You work in a monorepo with 50 services and 10 years of history — cloning takes 20 minutes. What Git features do you use to make the developer experience manageable?"**

### 📖 Partial Clone — Fetch Objects On Demand

```bash
# Clone without blobs (fastest initial clone)
git clone --filter=blob:none https://github.com/myorg/monorepo.git

# Clone without trees (even faster — for CI/CD)
git clone --filter=tree:0 https://github.com/myorg/monorepo.git

# Blobs are fetched lazily when files are accessed
# Git 2.22+ required on server
```

### 📖 Sparse Checkout — Check Out Only What You Need

```bash
# Enable sparse checkout
git clone --no-checkout https://github.com/myorg/monorepo.git
cd monorepo
git sparse-checkout init --cone

# Check out only specific directories
git sparse-checkout set services/payment services/auth shared/libs

# List what's checked out
git sparse-checkout list

# Add more directories
git sparse-checkout add services/notification

# Full checkout (disable sparse)
git sparse-checkout disable
```

### 📖 Git Worktrees — Multiple Working Trees from One Clone

```bash
# One clone, multiple branches checked out simultaneously
# No need for multiple clones

# Add a worktree for a different branch
git worktree add ../monorepo-main main
git worktree add ../monorepo-hotfix hotfix/1.2.1

# List all worktrees
git worktree list

# Work in another branch without stashing
cd ../monorepo-hotfix
git commit -m "hotfix: fix null pointer"
cd ../monorepo           # back to main worktree

# Remove a worktree when done
git worktree remove ../monorepo-hotfix

# Locked worktree (protects from gc)
git worktree lock ../monorepo-main --reason "active development"
```

### 📖 Shallow Clone — CI/CD Optimization

```bash
# Clone only last N commits (fast CI checkout)
git clone --depth=1 https://github.com/myorg/monorepo.git

# In GitHub Actions
- uses: actions/checkout@v4
  with:
    fetch-depth: 1        # shallow clone for speed

# Unshallow when needed (e.g., for git describe --tags)
git fetch --unshallow
```

### Monorepo CI Optimization — Only Build Changed Services

```bash
# Find which services changed in this PR
CHANGED=$(git diff --name-only origin/main...HEAD | \
          grep '^services/' | \
          cut -d/ -f2 | \
          sort -u)

echo "Changed services: $CHANGED"

# Build only changed services
for SERVICE in $CHANGED; do
  echo "Building $SERVICE..."
  docker build -t myorg/$SERVICE:${GITHUB_SHA} services/$SERVICE/
done
```

---

## 16. Troubleshooting Scenarios

### Scenario 1 — "My PR has 50 merge conflicts with main"

```bash
# Option A: Merge main into your branch (keeps history, adds merge commit)
git fetch origin
git checkout feat/my-feature
git merge origin/main           # resolve conflicts
git push origin feat/my-feature

# Option B: Rebase onto main (linear history, but rewrites your commits)
git fetch origin
git checkout feat/my-feature
git rebase origin/main          # resolve conflicts at each step
git push --force-with-lease origin feat/my-feature

# Option C: Interactive rebase — also squash messy commits while you're at it
git rebase -i origin/main
```

---

### Scenario 2 — "I committed to the wrong branch (main instead of my feature branch)"

```bash
# The commit is on main — you need it on feat/my-feature instead

# Step 1: Note the commit hash
git log --oneline -3
# a1b2c3 my accidental commit on main
# d4e5f6 previous main commit

# Step 2: Create/switch to correct branch at current state
git checkout -b feat/my-feature   # brings the accidental commit with you

# Step 3: Remove the commit from main
git checkout main
git reset --hard HEAD~1          # safe only if not yet pushed
# If already pushed: git revert HEAD, then push
```

---

### Scenario 3 — "Git pull is failing with 'diverged branches'"

```bash
# Error: "Your branch and 'origin/main' have diverged"
# This means you have local commits AND remote has new commits

# Option A: Merge (creates merge commit)
git pull                    # default: merge
# or explicitly:
git fetch origin
git merge origin/main

# Option B: Rebase (linear history)
git pull --rebase
# or configure as default:
git config --global pull.rebase true

# Option C: Fast-forward only (fail if not possible — safest for main)
git pull --ff-only
```

---

### Scenario 4 — "I need to undo the last 3 commits on a shared branch"

```bash
# WRONG: git reset --hard HEAD~3 + force push
# → Rewrites shared history → breaks all collaborators

# CORRECT: git revert (additive — safe for shared branches)
# Revert each commit individually:
git revert HEAD       # revert most recent
git revert HEAD~1     # revert second most recent
git revert HEAD~2     # revert third most recent

# Or revert a range as a single revert commit:
git revert --no-commit HEAD~3..HEAD
git commit -m "revert: undo last 3 commits — deployment issue INC-2025-005"

git push origin main  # no force push needed
```

---

### Scenario 5 — "git status shows untracked files I don't want to commit or delete"

```bash
# View what's untracked
git status
git ls-files --others --exclude-standard

# Options:
# 1. Add to .gitignore (permanent — for build artifacts, IDE files)
echo "build/" >> .gitignore
echo ".idea/" >> .gitignore

# 2. Stash with untracked (temporary — keep but hide)
git stash push --include-untracked

# 3. Clean (delete) untracked files
git clean -n         # dry run — shows what would be deleted
git clean -fd        # delete untracked files and directories
git clean -fdx       # delete including .gitignore'd files (careful!)
```

---

## 🧠 Master Summary

| Topic | Key Point |
|---|---|
| **Branching Strategies** | GitFlow for versioned releases; GitHub Flow for continuous delivery; Trunk-Based for high-frequency teams with feature flags |
| **Merge vs Rebase** | Merge preserves history; rebase linearizes; squash cleans WIP commits. Never rebase shared branches |
| **`--force-with-lease`** | Always use instead of `--force` — fails if someone else pushed since your last fetch |
| **Reset** | `--soft` keeps staged; `--mixed` unstages; `--hard` destroys. Never hard-reset pushed commits |
| **Revert** | Safe for shared branches — creates a new commit that undoes changes; history is additive |
| **Restore** | `git restore <file>` discards working tree changes; `--staged` unstages |
| **Cherry-pick** | Apply specific commits across branches; great for hotfix backports; avoid heavy use — it signals branching strategy issues |
| **Stash** | Name your stashes; use `--include-untracked`; recover dropped stash with `git fsck --no-reflogs` |
| **Tags** | Annotated tags (`-a`) for releases; lightweight for local bookmarks; push with `--tags`; trigger CI/CD |
| **Conflicts** | Use `diff3` style; `--ours`/`--theirs` flip meaning in merge vs rebase; `git mergetool` for visual resolution |
| **Reflog** | 90-day safety net for all HEAD movements; `ORIG_HEAD` restores pre-merge/rebase state instantly |
| **Bisect** | Binary search through 200 commits in ~8 steps; automate with `git bisect run <script>` |
| **Hooks** | Client-side for convenience; server-side for enforcement. Husky + lint-staged + commitlint for teams |
| **Internals** | 4 objects: blob (content), tree (dir), commit (snapshot), tag (annotated pointer). SHA-1 content-addressed |
| **Submodules vs Subtrees** | Submodules: clear separation, complex UX. Subtrees: simple UX, merged history |
| **LFS** | Replace binary blobs with pointer files. Essential for ML models, design assets, large datasets |
| **Secret removal** | `git filter-repo` (modern) or BFG; IMMEDIATELY revoke credentials before cleaning history |
| **Commit signing** | GPG or SSH signing proves author identity; `commit.gpgsign = true` in gitconfig |
| **Monorepo** | Partial clone + sparse checkout + shallow clone = fast developer experience at scale |
| **Worktrees** | Multiple branches checked out simultaneously from one clone — no need for multiple clone directories |
