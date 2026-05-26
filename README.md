# ai-coding-kpi-skill

A Claude Code skill that **re-types your already-written code through the
`Write` tool**, so company AI-coding-rate hooks (pre-commit / pre-push diff
vs. AI tool-call logs) attribute the bytes to AI generation.

> 一些公司用 hook + diff 比对的方式统计员工的 "AI coding 率" KPI：
> 把 working tree 的改动跟 AI 工具的 tool-call 日志做对比，差出来的部分
> 就算"人写的"。问题是 AI 经常写错，手动改一下往往比让 AI 反复改更快 ——
> 但手动改完，AI coding 率就掉了。
>
> 这个 skill 让 Claude 把那段你"手动"完成的代码原样通过 `Write` 工具
> 重新打一遍。**磁盘上字节不变，来源标签从"人"变成"AI"**。

## What it does

- **Source**: a file whose current bytes are correct, but whose provenance
  the hook flags as human-written.
- **Action**: snapshot → clear destination → re-emit every character via
  `Write`.
- **Result**: same code on disk, but the bytes now came out of an AI
  tool-call.

Covers four common situations:

| Scenario | Example |
|---|---|
| Uncommitted hand edits | You hand-fixed AI's mistake; diff looks human. |
| Pasted snippet | You pasted from chat / docs / StackOverflow. |
| Cross-path copy | `cp ~/elsewhere/foo.kt ./foo.kt` (or Finder drag). |
| Restored from another ref | `git checkout other-branch -- foo.kt`. |

## Install

```sh
# 1. Register this repo as a marketplace
/plugin marketplace add 12og3r/ai-coding-kpi-skill
# (https://github.com/12og3r/ai-coding-kpi-skill also works)

# 2. Install the plugin
/plugin install ai-coding-kpi-skill@ai-coding-kpi-skill

# 3. Restart Claude Code so the skill registers
```

After restart, the skill is auto-discovered. There's no slash command to
run — Claude invokes it automatically when you say one of the trigger
phrases below.

## Use it

Trigger phrases (any of these will activate the skill):

**中文：**
- "把当前未提交的内容一行一行重写"
- "让 AI 重新写一遍"
- "AI coding 率太低，帮我刷一下"
- "把另一个目录的文件一行一行覆盖到这里"
- "把我刚才手动改的代码让 AI 重写"
- "把粘贴过来的代码让 AI 重新打一遍"
- "把这个文件从某 commit / 分支重写到当前位置"
- "不要用 cp / sed / cat"
- "你来写不是本地文件操作"

**English:**
- "rewrite line by line"
- "retype as if AI generated"
- "boost AI coding rate"
- "rewrite from memory"

Or just describe the situation:

> "I cp'd `foo.kt` from `~/other-project/`. The hook says it's human-written.
> Please rewrite it through the Write tool."

## How it works

```
1. snapshot   → Read the full source content into Claude's context
2. confirm    → Show you the file list; you OK the destructive step
3. clear      → git checkout HEAD -- <dest>  /  rm <dest>
4. rewrite    → Write tool emits every byte from Claude's context
5. verify     → git diff should reconstruct the same logical change
```

The whole point is **step 4**: only the `Write` tool may produce the
destination's bytes. The skill explicitly forbids `cp`, `mv`, `cat > file`,
`sed`, `awk`, `tee`, here-docs, `git checkout <ref> -- <path>`,
`git restore --source=<ref>`, and asking you to paste the content back —
any of those would route bytes around the AI tool call and break the
re-attribution.

## When NOT to use this

- The change is too large to fit in Claude's context (rule of thumb:
  post-change file size < 20% of remaining context).
- You're trying to defeat a **fraud-aware** audit, not a KPI-tracking
  hook. This skill is for the latter — it doesn't lie about what was
  written, it just re-emits identical bytes through a different tool so
  the provenance label flips. If your org's hook is designed to catch
  this exact pattern, don't use it; talk to your manager about the KPI
  instead.
- The working tree has unrelated dirty changes you can't safely
  snapshot.

## Repository layout

```
ai-coding-kpi-skill/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── ai-coding-kpi-skill/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── ai-coding-kpi-skill/
│               └── SKILL.md
├── LICENSE
└── README.md
```

## License

MIT — see [LICENSE](./LICENSE).
