# Context & Resource Optimization Directives

These directives govern conversation context management, past session handoffs, and tool exploration scope to prevent context window bloat and unnecessary high-volume telemetry re-transmission.

---

## Directive 1: Past Conversation UUID Investigation & Handoff (Artifacts First)

Whenever a user provides a conversation UUID (or asks to inspect, resume, or debug a previous session):

1. **Step 1: Check Version Control State First**
   - Run `git status`, `git log -n 5 --stat`, or `git diff` on the target workspace.
   - The repository code and commit history often already reflect the final state of the previous session.

2. **Step 2: Checkpoint Artifacts First**
   - Look in the previous session's artifact folder: `~/.gemini/antigravity/brain/<uuid>/` (or platform equivalent).
   - Read concise summary artifacts first:
     - `walkthrough.md`: Changes made, validation results, and verification status.
     - `task.md`: Remaining checklist and open tasks.
     - `implementation_plan.md`: Technical architectural decisions.
   - These structured artifacts summarize hundreds of steps in < 15 KB.

3. **Step 3: Strictly Prohibit Full Transcript Dumps**
   - **DO NOT** call `view_file` to read `transcript.jsonl` or `transcript_full.jsonl` in full.
   - If specific details (e.g. an exact error message, traceback, or command output) are missing from the artifacts:
     - Use targeted `grep_search` for specific keywords on `transcript.jsonl`.
     - Read only bounded line ranges (`StartLine`/`EndLine`).
   - If deep, multi-step investigation of raw logs is unavoidable, **delegate it to an isolated subagent** (see Directive 2).

---

## Directive 2: Search Scope Assessment & Mandatory Subagent Isolation

Before executing wide searches, repository exploration, or multi-file inspections, the agent must estimate the search scope and prioritize indexed lookups:

1. **Knowledge Graph & Index Pre-Query (Graphify / AST First)**:
   - If a knowledge graph or semantic index exists in the project (e.g., `graphify-out/graph.json` or `graphify-out/wiki/index.md`):
     - **Query the graph first** via CLI/MCP (`graphify query`, `graphify path`, `graphify explain`) to retrieve a scoped subgraph.
     - A successful graph query typically resolves the target context in < 20 lines, bypassing multi-file scans entirely.
     - If `graphify-out/wiki/index.md` exists, navigate relevant community articles instead of scanning raw codebase trees.

2. **Scope Thresholds (When to Delegate to Subagents)**:
   - Touching or reading **more than 8–10 files**.
   - Scanning large directory trees or unstructured logs.
   - Performing full codebase extractions or unindexed multi-document surveys (e.g. running full `/graphify` semantic extraction).
   - Investigating another conversation's raw logs or trajectory (`brain/<uuid>/`).
   - Deep codebase research or broad external documentation surveys.

3. **Mandatory Subagent Delegation**:
   - When the search scope exceeds the threshold, the agent **MUST invoke an isolated subagent** (e.g., `invoke_subagent` with `TypeName="research"` or `TypeName="self"` and `Workspace="inherit"`).
   - **Why**: Subagents run with their own isolated context. All intermediate tool calls, verbose file reads, and search outputs stay inside the subagent's sandbox.
   - **Handoff Requirement**: The subagent must return **only a compact summary** (findings, target file paths, specific line numbers, or synthesized answers) back to the parent agent. The parent agent's context remains clean and lightweight.

---

## Directive 3: Bounded File Reading & Output Hygiene

1. **Slice Large Files**:
   - Avoid reading entire files exceeding 200 lines. Use `StartLine` and `EndLine` to read only relevant symbol ranges or functions.
2. **Command Output Limiting**:
   - Limit verbose shell outputs using pipes or flags (e.g. `git log -n 5`, `head -n 50`, `Select-Object -First 30`).
3. **No Minified / Binary / Bundle / Raw Graph Dumps**:
   - Never load minified bundles, build outputs (`dist/`, `build/`, `bin/`), raw database files, or raw serialized graphs (e.g. calling `view_file` on `graphify-out/graph.json` or an un-sliced `GRAPH_REPORT.md`) into context. Always query graphs through their dedicated CLI/MCP tools.

---

## Directive 4: Milestone Checkpointing (The 250–300 Step Rule)

- Autonomous multi-turn conversations compound context size on every step.
- When an ongoing session approaches **250–300 steps** or completes a significant milestone:
  1. Commit working code to git.
  2. Keep persistent knowledge graphs synchronized (e.g. run `graphify update .` to refresh AST state without LLM token cost).
  3. Write/update `walkthrough.md` and `task.md` summarizing progress and next steps.
  4. Suggest rolling over to a fresh session: *"Milestone complete. A fresh session can pick up from `task.md` and git commits with zero context overhead."*

---

## Directive 5: Context Size Threshold Warning & Proactive Rollover Alert

To prevent severe latency degradation and multi-gigabyte upload compounding, the agent must actively monitor conversation size metrics.

1. **Threshold Criteria**:
   A session is considered to exceed safe context bounds when ANY of the following are met:
   - Trajectory log (`brain/<uuid>/.system_generated/logs/transcript_full.jsonl` or `transcript.jsonl`) exceeds **1.5 MB – 2.0 MB** (~500k tokens).
   - Conversation database (`conversations/<uuid>.db`) exceeds **15 MB – 20 MB**.
   - Session step count reaches or exceeds **250 – 300 steps**.

2. **Mandatory Notification Upon Prompt Completion**:
   When the running prompt finishes and the threshold has been crossed, the agent **MUST deliver a two-channel warning**:
   
   - **Channel A: In-Chat Warning Alert**:
     Include a prominent GitHub-style warning block at the end of the chat response:
     > [!WARNING] **Context Compounding Alert**
     > This conversation has reached significant context accumulation (> 250 steps / > 1.5 MB trajectory / > 15 MB DB). 
     > Every subsequent turn re-transmits this accumulated history to the API, increasing latency and network usage.
     > **Recommended Action**: Checkpoint current work (`task.md` & git commit) and continue in a fresh conversation.

   - **Channel B: Walkthrough Artifact Notice**:
     If a `walkthrough.md` artifact is being created or updated in this turn, include an alert banner at the top of the document:
     > [!IMPORTANT] **Context Rollover Checkpoint**
     > This session has crossed the 250+ step / 15MB threshold. A fresh conversation can resume seamlessly using this `walkthrough.md` and `task.md` with zero context overhead.
