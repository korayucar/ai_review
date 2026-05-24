---
name: diffity-resolve-tree-traced
description: Wrapper around the upstream `diffity-resolve-tree` skill that, after every code edit lands, leaves an inline `@SEEN @AI:` / `@AI-reply:` trail at the edit site in the file's native comment syntax. Same idea as [[diffity-resolve-traced]] but for the diffity tree browser (whole-repo file comments rather than diff-anchored comments). Use when the user types `/diffity-resolve-tree-traced`, "resolve diffity tree with trail", "diffity tree resolve traced", or wants the tree-browser resolution flow plus an in-source review trail. Reads git state but never commits.
---

# diffity-resolve-tree-traced

Thin wrapper around the upstream [`diffity-resolve-tree`](https://github.com/nilbuild/diffity) skill (typically installed at `.agents/skills/diffity-resolve-tree/SKILL.md`; may live elsewhere under a global skills install). Delegates the full diffity tree-browser resolution flow, then augments each landed edit with the same `@SEEN @AI:` / `@AI-reply:` inline trail that [[codereview-inline]] produces — so [[codereview-clean]] can sweep tree-browser-driven reviews with no special-casing.

Sibling of [[diffity-resolve-traced]]. The trail format is identical; only the upstream skill differs (tree browser, comments on whole-repo files rather than on a diff).

## Trigger

The user says one of: `/diffity-resolve-tree-traced`, `diffity tree resolve traced`, `resolve diffity tree with trail`, `diffity-resolve-tree with trail`. Optional `thread-id` argument is forwarded verbatim to the upstream skill (e.g. `/diffity-resolve-tree-traced abc123`).

## Delegation

1. Locate the upstream skill. Standard locations, in order:
   - `.agents/skills/diffity-resolve-tree/SKILL.md` (per-project install via `npx skills add nilbuild/diffity`)
   - any global skills install path the host resolves `diffity-resolve-tree` from
2. Read the upstream `SKILL.md` end-to-end at delegation time. **Do not** copy its steps into this file — execute them by reference so upstream changes flow through automatically.
3. Run every step in the upstream `## Instructions` block (prerequisites, JSON list, per-thread classification including `[question]` reply-then-resolve flow, edit, `diffity agent resolve --summary`, final `diffity agent list`). Forward the optional `thread-id` argument unchanged.
4. While executing each upstream step that calls `Edit` (or any other file-write tool) on a source file, capture:
   - the **resolved thread's comment body** (verbatim text the user left in the diffity tree browser, including any `[nit]` / `[question]` tag and the leading author marker if present)
   - the **resolve summary** you are about to pass to `diffity agent resolve --summary "..."`
   - the **file path and line range** of each edit site
5. After the upstream step that runs `diffity agent resolve <thread-id> --summary "..."` succeeds for that thread, append the trail (next section) at every edit site that thread produced. Do this **before** moving on to the next thread, so trails interleave with resolutions and stay anchored.

## Trail to leave

Identical to [[diffity-resolve-traced]]'s "Trail to leave" section. Reproduced briefly:

1. At each edit site, insert in the filetype's comment syntax, immediately above the edited line:
   ```
   @SEEN @AI: <verbatim diffity comment body, collapsed to one line>
   @AI-reply: <one-line summary of what was changed and why>
   ```
2. Same comment syntax for both lines (block-only filetypes use block comments for both; line-comment languages use line comments for both).
3. The `@SEEN @AI:` form (with colon) signals a synthesized marker carrying a quoted body — distinct from `codereview-inline`'s `@SEEN @AI` which prepends to a pre-existing user comment.
4. Collapse newlines/whitespace in the diffity body; truncate at ~200 characters with `…` if needed.
5. Match the edited line's indentation.

## Filetype → comment syntax

Use the exact map defined in [[codereview-inline]] under "Filetype → comment syntax map". Do not redefine it. Fallback rule for unmapped filetypes is the same as [[diffity-resolve-traced]]: try the file's dominant comment style; if no comments anywhere, skip and report.

## Question-thread special case

The upstream `diffity-resolve-tree` skill differs from `diffity-resolve` in one place: for `[question]` threads from the user, it replies with `diffity agent reply` first and then resolves with a summary. If answering the question requires **no code edit** (the answer is purely informational), there is no edit site and therefore no trail — the answer lives entirely in the diffity thread. If answering does require a code edit (e.g. "should we add X?" → yes, and you add X), trail the edit normally.

## Output

Forward the upstream skill's own output, then print:

- `Traced N edit sites across M files` (count of trail insertions, not threads).
- One line per trail insertion: `<path>:<line> ← <thread-id-prefix> "<comment gist>"`.
- One line per skip: `Skipped trail at <path>:<line> — <reason>`.

## Edge cases

- **Upstream skill not installed.** Abort with `Upstream diffity-resolve-tree not found. Install with: npx skills add nilbuild/diffity`. Do not reimplement.
- **Upstream skill aborts** (no diffity tree session, no open threads, `diffity` CLI missing): forward its abort message verbatim and exit.
- **Pure-reply resolution** (question answered without an edit): no trail. See "Question-thread special case" above.
- **Upstream skill dismisses** a thread: no trail.
- **Edit lands in a filetype not covered by the map**: dominant-style fallback, then skip if none.
- **Same file edited by multiple threads in one run**: one trail per thread at its respective edit site. Adjacent stacked trails are fine — [[codereview-clean]] handles them.
- **Upstream `SKILL.md` evolves**: wrapper still works — it depends only on the contract "diffity-resolve-tree makes edits and resolves with a summary."
- **No edits land at all**: print `No edits to trace.` after upstream output.

## Boundaries

- Never modify or duplicate upstream `diffity-resolve-tree` logic. Execute it by reference and only add trail comments after edits land.
- Never commit. Match the rest of this plugin.
- Never invoke [[codereview-clean]] from here.
- Never rewrite or remove an existing `@SEEN @AI:` / `@AI-reply:` block. Append below if a follow-up thread touches the same site.
- Never insert the trail before `diffity agent resolve` succeeds for that thread.
