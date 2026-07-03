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
- Always branch off `uat` for new work (except urgent hotfixes — see Section 7)
- Open PRs for every change, with a clear title and description
- Write/update tests where applicable
- Request review from the team lead or another developer before merging
- Keep branches small and focused — one feature or fix per branch

---

## 4. Project Init (Day One Setup)

```cmd
git init
git branch -M main
git commit -m "Initial commit"
git remote add origin https://github.com/yourcompany/server.git
git push -u origin main
git checkout -b uat
git push -u origin uat
```

Repeat for each repo (`crm`, `website`, `server`). Then set branch protection on `main` and `uat` (see Section 9) before anyone starts pushing work.

---

## 5. Branching Model

```
main      → production (live)
uat       → UAT / testing environment
feature/* → new functionality, branches off uat
fix/*     → non-urgent bug fixes, branches off uat
hotfix/*  → urgent production bug fixes, branches off main
chore/*   → config, docs, cleanup, branches off uat
```

### Branch naming convention

| Prefix | Use case | Branches from |
|---|---|---|
| `feature/xyz` | New functionality | `uat` |
| `fix/xyz` | Non-urgent bug fix | `uat` |
| `hotfix/xyz` | Urgent production bug | `main` |
| `chore/xyz` | Docs, config, cleanup | `uat` |

---

## 6. Complete Flow Reference

Three views of the same process: the full end-to-end flow, then the same flow split by who does which part.

### 6.1 Full Flow (start to production, step by step)

```bash
# 1. Sync with latest before starting work
git checkout uat
git pull origin uat

# 2. Create a new branch
git checkout -b feature/branch-name

# 3. Stage, commit, push
git add .
git commit -m "Clear, descriptive message"
git push -u origin feature/branch-name

# 4. Open a PR into uat
gh pr create --base uat --head feature/branch-name --title "Title" --body "Description"

# 5. Merge the PR (squash + delete branch)
gh pr merge feature/branch-name --squash --delete-branch

# 6. Test it on the UAT environment
#    (manual step — verify it actually works before going further)

# 7. Promote uat into main
git checkout main
git pull origin main
git merge uat
git push origin main
#    (this triggers the production deploy)

# 8. Tag the release
git tag -a v1.2.0 -m "Release notes here"
git push origin v1.2.0

# 9. Sync main back into uat (keeps both branches aligned)
git checkout uat
git pull origin uat
git merge main
git push origin uat
```

### 6.2 Team Lead Flow

The team lead owns everything from PR review onward — not the day-to-day coding branches.

```bash
# Review a developer's PR (on GitHub, or via CLI)
gh pr view feature/branch-name
gh pr diff feature/branch-name

# Approve
gh pr review feature/branch-name --approve

# Merge once approved + CI passed
gh pr merge feature/branch-name --squash --delete-branch

# --- After enough features are tested on UAT, promote to production ---
git checkout main
git pull origin main
git merge uat
git push origin main

# Tag the release
git tag -a v1.2.0 -m "Release notes here"
git push origin v1.2.0

# Sync main back into uat
git checkout uat
git pull origin uat
git merge main
git push origin uat

# --- If a production hotfix was merged directly into main ---
git checkout uat
git pull origin uat
git merge main
git push origin uat
```

**Team lead also owns (not repeated per release):**
- Setting branch protection rules on `main` and `uat`
- Managing GitHub Teams / repo access
- Managing GitHub Secrets and Variables
- Approving/merging `hotfix/*` PRs into `main`

### 6.3 Developer Flow

Developers only ever touch `uat` as a starting point (or `main` for a genuine hotfix) — they never merge into `main` themselves and never promote a release.

```bash
# 1. Sync with latest before starting work
git checkout uat
git pull origin uat

# 2. Create a new branch
git checkout -b feature/branch-name

# 3. Work, then stage and commit only what you intend to change
git add src/path/to/file.js
git commit -m "Clear, descriptive message"

# 4. Push branch
git push -u origin feature/branch-name

# 5. Open a PR into uat
gh pr create --base uat --head feature/branch-name --title "Title" --body "Description"

# 6. Respond to review comments if requested, then push updates
git add .
git commit -m "Address review feedback"
git push

# Done — team lead (or another approved reviewer) takes it from here
```

**Developers do not:**
- Push directly to `main` or `uat`
- Merge their own PRs without approval
- Tag releases
- Promote `uat` to `main`

---

## 7. Bug Handling

### 7.1 Bug found in UAT (not live yet)

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

### 7.2 Bug found in production (urgent — hotfix)

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

### 7.3 Decision table

| Where bug is found | Branch from | Merge into | Then sync to |
|---|---|---|---|
| UAT / testing | `uat` | `uat` | flows to `main` naturally later |
| Production (urgent) | `main` | `main` | `uat` (mandatory — prevents drift) |

---

## 8. Pull Request Rules

- Every PR requires at least **1 approval** before merging
- CI checks (tests/build) must pass before merge is allowed
- Use **squash merge** to keep history clean (one commit per feature/fix)
- Delete the branch after merging — its job is done
- PR description should explain **what** changed and **why**, not just restate the title

---

## 9. Branch Protection Rules (set by Team Lead)

Applied to both `main` and `uat` in repo Settings → Branches:

- Require pull request before merging
- Require at least 1 approval
- Require status checks (CI) to pass before merging
- Block force-pushes
- Block direct pushes (even from admins, where possible)

CLI equivalent:

```bash
gh api repos/yourcompany/server/branches/main/protection \
  --method PUT \
  --field required_pull_request_reviews[required_approving_review_count]=1 \
  --field enforce_admins=true \
  --field required_status_checks=null \
  --field restrictions=null
```

Repeat for `uat`, and for each repo (`crm`, `website`, `server`).

---

## 10. CI/CD (GitHub Actions)

Each repo has its own workflow file at `.github/workflows/deploy.yml`:

- Push/merge to `uat` → deploys automatically to the **UAT server**
- Push/merge to `main` → deploys automatically to the **production server**

Secrets (DB credentials, API keys, deploy tokens) are stored in **GitHub Actions Secrets** — never committed to the repo. Each repo includes a `.env.example` file documenting required variables without real values.

---

## 11. Local Secrets / Environment Variables

GitHub Secrets and Variables only inject into GitHub Actions — they are **not** automatically available on a developer's local machine. Options for local use:

- **Manual `.env`** (simplest): commit a `.env.example` template, share real values via a password manager, each dev copies into their own local `.env`
- **Doppler** (free tier: 5 users, 3 projects): syncs secrets to both local dev and CI/CD
  ```bash
  doppler login
  doppler setup
  doppler run -- npm run dev
  ```
- **1Password CLI / Bitwarden CLI**: works if the company already uses one of these

---

## 12. Naming & Commit Message Conventions

- Branch names: lowercase, hyphen-separated, prefixed (`feature/`, `fix/`, `hotfix/`, `chore/`)
- Commit messages: short, present tense, descriptive
  - Good: `Fix invoice total calculation`
  - Avoid: `fixed stuff`, `update`, `wip`

---

## 13. Onboarding Checklist (New Developer)

- [ ] Added to the GitHub Organization and correct Team (`frontend` / `backend`)
- [ ] `git config --global user.name` and `user.email` set
- [ ] GitHub CLI (`gh`) installed and authenticated (`gh auth login`)
- [ ] Repo(s) cloned locally
- [ ] `.env.example` reviewed and local `.env` created with real values
- [ ] Confirmed access to UAT environment for testing
- [ ] Read this document

---

## 14. Golden Rules

1. Never push directly to `main` or `uat`.
2. Every change goes through a Pull Request.
3. Hotfixes go to `main` first, then must be merged back into `uat`.
4. Keep branches small and short-lived.
5. Don't commit secrets — ever.
