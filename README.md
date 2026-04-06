---
name: ultraplan
description: Deep planning mode — spawn an isolated Opus sub-agent for 10-30 minutes of focused strategic thinking. Produces a detailed, reviewable plan before any execution. Use for complex tasks that need architectural thinking, multi-step strategies, or risk analysis before committing to action. Complements pipeline-orchestrator (ULTRAPLAN plans, pipeline executes).
---

# ULTRAPLAN — Deep Strategic Planning

Inspired by Claude Code's ULTRAPLAN feature. Offloads complex planning to a dedicated Opus session with extended timeout, producing a detailed plan you review and approve before execution.

## When to Use

**Use ULTRAPLAN when:**
- Task is complex enough that jumping in would waste time
- Architecture decisions needed before coding
- Multi-week project needs a roadmap
- Risk analysis needed before a big change
- You want to think through all angles before committing
- Task would benefit from 10-30 minutes of uninterrupted deep thought

**Don't use when:**
- Task is straightforward (just do it)
- You already know the plan
- Quick fix or small change
- Time-sensitive (ULTRAPLAN takes 10-30 min)

## How It Works

```
User: "ultraplan: migrate council-room from Fastify to Hono"
         ↓
  ┌──────────────────────────────┐
  │  Spawn isolated Opus session │
  │  Timeout: 30 minutes         │
  │  Mode: PLAN ONLY (no exec)   │
  └──────────────────────────────┘
         ↓
  Agent reads context files, analyzes codebase,
  considers risks, explores alternatives...
  (10-30 minutes of deep thinking)
         ↓
  ┌──────────────────────────────┐
  │  Output: Structured Plan     │
  │  saved to state/ultraplan/   │
  └──────────────────────────────┘
         ↓
  User reviews plan → Approve / Reject / Modify
         ↓
  If approved → Feed into pipeline-orchestrator for execution
```

## Invocation

**Trigger phrases:**
- `"ultraplan: <task description>"`
- `"deep plan: <task>"`
- `"think deeply about: <task>"`

**With context files:**
- `"ultraplan: refactor auth system"` + attach relevant files

## The ULTRAPLAN Prompt

When spawning the sub-agent, use this prompt structure:

```
# ULTRAPLAN: Deep Strategic Planning

You have up to 30 minutes to think deeply about this task. Do NOT execute anything.
Your job is to produce a comprehensive, actionable plan.

## Task
{user's task description}

## Context
{relevant files, codebase structure, constraints}

## Your Output Must Include

### 1. Problem Analysis
- What exactly needs to be done?
- What are the constraints?
- What's the current state?
- What are the unknowns?

### 2. Approach Options
For each viable approach:
- Description
- Pros / Cons
- Risk level (low/medium/high)
- Estimated effort
- Dependencies

### 3. Recommended Approach
- Which option and why
- Key assumptions (state them explicitly)

### 4. Detailed Execution Plan
For each phase:
- What to do
- Expected output
- Dependencies on prior phases
- Verification criteria
- Estimated time

### 5. Risk Mitigation
- What could go wrong?
- How to detect it early?
- Fallback plan for each risk

### 6. Resource Requirements
- Files to modify
- New files to create
- Dependencies to install
- External services needed
- Estimated total time

### 7. Open Questions
- What do you need clarified before execution?
- What assumptions need validation?

## Rules
- DO NOT execute any code or make any changes
- DO NOT use tools that modify files (write, edit, exec)
- You MAY use read, web_search, web_fetch to gather information
- Focus on THINKING, not DOING
- Be thorough — this plan will be executed by another agent
- State assumptions explicitly
- If the task is too vague, list what needs clarification
```

## Agent Procedure

When user triggers ULTRAPLAN:

```
1. Parse task description
2. Identify relevant context files (ask if unclear)
3. Spawn isolated sub-agent:
   - Model: Opus (highest capability)
   - Timeout: 1800 seconds (30 min)
   - Mode: run (one-shot)
   - Task: ULTRAPLAN prompt + context
4. Wait for completion (sessions_yield)
5. Save plan to state/ultraplan/{id}/plan.md
6. Present summary to user
7. Ask: Approve / Reject / Modify?
8. If approved and pipeline-orchestrator is available:
   - Offer to feed plan into pipeline for execution
```

### Spawning the Sub-Agent

```
sessions_spawn:
  mode: run
  runTimeoutSeconds: 1800
  task: [ULTRAPLAN prompt with task + context]
```

## State Management

Plans saved to `~/.openclaw/workspace/state/ultraplan/`:

```
state/ultraplan/
├── {id}/
│   ├── plan.md          — The full plan document
│   ├── meta.json        — Metadata (task, timestamp, status)
│   └── feedback.md      — User's feedback/modifications (if any)
```

**meta.json:**
```json
{
  "id": "up-abc123",
  "task": "Original task description",
  "status": "planning | review | approved | rejected | executing",
  "created": "ISO timestamp",
  "completedAt": "ISO timestamp",
  "approvedAt": null,
  "executionPipelineId": null,
  "thinkingTimeMinutes": 12.5
}
```

## Plan Quality Checklist

Before presenting to user, verify the plan includes:
- [ ] Clear problem statement
- [ ] At least 2 approach options compared
- [ ] Recommended approach with justification
- [ ] Phase-by-phase execution steps
- [ ] Risk assessment with mitigations
- [ ] Time estimates per phase
- [ ] List of files to modify/create
- [ ] Open questions flagged

If any are missing, note it when presenting.

## Integration with Pipeline Orchestrator

After approval, the plan can be fed directly into pipeline-orchestrator:

```
Plan phases → Pipeline PLAN stage (pre-filled)
Plan acceptance criteria → Pipeline PRD stage (pre-filled)
```

This skips the pipeline's own planning phase — ULTRAPLAN already did the thinking.

## Examples

### Example 1: Architecture Decision
```
User: ultraplan: should we migrate from CouchDB to PostgreSQL for the council memory?

→ Agent spends 15 min analyzing:
  - Current CouchDB usage patterns
  - Data model compatibility
  - Migration complexity
  - Performance implications
  - Cost differences
  
→ Produces plan with 3 options: stay, migrate, hybrid
→ Recommends hybrid with migration timeline
```

### Example 2: Feature Development
```
User: ultraplan: add real-time collaboration to MAGI vault

→ Agent spends 25 min thinking through:
  - WebSocket vs SSE vs polling
  - Conflict resolution (CRDT vs OT)
  - Current architecture constraints
  - Incremental rollout plan
  
→ Produces 6-phase plan with risk matrix
→ User approves, feeds into pipeline
```

### Example 3: Cost Optimization
```
User: ultraplan: reduce our Bedrock costs by 50% without losing quality

→ Agent spends 20 min analyzing:
  - Current usage patterns per agent
  - Model routing opportunities
  - Caching strategies
  - Prompt optimization
  
→ Produces tiered plan: quick wins (week 1), medium effort (week 2-3), major refactors (month 2)
```

## Cost Advisory

ULTRAPLAN sessions are expensive:
- 30 min Opus session ≈ $0.50-2.00 depending on context size
- Use judiciously — not for simple tasks
- Consider: "Is this task complex enough to justify 30 min of Opus thinking?"
- Rule of thumb: if execution would take >4 hours, ULTRAPLAN is worth it
