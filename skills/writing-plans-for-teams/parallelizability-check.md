# Parallelizability Fitness Check Guide

## Overview

This guide helps determine whether a plan is suitable for team-driven (parallel) execution or should use sequential execution.

## Quick Assessment

| Factor | Team-Ready | Sequential Preferred |
|--------|------------|---------------------|
| Independent tasks | 3+ can start immediately | < 3 independent tasks |
| File boundaries | Each task has own files | Tasks share files heavily |
| Dependencies | < 30% of tasks blocked | > 70% of tasks blocked |
| Integration | Clear interfaces between tasks | Tight coupling between tasks |

## Detailed Criteria

### High Parallelizability → Use agent-team-driven-development

**Characteristics:**
- 3+ tasks with `Depends on: None`
- Tasks grouped by subsystem/file area (e.g., "auth/", "api/", "ui/")
- Each task creates or modifies distinct files
- Clear interfaces between components defined upfront
- Reviews can happen asynchronously

**Example pattern:**
```
Task 1: Auth types (auth/types.py) - Depends on: None
Task 2: API client (api/client.py) - Depends on: None
Task 3: UI components (ui/auth.tsx) - Depends on: None
Task 4: Integration (app/auth.py) - Depends on: 1, 2, 3
```

**Why it works:** Tasks 1-3 can run in parallel. Each implementer claims one task. No file conflicts because each task touches different files.

### Medium Parallelizability → Either approach works

**Characteristics:**
- 2-4 parallel paths possible
- Some independent tasks mixed with sequential chains
- Dependencies exist but don't fully block parallelization
- Some shared configuration or types between tasks

**Example pattern:**
```
Task 1: Core types (types.py) - Depends on: None
Task 2: API client (api.py) - Depends on: 1
Task 3: Database layer (db.py) - Depends on: 1
Task 4: UI (ui.tsx) - Depends on: 1
Task 5: Integration - Depends on: 2, 3, 4
```

**Why it works:** After Task 1 completes, Tasks 2-4 can run in parallel. Partial parallelization.

**Decision factors:**
- Time pressure? → Team-driven for faster completion
- Complexity? → Sequential may be safer
- Team familiarity with codebase? → Team-driven if experienced

### Low Parallelizability → Use subagent-driven-development

**Characteristics:**
- Mostly sequential chain (> 70% of tasks depend on previous)
- Each task depends on previous task's output
- Shared state modifications between tasks
- Complex integration requirements
- Refactoring existing tightly-coupled code

**Example pattern:**
```
Task 1: Extract interface - Depends on: None
Task 2: Implement new backend - Depends on: 1
Task 3: Migrate data - Depends on: 2
Task 4: Update callers - Depends on: 2
Task 5: Remove old code - Depends on: 4
Task 6: Integration test - Depends on: 5
```

**Why sequential is better:** Tasks 2-6 form a chain. Parallelization offers little benefit. Sequential execution is simpler and avoids coordination overhead.

## File Conflict Detection

**Parallel tasks MUST NOT modify the same files.**

**Safe parallelization:**
```
Task A: Create src/auth/login.py
Task B: Create src/api/users.py
→ No conflict, can run in parallel
```

**Unsafe parallelization:**
```
Task A: Modify src/config.py (add AUTH_SETTINGS)
Task B: Modify src/config.py (add API_SETTINGS)
→ Conflict! Must be sequential or merged into one task
```

**Resolution options:**
1. Merge tasks into one
2. Make tasks sequential (add dependency)
3. Split config into separate files

## Dependency Graph Validation

**Valid dependencies:**
- Earlier task defines types/interfaces used by later task
- Earlier task creates module that later task imports
- Earlier task sets up infrastructure that later task uses

**Invalid dependencies (circular):**
```
Task A depends on Task B
Task B depends on Task A
→ Impossible! Plan has an error.
```

**Overly cautious dependencies:**
```
Task A: Create button component
Task B: Create input component (Depends on: A)
→ These are independent! Dependency is unnecessary.
```

## Integration Checkpoint Requirements

**When parallel tasks complete, verify integration:**

| Check | How to Verify |
|-------|---------------|
| Type compatibility | TypeScript compiler passes, or type checks in tests |
| Import resolution | All imports resolve, no missing modules |
| API contracts | Mock tests pass, contract tests pass |
| Data flow | Integration tests that span components |
| Configuration | Shared config is consistent |

**Add checkpoint after each parallel group:**
```markdown
## Integration Checkpoint: Auth Feature

After Tasks 1, 2, 3 complete:
- [ ] Types from Task 1 match imports in Tasks 2, 3
- [ ] API client (Task 2) and UI (Task 3) can communicate
- [ ] Run: `pytest tests/integration/test_auth.py`
```

## Decision Flowchart

```
                    ┌─────────────────┐
                    │ 3+ independent  │
                    │     tasks?      │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │ Yes                         │ No
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │ Clear file      │           │ Sequential      │
    │ boundaries?     │           │ execution       │
    └────────┬────────┘           │ recommended     │
             │                    └─────────────────┘
    ┌────────┴────────┐
    │ Yes             │ No
    ▼                 ▼
┌─────────────────┐  ┌─────────────────┐
│ < 30% tasks     │  │ Medium: Either  │
│ blocked?        │  │ works, prefer   │
└────────┬────────┘  │ sequential for  │
         │           │ safety          │
    ┌────┴────┐      └─────────────────┘
    │ Yes     │ No
    ▼         ▼
┌────────┐  ┌─────────────────┐
│ Team   │  │ Medium: Either  │
│ driven │  │ works, consider │
│        │  │ task grouping   │
└────────┘  └─────────────────┘
```

## Anti-Patterns

**Over-parallelization:**
- Don't force parallelization on tightly-coupled tasks
- Coordination overhead > time savings
- Integration bugs from parallel work > sequential bugs

**Under-parallelization:**
- Missing opportunities when tasks are truly independent
- Sequential execution of tasks that could run concurrently
- Wasted time waiting when work could proceed in parallel

**File conflict ignorance:**
- Not checking file ownership between parallel tasks
- Results in merge conflicts and broken builds
- Always verify file boundaries before declaring parallelizable

## Summary

| Rating | Tasks | Dependencies | Files | Recommendation |
|--------|-------|--------------|-------|----------------|
| High | 3+ independent | < 30% blocked | Distinct per task | Team-driven |
| Medium | 2-4 paths | Mixed | Some overlap | Either, prefer team if time-critical |
| Low | Mostly sequential | > 70% blocked | Shared heavily | Sequential |
