# Context & Resource Optimization Rules for Cursor / Windsurf

Add these rules to your `.cursorrules` or system prompt to prevent context bloat and keep agent responses fast.

```markdown
# Context & Token Optimization Directives

## 1. Bounded File Reading & Output Hygiene
- Do not read entire files exceeding 150-200 lines if only a specific function or block is needed. Read target line ranges.
- Truncate terminal command outputs with pipes/filters (e.g., `git log -n 5`, `head -n 40`, `Select-Object -First 30`).
- Never load minified bundles, build outputs (`dist/`, `build/`, `bin/`), lockfiles, or binary files into context.

## 2. Broad Search Sandboxing
- If a task requires scanning more than 8-10 files or broad codebase exploration, run targeted ripgrep searches or AST queries rather than loading multiple whole files into the context window.
- Extract only pertinent lines, symbols, or interfaces before proposing edits.

## 3. Session Checkpointing & Rollover
- Multi-turn conversations compound token overhead on every turn.
- When reaching major milestones or after ~25-30 complex turns:
  1. Commit working code to git.
  2. Maintain a concise `task.md` or session summary outlining current status and next steps.
  3. Proactively recommend restarting a fresh chat thread to reset the context window to zero overhead while continuing from git and `task.md`.
```
