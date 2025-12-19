# Interactive Rebase for Cleaner History

Use `git rebase -i` to squash, edit, or reorder commits before pushing.

## Start Interactive Rebase

```bash
# Rebase last 3 commits
git rebase -i HEAD~3

# Rebase from a specific commit
git rebase -i abc1234
```

## Available Commands

```
pick   = keep commit as-is
reword = keep commit, edit message
edit   = pause to amend commit
squash = merge into previous commit (keep message)
fixup  = merge into previous commit (discard message)
drop   = delete commit
```

## Example: Squash Commits

```bash
# Before rebase (shown in editor):
pick abc1234 Add user model
pick def5678 Fix typo in user model
pick ghi9012 Add validation to user model

# Change to:
pick abc1234 Add user model
fixup def5678 Fix typo in user model
squash ghi9012 Add validation to user model

# Result: 1 clean commit with combined changes
```

## Editing a Commit

```bash
# In rebase editor, change pick to edit:
edit abc1234 Add user model

# Git pauses at that commit
git add .
git commit --amend
git rebase --continue
```

## Abort if Something Goes Wrong

```bash
git rebase --abort
```

## Golden Rules

- ⚠️ Never rebase pushed commits (unless force push is OK)
- ✅ Use for local cleanup before PR
- ✅ Squash "fix typo" commits
- ✅ Reorder for logical flow

---

📅 *Learned: 2024*
🏷️ *Tags: Git, Rebase, Clean History*
