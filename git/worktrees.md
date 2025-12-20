# Git Worktrees for Parallel Development

<div align="center">

![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![Worktrees](https://img.shields.io/badge/Feature-Worktrees-blue?style=for-the-badge)

*Worktrees let you work on multiple branches simultaneously in different directories.*

![Git](https://upload.wikimedia.org/wikipedia/commons/e/e0/Git-logo.svg)

</div>

## The Problem

```bash
# You're working on feature-a
# Suddenly need to fix a bug on main
# Options:
# 1. Stash changes (messy)
# 2. Commit WIP (ugly history)
# 3. Clone repo again (slow, uses disk space)
```

## The Solution: Worktrees

```bash
# Create a worktree for hotfix
git worktree add ../hotfix main

# Now you have:
# /project          <- feature-a branch
# /hotfix           <- main branch

# Work on hotfix without touching feature-a!
```

## Basic Commands

```bash
# Add worktree for existing branch
git worktree add ../bugfix bugfix-branch

# Add worktree with new branch
git worktree add -b new-feature ../new-feature main

# List all worktrees
git worktree list

# Remove worktree
git worktree remove ../hotfix

# Prune stale worktrees
git worktree prune
```

## Use Cases

### 1. Hotfixes While Working on Features
```bash
git worktree add ../hotfix main
cd ../hotfix
# fix bug, commit, push
cd ../project
git worktree remove ../hotfix
```

### 2. Comparing Branches Side by Side
```bash
git worktree add ../v1 release/v1
git worktree add ../v2 release/v2
# Open both in different IDE windows
```

### 3. Running Tests on Different Branches
```bash
git worktree add ../test-main main
cd ../test-main && npm test
```

## Benefits

- No stashing needed
- Faster than cloning
- Shared .git directory (saves disk space)
- Clean mental separation

---

*Learned: December 20, 2025*
*Tags: Git, Worktrees, Productivity*
