# Week 4 Challenge — Git & GitHub Basics Challenge

## Overview

This document covers all tasks completed for the Week 4 Git & GitHub challenge, including commands used, explanations, and a discussion on branching strategies.

---

## Task 1: Fork and Clone the Repository

### Steps
1. Forked the repository from the original source to my GitHub account.
2. Cloned the forked repository locally using HTTPS:

```bash
git clone https://github.com/<your-username>/90DaysOfDevOps.git
cd 2025/git/01_Git_and_Github_Basics
```

---

## Task 2: Initialize a Local Repository and Create a File

### Steps

```bash
# Create and navigate to the challenge directory
mkdir week-4-challenge
cd week-4-challenge

# Initialize a new git repository
git init

# Create info.txt with introductory content
echo "Name: <Your Name>
Introduction: I am learning DevOps through the 90DaysOfDevOps challenge.
Goal: Master Git, GitHub, CI/CD, and cloud tools." > info.txt

# Stage the file
git add info.txt

# Commit with a descriptive message
git commit -m "Initial commit: Add info.txt with introductory content"
```

### What Happened
- `git init` created a hidden `.git` directory, turning the folder into a tracked Git repository.
- `git add info.txt` moved the file to the staging area (index).
- `git commit` saved a snapshot of the staged changes to the local repository history.

---

## Task 3: Configure Remote URL with PAT and Push/Pull

### Steps

```bash
# Add remote origin with PAT embedded in the URL
git remote add origin https://<your-username>:<your-PAT>@github.com/<your-username>/90DaysOfDevOps.git

# If origin already existed, update it
git remote set-url origin https://<your-username>:<your-PAT>@github.com/<your-username>/90DaysOfDevOps.git

# Push the main branch and set upstream tracking
git push -u origin main

# Pull to verify the remote is correctly configured
git pull origin main
```

> ⚠️ **Security Note:** Embedding a PAT directly in the remote URL is done here for learning purposes only. In production, use a credential manager or SSH keys instead.

---

## Task 4: Explore Commit History

### Steps

```bash
git log
```

### Sample Output

```
commit a1b2c3d4e5f6... (HEAD -> main, origin/main)
Author: Your Name <you@example.com>
Date:   Mon Apr 05 2026

    Initial commit: Add info.txt with introductory content
```

### What to Note
- **Commit hash** — unique SHA-1 identifier for each snapshot.
- **Author & Date** — who made the change and when.
- **Commit message** — describes what was changed and why.

---

## Task 5: Advanced Branching and Switching

### Steps

```bash
# Create a new feature branch
git branch feature-update

# Switch to the new branch
git switch feature-update
# Alternative:
# git checkout feature-update

# Edit info.txt with additional content
echo "
Skills: Git, Linux, Docker, Kubernetes
Status: Week 4 challenge in progress" >> info.txt

# Stage and commit the changes
git add info.txt
git commit -m "Feature update: Enhance info.txt with additional details"

# Push the branch to remote
git push origin feature-update
```

After pushing, a **Pull Request (PR)** was opened on GitHub to merge `feature-update` into `main`.

---

### (Optional) Simulating a Merge Conflict

```bash
# Create an experimental branch from main
git checkout main
git branch experimental
git switch experimental

# Make a conflicting change to info.txt
echo "Experimental change: Testing conflict resolution." >> info.txt
git add info.txt
git commit -m "Experimental: Add conflicting line to info.txt"

# Switch back to feature-update and merge experimental
git switch feature-update
git merge experimental
```

Git will flag a conflict in `info.txt`. Open the file, manually resolve it by keeping the desired lines, then:

```bash
git add info.txt
git commit -m "Resolve merge conflict between feature-update and experimental"
```

---

## Task 6: Branching Strategies in Collaborative Development

### Why Branching Strategies Matter

In collaborative software development, multiple developers work on the same codebase simultaneously. Without a clear branching strategy, changes can overwrite each other, bugs can be introduced into production, and code reviews become chaotic. A well-defined branching strategy solves all of this.

### Key Benefits

**1. Isolating Features and Bug Fixes**
Each feature or bug fix lives on its own branch, completely isolated from the main codebase. This means an in-progress feature cannot accidentally break the production build. If a feature is abandoned, the branch is simply deleted — no cleanup needed on `main`.

**2. Facilitating Parallel Development**
Multiple team members can work on separate branches at the same time without stepping on each other's work. Developer A can build a login feature while Developer B fixes a payment bug — both independently, both safely.

**3. Reducing Merge Conflicts**
Short-lived, focused branches reduce the chance of conflicts. The longer a branch diverges from `main`, the harder the merge. Strategies like **Git Flow** (with `develop`, `feature/*`, `release/*`, and `hotfix/*` branches) and **trunk-based development** (frequent small merges) both minimize this risk in different ways.

**4. Enabling Effective Code Reviews**
Pull Requests (PRs) tie directly to branches. A PR represents a discrete, reviewable unit of work. Reviewers can comment on specific lines, request changes, and approve or reject before anything reaches `main`. This makes quality control systematic rather than ad hoc.

### Common Branching Models

| Model | Best For |
|---|---|
| **Git Flow** | Versioned software with scheduled releases |
| **GitHub Flow** | Continuous deployment with frequent merges |
| **Trunk-Based Development** | High-velocity teams with strong CI/CD pipelines |

The right model depends on team size, release cadence, and project complexity — but all of them share the same core principle: **protect the main branch and keep it stable at all times**.

---

## Bonus Task: SSH Authentication

### Steps

```bash
# Generate an SSH key pair
ssh-keygen -t ed25519 -C "your-email@example.com"

# Display the public key to copy to GitHub
cat ~/.ssh/id_ed25519.pub
```

1. Copied the public key output.
2. Went to **GitHub → Settings → SSH and GPG Keys → New SSH Key** and pasted the key.

```bash
# Switch remote URL from HTTPS to SSH
git remote set-url origin git@github.com:<your-username>/90DaysOfDevOps.git

# Test the SSH connection
ssh -T git@github.com

# Push using SSH
git push origin feature-update
```

### Why SSH Over HTTPS?
SSH authentication uses a public/private key pair instead of a username and password (or PAT). Once configured, it requires no credentials on every push/pull, making it both more convenient and more secure for daily use.

---

## Summary of Git Commands Used

| Command | Purpose |
|---|---|
| `git init` | Initialize a new local repository |
| `git add <file>` | Stage changes for commit |
| `git commit -m "msg"` | Save a snapshot with a message |
| `git clone <url>` | Copy a remote repository locally |
| `git remote add origin <url>` | Link local repo to a remote |
| `git remote set-url origin <url>` | Update an existing remote URL |
| `git push -u origin main` | Push branch and set upstream |
| `git pull origin main` | Fetch and merge remote changes |
| `git log` | View commit history |
| `git branch <name>` | Create a new branch |
| `git switch <name>` | Switch to a branch |
| `git checkout <name>` | Switch to a branch (older syntax) |
| `git merge <branch>` | Merge a branch into the current one |
| `ssh-keygen` | Generate an SSH key pair |
| `git remote set-url origin git@...` | Switch remote to SSH |

---
