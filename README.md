# codereview-skills

Lightweight inline code-review for Claude Code. Drop `@AI` in any comment, say `go`, and the agent answers in place. The exchange stays embedded in the source as a review trail until you sweep it.

## Install

```
/plugin install <repo-url>
```

(Replace `<repo-url>` with this repo's GitHub URL.)

## Quick usage

1. Drop a comment containing `@AI` anywhere in any source file:
   ```python
   # @AI why this magic number?
   TIMEOUT = 47
   ```
2. Say `go` (or `/codereview`). The agent scans uncommitted changes repo-wide and answers every open `@AI`:
   ```python
   # @SEEN @AI why this magic number?
   # @AI-reply: legacy API page size; see PR #4421
   TIMEOUT = 47
   ```
3. Follow up by adding another `@AI` below — thread continuity is preserved.
4. When done: `/codereview-clean .` sweeps the trail.

`codereview-inline` and `codereview-clean` only read git state — they never stage, commit, or push. `commit-reply-review` is the deliberate exception: it creates exactly one snapshot commit (after preflight checks) so the AI's reply edits stay isolated from your work. Every other commit is still yours.

## Skills

| Skill | Purpose |
|---|---|
| [`codereview-inline`](skills/codereview-inline/SKILL.md) | Scans uncommitted changes, answers every open `@AI` in place. |
| [`codereview-clean`](skills/codereview-clean/SKILL.md) | Removes closed `@SEEN @AI` blocks and adjacent `@AI-reply:` comments. Optional `--all` / `--replies-only` flags. |
| [`commit-reply-review`](skills/commit-reply-review/SKILL.md) | Niche: snapshots the dirty tree as a WIP commit, then runs the inline reply pass so the AI's edits land as their own uncommitted change. For users who want git history to cleanly separate "what I wrote" from "what the AI added." |

### Diffity bridges

Wrappers around the [`nilbuild/diffity`](https://github.com/nilbuild/diffity) skill pack that augment its browser-driven resolution flows with the same `@SEEN @AI:` / `@AI-reply:` inline trail this plugin uses elsewhere — so `codereview-clean` can sweep diffity-driven reviews with no special-casing. Install diffity first: `npx skills add nilbuild/diffity`.

| Skill | Purpose |
|---|---|
| [`diffity-resolve-traced`](skills/diffity-resolve-traced/SKILL.md) | Delegates to upstream `diffity-resolve` (diff-viewer resolution), then leaves an `@SEEN @AI:` / `@AI-reply:` trail at every edit site. |
| [`diffity-resolve-tree-traced`](skills/diffity-resolve-tree-traced/SKILL.md) | Delegates to upstream `diffity-resolve-tree` (tree-browser resolution), then leaves the same trail at every edit site. |
| [`diffity-commit-resolve-traced`](skills/diffity-commit-resolve-traced/SKILL.md) | Niche: snapshots the dirty tree as a WIP commit, then runs `diffity-resolve-traced` so the AI's edits + inline trail land as their own uncommitted change. Diffity-flow counterpart to `commit-reply-review`. |

> **Heads up:** `codereview-clean` and `commit-reply-review` are both lightly tested. Commit (or stash) before invoking so you can `git diff` / revert if anything surprises you. `commit-reply-review` will refuse to run if it detects a no-auto-commit directive in your repo (CLAUDE.md / AGENTS.md / settings hooks) unless you reconfirm.

## Marker reference

| Token | Meaning |
|---|---|
| `@AI` | Open thread. Agent attends on next `go`. |
| `@SEEN` | Prepended in front of `@AI` when the agent processes the thread. Delete it to reopen. |
| `@AI-reply:` | Agent's reply comment, placed immediately after your comment. |

Tokens recognized only inside comments. Per-filetype comment syntax: `#`, `//`, `/* */`, `<!-- -->`, `--`, `;;`, `%% %%` — see the inline skill spec for the full map.

## Full spec

- [`skills/codereview-inline/SKILL.md`](skills/codereview-inline/SKILL.md) — processing flow, thread continuity, edge cases.
- [`skills/codereview-clean/SKILL.md`](skills/codereview-clean/SKILL.md) — modes, path safety, edge cases.
