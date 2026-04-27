# Contributing

## Commits: one file per commit (with narrow exceptions)

Keep history easy to revert, cherry-pick, and bisect.

1. **Default:** one **logical** change per commit. Prefer **one tracked file per commit** when changes are independent (different skills or different concerns).

2. **When to combine files in one commit:** only when they are **tightly coupled**—for example the same user-facing change split across a skill and its `references/` file, or a rename that must touch two paths atomically. Do not bundle unrelated skills “because it was one PR.”

3. **Pull requests** may contain many commits; reviewers prefer several small commits over one large squash that mixes unrelated files.

4. **Messages:** use [Conventional Commits](https://www.conventionalcommits.org/) with a scope when helpful, e.g. `docs(html-table-components): clarify Step 7 test selectors`.

5. **README or index files:** if a commit only updates the root `README.md` table to reflect another skill’s change, that README commit can land **after** the skill commits it summarizes, still as its **own** commit.

6. **Coding agents:** the [`skills/granular-commits/SKILL.md`](skills/granular-commits/SKILL.md) file encodes the same policy in an executable workflow (group diff → stage one group → Conventional `git commit` → repeat). Use it in agent or IDE skill packs when automating commits against this repository.
