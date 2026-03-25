---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task, with two-stage review after each using skeptical-architect-reviewer.

**Why subagents:** You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "Tasks mostly independent?" [shape=diamond];
    "Stay in this session?" [shape=diamond];
    "subagent-driven-development" [shape=box];
    "executing-plans" [shape=box];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "Tasks mostly independent?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "Tasks mostly independent?" -> "Stay in this session?" [label="yes"];
    "Tasks mostly independent?" -> "Manual execution or brainstorm first" [label="no - tightly coupled"];
    "Stay in this session?" -> "subagent-driven-development" [label="yes"];
    "Stay in this session?" -> "executing-plans" [label="no - parallel session"];
}
```

## The Process

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "Dispatch implementer subagent" [shape=box];
        "Implementer asks questions?" [shape=diamond];
        "Answer questions" [shape=box];
        "Implementer implements, tests" [shape=box];
        "Dispatch skeptical-architect-reviewer (spec)" [shape=box];
        "Spec compliant?" [shape=diamond];
        "Implementer fixes" [shape=box];
        "Dispatch skeptical-architect-reviewer (code)" [shape=box];
        "Code quality OK?" [shape=diamond];
        "Mark task complete" [shape=box];
    }

    "Read plan, extract tasks, create TodoWrite" [shape=box];
    "More tasks?" [shape=diamond];
    "Dispatch skeptical-architect-reviewer (final)" [shape=box];
    "All done" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, extract tasks, create TodoWrite" -> "Dispatch implementer subagent";
    "Dispatch implementer subagent" -> "Implementer asks questions?";
    "Implementer asks questions?" -> "Answer questions" [label="yes"];
    "Answer questions" -> "Dispatch implementer subagent";
    "Implementer asks questions?" -> "Implementer implements, tests" [label="no"];
    "Implementer implements, tests" -> "Dispatch skeptical-architect-reviewer (spec)";
    "Dispatch skeptical-architect-reviewer (spec)" -> "Spec compliant?";
    "Spec compliant?" -> "Implementer fixes" [label="no"];
    "Implementer fixes" -> "Dispatch skeptical-architect-reviewer (spec)";
    "Spec compliant?" -> "Dispatch skeptical-architect-reviewer (code)" [label="yes"];
    "Dispatch skeptical-architect-reviewer (code)" -> "Code quality OK?";
    "Code quality OK?" -> "Implementer fixes" [label="no"];
    "Implementer fixes" -> "Dispatch skeptical-architect-reviewer (code)";
    "Code quality OK?" -> "Mark task complete" [label="yes"];
    "Mark task complete" -> "More tasks?";
    "More tasks?" -> "Dispatch implementer subagent" [label="yes"];
    "More tasks?" -> "Dispatch skeptical-architect-reviewer (final)" [label="no"];
    "Dispatch skeptical-architect-reviewer (final)" -> "All done";
}
```

## Reviews with skeptical-architect-reviewer

All reviews use the skeptical-architect-reviewer agent. Pass appropriate CLAIM:

**Spec compliance review:**
```
Agent({
    name: "skeptical-architect-reviewer",
    prompt: "CLAIM: Implementation at [base_sha]..[head_sha] matches Task N spec: [task_spec_text]"
})
```

**Code quality review:**
```
Agent({
    name: "skeptical-architect-reviewer",
    prompt: "CLAIM: Code at [base_sha]..[head_sha] is well-built and follows project standards"
})
```

**Final review:**
```
Agent({
    name: "skeptical-architect-reviewer",
    prompt: "CLAIM: All tasks from plan [plan_path] are complete and integrated"
})
```

## Model Selection

Use the least powerful model that can handle each role:

**Mechanical implementation** (isolated functions, clear specs): fast, cheap model

**Integration and judgment** (multi-file, debugging): standard model

**Architecture, design, and review**: most capable model

## Handling Implementer Status

**DONE:** Proceed to spec compliance review.

**DONE_WITH_CONCERNS:** Read concerns. If about correctness, address before review. If observations, note and proceed.

**NEEDS_CONTEXT:** Provide missing context and re-dispatch.

**BLOCKED:** Assess blocker:
1. Context problem → provide more context, re-dispatch same model
2. Reasoning problem → re-dispatch with more capable model
3. Task too large → break into smaller pieces
4. Plan wrong → escalate to human

**Never** ignore an escalation or force retry without changes.

## Prompt Templates

- `./implementer-prompt.md` - Dispatch implementer subagent

## Red Flags

**Never:**
- Start implementation on main/master without explicit consent
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed issues
- Dispatch multiple implementers in parallel (conflicts)
- Make subagent read plan file (provide full text)
- Skip scene-setting context
- Ignore subagent questions
- Accept "close enough" on spec compliance
- Skip review loops
- Let implementer self-review replace actual review
- Start code quality review before spec compliance passes
- Move to next task while review has open issues

**Always:**
- Two-stage review per task (spec → code quality)
- Re-review after fixes
- Answer implementer questions before they proceed

<HARD-GATE>
**Do NOT proceed to the next task until both skeptic reviews (spec + code quality) PASS for the current task.**

**Why:** A small misunderstanding in Task 1 becomes wrong assumptions in Task 2, wrong interfaces in Task 3, and by Task 15 you're debugging a cascade of compounding issues. Early reviews are cheap; late debugging is expensive.

No exceptions. No "this task is simple." No "I'll review after." Both reviews must pass before moving on.
</HARD-GATE>

## Integration

**Required workflow skills:**
- **superpowers:writing-plans** - Creates the plan this skill executes

**Subagents should use:**
- **superpowers:test-driven-development** - TDD for each task

**Alternative workflow:**
- **superpowers:executing-plans** - Parallel session instead of same-session
