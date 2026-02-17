---
created: 2026-02-16
tags:
  - git
  - vcs
  - workflow
aliases:
  - Git Workflows
  - Git Best Practices
related:
  - CI-CD-Pipeline
  - Testing-Patterns
---

# Git Workflows

> [!SUMMARY] Обзор
> Git workflows, branching strategies, rebase vs merge, cherry-pick, recovery. Практики для командной разработки.

---

## 📚 Branching Strategies

### Git Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  main (production)                                               │
│    ↑                                                             │
│  develop (staging)                                               │
│    ↑        ↑        ↑                                           │
│  feature   release  hotfix                                       │
│                                                                  │
│  feature/* → Новые функции                                      │
│  release/* → Подготовка релиза                                  │
│  hotfix/*  → Срочные исправления production                     │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Feature
git checkout develop
git checkout -b feature/new-feature
# ... work ...
git commit -m "feat: add new feature"
git checkout develop
git merge feature/new-feature
git branch -d feature/new-feature

# Release
git checkout -b release/1.0.0 develop
# ... version bump, final fixes ...
git commit -m "chore: version 1.0.0"
git checkout main
git merge release/1.0.0
git tag v1.0.0
git checkout develop
git merge release/1.0.0
git branch -d release/1.0.0

# Hotfix
git checkout -b hotfix/fix-bug main
# ... fix ...
git commit -m "fix: critical bug fix"
git checkout main
git merge hotfix/fix-bug
git tag v1.0.1
git checkout develop
git merge hotfix/fix-bug
git branch -d hotfix/fix-bug
```

### GitHub Flow (Simpler)

```
┌─────────────────────────────────────────────────────────────────┐
│  main (always deployable)                                        │
│    ↑                                                             │
│  feature/* → PR → Review → Merge → Delete                       │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Create branch from main
git checkout main
git checkout -b feature/new-feature

# Push and create PR
git push -u origin feature/new-feature
# Create PR on GitHub

# After review and merge
git checkout main
git pull origin main
git branch -d feature/new-feature
```

### Trunk-Based Development

```
┌─────────────────────────────────────────────────────────────────┐
│  main (trunk) - developers commit directly                       │
│    ↑                                                             │
│  Short-lived feature branches (< 1 day)                         │
│  Feature flags for incomplete features                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Essential Commands

### Daily Workflow

```bash
# Start work
git checkout main
git pull origin main
git checkout -b feature/my-feature

# During work
git status
git add file.ts
git commit -m "feat: add feature"

# Sync with main
git fetch origin
git rebase origin/main

# Push
git push -u origin feature/my-feature

# After PR merge
git checkout main
git pull origin main
git branch -d feature/my-feature
```

### Rebase vs Merge

```bash
# Merge (preserves history)
git checkout main
git merge feature/my-feature
# Creates merge commit

# Rebase (linear history)
git checkout feature/my-feature
git rebase main
# Rewrites history

# Interactive rebase (squash, edit, reorder)
git rebase -i HEAD~3
# Options: pick, squash, fixup, edit, drop
```

### Recovery Commands

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Undo committed and pushed
git revert HEAD

# Restore deleted file
git checkout HEAD -- file.ts

# Find lost commit
git reflog

# Recover from reflog
git checkout -b recovery-branch <commit-hash>

# Stash changes
git stash
git stash pop
git stash list
git stash drop
```

---

## 🎯 Advanced Techniques

### Cherry-Pick

```bash
# Pick specific commit
git cherry-pick abc123

# Pick range of commits
git cherry-pick abc123..def456

# Pick without committing
git cherry-pick --no-commit abc123
```

### Bisect (Find Bug)

```bash
# Start bisect
git bisect start
git bisect bad           # Current is bad
git bisect good v1.0.0   # v1.0.0 is good

# Git checks out commits, you mark good/bad
git bisect good
git bisect bad

# When found
git bisect reset
```

### Worktree

```bash
# Create additional working directory
git worktree add ../hotfix-fix hotfix/fix-bug

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../hotfix-fix
```

---

## 📝 Commit Conventions

### Conventional Commits

```
feat:     New feature
fix:      Bug fix
docs:     Documentation
style:    Formatting
refactor: Code restructuring
test:     Tests
chore:    Maintenance

Scope (optional):
feat(api): add new endpoint
fix(ui): fix button alignment

Breaking changes:
feat!: change API
feat: change API

BREAKING CHANGE: description
```

### Examples

```bash
# Good commits
git commit -m "feat(auth): add JWT refresh tokens"
git commit -m "fix(api): handle null user in middleware"
git commit -m "docs: update API documentation"
git commit -m "refactor(users): extract validation logic"

# Bad commits
git commit -m "fix"
git commit -m "stuff"
git commit -m "WIP"
git commit -m "asdfasdf"
```

---

## 🔗 Remote Operations

### Multiple Remotes

```bash
# Add remote
git remote add upstream https://github.com/original/repo.git

# List remotes
git remote -v

# Fetch from upstream
git fetch upstream

# Merge upstream changes
git checkout main
git merge upstream/main

# Push to specific remote
git push origin feature/my-feature
git push upstream main
```

### Pull Request Workflow

```bash
# Fork workflow
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git checkout -b feature/my-feature upstream/main

# Work and commit
git add .
git commit -m "feat: add feature"

# Sync with upstream
git fetch upstream
git rebase upstream/main

# Push to your fork
git push origin feature/my-feature

# Create PR on GitHub
```

---

## 🎯 Best Practices

### ✅ Делать

```bash
# 1. Small commits
git add -p  # Interactive staging

# 2. Descriptive messages
git commit -m "feat: add user authentication with JWT"

# 3. Rebase before merge
git fetch origin
git rebase origin/main

# 4. Delete merged branches
git branch -d feature/my-feature

# 5. Use .gitignore
node_modules/
.env
dist/
*.log
```

### ❌ Не делать

```bash
# 1. Don't commit secrets
# Use .env and add to .gitignore

# 2. Don't force push shared branches
git push --force  # ❌ on shared branches

# 3. Don't huge commits
# Split into logical units

# 4. Don't ignore merge conflicts
# Resolve properly

# 5. Don't commit generated files
# dist/, build/, node_modules/
```

---

## 🔗 Связанные заметки

- [[CI-CD-Pipeline]] — Git hooks в CI/CD
- [[Testing-Patterns]] — Тесты перед коммитом

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Commit often** — маленькие коммиты
> 2. **Rebase locally** — линейная история
> 3. **Merge remotely** — PR workflow
> 4. **Delete branches** — чистота
> 5. **Conventional commits** — авто-CHANGELOG

> [!INFO] Aliases
> ```bash
> # ~/.gitconfig
> [alias]
>   co = checkout
>   br = branch
>   st = status
>   ci = commit
>   last = log -1 HEAD
>   unstage = reset HEAD --
>   amend = commit --amend
>   graph = log --oneline --graph --all
> ```
