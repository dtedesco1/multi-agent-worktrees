# Git Worktree Multi‑Agent **Setup Guide** (v1.3)

> **Audience:** Human maintainers boot‑strapping a repository where many AI or human agents work concurrently.
> **Goal:** Provide a bullet‑proof, repeatable recipe to create one primary repo plus *N* lightweight worktrees—one per agent—without clone‑bloat or branch collisions.

---

## 0 · At‑a‑Glance

```text
repo-root/
├─ project-main/               ← integration worktree  (branch: main)
├─ project-scripts/            ← helper scripts & shared hooks
└─ project-<agent-id>/         ← one folder per agent, created on demand
```

### Disk‑Space Reality Check

```
Total ≈ 1 × <object store size>  +  N × <size of a full checkout>
```

* All worktrees share `.git/objects` → huge saving compared with *N* clones.
* Each worktree still needs a complete working copy of tracked files. Expect \~1.2× repo size for a *few* worktrees; growth becomes roughly linear with **N**.

---

## 1 · Prerequisites

| Item                         | Why                                            |
| ---------------------------- | ---------------------------------------------- |
| Git ≥ 2.40                   | Stable `git worktree`, sparse‑checkout updates |
| POSIX shell / Git Bash       | For helper scripts                             |
| Remote (GitHub/Lab)          | CI, review, backup                             |
| Disk space per formula above | Avoid surprises                                |

---

## 2 · Initial Repository Bootstrap

```bash
cd /path/to/repo-root

git clone git@github.com:org/project.git project-main
cd project-main

# Ensure default branch ‘main’ exists & tracks remote
if git ls-remote --exit-code --heads origin main >/dev/null 2>&1; then
  git checkout main && git pull origin main
else
  git switch -c main
  git push -u origin main        # establish tracking
fi

# ---------------- Shared hooks ----------------
# 1. Create committed hooks directory
mkdir -p ../project-scripts/.githooks
#   (Add your hooks, e.g. cp path/to/pre-commit ../project-scripts/.githooks/)

# 2. Point Git to it once; all worktrees inherit this
git config core.hooksPath ../project-scripts/.githooks

# ---------------- Helper scripts -------------
mkdir -p ../project-scripts
# Create scripts new-agent, remove-agent, gc-all (Section 3) inside this folder.

# ---------------- First commit ---------------
cat <<'EOT' > README.md
Multi‑agent worktree repo. See project-scripts/README.md for helper utilities.
EOT

git add README.md ../project-scripts
# Add .gitignore from repo‑root (see Appendix)
git add ../.gitignore

git commit -m "chore: bootstrap multi-agent worktree layout"
git push
```

---

## 3 · Helper Scripts  (`project-scripts/`)

Create each file below and `chmod +x` it. A `README.md` inside `project-scripts/` should describe usage.

### 3.1 `new-agent`

```bash
#!/usr/bin/env bash
set -euo pipefail
AGENT=${1:?Usage: new-agent <agent-id>}
BRANCH="agent/${AGENT}"
WORKTREE_PATH="../project-${AGENT}"

# Safety check: refuse if path already exists
if [ -d "${WORKTREE_PATH}" ]; then
  echo "Error: Worktree path ${WORKTREE_PATH} already exists." >&2
  echo "Use './remove-agent ${AGENT}' first or pick a new AGENT_ID." >&2
  exit 1
fi

cd ../project-main

git switch main
# Reset or create fresh branch from current main
if git show-ref --quiet --verify "refs/heads/${BRANCH}"; then
  git branch -D "${BRANCH}"
fi
git branch "${BRANCH}"

git worktree add "${WORKTREE_PATH}" "${BRANCH}"

echo "✅  ${WORKTREE_PATH} on ${BRANCH} ready"
```

### 3.2 `remove-agent`

```bash
#!/usr/bin/env bash
set -euo pipefail
AGENT=${1:?Usage: remove-agent <agent-id>}
BRANCH="agent/${AGENT}"
WORKTREE_PATH="../project-${AGENT}"

cd ../project-main

git worktree remove -f "${WORKTREE_PATH}" 2>/dev/null || true
git branch  -D "${BRANCH}" 2>/dev/null || true
git worktree prune -v
```

### 3.3 `gc-all`  (⚠ *Dry‑Run by default*)

```bash
#!/usr/bin/env bash
# Reports worktrees idle >60 days and suggests removal commands.
set -euo pipefail
cd ../project-main

git fetch --all --prune
git worktree prune -v

echo "🔎 Scanning for worktrees idle >60 days…"
find .. -maxdepth 1 -type d -name "project-*" -mtime +60 | while read -r dir; do
  id=$(basename "$dir" | sed 's/^project-//')
  printf "⚠️  Stale: %s  →  run: ./remove-agent \"%s\"\n" "$dir" "$id"
done

git gc --aggressive --prune=now
```

---

## 4 · On‑Boarding New Agents

```bash
cd repo-root/project-scripts
./new-agent alice       # repeat per contributor
```

Each agent opens *only* their folder (`project-alice/`), already on branch `agent/alice`, inheriting shared hooks.

---

## 5 · Daily Ops & Integration

| Role           | Task                               | Action                                                                      |
| -------------- | ---------------------------------- | --------------------------------------------------------------------------- |
| **Agent**      | Pull → Rebase → Commit → Push loop | `git fetch origin && git rebase origin/main && git push --force-with-lease` |
| **Agent**      | Draft PR → tag maintainer          | Standard GitHub/GitLab flow                                                 |
| **Integrator** | Review & merge approved PRs        | Click *Merge*, or locally `git merge --no-ff agent/<id>` after review       |
| **Integrator** | Retire finished sandbox            | `./remove-agent <id>`                                                       |

*Policy:* Every active branch **must** rebase onto `main` at least once per 24 h. CI fails builds more than 24 h out‑of‑date.

---

## 6 · CI / Governance Essentials

* Branch pattern `agent/**` → full test + lint matrix.
* Required status checks before merge.
* Dependabot/Renovate runs on `main`.
* Stale‑PR bot pings after 48 h inactivity.

---

## 7 · Troubleshooting & Recovery

| Symptom                  | Likely Cause                 | Resolution                                                  |
| ------------------------ | ---------------------------- | ----------------------------------------------------------- |
| `fatal: cannot lock ref` | Branch already checked out   | Decide owner or `./remove-agent` old worktree.              |
| IDE “not a Git repo”     | Project opened at wrong path | Open `project-<id>/` (worktree root) or update IDE version. |
| Endless merge conflicts  | Long‑lived branch            | Rebase earlier & split work.                                |
| Lost commits after GC    | Commit not pushed/tagged     | `git reflog` then `git cherry-pick <hash>`.                 |

---

## 8 · House‑Keeping Schedule

| Interval | Action                                      |
| -------- | ------------------------------------------- |
| Daily    | Verify all branches <24 h behind `main`.    |
| Weekly   | Merge green PRs, tag release, run `gc-all`. |
| Monthly  | Audit stale worktrees & rotate secrets.     |

---

## 9 · Appendix & Resources

* Anthropic Claude‑Code best practices – [https://www.anthropic.com/engineering/claude-code-best-practices](https://www.anthropic.com/engineering/claude-code-best-practices)
* `git help worktree`, `git sparse-checkout`

### Sample `.gitignore` (commit at repo‑root)

```gitignore
# ---- IDE / Editor ----
.idea/
.vscode/
*.swp
.DS_Store

# ---- Build & dependency dirs (apply in *any* worktree) ----
**/node_modules/
**/target/
**/build/
**/dist/
**/out/
**/venv/
**/.venv/
**/__pycache__/

# ---- Logs ----
*.log

# ---- OS stuff ----
Thumbs.db
```

---

> **Mantra:** *One object store · Many sandboxes · Pull, Rebase, Push.*
