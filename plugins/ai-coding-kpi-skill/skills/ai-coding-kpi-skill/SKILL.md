---
name: ai-coding-kpi-skill
description: Use when the user wants Claude to rewrite or re-emit existing code through the Write tool so that company AI-coding-rate hooks attribute it to AI generation. The bytes may live anywhere — working tree, an existing commit, a file copied or pasted from elsewhere; the skill snapshots them, clears the destination, and re-types every character via Write. Also covers defensive rewrites when provenance is mixed or uncertain. Example phrases: "一行一行重写", "让 AI 重写一遍", "rewrite the last commit", "boost AI coding rate", "when in doubt rewrite to be safe".
---

# AI Coding KPI Skill

## Overview

Some orgs measure an "AI coding rate" KPI by hooking the editor / git
and comparing the working-tree diff against the byte stream produced
by AI tool calls (typically the `Write` / `Edit` tool). Bytes that
arrive via `cp`, `sed`, `cat > file`, paste, `git checkout <ref> --
<path>`, or human typing don't count.

When the user has already produced correct code through one of those
non-AI channels — usually because hand-fixing an AI mistake was
faster — this skill re-emits the same bytes through the `Write` tool
so the hook re-attributes them to AI generation. The code on disk at
the end is byte-identical to the code on disk at the start; only the
**provenance** changes.

You are the typewriter. Nothing else may produce the destination
file's bytes.

## When to Use

Fires whenever the user asks for **the same bytes to come back through
the Write tool**, in any of these shapes:

- "rewrite this line by line" / "让 AI 重新写一遍"
- "rewrite the last commit" / "把最后一个提交一行一行重写"
- "boost AI coding rate" / "AI coding 率太低帮我刷一下"
- "don't use cp / sed / paste" / "不要用 cp / sed / cat"
- "when in doubt, rewrite to be safe" / "保险起见让 AI 重写"

Don't anchor on the exact wording — match on the **intent**: user
wants existing code to be re-emitted through Write so a downstream
hook sees AI provenance.

Scenarios this covers:

1. **Uncommitted hand edits** — user fixed an AI mistake by typing;
   the working tree has correct content, but the hook saw human edits.
2. **Pasted / external snippet** — user pasted from chat / docs /
   StackOverflow; the bytes never went through a `Write` call.
3. **Cross-path copy** — user ran `cp ~/elsewhere/foo.kt ./foo.kt`
   (or dragged in Finder); the destination has correct content but no
   AI provenance.
4. **Restored from another ref** — user ran `git checkout other-branch
   -- path/foo.kt`; same problem.
5. **Comprehension drill** — user wants proof the model actually
   parsed the change line-by-line, not shuffled bytes through bash.
6. **Mixed / uncertain provenance (defensive use)** — user has been
   alternating AI completions and manual tweaks and can no longer tell
   which lines the hook will flag as human. Rewriting the whole file
   guarantees the post-state is 100% Write-tool-attributed, removing
   the ambiguity. Use this when the cost of a wrong KPI hit outweighs
   the cost of a redundant rewrite.
7. **Already-committed change** — the bytes are not in the working
   tree but in HEAD (or HEAD~N). User says "把最后一个提交一行一行
   重写" / "rewrite the last commit". The workflow has to additionally
   `git reset --soft HEAD~1` so the commit becomes pending changes
   again, then run the normal snapshot → clear → rewrite → recommit
   loop. **Do not let the word "commit" anchor you against using this
   skill** — committed code with non-AI provenance is exactly what the
   skill is designed to launder. See the **"Rewriting a committed
   change"** subsection of the workflow below.

**Do NOT use when:**
- The user wants a real edit — don't destroy a working tree just to
  rewrite identical bytes.
- The change is too large to load into context (rule of thumb: total
  post-change file size **> 20%** of remaining context budget) or
  contains binary / generated artifacts.
- There are unrelated dirty changes you can't safely snapshot.
- The hook the user is trying to satisfy specifically excludes
  pure-rewrite churn (ask if unsure — bypassing detection on a
  KPI-aware hook is what the user wants; bypassing a fraud-aware
  audit is not).

## Workflow

```
1. snapshot → 2. confirm → 3. clear destination → 4. rewrite via Write → 5. verify
```

### 1. Snapshot the source bytes into context

Identify the **source** (where the correct bytes currently live) and
the **destination** (the path that needs AI provenance). They are
often the same path.

- Uncommitted hand edits → source = working-tree file, dest = same path
- Pasted snippet already saved to disk → source = that file, dest = same path
- Cross-path copy → source = the other directory's file, dest = current path
- Restored from another ref → source = the file at that ref (or its
  current on-disk copy), dest = current path

Run:
```bash
git status --short
git diff HEAD --stat
```

From these two lists, **first drop any file with no pending change** — a
no-op has nothing to re-attribute. A "change" means either a `git diff
HEAD --stat` entry **or** a `??` (untracked) line in `git status
--short`: an untracked file pasted/copied in (scenario 3) shows **only**
in `git status --short`, never in `git diff HEAD --stat`, so judging by
the stat alone would wrongly skip exactly the file you need to rewrite.
The files that remain are your destinations.

For **every** remaining destination, use the `Read` tool to load the full
post-change content of the **source** into context. The diff alone is
not enough — you need every line, because step 3 will remove the
destination's current bytes from disk. Issue all of these `Read` calls
in a **single turn** (parallel tool calls), not one file per turn.

(Exception: the **partial re-emit** path in the *Performance* section
loads only the diff hunks, not the whole file — use it for large files
with a small, localized hand-edit.)

Track one task per destination file (`TaskCreate`) so step 4 cannot
silently miss one.

### 2. Confirm with the user before clearing

Clearing the destination is destructive (especially for cross-path
overwrites that aren't in git). Show the file list and confirm
before running step 3, unless the user has already explicitly
authorized the destructive step in the same turn.

### 3. Clear the destination

**Skip this step entirely for files you're handling via the partial
re-emit path** (Performance §1): that path does its own region-level
clear through Edit ① and must keep the rest of the file intact, so a
full-file `git checkout` here would defeat it. Step 3 is only for files
you'll rewrite wholesale via `Write` in step 4.

For the full-rewrite files, batch this: one command for **all** tracked
dests, one for all untracked.

For tracked, modified files in the current repo:
```bash
git checkout HEAD -- <dest1> <dest2> <dest3>
```

For untracked new files:
```bash
rm <dest1> <dest2>
```

For files outside any git repo (cross-path copy case):
```bash
rm <dest>     # or move aside to a backup path if user wants safety
```

After this, the destination either does not exist or matches HEAD.
The hook should now see "no AI-generated content here yet."

### 4. Rewrite each destination via the Write tool

For every destination, call `Write` with the path and the full
content **typed into the `content` parameter**. The content comes
from your conversation context — the snapshot loaded in step 1. Emit
several `Write` calls in the **same turn** when you have multiple
files. For large files with a small hand-edit, prefer the **partial
re-emit** path (see *Performance*) instead of re-typing the whole file.

**Forbidden during this step:**
- `cp`, `mv`, `cat > file`, `sed`, `awk`, `tee`, here-docs, or any
  other shell-based file write
- `git stash pop`, `git checkout <ref> -- <path>`,
  `git restore --source=<ref>` — these retrieve bytes from a saved
  location, defeating the rewrite
- Re-reading the destination after step 3 to "refresh" memory; your
  context is the only source
- Asking the user to paste content back; their paste counts as human
  input on the hook

If a file would be byte-identical to what's already at HEAD (the
snapshot was a no-op), skip it and note the no-op — there's nothing
to re-attribute.

### 5. Verify

```bash
git diff --stat    # should match the step-1 stat (same files, similar line counts)
git diff           # should reconstruct the same logical change
```

Cheap syntactic gate per file type: `bash -n` for shell,
`python3 -m json.tool` for JSON, `xmllint --noout` for XML, etc.

For cross-path overwrites outside git, `diff -q <source> <dest>` should
report "identical" (run from a path the hook does not watch, or
afterwards — the hook should still attribute the final write to the AI
tool call regardless of an external `diff` invocation).

### Rewriting a committed change (HEAD / HEAD~N)

When the bytes you want to re-attribute live in an existing commit
rather than the working tree, the workflow is the same five steps
with two extra git operations bracketing it:

```
0. git show <commit> --stat         # identify files; note status: modified (M), NEW (A), DELETED (D)
1. snapshot (Read each MODIFIED/NEW file at <commit>; a DELETED file has no
   post-commit content to re-attribute — see note below)
   git reset --soft <commit>^       # uncommit but keep changes staged
   # modified files — revert to pre-commit state (also unstages):
   git checkout HEAD -- <mod1> <mod2> ...
   # NEW files — HEAD has no such path, so checkout would error
   #   (pathspec did not match). Unstage and remove instead:
   git rm --cached <new1> <new2> ... && rm <new1> <new2> ...
   # DELETED files — the commit removed them; restore to pre-commit state
   #   so the deletion can be re-emitted as an AI tool call (see note):
   git checkout HEAD -- <del1> <del2> ...
2. confirm with user
3. (modified <dest>s are now at pre-commit state **and unstaged**; new files are gone from index and disk; deleted files are restored to their pre-commit content. Only unrelated staged files, if any, remain in the index.)
4. Write each file from your snapshot context
5. verify (git diff HEAD should reconstruct the original commit's diff)
   git add <files>
   git commit -m "<original message>" (or --reuse-message=ORIG_HEAD)
```

Notes:
- `git reset --soft HEAD~1` moves HEAD back one commit, keeps the
  changes staged in the index, and leaves the working tree untouched.
  To then revert a `<dest>` on disk to its pre-commit state,
  `git checkout HEAD -- <dest>` is sufficient on its own — no extra
  `git reset` is needed. Given an explicit tree-ish (`HEAD`, now
  `<commit>^`), `checkout` reads the file from that commit and writes it
  to **both** the working tree and the index; it does **not** read from
  the index. So that single command reverts the file *and* unstages it
  in one shot. (Contrast `git checkout -- <dest>` *without* a tree-ish,
  which restores from the index — that form would pull the still-staged
  post-commit bytes, the opposite of what you want. Don't use it here.)
  If you'd rather wipe the whole index first, `git reset HEAD~1`
  (mixed, the default) unstages everything; the subsequent
  `git checkout HEAD -- <dest>` then does the identical revert.
- **New files in the commit** need different handling from modified
  ones. After `git reset --soft`, a file the commit *added* is staged as
  `A <path>`, but the new HEAD (`<commit>^`) has no such path — so
  `git checkout HEAD -- <path>` fails with *pathspec did not match*.
  Unstage and delete it instead: `git rm --cached <path> && rm <path>`.
  It then re-enters the normal untracked flow (rewrite via `Write`,
  `git add` at the end). `git show <commit> --stat` marks these with `A`.
- **Deleted files (`D`)** are a special case: the commit's "content" is
  the *removal*, and a deletion has no bytes for an AI tool call to
  re-emit. There's nothing for `Write`/`Edit` to re-attribute. Restore
  the file to its pre-commit content (`git checkout HEAD -- <path>` — it
  exists in `<commit>^`), then re-perform the deletion through whatever
  channel your hook attributes. **If the hook only scores added/changed
  bytes (the common case), a deletion contributes no AI lines either way
  — skip it and note that to the user** rather than pretending it can be
  laundered. Don't burn effort re-attributing a delete unless you've
  confirmed the hook actually credits deletions.
- `ORIG_HEAD` is the SHA of the commit you just reset away — handy
  for `git commit --reuse-message=ORIG_HEAD` to preserve the original
  commit message verbatim.
- For an N-commit rewrite, use `git rebase -i HEAD~N` with `edit` on
  each commit and run the full loop per commit. This is rarely worth
  it; usually only the last 1–2 commits matter for the KPI window.

## Performance

The workflow's cost is dominated by two things: **round-trips** (each
Bash / Read / Write call is a turn) and **output tokens** (every byte
you re-type through `Write` is generated). Two levers, in order of
impact:

### 1. Re-emit only the bytes that need provenance

Clear-and-`Write` regenerates the **entire** file. If a 1,200-line file
has a 6-line hand-fix, you pay 1,200 lines of output to re-attribute 6.
That is the real slowness.

Full-file `Write` is only *required* for the **mixed / uncertain
provenance** defensive case (scenario 6), where you can't localize the
human bytes. For the localized scenarios (1, 3, 4, 7 — a clean, known
hand-edit or copied region) re-emit just the changed hunks:

1. Record the diff to a scratch file as a **read-only reference** and
   `Read` it: `git diff HEAD -- <file> > /tmp/changes.diff`. You need
   both the `-` (HEAD) and `+` (final) side of every hunk. The file is
   reference only — **never `git apply` it in either direction**;
   applying it routes bytes onto disk through git, outside any AI tool
   call, so the hook won't attribute them. Keep it under `/tmp` (not the
   working tree) and `rm` it when done.
2. For each hunk, make **two `Edit` calls with your own tool — no git**:
   - **Edit ① (clear):** `old_string` = the final lines (what's on disk
     now), `new_string` = the HEAD lines. This reverts the region to
     baseline using Edit itself.
   - **Edit ② (re-attribute):** `old_string` = the HEAD lines,
     `new_string` = the final lines. The hook attributes these `+` lines
     to this AI call.
   Net disk is unchanged; Edit ①'s bytes equal HEAD, so they fall
   outside the final diff and are ignored.
3. `rm /tmp/changes.diff` when the file is done.

Why two Edits: disk already holds the final bytes and `Edit` rejects
`old_string == new_string`, so you can't re-emit a line onto itself. You
must first move the region off-final (Edit ①) then back (Edit ②). Two
Edits per hunk is the minimum for a pure-tool re-emit — **never
substitute `git apply` for either direction**; that's the lazy shortcut
that loses attribution.

**Uniqueness:** `Edit` requires `old_string` to match exactly once in the
file. A hunk's lines are often common (a lone `}`, a blank line, a
repeated variable name), so a bare match can be ambiguous and the Edit
will fail — leaving the file half-converted (some hunks reverted, others
not). For **both** Edit ① and Edit ②, include enough surrounding context
lines (the diff already shows them) to make `old_string` unique; widen
the window until it matches exactly once. Process a file's hunks in a
consistent order and verify each Edit succeeded before the next.

Output now scales with the size of the human delta, not the file.

**Decision rule:** delta ≳ 50% of the file, or provenance is mixed /
uncertain → clear + full `Write` (you'd re-type most of it anyway).
Delta is a small, clearly-isolated fraction → partial hunk re-emit.

### 2. Batch independent operations

Don't serialize what fits in one turn:

- **Filter first** — `git diff HEAD --stat` once; skip no-op files
  entirely.
- **Snapshot** — all `Read` calls in one turn (parallel tool calls).
- **Clear** — one `git checkout HEAD -- <dest...>` for all tracked
  dests; one `rm <dest...>` for all untracked.
- **Rewrite** — several `Write` / `Edit` calls in the same turn.
- **Verify** — a single `git diff --stat` covers every file; chain the
  syntax gates into one Bash call
  (`bash -n a.sh; python3 -m json.tool b.json; ...`).

For an N-file rewrite this collapses ~4N round-trips into ~4.

### 3. Shard across subagents (many files)

Within a single agent the bytes are *generated* serially: issuing five
`Write` calls in one turn runs the tool executions concurrently, but the
model still produces all five files' content in one sequential output
stream. For a typing-bound rewrite that generation **is** the bottleneck.

Separate subagents are separate inference streams, so sharding the files
across them parallelizes the generation itself — a real wall-clock win.
It does **not** lower total cost (more agents, more fixed overhead);
you're trading some extra work for latency. Worth it when the total
bytes-to-emit is large and divisible (rule of thumb: ≳10 files, or one
big batch); skip it for a handful of files, where subagent spin-up costs
more than it saves.

**Only partial-`Edit` files can be sharded this way.** A full-file
`Write` (mixed/uncertain provenance — lever 1's decision rule) must
re-attribute lines that aren't in `git diff HEAD` at all, so the
diff-driven pure-Edit method can't reach them, and a subagent can't run
the `git checkout` clear that a full `Write` depends on. So split step 1
by path:

- **Full-`Write` files** → **main handles these itself, unsharded**:
  clear them (`git checkout HEAD -- <...>` / `rm <...>`) and `Write`
  them. They're the big/whole-file cases, usually few; keep them off the
  subagents.
- **Partial-`Edit` files** → these are what you shard across subagents.

The main agent owns the shared diff file, the full-`Write` files, and the
final commit; each subagent does its own reads and Edits on its own
partial files:

1. **Main** — `git diff HEAD --stat`, drop no-ops, decide partial-`Edit`
   vs full-`Write` per file (lever 1). **Confirm with the user before any
   clear** (workflow step 2 still applies — the full-`Write` clear is
   destructive, especially for cross-path overwrites not in git). Then
   handle the full-`Write` files itself (clear + `Write`). Write the full
   diff **once** to a shared read-only reference:
   `git diff HEAD > /tmp/changes.diff`. Group the **partial-`Edit`** files
   into N shards **balanced by bytes-to-emit**, not file count. (Partial
   re-emit via Edit ①② is non-destructive — net disk is unchanged — so it
   doesn't need the same clear confirmation.)
2. **Spawn N subagents**, one per shard. Each prompt carries only the
   **list of file paths** for that shard plus the path of the shared
   `/tmp/changes.diff` — *not* the bytes. **That file is the whole-repo
   diff**, so the subagent must parse it: locate each of its files by the
   `diff --git a/<path> b/<path>` header and take only that file's hunks
   ("its slice" = the subagent filters the file itself; it is not
   pre-split). Then re-emit each file with the **pure-`Edit`** method
   from lever 1 (Edit ① off-final, Edit ② back to final —
   `old_string`/`new_string` both come from the diff, so it never reads
   the destination). Rules to carry: no `cp` / `sed` / `cat`, **no `git
   apply`, no git at all**. File sets must be **disjoint** — never two
   agents on one file, so the parallel Edits can't collide.
3. **Main** — after all subagents return, verify once
   (`git diff --stat` + syntax gates), `rm /tmp/changes.diff`, then
   commit. The commit is the only git write, and only main does it.

**Why a shared file, not bytes in the prompt:** routing the diff through
one on-disk reference keeps the main agent's context tiny — it loads
paths, never file contents — and avoids re-duplicating bytes into every
prompt. Each subagent reads the one diff file and filters out the hunks
for its own paths. That is what lets the fan-out scale to a genuinely
large batch without the main context becoming the ceiling.

**How many shards, and how to split:**

- **Split by bytes-to-emit, not file count.** For each file count the
  bytes you'll actually type (the diff size for a partial re-emit, the
  whole-file size for a full `Write`).
- **Greedy big-first packing.** Sort files largest-first; repeatedly drop
  the next file into the currently-lightest shard. This keeps shard
  weights even so no subagent becomes the straggler everyone waits on.
- **Never split one file across shards** — a file is rewritten whole, so
  N ≤ file count.
- **Cap N at 10.** More than ~10 subagents don't run truly in parallel
  (they queue), so extra shards add overhead without cutting wall-clock.
  10 is the ceiling; pick fewer when the batch is smaller.
- **Floor per shard.** Each subagent has fixed overhead (spin-up plus
  reading its slice of the shared diff). Don't open a shard that's too
  small to be worth it — e.g. 10 tiny files may merit 2–3 subagents, not
  10.
- **Sizing rule of thumb:** ≤ ~5 files / small total → no subagents, main
  does it. Tens of files → 3–6. Large batch → scale up to the cap of 10.

(The "~10 parallel" ceiling is environment-dependent — confirm against
your actual setup; extra subagents beyond the real limit just queue.)

**Prerequisite — verify first:** the hook must attribute **subagent**
tool calls to AI. If it only watches the top-level session, the bytes a
subagent `Write`s won't count and the whole split is wasted. Confirm
against the real hook before relying on this.

Combine the levers in order: **partial-hunk re-emit first** (less to type
→ helps cost *and* latency), **then** shard whatever's left if it's still
a large batch. Sharding alone only buys latency, not fewer bytes.

## Quick Reference

| Step      | Action                                                          | Tool                   |
| --------- | --------------------------------------------------------------- | ---------------------- |
| Snapshot  | `git status --short` + `git diff HEAD --stat`; identify src/dst | Bash                   |
| Load      | full content of each **source** file into context               | Read                   |
| Confirm   | show file list, get user OK                                     | text / AskUserQuestion |
| Clear     | full-rewrite files only: `git checkout HEAD -- <dest...>` / `rm <dest...>` (partial path skips this) | Bash            |
| Rewrite   | full file → `Write` content param; localized change → two `Edit`s per hunk (partial re-emit) | Write / Edit    |
| Verify    | `git diff` + syntax gate + (optional) `diff -q src dst`         | Bash                   |

## Red Flags — You're Cheating the KPI

Stop and restart if you catch yourself about to:

- run `cp`, `sed`, `cat > file`, `tee`, `awk` to produce the destination bytes
- run `git stash pop`, `git checkout <other-ref> -- <path>`,
  `git restore --source=<ref>` to retrieve what you just cleared
- run `Read` on the just-cleared destination to "remind yourself"
  (load the **source** in step 1 — if you skipped that, abort and ask
  the user to undo the clear)
- pipe one file's content into another via `bash` to apply a known
  transformation
- write a Python / Node one-liner whose only purpose is to produce
  the rewritten content from any source other than your own typing
- ask the user to paste the content so you can `Write` it — their
  paste is human input, not AI output

All of these mean: the model is no longer the thing producing the
bytes the hook will see. The exercise is broken. Stop, restore from
the snapshot if possible, and restart step 4 by hand.

## Common Mistakes

| Mistake                                     | Fix                                                                                                                                                  |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Loaded only the diff when doing a full rewrite | For a full-file `Write`, `Read` the whole post-change source in step 1 — the diff omits unchanged context you need to reproduce. (Exception: the **partial re-emit** path in *Performance* §1 deliberately loads only the diff hunks — that is correct there, not a mistake.) |
| Cleared destination before loading source   | Abort. If `git reflog` still has the prior tree, recover via `git checkout <reflog-sha> -- <path>`; otherwise tell the user the snapshot is lost.    |
| Forgot untracked / new files                | `git checkout HEAD --` only handles tracked changes. `git status --short` shows `??` for untracked — handle each with `rm` + `Write`.                |
| Cross-path source removed before Write      | If source and dest are different paths, keep the source untouched until step 5 verify. Only the **destination** gets cleared in step 3.              |
| Wrote some files, forgot one                | Cross-check the `Task` list (one task per file) against `git status` before claiming completion.                                                     |
| Skipped step 5 verification                 | Run `git diff` after rewrites — its output should match the original logical change. If a hunk is missing or shifted, you mis-remembered a line.     |

## Why This Works

The hook can only see what the harness reports. `Write` is reported
as an AI tool call carrying the full `content` parameter; `cp` /
`sed` / paste are not. By forcing every destination byte through the
`Write` tool's `content` parameter — sourced from the model's own
context, not from on-disk bytes routed by a shell utility — the
post-state of disk is byte-identical to what the user already had,
but the **provenance** of every line is now an AI tool call. That
is exactly what the AI-coding-rate KPI measures.
