# Context & Resource Optimization Directives for Claude Code

Drop the following block into your project's `CLAUDE.md` to prevent conversation bloating and keep token latency minimal.

```markdown
# Resource & Context Optimization Directives

## 1. Inspection & Handoff (Artifacts First)
- Before diving into deep debug logs, inspect `git status` and `git diff` first.
- If resuming work, inspect summary notes or documentation before scanning extensive raw logs.
- Never cat entire verbose log files or unbounded database dumps into context. Grep for exact error messages or use bounded head/tail commands.

## 2. Search Scope Management & Subagent Isolation
- If a project index or knowledge graph exists (e.g. `graphify`, ctags), query the graph first to locate target symbols rather than running wide searches.
- When exploring unfamiliar codebases across > 10 files, do not read full files sequentially. Delegate deep scans to a subagent or use targeted grep.
- Summarize findings concisely before proceeding to modifications.

## 3. Bounded File Reading & Command Output
- Read only relevant function or class ranges for files over 200 lines.
- Limit terminal output: append `| head -n 40` or equivalent filtering.
- Avoid ingesting minified assets, compiled binaries, raw database dumps, or raw serialized graphs (e.g. `graph.json`).

## 4. Milestone Checkpointing
- In long multi-step workflows, create regular git commits at working milestones.
- Keep persistent indexes updated (e.g. `graphify update .`).
- Keep a clean `task.md` checklist.
- When reaching significant context accumulation, prompt the user to run `/compact` or start a new session referencing the latest git commit and checklist.
```
