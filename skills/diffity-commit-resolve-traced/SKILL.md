---
name: diffity-commit-resolve-traced
description: Niche variant of [[diffity-resolve-traced]] for users who want a clean repo state between diffity review cycles. Snapshots the current dirty tree as its own commit, then runs the traced diffity resolve pass on open browser threads, leaving the AI's edits + `@SEEN @AI:` / `@AI-reply:` trail uncommitted so they land as their own follow-up commit. Use when the user types `/diffity-commit-resolve-traced`, "diffity commit then resolve", "snapshot and diffity resolve", or similar. Refuses to run when the repo declares a no-auto-commit policy unless the user reconfirms.
---

# diffity-commit-resolve-traced

Same idea as [[commit-reply-review]] — snapshot dirty tree first, then run the reply pass on top — but the reply pass is [[diffity-resolve-traced]] instead of [[codereview-inline]]. The point: separate "what I wrote" from "what the AI added while resolving diffity threads," at git-history granularity, across many review cycles, without manual `git add -p` gymnastics.

This is a deliberate, niche workflow. Most users should keep using plain [[diffity-resolve-traced]] and commit whenever they like. The commit-variant exists for users who want every diffity resolution cycle to produce exactly two commits: the snapshot they wrote, and the AI's resolution edits + inline trail.

## Trigger

The user says one of: `/diffity-commit-resolve-traced`, `diffity commit then resolve`, `snapshot and diffity resolve`, `clean-state diffity resolve`. Optional `thread-id` argument forwarded to the underlying [[diffity-resolve-traced]] (e.g. `/diffity-commit-resolve-traced abc123`).

## Preflight (in order; abort on any failure unless noted)

1. **Open diffity threads exist.** Run `diffity agent list --status open --json` (the same gate [[diffity-resolve-traced]] uses). If the command errors with "No active review session," abort with `No diffity session — start one with \`diffity\` first.` If there are zero open threads, print `No open diffity threads — nothing to snapshot.` and exit. Never commit "just in case."
2. **Repo permits automatic commits.** Same check as [[commit-reply-review]] preflight 2 — grep `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `.cursorrules`, `.github/copilot-instructions.md` for phrases like "do not commit", "never auto-commit", "no automatic commits", "commits require approval", "manual commit only"; check `.claude/settings.json` / `.claude/settings.local.json` for `PreToolUse` hooks gating `Bash(git commit*)` or a `permissions.deny` entry. `.pre-commit-config.yaml` is not a block. If any directive is found, quote it and ask the user explicitly: *"This repo declares <quote>. `/diffity-commit-resolve-traced` will create a commit. Continue anyway? (yes / no)"* Default to **no**.
3. **Working tree has something to snapshot.** `git status --porcelain`. If empty, skip the snapshot step and go straight to the resolve pass, printing `Working tree clean — proceeding directly to diffity resolve pass.`
4. **No in-progress rebase / merge / cherry-pick / bisect.** Check `.git/MERGE_HEAD`, `.git/CHERRY_PICK_HEAD`, `.git/rebase-merge/`, `.git/rebase-apply/`, `.git/BISECT_LOG`. If any exists, abort with `Refusing to commit during in-progress <op>. Finish or abort it first.`
5. **HEAD is on a branch, not detached.** `git symbolic-ref -q HEAD` must succeed. If detached, abort with `Refusing to commit on detached HEAD.`
6. **Upstream wrapper present.** Verify `diffity-resolve-traced` skill is resolvable (this plugin ships it; should always succeed). If not, abort with `diffity-resolve-traced not found — reinstall the codereview-skills plugin.`

## Snapshot

1. Stage everything in the working tree: `git add -A`. Includes untracked files — the snapshot's purpose is "everything that existed before the AI touched it." If the user has secrets or junk they don't want committed, they `.gitignore` them first or use plain [[diffity-resolve-traced]].
2. Commit with this message (verbatim, substituting `N`):

   ```
   wip: snapshot pre-diffity-resolve pass

   Auto-committed by /diffity-commit-resolve-traced before processing N open diffity threads.
   Resolve edits and inline @SEEN @AI: / @AI-reply: trail will land as a separate, uncommitted change for review.
   ```

   Use `--no-verify` **only** if the user has already opted into it this session; otherwise let pre-commit hooks run normally. If they fail, abort the whole skill and surface the hook output. Do not retry, do not amend, do not skip.
3. Re-check `git status --porcelain` is empty after commit. If hooks auto-fixed files back into the tree, stage them and amend the snapshot **once**. If hooks keep changing files on repeated runs (unstable formatter), abort after the single amend.

## Resolve pass

Run the full [[diffity-resolve-traced]] flow (which itself delegates to upstream `diffity-resolve` and then inserts the `@SEEN @AI:` / `@AI-reply:` trail at each edit site). Forward the optional `thread-id` argument unchanged.

**Do not commit the resolve edits or the trail.** The user's whole point in invoking this skill is to inspect the AI's diff in isolation before committing it themselves.

## Output

Print, in order:

- `Snapshot: <short-sha> "<commit subject>"` (or `Snapshot: skipped — working tree was clean`).
- Forward [[diffity-resolve-traced]]'s output verbatim (upstream resolution lines + traced trail summary).
- A trailing hint: `Resolve edits + trail are uncommitted. Review with \`git diff\`, then commit when ready.`

## Edge cases

- **User runs the skill twice in a row** without committing the resolve pass: second run sees a dirty tree (the AI's resolve edits from the first run) and would snapshot them into a new "WIP" commit, defeating the separation. Detect this: if the dirty tree contains *only* `@SEEN @AI:` / `@AI-reply:` edits AND files diffity already touched (diff-only-touches-resolve-output check), abort with `Previous diffity resolve pass is still uncommitted. Commit or discard it before running again.`
- **Pre-commit hook reformats files mid-commit**: handled in Snapshot step 3 with a single amend. Same abort rule as [[commit-reply-review]] if hooks are unstable.
- **Snapshot commit succeeds but resolve pass errors midway** (diffity CLI dies, file unreadable, etc.): leave the snapshot in place (it's valid) and surface the partial state. Files already edited in the resolve pass stay edited; the user can `git diff` to see how far it got. Do not roll back the snapshot.
- **Repo is not a git repo at all**: abort early with `Not a git repository — \`/diffity-commit-resolve-traced\` needs git.` Suggest plain [[diffity-resolve-traced]] instead.
- **`.gitignore` excludes everything the user touched**: `git status --porcelain` will be empty, preflight 3 treats as "clean tree, skip snapshot." Resolve pass still runs. Correct behavior.
- **Diffity session exists but no edits land** (all threads are general comments, dismissed, or pure replies): snapshot is still made (preflight 1 saw open threads), but the resolve pass produces no edits and no trail. Result: a snapshot commit followed by a clean tree. Print `Snapshot: <sha> "..." / Resolve pass produced no edits — clean tree after snapshot.` so the user knows what happened.

## Boundaries

- Never commit without confirming the open-thread count is > 0 *and* the working tree is non-empty *and* the user passed preflight 2.
- Never use `git commit --amend` outside the single hook-fixup case in Snapshot step 3. The snapshot commit is the user's history now; treat it as immutable.
- Never push. Never `git reset`, `git restore`, `git checkout --`, or any other destructive op. This skill writes exactly one commit and edits files — nothing else.
- Never run if preflight 2 surfaces a no-auto-commit directive and the user has not explicitly said yes in the same turn.
- Never invoke [[codereview-clean]] from here — cleanup is always user-initiated.
- Never modify or duplicate upstream `diffity-resolve` logic. The resolve pass is delegated to [[diffity-resolve-traced]], which delegates further to upstream. Future upstream changes flow through automatically.
