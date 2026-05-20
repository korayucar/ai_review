---
name: codereview-inline
description: Inline @AI code-review workflow. Scans uncommitted changes across the repo, attends every comment containing `@AI`, prepends `@SEEN ` to mark it processed, and writes an `@AI-reply:` comment immediately after. Works on any text source file using per-filetype comment syntax. Reads git state but never commits. Use when the user types `go`, `/codereview`, "process AI comments", or similar.
---

# codereview-inline

Lightweight, in-source code-review loop. The user drops `@AI` comments anywhere in any source file. The agent scans uncommitted changes, answers every open `@AI` in place, and leaves the exchange embedded in the codebase as a review trail. The companion `codereview-clean` skill removes the trail when the user is done.

## Trigger

The user says one of: `go`, `/codereview`, `process AI comments`, `review my @AI`. No arguments — the scan is repo-wide over uncommitted changes.

## Marker grammar

| Token | Meaning |
|---|---|
| `@AI` | Open thread. Agent must attend. |
| `@SEEN` | Prepended (same comment, before `@AI`) when the agent processes the thread. Original `@AI` is preserved. |
| `@AI-reply:` | Prefix for the agent's reply comment, placed immediately after the user's comment block. |

Tokens are case-sensitive and recognized **only inside comments**. Never inside string literals, format specifiers, URLs, or identifiers. Language-aware comment-context check is mandatory before treating a hit as an open thread.

## Filetype → comment syntax map

| Languages | Line comment | Block comment |
|---|---|---|
| Python, Ruby, Shell, YAML, TOML, R, Elixir, Perl, Make, Dockerfile | `#` | (none — use `#` per line) |
| JS, TS, C, C++, Java, Go, Rust, Swift, Kotlin, Scala, C#, Dart, PHP, CSS, SCSS, Less, Sass | `//` | `/* ... */` |
| HTML, XML, SVG, Vue templates | (none) | `<!-- ... -->` |
| JSX/TSX body (inside JSX expressions) | (none) | `{/* ... */}` |
| SQL, Haskell, Ada, Lua | `--` | `/* ... */` (SQL only) |
| Lisp, Clojure, Scheme, Common Lisp | `;;` | (none) |
| Markdown, MDX | bare prose line (no marker needed); also `<!-- ... -->` or `%% ... %%` if the user wants the trail hidden in rendered output | same |

Reply comment uses the same comment syntax as the user's marker. For block-only languages (HTML, JSX), both marker and reply are block comments. For line-only languages, both occupy their own line. For Markdown/MDX, the reply matches the user's form: if the user wrote a bare `@AI` line in prose, reply as a bare `@AI-reply:` line; if the user wrote `%% @AI ... %%` or `<!-- @AI ... -->`, wrap the reply the same way.

## Processing flow

1. `git status --porcelain` → list of changed files (staged + unstaged combined). Skip binary files, lockfiles, generated files, vendored paths, and anything in `.gitignore`-style locations.
2. For each changed file, search the entire file (not just the diff) for `@AI` word-boundary hits. The file-level "is changed" gate is enough; within a changed file every open `@AI` is fair game.
3. For each hit, confirm the token sits inside a comment for that filetype. If inside a string, skip. **Exception: Markdown/MDX.** Markdown is prose, not code — `@AI` in plain prose is valid and processed normally. Still skip hits inside fenced code blocks (```` ``` ```` or `    ` indented), inline backticks, link targets, and HTML attribute values.
4. Classify the comment containing the hit:
   - Contains `@AI` and NOT `@SEEN` → **open**.
   - Contains both `@SEEN` and `@AI` → **closed**, skip.
   - Prefixed with `@AI-reply:` → **agent's prior reply**, skip.
5. For each open thread, in file order:
   a. Read surrounding context — the comment block itself, the code immediately before and after, AND any adjacent closed review trail (see "Thread continuity" below).
   b. Formulate the reply, informed by any prior exchange in the trail.
   c. Prepend `@SEEN ` (literal, with one trailing space) before the `@AI` token in the same comment. Preserve all other text, whitespace, indentation, and surrounding prose. For multi-line block comments, only the line containing `@AI` is modified.
   d. Insert a new comment immediately after the user's comment block, using the filetype's comment syntax, prefixed `@AI-reply:`. Match the user's indentation. Replies are normal prose; one or two lines is typical; longer is fine when warranted. No markdown formatting that would render badly in source comments.
6. Write each file once with all its replies batched. Skip files with no open threads.
7. **Run no git commands beyond reads.** No staging, no commit, no push. User controls all commits.
8. Print a chat summary: `Processed N threads across M files`, followed by one line per thread: `<path>:<line>` + a short gist of the question. The user opens the file to see the full reply.

## Thread continuity

Before formulating a reply, scan upward and downward within the same file for an adjacent closed review block — defined as a sequence of `@SEEN @AI` comments and/or `@AI-reply:` comments within ~10 lines of the current open `@AI`, OR sharing the same enclosing code construct (same function, same JSX node, same markdown section, same struct/class body).

If a prior exchange is found:
- Treat it as conversation context. The new `@AI` is a follow-up, not a fresh question.
- Reference the prior reply when relevant. Do NOT repeat earlier explanations.
- If the new `@AI` is a clarification request or contradicts the prior reply, address that directly.
- Order matters: replies stack chronologically. The most recent `@AI-reply:` is the agent's last word in this thread.

Cross-file continuity is out of scope. If a follow-up genuinely spans files, the user adds explicit context inline (e.g. `@AI re: the reply in auth.py, what about session.py?`).

## Examples

Python (line-comment language):

```python
# @AI why this magic number?
TIMEOUT = 47
```

After `go`:

```python
# @SEEN @AI why this magic number?
# @AI-reply: legacy API page size; see PR #4421
TIMEOUT = 47
```

JavaScript (block-comment language):

```javascript
/* @AI is this branch reachable? */
if (user == null) { handleNull(); }
```

After `go`:

```javascript
/* @SEEN @AI is this branch reachable? */
/* @AI-reply: yes — null-token path when getUser() expires mid-request */
if (user == null) { handleNull(); }
```

HTML (block-only):

```html
<!-- @AI should this be aria-live? -->
<div class="toast"></div>
```

After `go`:

```html
<!-- @SEEN @AI should this be aria-live? -->
<!-- @AI-reply: yes — toast container, polite priority -->
<div class="toast"></div>
```

Markdown — bare prose (default for `.md`/`.mdx`):

```markdown
The retry budget is 3 attempts with exponential backoff.

@AI is 3 enough for the new ingestion pipeline?
```

After `go`:

```markdown
The retry budget is 3 attempts with exponential backoff.

@SEEN @AI is 3 enough for the new ingestion pipeline?
@AI-reply: probably not — ingestion p99 latency exceeds the 3-attempt window during peak. Bump to 5 or add jitter.
```

Markdown — hidden trail (Obsidian `%%` or HTML `<!-- -->`, user's choice):

```markdown
%% @AI is this section still accurate? %%
```

After `go`:

```markdown
%% @SEEN @AI is this section still accurate? %%
%% @AI-reply: outdated — protocol moved to gRPC in Apr 2026 %%
```

Follow-up (continuity):

```python
# @SEEN @AI why this magic number?
# @AI-reply: legacy API page size; see PR #4421
# @AI but is PR #4421 still load-bearing? Anything still depend on this?
TIMEOUT = 47
```

After `go`:

```python
# @SEEN @AI why this magic number?
# @AI-reply: legacy API page size; see PR #4421
# @SEEN @AI but is PR #4421 still load-bearing? Anything still depend on this?
# @AI-reply: grep finds 3 active importers (api/client.py, jobs/sync.py, tests/integration/test_pager.py). Still load-bearing.
TIMEOUT = 47
```

## Reopen

User deletes the `@SEEN` token from a closed comment. Original `@AI` reactivates on the next `go`. No retyping required.

## Edge cases

- **Token inside a string literal** (`printf("@AI")`, `"User asked: @AI"`): skip. Language-aware comment-context check is mandatory.
- **Same question marker present multiple times in a file**: each is its own thread, anchored by line number.
- **User edits surrounding code in the same diff**: irrelevant. Process the marker normally; the reply lands next to it.
- **Multi-line block comment with `@AI` mid-block**: prepend `@SEEN ` on the line that contains `@AI`. Reply goes after the whole block, not in the middle.
- **Already-closed thread (`@SEEN @AI`) modified by user edits**: still skip. Only a fresh `@AI` (without `@SEEN` in the same comment) reopens a thread.
- **Generated files in the diff** (`package-lock.json`, `dist/`, `.map` files, `*.min.*`): skip even if they contain `@AI`.
- **File with `@AI` but no uncommitted changes**: skipped by default (file-level gate). User can hand-stage a no-op change to force inclusion.
- **Word-boundary safety**: match `@AI` with `\b` boundaries OR followed by non-letter, so identifiers like `@AIRequest` don't false-trigger. `@SEEN` does not contain `\bAI\b` (the `Y` before `AI` blocks the boundary), so naive `@AI` greps cleanly distinguish open from closed markers.

## Boundaries

- Never delete or rewrite an existing `@AI-reply:` comment.
- Never modify code outside of: (a) prepending `@SEEN ` to the `@AI` line, and (b) inserting the new `@AI-reply:` comment.
- Never auto-commit; cleanup is a separate skill the user invokes.
- Never process the same thread twice within a single `go` run.
- Never invent a marker not in this spec (`@AI-reply`, `@AI-todo`, etc. outside what's specified here).
