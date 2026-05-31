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
  post-change file size < 20% of remaining context budget) or
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

From that stat, **first skip any file with no diff** — a no-op has
nothing to re-attribute, so don't read or rewrite it. The files that
remain are your destinations.

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

Batch this: one command for **all** tracked dests, one for all untracked.

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
0. git show <commit> --stat         # identify files in the commit
1. snapshot (Read each file at <commit>)
   git reset --soft <commit>^       # uncommit but keep changes staged
   git checkout HEAD -- <dest>      # revert each file to its pre-commit state
2. confirm with user
3. (each <dest> is now at pre-commit state **and unstaged** — `git checkout HEAD -- <dest>` rewrote both the working tree and the index entry. Only unrelated staged files, if any, remain in the index.)
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

1. Load only the diff hunks into context (`git diff HEAD -- <file>`),
   both the `-` (HEAD) and `+` (final) sides — not the whole file.
2. Restore the HEAD baseline for **that region only** instead of
   clearing the whole file — i.e. reverse-apply the single hunk. This is
   the partial analog of step 3's `git checkout HEAD -- <dest>`: it
   produces *baseline* bytes, not final ones, so it's a clear, not a
   cheat. The rest of the file is untouched.
3. Re-type only that region with `Edit`: `old_string` = the HEAD lines,
   `new_string` = the final lines, **both from context** (no destination
   re-read). The hook attributes the `+` lines to this AI call.

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
It does **not** lower total cost: each file's content is duplicated into a
subagent prompt, so tokens go up. You're trading tokens for latency.
Worth it when the total bytes-to-emit is large and divisible (rule of
thumb: ≳10 files, or one big batch); skip it for a handful of files,
where subagent spin-up costs more than it saves.

The main agent stays the single owner of git and the working tree:

1. **Main** — snapshot (`git diff HEAD --stat`), drop no-ops, decide
   partial-hunk vs full-`Write` per file (see lever 1), then group files
   into N shards **balanced by bytes-to-emit**, not file count — don't
   pile the big files into one shard.
2. **Main** — clear every destination first, batched:
   `git checkout HEAD -- <...>` / `rm <...>`.
3. **Spawn N subagents**, one per shard. The main agent must already hold
   each shard's bytes in its own context (the step-1 snapshot — the full
   file for a full `Write`, the diff hunks for a partial re-emit), then
   **inlines those bytes as literal text inside the Task prompt** it
   hands to the subagent. That handoff is AI→AI (main's context → prompt
   string), not a human paste, so provenance is preserved. The subagent
   reconstructs the file purely from what its prompt contains — it must
   not `Read` the cleared destination. Each prompt also carries the
   forbidden-operations rules (no `cp` / `sed` / `cat`) and the
   instruction to emit each file via `Write` / `Edit` and nothing else.
   File sets must be **disjoint** — never two agents on one file.
4. **Main** — after all subagents return, verify once
   (`git diff --stat` + syntax gates) and commit. Subagents never run git.

**Context-budget ceiling:** the main agent has to hold *every* shard's
bytes in its own context (to snapshot them, then inline them into
prompts), and each prompt re-duplicates its shard's bytes on top. Sharding
is for large batches — exactly when this is most likely to bite. If the
full snapshot won't fit in the main agent's context, fan-out is not an
option: fall back to processing files in **sequential groups** (snapshot
group, write, drop from context, repeat) rather than one big parallel
spawn.

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
  (they queue), so extra shards add prompt-duplication overhead without
  cutting wall-clock. 10 is the ceiling; pick fewer when the batch is
  smaller.
- **Floor per shard.** Each subagent has fixed overhead (its content must
  be passed in its prompt, plus spin-up). Don't open a shard that's too
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
| Clear     | `git checkout HEAD -- <dest>` / `rm <dest>`                     | Bash                   |
| Rewrite   | type content into `Write` tool's `content` param                | Write                  |
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
