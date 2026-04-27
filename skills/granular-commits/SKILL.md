---
name: granular-commits
description: >-
  Enforces one logical change per commit (prefer one file when changes are independent),
  Conventional Commits (feat, fix, chore, etc.), and staged-file discipline so unrelated
  edits are never squashed on one message. Use when the user asks to commit, stage,
  "granular" commits, split a diff, or write commit messages; and whenever committing
  multiple file paths in one session.
---

# Granular commits & Conventional Commits

**Non-negotiable** when the user is committing work, unless they explicitly request a **single** commit (or a dedicated squash) for a narrow reason.

Repositories that publish this skill may also have a root `CONTRIBUTING.md` with the same policy in prose — follow **both** this procedure and that file when present.

---

## Before you stage anything

1. Run `git status` and, if needed, `git diff` / per-file `git diff -- <path>`.
2. **Group** paths by logical change (feature A, fix B, docs for C, rename pair, and so on).
3. A **logical change** that touches several paths *may* be one commit only when the paths are **tightly coupled** (e.g. the same feature needs `foo.ts` + `foo.test.ts` in the same change; a skill and its `references/*` for one user-facing edit; a rename in two files that must be atomic). Do **not** use “it’s one PR” to justify one commit across unrelated files.

---

## Staging and committing (repeat per group)

1. Stage only the paths for **one** group: e.g. `git add -p` or `git add -- <path1> <path2>` with **no** unrelated files.
2. Reread `git diff --cached` — if anything unrelated is staged, unstage and split (`git reset -p` or `git reset HEAD -- <path>`).
3. Write a **Conventional Commits** message (see next section) that describes **this staged set only** — not the rest of the working tree.
4. `git commit` (one commit per group).
5. Return to step 1 for the next group until the working tree for this request is clean or only intentionally left dirty.

**Default when changes are independent:** one **file** = one commit (unless a tight-coupling exception applies, as in repo `CONTRIBUTING`).

**Pull requests** may (and often should) contain **many** commits. Do **not** replace several correct commits with one big commit that mixes unrelated files just to “finish the PR in one go.”

---

## Conventional Commits format

```text
<type>[optional scope]: <short description>
```

- **type** (required) — one of: `feat`, `fix`, `refactor`, `chore`, `docs`, `style`, `perf`, `test`, `ci`, `build`
- **scope** (optional) — area, e.g. `invoice`, `skills`, `e2e` (kebab or short name per project)
- **short description** — imperative mood, **not** a sentence: lowercase start after the colon, **no** trailing period

**Examples (good):**

- `feat(teams): add author to card schema`
- `fix(downloads): respect empty file list`
- `docs(html-table-components): clarify step 7 test selectors`
- `chore: bump eslint config`

**Anti-patterns:**

- A single `chore: update` / `fix stuff` with unrelated hunks
- A `feat` message body that lists several unrelated product changes
- Staging the whole project with `git add .` when the diff mixes concerns — split first

---

## When the user is explicit

- If they ask for **granular** / **one file per commit** (where that makes sense) / **separate commits**, treat that as a **hard** requirement: split groups accordingly and do not merge into one commit to save time.
- If they **explicitly** want **one commit** (e.g. "squash to one" or "single commit for the whole review"), you may do that **after** they are explicit — and the message should still be Conventional where possible, or as they direct.

---

## Checklist (copy for complex sessions)

```text
- [ ] Grouped by logical change; coupled-only exceptions noted
- [ ] Staging matches exactly one group
- [ ] `git diff --cached` has no stowaway paths
- [ ] Message is Conventional and describes only the staged set
- [ ] Repeated until done or user stops with partial WIP
```

---

## Related

- The **myf-agent-skills** host repo’s [CONTRIBUTING.md](https://github.com/obaDev95/myf-agent-skills/blob/main/CONTRIBUTING.md) states the same one-file / coupled-file rules for humans; agents follow this skill plus that file.
- In a `skills/<name>/` tree, the policy file is at `../../CONTRIBUTING.md` relative to this `SKILL.md` (e.g. `skills/granular-commits/SKILL.md` → repository root `CONTRIBUTING.md`).
- If you also keep a **flat** `myf-agent-skills/<name>/` copy inside ui-myfinance, mirror the same `SKILL.md` text as the host’s `skills/granular-commits/SKILL.md`.
