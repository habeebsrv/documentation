# Company GitHub Workflow — Team Guide

**Audience:** Team Lead + Developers
**Projects covered:** CRM, Website, Server

---

## 1. Overview

This document defines how our team uses GitHub across all projects. Everyone — team lead and developers — follows the same structure so code stays safe, reviewed, and easy to deploy.

**Core principle:** `main` is always live/production. `uat` is always testing. Nobody pushes directly to either — everything goes through a Pull Request (PR).

---

## 2. Organization Structure

- All repos live under the company GitHub Organization: `github.com/yourcompany`
- Repos (one per project):
  - `yourcompany/crm`
  - `yourcompany/website`
  - `yourcompany/server`
- Access is managed through **GitHub Teams**, not individual permissions:
  - `@yourcompany/backend` → write access to `server`
  - `@yourcompany/frontend` → write access to `website`, `crm`
  - `@yourcompany/admins` → admin access (team lead only)

---

## 3. Roles & Responsibilities

### Team Lead

- Owns `main` and `uat` branch protection rules
- Reviews and approves PRs into `main` (production)
- Manages GitHub Teams and repo access
- Manages secrets (API keys, deploy tokens) in GitHub Actions
- Decides when `uat` gets promoted to `main`
- Handles or approves hotfixes to production
- Keeps `uat` and `main` in sync after hotfixes

### Developers

- Never push directly to `main` or `uat`
- Always branch off `uat` for new work (except urgent hotfixes — see Section 6)
- Open PRs for every change, with a clear title and description
- Write/update tests where applicable
- Request review from the team lead or another developer before merging
- Keep branches small and focused — one feature or fix per branch

---

## 4. Branching Model

```
main      → production (live)
uat       → UAT / testing environment
feature/* → new functionality, branches off uat
fix/*     → non-urgent bug fixes, branches off uat
hotfix/*  → urgent production bug fixes, branches off main
chore/*   → config, docs, cleanup, branches off uat
```

### Branch naming convention

| Prefix          | Use case              | Branches from |
| --------------- | --------------------- | ------------- |
| `feature/xyz` | New functionality     | `uat`       |
| `fix/xyz`     | Non-urgent bug fix    | `uat`       |
| `hotfix/xyz`  | Urgent production bug | `main`      |
| `chore/xyz`   | Docs, config, cleanup | `uat`       |

---

## 5. Standard Development Flow (Feature or Non-Urgent Fix)

```bash
# 1. Start from latest uat
git checkout uat
git pull origin uat

# 2. Create a branch
git checkout -b feature/add-invoice-export

# 3. Work, then commit
git add .
git commit -m "Add invoice export to CSV"

# 4. Push branch
git push -u origin feature/add-invoice-export

# 5. Open a PR into uat
gh pr create --base uat --head feature/add-invoice-export \
  --title "Add invoice export" \
  --body "Adds CSV export button on invoice page"

# 6. After approval + CI passes, merge and delete branch
gh pr merge feature/add-invoice-export --squash --delete-branch
```

Once tested and confirmed working on the UAT environment, the team lead promotes `uat` into `main`:

```bash
git checkout main
git pull origin main
git merge uat
git push origin main
```

This triggers deployment to production.

---

## 6. Bug Handling

### 6.1 Bug found in UAT (not live yet)

Treat like a normal fix — branch off `uat`:

```bash
git checkout uat
git pull origin uat
git checkout -b fix/invoice-total-wrong

git add .
git commit -m "Fix incorrect invoice total calculation"
git push -u origin fix/invoice-total-wrong

gh pr create --base uat --head fix/invoice-total-wrong \
  --title "Fix invoice total bug"
```

### 6.2 Bug found in production (urgent — hotfix)

Branch directly off `main`, fix, deploy immediately:

```bash
git checkout main
git pull origin main
git checkout -b hotfix/login-crash

git add .
git commit -m "Fix login crash on empty password"
git push -u origin hotfix/login-crash

gh pr create --base main --head hotfix/login-crash \
  --title "Hotfix: login crash"
```

Merge → auto-deploys to production.

**Required follow-up — sync the fix back into `uat`** so it doesn't reappear later:

```bash
git checkout uat
git pull origin uat
git merge main
git push origin uat
```

### 6.3 Decision table

| Where bug is found  | Branch from | Merge into | Then sync to                          |
| ------------------- | ----------- | ---------- | ------------------------------------- |
| UAT / testing       | `uat`     | `uat`    | flows to`main` naturally later      |
| Production (urgent) | `main`    | `main`   | `uat` (mandatory — prevents drift) |

---

## 7. Pull Request Rules

- Every PR requires at least **1 approval** before merging
- CI checks (tests/build) must pass before merge is allowed
- Use **squash merge** to keep history clean (one commit per feature/fix)
- Delete the branch after merging — its job is done
- PR description should explain **what** changed and **why**, not just restate the title

---

## 8. Branch Protection Rules (set by Team Lead)

Applied to both `main` and `uat` in repo Settings → Branches:

- Require pull request before merging
- Require at least 1 approval
- Require status checks (CI) to pass before merging
- Block force-pushes
- Block direct pushes (even from admins, where possible)

---

## 9. CI/CD (GitHub Actions)

Each repo has its own workflow file at `.github/workflows/deploy.yml`:

- Push/merge to `uat` → deploys automatically to the **UAT server**
- Push/merge to `main` → deploys automatically to the **production server**

Secrets (DB credentials, API keys, deploy tokens) are stored in **GitHub Actions Secrets** — never committed to the repo. Each repo includes a `.env.example` file documenting required variables without real values.

---

## 10. Everyday Commands Cheat Sheet

```bash
# Sync with latest before starting work
git checkout uat
git pull origin uat

# Create a new branch
git checkout -b feature/branch-name

# Stage, commit, push
git add .
git commit -m "Clear, descriptive message"
git push -u origin feature/branch-name

# Open a PR (GitHub CLI)
gh pr create --base uat --head feature/branch-name --title "Title" --body "Description"

# Merge a PR (squash + delete branch)
gh pr merge feature/branch-name --squash --delete-branch

# Tag a production release
git checkout main
git tag -a v1.2.0 -m "Release notes here"
git push origin v1.2.0
```

---

## 11. Naming & Commit Message Conventions

- Branch names: lowercase, hyphen-separated, prefixed (`feature/`, `fix/`, `hotfix/`, `chore/`)
- Commit messages: short, present tense, descriptive
  - Good: `Fix invoice total calculation`
  - Avoid: `fixed stuff`, `update`, `wip`

---

## 12. Onboarding Checklist (New Developer)

- [ ] Added to the GitHub Organization and correct Team (`frontend` / `backend`)
- [ ] `git config --global user.name` and `user.email` set
- [ ] GitHub CLI (`gh`) installed and authenticated (`gh auth login`)
- [ ] Repo(s) cloned locally
- [ ] `.env.example` reviewed and local `.env` created with real values
- [ ] Confirmed access to UAT environment for testing
- [ ] Read this document

---

## 13. Golden Rules

1. Never push directly to `main` or `uat`.
2. Every change goes through a Pull Request.
3. Hotfixes go to `main` first, then must be merged back into `uat`.
4. Keep branches small and short-lived.
5. Don't commit secrets — ever.
