# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a fork of **Superpowers** - a Claude Code plugin that provides software development workflow skills. The plugin consists of markdown-based "skills" that are automatically loaded into Claude Code sessions via hooks.

## Key Directories

- `skills/` - Core skills (SKILL.md files with YAML frontmatter)
- `agents/` - Specialized subagents (e.g., skeptical-architect-reviewer)
- `.claude/skills/` - Local skills for developing this plugin (writing-skills)
- `hooks/` - Session start hooks that inject skills into Claude
- `tests/` - Skill tests (subagent-driven pressure scenarios)

## Skill Architecture

Each skill is a directory containing `SKILL.md`:

```
skills/
  skill-name/
    SKILL.md              # Required: Main skill file
    supporting-file.*     # Optional: Reference docs, examples
```

### SKILL.md Structure

```markdown
---
name: skill-name
description: Use when [specific triggering conditions]
---

# Skill content...
```

**Frontmatter rules:**
- Only `name` and `description` fields
- Max 1024 characters total
- Description: "Use when..." format, describes ONLY triggers (not workflow)
- Name: letters, numbers, hyphens only

## Developing Skills

**CRITICAL: Skills are developed using TDD for documentation.**

Read `.claude/skills/writing-skills/SKILL.md` for the complete methodology.

### TDD Cycle for Skills

1. **RED** - Run pressure scenario with subagent WITHOUT skill, document failures
2. **GREEN** - Write minimal skill addressing those failures
3. **REFACTOR** - Close loopholes found in testing, re-verify

**No skill without a failing test first.**

### Testing Skills

Skills are tested via subagent pressure scenarios in `tests/`:

- `tests/skill-triggering/` - Skill discovery tests
- `tests/explicit-skill-requests/` - Explicit invocation tests
- `tests/subagent-driven-dev/` - Implementation workflow tests

Run tests by dispatching subagents with pressure scenarios (see `writing-skills/testing-skills-with-subagents.md`).

## Skill Workflow

The standard development workflow:

1. **brainstorming** → Design/spec creation
2. **writing-plans** → Implementation plan
3. **subagent-driven-development** or **executing-plans** → Implementation
4. **test-driven-development** → RED-GREEN-REFACTOR cycle
5. **requesting-code-review** → Review between tasks
6. **finishing-a-development-branch** → Merge/PR workflow

## Plugin Mechanics

The `hooks/session-start` script:
1. Reads `skills/using-superpowers/SKILL.md`
2. Escapes content for JSON
3. Injects as `additionalContext` into Claude's system prompt

Skills are auto-discovered by Claude based on `description` matching the current task.

## Making Changes

1. Edit skill files directly (no build step)
2. Test changes via subagent scenarios before committing
3. Follow the writing-skills TDD methodology for new/modified skills
4. Spec design docs go to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
