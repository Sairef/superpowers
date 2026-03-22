---
name: dispatching-parallel-agents
description: Use when facing 2+ independent tasks that can be worked on without shared state or sequential dependencies
---

# Dispatching Parallel Agents

## Overview

You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

When you have multiple unrelated failures (different test files, different subsystems, different bugs), investigating them sequentially wastes time. Each investigation is independent and can happen in parallel.

**Core principle:** Dispatch one agent per independent problem domain. Let them work concurrently.

## When to Use

```dot
digraph when_to_use {
    "Multiple failures?" [shape=diamond];
    "Are they independent?" [shape=diamond];
    "Single agent investigates all" [shape=box];
    "One agent per problem domain" [shape=box];
    "Can they work in parallel?" [shape=diamond];
    "Sequential agents" [shape=box];
    "Parallel dispatch" [shape=box];

    "Multiple failures?" -> "Are they independent?" [label="yes"];
    "Are they independent?" -> "Single agent investigates all" [label="no - related"];
    "Are they independent?" -> "Can they work in parallel?" [label="yes"];
    "Can they work in parallel?" -> "Parallel dispatch" [label="yes"];
    "Can they work in parallel?" -> "Sequential agents" [label="no - shared state"];
}
```

**Use when:**
- 3+ test files failing with different root causes
- Multiple subsystems broken independently
- Each problem can be understood without context from others
- No shared state between investigations

**Don't use when:**
- Failures are related (fix one might fix others)
- Need to understand full system state
- Agents would interfere with each other

## The Pattern

### 1. Identify Independent Domains

Group failures by what's broken:
- File A tests: Tool approval flow
- File B tests: Batch completion behavior
- File C tests: Abort functionality

Each domain is independent - fixing tool approval doesn't affect abort tests.

### 2. Create Focused Agent Tasks

Each agent gets:
- **Specific scope:** One test file or subsystem
- **Clear goal:** Make these tests pass
- **Constraints:** Don't change other code
- **Expected output:** Summary of what you found and fixed

### 3. Dispatch in Parallel

```typescript
// In Claude Code / AI environment
Task("Fix agent-tool-abort.test.ts failures")
Task("Fix batch-completion-behavior.test.ts failures")
Task("Fix tool-approval-race-conditions.test.ts failures")
// All three run concurrently
```

### 4. Review and Integrate

When agents return:
- Read each summary
- Verify fixes don't conflict
- Run full test suite
- Integrate all changes

## Agent Prompt Structure

Good agent prompts are:
1. **Focused** - One clear problem domain
2. **Self-contained** - All context needed to understand the problem
3. **Specific about output** - What should the agent return?

```markdown
Fix the 3 failing tests in src/agents/agent-tool-abort.test.ts:

1. "should abort tool with partial output capture" - expects 'interrupted at' in message
2. "should handle mixed completed and aborted tools" - fast tool aborted instead of completed
3. "should properly track pendingToolCount" - expects 3 results but gets 0

These are timing/race condition issues. Your task:

1. Read the test file and understand what each test verifies
2. Identify root cause - timing issues or actual bugs?
3. Fix by:
   - Replacing arbitrary timeouts with event-based waiting
   - Fixing bugs in abort implementation if found
   - Adjusting test expectations if testing changed behavior

Do NOT just increase timeouts - find the real issue.

Return: Summary of what you found and what you fixed.
```

## Common Mistakes

**❌ Too broad:** "Fix all the tests" - agent gets lost
**✅ Specific:** "Fix agent-tool-abort.test.ts" - focused scope

**❌ No context:** "Fix the race condition" - agent doesn't know where
**✅ Context:** Paste the error messages and test names

**❌ No constraints:** Agent might refactor everything
**✅ Constraints:** "Do NOT change production code" or "Fix tests only"

**❌ Vague output:** "Fix it" - you don't know what changed
**✅ Specific:** "Return summary of root cause and changes"

## When NOT to Use

**Related failures:** Fixing one might fix others - investigate together first
**Need full context:** Understanding requires seeing entire system
**Exploratory debugging:** You don't know what's broken yet
**Shared state:** Agents would interfere (editing same files, using same resources)

## Task Tool vs Teammates

When you have parallel work, choose the right tool:

### Use Task Tool (this skill) when:

- **One-shot investigations** - Each agent does one thing and done
- **No coordination needed** - Agents work independently, no communication
- **Truly isolated** - No shared context benefit between agents
- **Simple parallel dispatch** - Just fire and forget

```
Task({ description: "Fix test A", prompt: "..." })
Task({ description: "Fix test B", prompt: "..." })
Task({ description: "Fix test C", prompt: "..." })
// All run in parallel, no coordination
```

### Use Teammates (agent-team-driven-development) when:

- **Multiple related tasks per agent** - Agents do several things
- **Shared context helps** - Agents benefit from accumulated knowledge
- **Coordination needed** - Agents communicate about work
- **Integration verification** - Need to verify parallel work integrates correctly

```
TeamCreate({ team_name: "feature-team" })
Agent({ team_name: "feature-team", name: "implementer-1", prompt: "..." })
Agent({ team_name: "feature-team", name: "implementer-2", prompt: "..." })
// Teammates persist, share TaskList, coordinate via SendMessage
```

### Decision Flowchart

```dot
digraph task_vs_teammate {
    "Parallel work needed?" [shape=diamond];
    "Multiple tasks per agent?" [shape=diamond];
    "Need coordination?" [shape=diamond];
    "Use Task tool\n(this skill)" [shape=box];
    "Use Teammates\n(agent-team-driven-development)" [shape=box style=filled fillcolor=lightgreen];

    "Parallel work needed?" -> "Multiple tasks per agent?" [label="yes"];
    "Parallel work needed?" -> "No parallel dispatch needed" [label="no"];
    "Multiple tasks per agent?" -> "Need coordination?" [label="yes"];
    "Multiple tasks per agent?" -> "Use Task tool\n(this skill)" [label="no"];
    "Need coordination?" -> "Use Teammates\n(agent-team-driven-development)" [label="yes"];
    "Need coordination?" -> "Use Task tool\n(this skill)" [label="no"];
}
```

### Comparison

| Factor | Task Tool | Teammates |
|--------|-----------|-----------|
| Persistence | Disposable | Persist across tasks |
| Coordination | None | SendMessage, shared TaskList |
| Context | Isolated | Shared team context |
| Best for | One-shot investigations | Multi-task implementation |
| Overhead | Low | Higher (team setup) |
| Integration checks | Manual | Built-in workflow |

### Example Scenarios

**Task Tool - Debugging multiple test failures:**
```
3 test files failing independently →
Task("Fix test A") + Task("Fix test B") + Task("Fix test C")
→ Each agent fixes one file, done
```

**Teammates - Implementing a feature with parallel components:**
```
Feature with auth, API, UI components →
TeamCreate → implementer-1 (auth) + implementer-2 (API) + implementer-3 (UI)
→ Reviewers verify integration, teammates coordinate on interfaces
```

## Real Example from Session

**Scenario:** 6 test failures across 3 files after major refactoring

**Failures:**
- agent-tool-abort.test.ts: 3 failures (timing issues)
- batch-completion-behavior.test.ts: 2 failures (tools not executing)
- tool-approval-race-conditions.test.ts: 1 failure (execution count = 0)

**Decision:** Independent domains - abort logic separate from batch completion separate from race conditions

**Dispatch:**
```
Agent 1 → Fix agent-tool-abort.test.ts
Agent 2 → Fix batch-completion-behavior.test.ts
Agent 3 → Fix tool-approval-race-conditions.test.ts
```

**Results:**
- Agent 1: Replaced timeouts with event-based waiting
- Agent 2: Fixed event structure bug (threadId in wrong place)
- Agent 3: Added wait for async tool execution to complete

**Integration:** All fixes independent, no conflicts, full suite green

**Time saved:** 3 problems solved in parallel vs sequentially

## Key Benefits

1. **Parallelization** - Multiple investigations happen simultaneously
2. **Focus** - Each agent has narrow scope, less context to track
3. **Independence** - Agents don't interfere with each other
4. **Speed** - 3 problems solved in time of 1

## Verification

After agents return:
1. **Review each summary** - Understand what changed
2. **Check for conflicts** - Did agents edit same code?
3. **Run full suite** - Verify all fixes work together
4. **Spot check** - Agents can make systematic errors

## Real-World Impact

From debugging session (2025-10-03):
- 6 failures across 3 files
- 3 agents dispatched in parallel
- All investigations completed concurrently
- All fixes integrated successfully
- Zero conflicts between agent changes
