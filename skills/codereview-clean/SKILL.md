---
name: codereview-clean
description: Sweep inline @AI review trail left by codereview-inline. Removes closed-thread comments (`@SEEN @AI` blocks + adjacent `@AI-reply:` comments) from a file or path. Defaults to closed threads only. `--all` also removes still-open `@AI` markers; `--replies-only` removes only agent replies, leaving user comments intact. Use when the user types `/codereview-clean <path>`, "clean AI review comments", or similar.
---

# codereview-clean

Companion to `codereview-inline`. Removes the embedded review trail when the user is done with it. Closed threads can stay in source indefinitely; this skill only runs on explicit invocation.

## Trigger

`/codereview-clean <path>`, optionally with a mode flag. `<path>` can be a file, directory, or `.` for the whole repo.

## Modes

| Flag | Behavior |
|---|---|
| (default) | Remove closed comment blocks (`@SEEN @AI ...`) and their adjacent `@AI-reply:` comments. Leave still-open `@AI` threads alone. |
| `--all` | All of the above, plus remove still-open `@AI` comments (use when the user has decided to drop a pending question). |
| `--replies-only` | Remove only `@AI-reply:` comments. Keep user comments intact, including `@SEEN` markers — useful when the user wants to preserve their own questions as inline documentation. |

## Processing flow

1. Resolve `<path>`. Refuse `/`, `~`, paths containing `..` that escape the repo, or anything outside the current repo root.
2. List candidate files under the path (skip binary, generated, lockfile, vendored — same filters as `codereview-inline`).
3. For each file, identify comments matching the active mode's target set. Comment recognition is language-aware via the same filetype map as `codereview-inline`.
4. Build the edited file in memory:
   - For closed-thread removal: drop the entire comment block containing `@SEEN @AI`, since that comment exists only as part of the review trail.
   - For `@AI-reply:` removal: drop the entire reply comment block.
   - For `--all` open-marker removal: drop the entire user comment block containing `@AI` (even without `@SEEN`).
5. Collapse stray blank lines: if removal leaves two or more consecutive blank lines, collapse to one. Don't leave dangling indentation.
6. Write each file once, only if changed.
7. Print summary: `Removed N comments across M files`, with a per-file count.
8. Run no git commands beyond reads. User commits the cleanup whenever they want.

## Edge cases

- **Comment block mixes review tokens with user prose** (e.g. `// @SEEN @AI TODO: revisit this later`): default mode treats the whole comment as review trail and removes it. If the user wants to keep surrounding prose, they should hand-edit before running clean, or use `--replies-only`.
- **Multi-line `@AI-reply:` reply**: remove the whole comment block, not just the first line.
- **Marker appears inside a string** despite the comment-context filter: skip. Same safety check as `codereview-inline`.
- **File contains only review markers, nothing else**: still safe; result is an emptier file with no special handling needed.
- **Last line of file is a removed comment**: trim trailing whitespace/blank lines after removal.

## Boundaries

- Never touch code outside of comment blocks containing the target tokens.
- Never run without an explicit `<path>` argument.
- Never auto-commit.
- Never remove a comment that doesn't contain any of `@AI`, `@SEEN`, or `@AI-reply:`.
