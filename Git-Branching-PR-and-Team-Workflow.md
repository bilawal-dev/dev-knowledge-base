# Git Branching, Rebase, PRs & Team Workflow

A practical guide to Git history, pull requests, conflicts, squash merges, and real-world team scenarios.

---

## 0. Core Mental Models (Read This First)

### What a Branch Is

A branch is just a **pointer to a commit**.

> "A branch is not a copy of the repo. It's a movable label."

### What a Commit Is

A commit is a **snapshot of the project** with a parent reference.

> "Commits are the real history. Branches just point at them."

### What a Pull Request (PR) Is

A PR is **not code**.
A PR is a **live comparison between two branch tips**.

> "Show me everything in `dev_patience` that is not in `main`."

Because of this:
- Any new commit pushed to the branch → appears automatically in the PR
- You never "update a PR" directly
- You update the branch, and the PR reflects it

---

## 1. What Happens When New Commits Are Pushed to a PR Branch

If Patience opens a PR from `dev_patience` → `main`:

```
main: A — B — C
dev:          \ — D — E
```

**PR shows:** D, E

If he later pushes more commits:

```
dev:          \ — D — E — F — G
```

👉 The same PR updates automatically
👉 No need to close
👉 No new PR

---

## 2. Divergence: When Main Moves Forward

If while he is working, main gets new commits:

```
main: A — B — C — X — Y
dev:          \ — D — E
```

Now branches **diverged**.

**Possible states:**
- ✅ No conflicts → PR can still be merged
- ❌ Conflicts → dev branch must be updated

> **Important:** A PR can exist even if branches diverged.

---

## 3. Conflict Resolution (Real Workflow)

**Conflicts are not fixed in PRs.**
**They are fixed in branches.**

### Correct Professional Flow

On Patience's machine:

```bash
git checkout dev_patience
git fetch origin
git merge origin/main
```

If conflicts exist, Git pauses.

He opens VS Code, fixes files, then:

```bash
git add .
git commit
git push
```

That commit:
- Is the **merge commit**
- Contains the conflict resolution
- Has **two parents**

👉 PR updates automatically
👉 Conflicts disappear
👉 Same PR, no recreation

---

## 4. Merge Commit vs Conflict Resolution Commit

**They are the same thing.**

There is no extra "conflict commit".

**Flow:**

```bash
git merge origin/main   # ← starts merge
# fix files
git add .
git commit              # ← creates the merge commit
```

That commit:
- Completes the merge
- Records conflict fixes
- Has two parents

---

## 5. What Happens When You Click "Merge PR"

GitHub offers 3 strategies:

### 1) Create a Merge Commit (Normal Merge)

**History:**

```
main: A — B — C — X — Y — P
                      /
dev:          \ — D — E — M
```

- `P` = PR merge commit
- `M` = merge main into dev (if conflicts were fixed)

**Main gets:**
- All feature commits
- Merge commits
- PR merge commit

| Pros | Cons |
|------|------|
| ✔ Preserves full history | ❌ Noisy graph |

### 2) Squash and Merge (Recommended for Your Setup)

**History:**

```
main: A — B — C — X — Y — S
dev:          \ — D — E — M
```

- `S` = one new commit containing all changes

**Main gets:**
- One clean commit

| Pros | Cons |
|------|------|
| ✔ Clean history | ❌ Rewrites history relationship |
| ✔ Easy rollback | |
| ✔ No junk merges | |

### 3) Rebase and Merge

Commits are replayed on main:

```
main: A — B — C — X — Y — D' — E'
```

| Pros | Cons |
|------|------|
| ✔ Linear history | ❌ Rewrites commit hashes |
| | ❌ More dangerous in teams |

---

## 6. Critical Rule After SQUASH Merge

After squash, Git does **NOT** know dev commits were merged.

So the dev branch becomes **logically broken**.

**You MUST do one of these:**

### Option A — Delete Branch (Best Practice)

On GitHub: delete branch

Next work:

```bash
git checkout main
git pull origin main
git checkout -b dev_patience
```

✔ Cleanest workflow.

### Option B — Hard Reset Branch

```bash
git checkout dev_patience
git fetch origin
git reset --hard origin/main
```

Now dev matches main.

### Option C — Rebase Branch

```bash
git checkout dev_patience
git fetch origin
git rebase origin/main
```

More complex. Risky if shared.

> ❌ **Never continue old dev branch after squash without syncing.**

---

## 7. Cherry-Pick (What You Used Correctly)

### What Cherry-Pick Does

Cherry-pick **copies commits** from one branch to another.

```bash
git cherry-pick <commit>
```

It:
- Replays the patch
- Creates a **new commit hash**
- Preserves original author
- You become the committer

> **Mental model:** "Apply this commit here."

### Cherry-Pick a Range

```bash
git cherry-pick A..B
```

Means: all commits **after A up to B**.

**Example:**

```bash
git cherry-pick origin/backup~2..origin/backup
```

✔ Last two commits

---

## 8. Backup Branch Strategy (Safe Recovery)

If a dev has local commits and history changed:

```bash
git push origin dev_patience:backup_dev_patience
```

Creates a remote backup.

Then you can:

```bash
git cherry-pick origin/backup_dev_patience~2..origin/backup_dev_patience
```

After recovery, dev resets:

```bash
git reset --hard origin/dev_patience
```

✔ No lost work
✔ Clean branch
✔ Controlled integration

---

## 9. What History Will Show in Different Strategies

| Strategy | Main History |
|----------|--------------|
| Normal merge | All commits + merge commits |
| Squash merge | One clean commit |
| Rebase merge | All commits, rewritten, no merges |

---

## 10. Two Workflow Strategies: Squash vs Normal Merge

### Strategy A: Squash Merge (Clean History)

**Best for:** Clean main history, easy rollback, short-lived feature branches.

**Flow:**

```bash
git checkout -b dev-bilawal
# work & commit
git push
# open PR → squash merge → delete branch
```

**After merge, you MUST:**
- Delete branch (best), OR
- `git reset --hard origin/main` (if keeping branch name)

> ❌ Never continue working on the branch without reset/delete.

**Main history:** One clean commit per PR.

---

### Strategy B: Normal Merge (Keep Branch Alive)

**Best for:** Long-lived branches, continuous work, no branch deletion.

**Flow:**

```bash
git checkout -b dev-bilawal
# work & commit (C, D)
git push
# open PR → normal merge (Create merge commit)
```

**After merge, sync your branch:**

```bash
git checkout dev-bilawal
git fetch origin
git merge origin/main
# creates a "merge main into dev" commit
git push
```

**Then continue working:**

```bash
# more commits (E, F)
git push
# open new PR → normal merge
```

**Key insight:** The "merge main into dev" commit does **NOT** appear in main's commit list.

```
main: A — B ————— M1 ————————————— M2
            \    /  \             /
dev:         C — D — X — E — F —
                     ↑
              "merge main into dev" (NOT in main's commit list)
```

GitHub follows **first-parent path** for main: `M2 → M1 → B → A`

The sync commit `X` stays on the side branch — main only shows PR merge commits.

---

### Comparison: When to Use Which

| Aspect | Squash Merge | Normal Merge |
|--------|--------------|--------------|
| Main history | One commit per PR | All commits + merge commits |
| Branch after merge | Must delete or reset | Can keep working |
| Sync commits | N/A (branch deleted) | Don't pollute main |
| Rollback | Easy (one commit) | Harder (multiple commits) |
| Best for | Feature branches, clean history | Long-lived dev branches |

---

## 11. Solo Dev Workflow

Even solo, PRs are useful.

**Branch structure:**

```
main
dev-bilawal
```

### Option 1: Squash Merge (Recommended for Solo)

```bash
git checkout -b dev-bilawal
# work & commit
git push
# open PR → squash merge → delete branch
# next feature: create new branch from main
```

**Benefits:**
- Ultra-clean history
- Easy to review your own changes
- Simple rollback

### Option 2: Normal Merge (Keep Branch)

```bash
git checkout dev-bilawal
# work & commit
git push
# open PR → normal merge
# sync: git merge origin/main
# continue working...
```

**Benefits:**
- No branch recreation
- Continuous workflow

---

## 12. Multi-Dev Workflow

**Branch structure:**

```
main
dev-bilawal
dev-patience
```

**Core rules (both strategies):**
- ✔ No direct push to main
- ✔ All work in branches
- ✔ PRs only
- ✔ Fix conflicts locally

### With Squash Merge

Additional rules:
- ✔ Delete branches after merge
- ✔ Never continue branch after squash without reset

### With Normal Merge

Additional rules:
- ✔ Sync branch after each PR merge: `git merge origin/main`
- ✔ Sync commits stay out of main's commit list

---

## 13. If a PR Already Exists and Main Changed

**Do NOT close PR.**

Do this instead:

```bash
git checkout dev_branch
git fetch origin
git merge origin/main
# fix conflicts if any
git add .
git commit
git push
```

PR updates automatically.

> This creates a "merge main into dev" commit. With normal merge PR, this commit won't appear in main's commit list.

---

## 14. How Conflicts Are Resolved in VS Code

1. Git marks files as conflicted
2. VS Code shows conflict UI
3. You choose:
   - Accept current
   - Accept incoming
   - Accept both
   - Manual edit

Then:

```bash
git add .
git commit
git push
```

PR becomes clean.

---

## 15. Final Team Rules (The "Git Law")

**Universal rules:**
- ✔ Feature branches only
- ✔ PRs for everything
- ✔ Fix conflicts locally
- ✔ Never force-push shared main

**If using Squash Merge:**
- ✔ Delete branch after merge
- ✔ Never continue dev branch after squash without reset

**If using Normal Merge:**
- ✔ Sync branch after PR merge: `git merge origin/main`
- ✔ Sync commits don't pollute main's commit list

---

## 16. Final Mental Model Summary

| Concept | Meaning |
|---------|---------|
| Branch | Pointer to commit |
| Commit | Snapshot + parent |
| PR | Live diff between branches |
| Merge conflict | Branch problem |
| Fixing conflicts | Completing a merge |
| Squash merge | New clean commit, must reset/delete branch |
| Normal merge | All commits preserved, can keep branch |
| Sync commit | "merge main into dev" — stays out of main's list |
| Cherry-pick | Copy commits |
| Reset --hard | Move branch pointer |
| Rebase | Replay commits |

---

This document is meant to be a long-term Git reference covering real production scenarios: solo dev, team dev, PR behavior, divergence, conflict resolution, squash merges, and history rewriting.
