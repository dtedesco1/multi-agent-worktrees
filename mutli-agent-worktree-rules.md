# 📜 Agent Worktree Rules (v1.1)

> **Audience:** Any AI or human agent operating in a dedicated worktree.

---

## 0 · Environment Variables

| Var             | Must equal                                         |
| --------------- | -------------------------------------------------- |
| `AGENT_ID`      | your unique identifier (`alice`, `cursor-ui-3`, …) |
| `WORKTREE_ROOT` | absolute path ending in `project-${AGENT_ID}`      |

The repository root for *all* commands is `WORKTREE_ROOT`.

---

## 1 · Branch Ownership

1. Your branch **must** be `agent/${AGENT_ID}`.
2. Do **not** create or modify any other branch.
3. Your branch may be force‑rebased or deleted by maintainers without notice.

---

## 2 · Allowed Git Commands

```bash
# Inspect
git status                   # check state

# Sync & update
git fetch origin             # update refs
git rebase origin/main       # bring branch current

# Commit & publish
git add <files>
git commit -m "<msg>"
git push --force-with-lease  # required after rebase
```

Anything else—ask a human first.

---

## 3 · Workflow

1. **Sync**  – every ≤4 h or when CI fails:

   ```bash
   git fetch origin
   git rebase origin/main
   git push --force-with-lease
   ```

2. **Develop** – make changes inside `WORKTREE_ROOT` only.
3. **Commit small** logical units; run tests & linter locally.
4. **Push** to trigger CI.
5. **Create Draft PR** to `main` and tag your maintainer when checklist ✓.

---

## 4 · Integration Readiness Checklist

* [ ] Rebased onto `origin/main` within last 4 h.
* [ ] `npm test` / `pytest` / etc. **green** locally.
* [ ] Linter **clean**.
* [ ] CI pipeline **passes**.
* [ ] Draft PR created; reviewer assigned.

---

## 5 · Prohibited Actions

* `git merge`, `git switch`, `git checkout` to any branch but your own.
* Force‑pushing *any* branch **other** than `agent/${AGENT_ID}`.
* Deleting branches without maintainer approval.
* Editing files outside `WORKTREE_ROOT`.

---

## 6 · Failsafe

If repeated Git errors occur:

1. Output exactly: `ERROR: <short description>`
2. Halt further edits.
3. Await maintainer instructions.

---

## 7 · Mantra

**“Stay in your sandbox · Pull, Rebase, Push · Ask when unsure.”**
