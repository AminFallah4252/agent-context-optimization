# Practical Examples: Before & After Optimization

This guide illustrates how the directives alter agent behavior in everyday engineering scenarios.

---

## Scenario 1: Resuming a Previous Session from UUID

**User Prompt**:
> *"Hey, can you inspect session `3f8a12bc-891d` and tell me where we left off on the database migration?"*

### ❌ Without Directives (Context Bloat)
1. The agent calls `view_file` on `brain/3f8a12bc-891d/logs/transcript_full.jsonl`.
2. The transcript is 2.8 MB (~750k tokens).
3. The API call times out or exhausts context limits with an HTTP 400 error.
4. If it succeeds, the agent spends 60 seconds processing raw JSON formatting, tool call payloads, and terminal spam before attempting to answer.

### ✅ With Directives (Directive 1 Compliance)
1. **Step 1**: Agent checks `git status` and `git log -n 3` in the workspace to see the last committed state.
2. **Step 2**: Agent reads `brain/3f8a12bc-891d/walkthrough.md` and `task.md` (< 10 KB total).
3. **Step 3**: Agent immediately identifies that Migration #4 was completed and Migration #5 had a pending index check.
4. **Result**: Response delivered in under 3 seconds with minimal token consumption.

---

## Scenario 2: Broad Codebase Survey (50+ Files)

**User Prompt**:
> *"Find all places across the repository where deprecated `fetchLegacyUser()` is called and summarize their error handling patterns."*

### ❌ Without Directives (Root Context Pollution)
1. The agent lists the directory and begins reading 35 different service files sequentially using `view_file`.
2. 180,000 tokens of boilerplate code flood the active context window.
3. For all subsequent turns in the session, every prompt incurs massive token re-transmissions and noticeable latency lag.

### ✅ With Directives (Directive 2: Mandatory Subagent Isolation)
1. Scope threshold estimated: > 10 files involved.
2. The agent launches a subagent (`TypeName="research"`, `Workspace="inherit"`).
3. Inside the subagent sandbox:
   - The subagent runs ripgrep, inspects the 35 files, and categorizes error handling.
   - All intermediate tool calls and file contents remain inside the subagent's sandbox.
4. The subagent returns a concise 30-line synthesized table to the parent agent.
5. The parent agent presents the final response to the user with its root context clean and nimble.

---

## Scenario 3: Inspecting a Large 1,500-Line Controller

**User Prompt**:
> *"Why is `updateUserProfile()` throwing a validation error on line 850 of `UserController.ts`?"*

### ❌ Without Directives
```bash
# Agent runs full read
view_file TargetFile="src/controllers/UserController.ts"
# Dumps 1,500 lines into history
```

### ✅ With Directives (Directive 3: Bounded File Reading)
```bash
# Agent slices relevant symbol range
view_file TargetFile="src/controllers/UserController.ts" StartLine=830 EndLine=890
```
- Ingests only 60 lines instead of 1,500 lines.
- Keeps conversation context tightly focused on the target function logic.

---

## Scenario 4: Reaching 250 Steps in a Complex Refactor

**Situation**: An autonomous agent has completed 265 steps across multiple feature branches. Local database size has grown to 18 MB.

### ❌ Without Directives
- The agent continues working in the bloated session.
- Token latency grows to 40+ seconds per turn.
- The model begins forgetting earlier architectural constraints and misinterprets user feedback.

### ✅ With Directives (Directives 4 & 5: Milestone Rollover)
The agent automatically finishes the immediate milestone and outputs:

```markdown
All unit tests pass for the authentication refactor.

> [!WARNING] **Context Compounding Alert**
> This conversation has reached significant context accumulation (265 steps / 18.2 MB DB). 
> Every subsequent turn re-transmits this accumulated history to the API, increasing latency and network usage.
> **Recommended Action**: Current progress has been checkpointed to git and `task.md`. Let's resume seamlessly in a fresh conversation with zero context overhead!
```
- `walkthrough.md` is updated with the rollover banner.
- The developer opens a fresh session: *"Continue from task.md and latest commit"*.
- The new session starts with pristine performance.
