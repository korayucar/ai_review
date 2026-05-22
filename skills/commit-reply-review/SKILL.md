---
name: commit-reply-review
description: Niche variant of `codereview-inline` for users who want a clean repo state between review cycles. Snapshots the current dirty tree as its own commit, then runs the inline reply pass on `@AI` threads, leaving the AI's reply edits uncommitted so they land as their own follow-up commit. Use when the user types `/commit-reply-review`, "commit then reply", or similar. Refuses to run when the repo declares a no-auto-commit policy unless the user reconfirms.
---

# commit-reply-review

Same surface as [[codereview-inline]] — scan uncommitted changes, attend every `@AI`, prepend `@SEEN`, append `@AI-reply:` — but bracketed with one extra step: before the reply pass, commit whatever the user already has staged + unstaged as a snapshot. The reply edits then sit on top as their own dirty tree, ready for the user to commit (or amend, or discard) separately.

This is a deliberate, niche workflow: some developers want git history to cleanly separate "what I wrote" from "what the AI added in the reply pass," and want that separation to hold across many review cycles without manual `git add -p` gymnastics each round. **Not a default.** Most users should keep using `codereview-inline` and commit whenever they like.

## Trigger

The user says one of: `/commit-reply-review`, `commit then reply`, `snapshot and reply`, `clean-state review`. No arguments — scope is the whole repo, same as `codereview-inline`.

## Preflight (in order; abort on any failure unless noted)

1. **Open threads exist.** Run the same scan as `codereview-inline` step 1–4 (changed-files gate, comment-context check). If zero open `@AI` threads, print `No open @AI threads — nothing to snapshot.` and exit. Never commit "just in case."
2. **Repo permits automatic commits.** Check for directives that forbid agent-initiated commits, in this order:
   - `CLAUDE.md` (repo root and any parent up to repo root) — grep for phrases like "do not commit", "never auto-commit", "no automatic commits", "commits require approval", "manual commit only".
   - `AGENTS.md`, `CONTRIBUTING.md`, `.cursorrules`, `.github/copilot-instructions.md` — same grep.
   - `.claude/settings.json` and `.claude/settings.local.json` — look for `PreToolUse` hooks gating `Bash(git commit*)`, or a `permissions.deny` entry blocking git commit.
   - Project-level `.pre-commit-config.yaml` is **not** a block — pre-commit hooks running is fine, the commit itself is still permitted.

   If any directive is found, print the matching file + line + quote, then ask the user explicitly: *"This repo declares <quote>. `/commit-reply-review` will create a commit. Continue anyway? (yes / no)"* Default to **no** if the user does not reply with an affirmative. Never assume silence is consent.
3. **Working tree has something to snapshot.** Run `git status --porcelain`. If output is empty, there's nothing to commit — skip step "Snapshot" and go straight to the reply pass, printing `Working tree clean — proceeding directly to reply pass.`
4. **No in-progress rebase / merge / cherry-pick / bisect.** Check `.git/MERGE_HEAD`, `.git/CHERRY_PICK_HEAD`, `.git/rebase-merge/`, `.git/rebase-apply/`, `.git/BISECT_LOG`. If any exists, abort with `Refusing to commit during in-progress <op>. Finish or abort it first.`
5. **HEAD is on a branch, not detached.** `git symbolic-ref -q HEAD` must succeed. If detached, abort with `Refusing to commit on detached HEAD.`

## Snapshot

1. Stage everything currently in the working tree: `git add -A`. This deliberately includes untracked files — the snapshot's purpose is "everything that existed before the AI touched it," and a half-snapshot defeats the point. If the user has secrets or junk they don't want committed, they `.gitignore` them first or use plain `codereview-inline`.
2. Commit with this message (verbatim):

   ```
   wip: snapshot pre-@AI-reply pass

   Auto-committed by /commit-reply-review before processing N open @AI threads.
   Reply edits will land as a separate, uncommitted change for review.
   ```

   Substitute `N` with the actual count. Use `--no-verify` **only** if the user has already opted into it in this session; otherwise let pre-commit hooks run normally — if they fail, abort the whole skill and surface the hook output. Do not retry, do not amend, do not skip.
3. After commit, re-check `git status --porcelain` is empty. If hooks added auto-fixed files back to the tree, stage them and amend the snapshot once — the goal is a clean working tree before the reply pass starts.

## Reply pass

Run the full [[codereview-inline]] flow from its step 5 onward (open-thread iteration, `@SEEN ` prepend, `@AI-reply:` insertion, batched per-file writes, same edge cases, same boundaries). **Do not commit the reply edits.** The user's whole point in invoking this skill is to inspect the AI's diff in isolation before committing it themselves.

## Output

Print, in order:
- `Snapshot: <short-sha> "<commit subject>"` (or `Snapshot: skipped — working tree was clean`)
- Then the standard `codereview-inline` summary: `Processed N threads across M files`, followed by one line per thread.
- A trailing hint: `Reply edits are uncommitted. Review with \`git diff\`, then commit when ready.`

## Edge cases

- **User runs the skill twice in a row** without committing the reply pass: second run sees a dirty tree (the reply edits from the first run) and would snapshot the AI's own output into the user's "WIP" commit, defeating the separation. Detect this: if the dirty tree contains *only* `@SEEN` / `@AI-reply:` edits (diff-only-touches-comment-trail check), abort with `Previous reply pass is still uncommitted. Commit or discard it before running again.` Don't auto-decide for the user.
- **Pre-commit hook reformats files** mid-commit: handled in Snapshot step 3 with a single amend. If the hook keeps changing files on repeated runs (unstable formatter), abort after one amend attempt — the repo has a bigger problem to fix first.
- **Snapshot commit succeeds but reply pass errors midway**: leave the snapshot in place (it's valid as-is) and surface the partial state. Files already written in the reply pass stay written; the user can `git diff` to see how far it got. Do not try to roll back the snapshot — that's a destructive op requiring explicit user consent.
- **Repo is not a git repo at all**: abort early with `Not a git repository — \`/commit-reply-review\` needs git.` Suggest plain `codereview-inline` instead.
- **`.gitignore` excludes everything the user touched**: `git status --porcelain` will be empty, treated by preflight 3 as "clean tree, skip snapshot." Reply pass still runs. This is correct behavior — there's nothing meaningful to snapshot.

## Boundaries

- Never commit without confirming the open-thread count is > 0 OR the working tree is non-empty *and* the user passed preflight 2.
- Never use `git commit --amend` outside the single hook-fixup case in Snapshot step 3. The snapshot commit is the user's history now; treat it as immutable.
- Never push. Never `git reset`, `git restore`, `git checkout --`, or any other destructive op. This skill writes exactly one commit and edits files — nothing else.
- Never run if preflight 2 surfaces a no-auto-commit directive and the user has not explicitly said yes in the same turn.
- Never invoke the [[codereview-clean]] skill from here — cleanup is always user-initiated.
