---
name: diffity-resolve-traced
description: Wrapper around the upstream `diffity-resolve` skill that, after every code edit lands, leaves an inline `@SEEN @AI:` / `@AI-reply:` trail at the edit site in the file's native comment syntax. Lets `codereview-clean` sweep diffity-driven reviews the same way it sweeps inline `@AI` threads. Use when the user types `/diffity-resolve-traced`, "resolve diffity with trail", "diffity resolve traced", or wants the diffity diff-viewer resolution flow plus an in-source review trail. Reads git state but never commits.
---

# diffity-resolve-traced

Thin wrapper around the upstream [`diffity-resolve`](https://github.com/nilbuild/diffity) skill (typically installed at `.agents/skills/diffity-resolve/SKILL.md`; may live elsewhere under a global skills install). Delegates the full diffity diff-viewer resolution flow, then augments each landed edit with the same `@SEEN @AI:` / `@AI-reply:` inline trail that [[codereview-inline]] produces — so [[codereview-clean]] can sweep diffity-driven reviews with no special-casing.

Diffity edits source but leaves no in-file evidence of which browser comment drove which change. This wrapper closes that gap.

## Trigger

The user says one of: `/diffity-resolve-traced`, `diffity resolve traced`, `resolve diffity with trail`, `diffity resolve then trace`. Optional `thread-id` argument is forwarded verbatim to the upstream skill (e.g. `/diffity-resolve-traced abc123`).

## Delegation

1. Locate the upstream skill. Standard locations, in order:
   - `.agents/skills/diffity-resolve/SKILL.md` (per-project install via `npx skills add nilbuild/diffity`)
   - any global skills install path the host resolves `diffity-resolve` from
2. Read the upstream `SKILL.md` end-to-end at delegation time. **Do not** copy its steps into this file — execute them by reference so upstream changes flow through automatically.
3. Run every step in the upstream `## Instructions` block (prerequisites, JSON list, per-thread classification, edit, `diffity agent resolve --summary`, final `diffity agent list`). Forward the optional `thread-id` argument unchanged.
4. While executing each upstream step that calls `Edit` (or any other file-write tool) on a source file, capture:
   - the **resolved thread's comment body** (verbatim text the user left in the diffity browser, including any `[nit]` / `[question]` tag and the leading author marker if present)
   - the **resolve summary** you are about to pass to `diffity agent resolve --summary "..."` (this is your one-line description of what was changed)
   - the **file path and line range** of each edit site
5. After the upstream step that runs `diffity agent resolve <thread-id> --summary "..."` succeeds for that thread, append the trail (next section) at every edit site that thread produced. Do this **before** moving on to the next thread, so trails interleave with resolutions and stay anchored.

## Trail to leave

For each edit site (one per file the thread touched; if multiple non-adjacent edits in the same file, leave a trail at each):

1. Insert, in the filetype's comment syntax, immediately above the edited line (or above the edit's first line for a multi-line edit):
   ```
   @SEEN @AI: <verbatim diffity comment body, collapsed to one line>
   @AI-reply: <one-line summary of what was changed and why>
   ```
2. Both lines use the same comment syntax. For block-only filetypes (HTML, JSX expressions), both are block comments. For line-comment languages, both are line comments. Markdown follows the same form the user uses elsewhere in the file — bare prose lines if the file has no review-trail convention yet, `%% ... %%` or `<!-- ... -->` if a convention is already established in that file.
3. The `@SEEN @AI:` form (with colon) is deliberate and distinct from `codereview-inline`'s `@SEEN @AI` (no colon). Reason: the inline workflow prepends `@SEEN ` to a pre-existing `@AI` comment the user wrote; here there is no pre-existing comment, so the marker is synthesized fresh and the colon clarifies it carries a quoted body.
4. Collapse the diffity comment body to one line: replace newlines with spaces, collapse runs of whitespace, and trim. If the body is longer than ~200 characters, truncate with `…` — full body is still in the resolved diffity thread.
5. Preserve indentation: match the indentation of the line the trail is placed above. Do not introduce tabs/spaces inconsistent with the rest of the file.
6. Word-boundary safety: the inserted `@AI` must sit inside a comment (the comment syntax guarantees this). Do not insert in strings, JSX attribute values, or fenced code blocks.

## Filetype → comment syntax

Use the exact map defined in [[codereview-inline]] under "Filetype → comment syntax map" — do not redefine it. If the upstream `Edit` touches a file whose extension is not covered there, default to that file's existing dominant comment style (read the file, find the most common comment marker, use that); if the file has no comments at all and no clear convention, skip the trail for that edit and print a one-line notice in the chat summary (`No comment syntax for <path> — trail skipped`).

## Examples

Python (line-comment language). Diffity thread body: `[nit] rename to TIMEOUT_SECONDS so the unit is obvious`. Resolve summary: `Renamed TIMEOUT → TIMEOUT_SECONDS for unit clarity`:

```python
# @SEEN @AI: [nit] rename to TIMEOUT_SECONDS so the unit is obvious
# @AI-reply: Renamed TIMEOUT → TIMEOUT_SECONDS for unit clarity
TIMEOUT_SECONDS = 47
```

JavaScript (block-comment language). Diffity thread body: `is this branch reachable?`. Resolve summary: `Added comment — yes, null-token path when getUser() expires`:

```javascript
/* @SEEN @AI: is this branch reachable? */
/* @AI-reply: Added comment — yes, null-token path when getUser() expires */
if (user == null) { handleNull(); }
```

HTML (block-only):

```html
<!-- @SEEN @AI: should this be aria-live? -->
<!-- @AI-reply: Added aria-live="polite" — it's a toast container -->
<div class="toast" aria-live="polite"></div>
```

Multi-file thread (same comment resolved by touching two files):

```python
# api/client.py
# @SEEN @AI: extract the retry config into a shared module
# @AI-reply: Moved retry config to common/retry.py; importers updated
from common.retry import RETRY_BUDGET
```

```python
# jobs/sync.py
# @SEEN @AI: extract the retry config into a shared module
# @AI-reply: Moved retry config to common/retry.py; importers updated
from common.retry import RETRY_BUDGET
```

Same trail body, repeated at each edit site — one diffity thread maps to one logical exchange, surfaced in every file it touched.

## Output

Forward the upstream skill's own output (it prints per-thread resolution status and a final `diffity agent list`). Then print, in order:

- `Traced N edit sites across M files` (count of trail insertions, not threads — a single thread may produce multiple).
- One line per trail insertion: `<path>:<line> ← <thread-id-prefix> "<comment gist>"`.
- If any edit was skipped (unknown filetype, no comment convention), one line per skip: `Skipped trail at <path>:<line> — <reason>`.

## Edge cases

- **Upstream skill not installed.** If `.agents/skills/diffity-resolve/SKILL.md` is missing and no global resolution succeeds, abort with `Upstream diffity-resolve not found. Install with: npx skills add nilbuild/diffity`. Do not attempt to reimplement diffity logic.
- **Upstream skill aborts** (no diffity session, no open threads, `diffity` CLI missing): forward its abort message verbatim and exit. Nothing to trace.
- **Upstream skill replies-only** (no edit, just `diffity agent reply` for clarification): no trail. Trail is keyed to source edits, not to back-and-forth on the browser side.
- **Upstream skill dismisses** a thread (`diffity agent dismiss`): no trail. Dismissal means no edit.
- **Upstream skill edits a file outside this filetype map's coverage** (e.g. an obscure DSL): see "Filetype → comment syntax" fallback — try dominant comment style; if none, skip and report.
- **Same file edited by multiple threads in one run**: leave one trail per thread at its respective edit site. Trails may end up adjacent; that's fine — `codereview-clean` handles stacked closed blocks.
- **Edit happens inside a string literal or generated/lockfile region**: defer to the upstream skill's own judgment about whether the edit was correct; the trail simply doesn't go in. Print a skip notice.
- **Upstream `SKILL.md` changes shape** (new steps, different CLI flags): wrapper still works — it only depends on the contract "diffity-resolve makes edits and resolves threads with a summary." Trail insertion happens at the observed edit sites regardless of how upstream got there.
- **No edits land at all** (every thread is skipped, replied-to, or dismissed): wrapper produces no trail and prints `No edits to trace.` after the upstream output.

## Boundaries

- Never modify or duplicate upstream `diffity-resolve` logic. The wrapper executes the upstream skill by reference and only adds trail comments after edits land.
- Never commit. Match the rest of this plugin — leave all changes (upstream edits + trail) uncommitted for the user to review.
- Never invoke [[codereview-clean]] from here. Cleanup is always user-initiated.
- Never rewrite or remove an existing `@SEEN @AI:` / `@AI-reply:` block left by a prior run. If a trail already exists at an edit site (e.g. a follow-up thread on the same line), append a new trail below the existing one rather than replacing it.
- Never insert the trail before the upstream `diffity agent resolve` call succeeds — if resolve fails, the edit may need to be reverted by the user, and a stale trail would lie.
