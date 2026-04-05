# Week 4: Git & GitHub Advanced Challenge 

## Overview

This document covers all six advanced Git tasks, including commands used, explanations, and best practices for real-world DevOps workflows.

---

## Task 1: Working with Pull Requests (PRs)

### Steps

```bash
# Clone the forked repository
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>

# Create a feature branch and make changes
git checkout -b feature-branch
echo "New Feature" >> feature.txt
git add .
git commit -m "Added a new feature"

# Push the branch to remote
git push origin feature-branch
```

After pushing, navigate to your repository on GitHub. You will see a prompt to **"Compare & pull request"** - click it to open the PR form.

### Steps to Create a PR on GitHub

1. Go to your forked repository on GitHub.
2. Click **"Compare & pull request"** next to your pushed branch.
3. Set the **base branch** (e.g., `main`) and **compare branch** (e.g., `feature-branch`).
4. Fill in a clear title and description (see best practices below).
5. Assign **reviewers**, **labels**, and a **milestone** if applicable.
6. Click **"Create Pull Request"**.
7. Once the reviewer approves, click **"Merge pull request"** and confirm.

### Best Practices for Writing PR Descriptions

A good PR description should answer three questions: *What* changed, *Why* it changed, and *How* to test it.

```
## What
Added a new feature to display user profile information on the dashboard.

## Why
Resolves #42 — users had no way to view their profile from the main screen.

## How to Test
1. Log in with any test account.
2. Navigate to /dashboard.
3. Confirm the profile card appears in the top-right corner.

```

**Additional tips:**
- Keep PRs small and focused - one feature or fix per PR.
- Reference related issues using `Closes #<issue-number>` to auto-close them on merge.
- Use a checklist for reviewers: `- [ ] Tests pass`, `- [ ] Docs updated`.

### Handling Review Comments

- **Address every comment** - either make the change or explain why you disagree.
- Reply with **"Done"** or **"Fixed in `<commit>`"** once a change is applied.
- Use **"Request changes"** thoughtfully — it blocks the merge until resolved.
- Re-request review after pushing updates so the reviewer is notified.
- For disagreements, keep discussion professional and focused on the code, not the person.

---

## Task 2: Undoing Changes — Reset & Revert

### Steps

```bash
# Create and commit a file by mistake
echo "Wrong code" >> wrong.txt
git add .
git commit -m "Committed by mistake"

# --- Option 1: Soft Reset ---
# Undoes the commit but keeps changes staged (ready to recommit)
git reset --soft HEAD~1

# --- Option 2: Mixed Reset ---
# Undoes the commit and unstages changes, but keeps the file
git reset --mixed HEAD~1

# --- Option 3: Hard Reset ---
# Undoes the commit AND deletes all changes permanently
git reset --hard HEAD~1

# --- Option 4: Revert ---
# Creates a NEW commit that undoes the last commit (safe for shared branches)
git revert HEAD
```

### Differences Between Reset and Revert

| | `git reset` | `git revert` |
|---|---|---|
| **What it does** | Moves the branch pointer backward, erasing commits | Creates a new commit that undoes changes |
| **History** | Rewrites history | Preserves history |
| **Safe for shared branches?** | ❌ No — breaks others' clones | ✅ Yes — non-destructive |
| **Use case** | Local cleanup before pushing | Undoing changes already pushed to remote |

### Reset Modes Compared

| Mode | Commit removed? | Staging area | Working directory |
|---|---|---|---|
| `--soft` | ✅ Yes | Changes kept staged | Unchanged |
| `--mixed` (default) | ✅ Yes | Changes unstaged | Unchanged |
| `--hard` | ✅ Yes | Cleared | Cleared (⚠️ irreversible) |

### When to Use Each Method

- **`--soft`** — You want to redo the commit with a better message or combine it with more changes.
- **`--mixed`** — You want to review and re-stage changes before recommitting.
- **`--hard`** — You want to completely discard work (use with caution — changes are gone).
- **`revert`** — The commit has already been pushed to a shared branch and you cannot rewrite history.

---

## Task 3: Stashing - Save Work Without Committing

### Steps

```bash
# Modify a file without committing
echo "Temporary Change" >> temp.txt
git add temp.txt

# Stash the staged changes
git stash

# Switch to another branch (working tree is now clean)
git checkout main

# Apply the stash and remove it from the stash list
git stash pop
```

### Additional Useful Stash Commands

```bash
# Save a stash with a descriptive label
git stash save "WIP: working on login validation"

# List all stashes
git stash list

# Apply a specific stash without removing it
git stash apply stash@{2}

# Drop a specific stash after you no longer need it
git stash drop stash@{0}

# Clear all stashes
git stash clear
```

### When to Use `git stash`

- You need to switch branches to fix an urgent bug but your current work is incomplete.
- You want to pull the latest changes from remote without committing half-done work.
- You want to test something quickly on a clean working tree and return to your work later.

### `git stash pop` vs `git stash apply`

| | `git stash pop` | `git stash apply` |
|---|---|---|
| **Applies stash** | ✅ Yes | ✅ Yes |
| **Removes from stash list** | ✅ Yes (after applying) | ❌ No (stash stays) |
| **Best for** | One-time use, done with the stash | Applying the same stash to multiple branches |

**Rule of thumb:** Use `pop` for everyday use. Use `apply` when you need the stashed changes on more than one branch.

---

## Task 4: Cherry-Picking - Selectively Apply Commits

### Steps

```bash
# Find the commit hash you want to apply
git log --oneline

# Example output:
# a3f9c12 Fix: resolve null pointer in payment module
# b1e7d45 Feature: add dark mode toggle
# c9f2a01 Chore: update dependencies

# Apply a specific commit to the current branch
git cherry-pick a3f9c12

# If conflicts arise, resolve them in the affected files, then:
git add <resolved-file>
git cherry-pick --continue

# To abort a cherry-pick in progress:
git cherry-pick --abort
```

### How Cherry-Picking Is Used in Bug Fixes

Cherry-picking is most commonly used when a bug fix is committed to a development or feature branch but needs to be applied to a production hotfix branch immediately — without pulling in all the unfinished development work.

**Example scenario:**
- `main` is the stable production branch.
- `develop` has several in-progress features that aren't ready for release.
- A critical bug fix is committed to `develop` as commit `a3f9c12`.
- Instead of merging all of `develop`, you cherry-pick just that commit onto `main`.

```bash
git checkout main
git cherry-pick a3f9c12
git push origin main
```

### Risks of Cherry-Picking

- **Duplicate commits** - The cherry-picked commit gets a new SHA on the target branch, so it now exists in two places. If branches are later merged, Git may apply the change twice.
- **Missing context** - A commit may depend on earlier commits that aren't on the target branch, causing conflicts or subtle bugs.
- **History fragmentation** - Overuse makes the commit graph harder to read and reason about.
- **Not a substitute for proper branching** - If you're cherry-picking frequently, it may signal that your branching strategy needs revision.

---

## Task 5: Rebasing - Keeping a Clean Commit History

### Steps

```bash
# Fetch the latest state of the remote
git fetch origin main

# Rebase the current feature branch on top of the latest main
git rebase origin/main

# If there are conflicts, resolve them in the affected files, then:
git add <resolved-file>
git rebase --continue

# To skip a conflicting commit (use with caution):
git rebase --skip

# To abort and return to the state before rebasing:
git rebase --abort
```

### Difference Between Merge and Rebase

| | `git merge` | `git rebase` |
|---|---|---|
| **History** | Preserves all original commits; creates a merge commit | Rewrites commits on top of the target; linear history |
| **Commit graph** | Non-linear (branches visible) | Linear (clean, straight line) |
| **Safe for shared branches?** | ✅ Yes | ❌ No — never rebase a public/shared branch |
| **Merge commit** | Creates one | Does not |
| **Best for** | Long-running branches, preserving context | Short-lived feature branches before merging |

**Visual difference:**

```
# After merge
main: A---B---C---M
                 /
feature:   D---E

# After rebase
main: A---B---C---D'---E'
```

### Best Practices for Rebasing

- **Never rebase a branch others are working on.** Rewriting shared history forces everyone else to re-sync and can cause serious confusion.
- **Rebase before merging** a feature branch into main to keep the history clean and linear.
- **Use interactive rebase** (`git rebase -i`) to squash, reorder, or edit commits before a PR:
  ```bash
  git rebase -i HEAD~3   # interactively edit the last 3 commits
  ```
- **Fetch first, then rebase** — always `git fetch` before `git rebase origin/main` to ensure you're rebasing onto the latest state.
- **Use `--force-with-lease` when pushing after a rebase** — safer than `--force` as it won't overwrite others' pushes:
  ```bash
  git push --force-with-lease origin feature-branch
  ```

---

## Task 6: Branching Strategies Used in Companies

### Simulating a Git Workflow

```bash
git branch feature-1
git branch hotfix-1
git checkout feature-1

# Simulate feature work
echo "Feature 1 work" >> feature1.txt
git add .
git commit -m "feat: implement feature 1"

# Simulate a hotfix
git checkout main
git checkout hotfix-1
echo "Critical fix" >> hotfix.txt
git add .
git commit -m "fix: patch critical production bug"
```

---

### Git Flow

Git Flow uses multiple long-lived branches with strict rules about what each one represents.

**Branch structure:**

| Branch | Purpose |
|---|---|
| `main` | Production-ready code only |
| `develop` | Integration branch for features |
| `feature/*` | Individual features branched from `develop` |
| `release/*` | Stabilization before merging to `main` |
| `hotfix/*` | Emergency production fixes branched from `main` |

**Workflow:**
1. Feature branches are created from `develop` and merged back into `develop` via PR.
2. When ready for release, a `release/*` branch is cut from `develop` for final testing.
3. The release branch merges into both `main` and `develop`.
4. Hotfixes branch from `main`, and merge back into both `main` and `develop`.

**Best for:** Projects with scheduled versioned releases (e.g., mobile apps, packaged software).

---

### GitHub Flow

A simpler model with only two concepts: `main` and short-lived feature branches.

**Workflow:**
1. Create a branch from `main` for any change — feature, fix, or experiment.
2. Make commits, push, open a PR.
3. Discuss and review.
4. Deploy from the branch to staging for final validation.
5. Merge into `main` — this is a deploy.

**Best for:** Teams deploying continuously (SaaS products, web apps). Requires strong automated testing.

---

### Trunk-Based Development (TBD)

Everyone commits directly to `main` (the "trunk") or uses very short-lived branches (< 1 day). Feature flags are used to hide incomplete features in production.

**Workflow:**
1. Developers commit small, frequent changes directly to `main` or via branches that live less than a day.
2. CI runs on every commit and must pass before merge.
3. Incomplete features are hidden behind feature flags rather than living in branches.

**Best for:** High-velocity teams with mature CI/CD pipelines and strong automated test coverage.

---

### Which Strategy Is Best for DevOps and CI/CD?

**For most DevOps teams: GitHub Flow or Trunk-Based Development.**

Git Flow introduces overhead (long-lived branches, complex merge rules) that conflicts with the DevOps principle of frequent, small deployments. The more often you deploy, the less value Git Flow's `release` branch adds.

| Criteria | Git Flow | GitHub Flow | Trunk-Based |
|---|---|---|---|
| **Deployment frequency** | Low–medium (versioned releases) | High (continuous delivery) | Very high (continuous integration) |
| **Team size** | Any | Small–medium | Medium–large with strong CI |
| **CI/CD fit** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Complexity** | High | Low | Low–medium |
| **Feature flags needed** | No | Sometimes | Yes |

**Recommendation:** Start with **GitHub Flow** — it's simple, PR-based, and maps directly to CI/CD pipelines. Migrate toward **Trunk-Based Development** as your team grows and test automation matures.

---

## Summary of All Git Commands Used

| Command | Purpose |
|---|---|
| `git clone <url>` | Clone a remote repository locally |
| `git checkout -b <branch>` | Create and switch to a new branch |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Commit staged changes |
| `git push origin <branch>` | Push branch to remote |
| `git reset --soft HEAD~1` | Undo commit, keep changes staged |
| `git reset --mixed HEAD~1` | Undo commit, unstage changes |
| `git reset --hard HEAD~1` | Undo commit and discard all changes |
| `git revert HEAD` | Create a new commit that undoes the last one |
| `git stash` | Temporarily save uncommitted work |
| `git stash pop` | Restore and remove the latest stash |
| `git stash apply` | Restore a stash without removing it |
| `git log --oneline` | View compact commit history |
| `git cherry-pick <hash>` | Apply a specific commit to current branch |
| `git cherry-pick --continue` | Continue after resolving cherry-pick conflict |
| `git fetch origin main` | Fetch latest changes from remote |
| `git rebase origin/main` | Rebase current branch onto latest main |
| `git rebase --continue` | Continue after resolving rebase conflict |
| `git rebase -i HEAD~N` | Interactive rebase of last N commits |
| `git branch <name>` | Create a new branch |
| `git checkout <branch>` | Switch to an existing branch |

---
