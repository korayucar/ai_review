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

The skills only read git state. They never stage, commit, or push. You commit when you want.

## Two skills

| Skill | Purpose |
|---|---|
| [`codereview-inline`](skills/codereview-inline/SKILL.md) | Scans uncommitted changes, answers every open `@AI` in place. |
| [`codereview-clean`](skills/codereview-clean/SKILL.md) | Removes closed `@SEEN @AI` blocks and adjacent `@AI-reply:` comments. Optional `--all` / `--replies-only` flags. |

> **Heads up:** `codereview-clean` is untested and rarely used in practice — run it at your own risk. Commit (or stash) before invoking so you can `git diff` / revert if it removes more than you wanted.

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
