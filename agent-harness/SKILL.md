---
name: agent-harness
description: Use when building a non-trivial application that benefits from separated planning, implementation, and evaluation. Orchestrates a 3-agent team (planner, generator, evaluator) using Claude Code agent teams for long-running application development with sprint-based iteration.
---

# Agent Harness: 3-Agent Team for Application Development

A GAN-inspired development harness that separates planning, generation, and evaluation into independent agent teammates. Based on the [Anthropic engineering blog post on harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps).

## When to Use

- Building a non-trivial application from a brief description (1-4 sentences)
- Any project that benefits from iterative build-test cycles
- Frontend-heavy work where design quality matters (the evaluator catches "AI slop")
- Projects where you want automated QA between implementation sprints

**When NOT to use:**

- Quick bug fixes or small changes (use a single session)
- Pure refactoring with no user-facing changes to evaluate
- Tasks where all work happens in a single file

## Prerequisites

### 1. Enable Agent Teams

Agent teams are experimental. Add to your `settings.json` or environment:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 2. Playwright MCP (recommended)

The evaluator works best with [Playwright MCP](https://github.com/anthropics/mcp-playwright) for browser-based testing. If unavailable, the evaluator falls back to: running test suites → inspecting logs/screenshots → reading code. See `evaluator-prompt.md` for the full fallback hierarchy.

### 3. Display Mode (optional)

For the best experience, use split-pane mode (requires tmux or iTerm2):

```bash
claude --teammate-mode tmux
```

In-process mode works too — use `Shift+Down` to cycle between teammates.

## Directory Structure

The harness uses `.harness/` in the project root for all inter-agent communication:

```
.harness/
├── spec.md                          # Product spec (written by planner)
├── sprints/
│   ├── sprint-1-contract.md         # Success criteria (generator proposes, evaluator reviews)
│   ├── sprint-1-complete.md         # Implementation summary (generator)
│   ├── sprint-1-evaluation.md       # QA assessment (evaluator)
│   ├── sprint-2-contract.md
│   └── ...
└── status.md                        # Current sprint number, state, blocker notes
```

## How to Run the Harness

Tell your Claude Code session to create the agent team. Here's the prompt pattern:

```
Create an agent team for building an application. Read the skill files in
.claude/skills/agent-harness/ for the full workflow.

The app I want: <your 1-4 sentence description>

Spawn three teammates:
1. "planner" — reads planner-prompt.md, expands my description into a full spec
2. "generator" — reads generator-prompt.md, implements sprints based on the spec
3. "evaluator" — reads evaluator-prompt.md, tests each sprint against its contract

Require plan approval for the generator before each sprint implementation.
Coordinate the sprint loop: planner writes spec → generator proposes contract →
evaluator reviews contract → generator implements → evaluator tests → loop.
```

### The Sprint Loop

**Phase 1: Planning**

1. Lead assigns the planner a task: "Expand the user's request into a product spec"
2. Planner reads `planner-prompt.md`, writes `.harness/spec.md`
3. Planner messages the lead when done
4. Lead reviews the spec (or asks the user to review)

**Phase 2: Sprint Contracts**

5. Lead assigns generator: "Propose a sprint contract for sprint N"
6. Generator reads the spec and `sprint-contract-template.md`, writes `.harness/sprints/sprint-N-contract.md`
7. Lead assigns evaluator: "Review this sprint contract for testability"
8. Evaluator messages generator with feedback; they iterate until agreement
9. Lead approves the contract

**Phase 3: Implementation**

10. Lead assigns generator: "Implement sprint N per the contract"
11. Generator implements, commits incrementally, writes `.harness/sprints/sprint-N-complete.md`
12. Generator messages evaluator: "Sprint N ready for evaluation"

**Phase 4: Evaluation**

13. Lead assigns evaluator: "Evaluate sprint N against its contract"
14. Evaluator uses Playwright MCP (or fallback) to test the running app
15. Evaluator writes `.harness/sprints/sprint-N-evaluation.md`
16. If PASS: move to sprint N+1, return to Phase 2
17. If FAIL: evaluator messages generator with specific failures, generator fixes, re-evaluate (max 3 retries)

**Phase 5: Completion**

18. After all sprints pass, lead synthesizes a summary
19. Lead cleans up the team

### Quality Gates via Hooks (Optional)

For stricter enforcement, add hooks to `.claude/hooks.json`:

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "command": "bash -c 'task_id=\"$TASK_ID\"; if echo \"$task_id\" | grep -q \"sprint.*implement\"; then echo \"Run evaluator before marking complete\" && exit 2; fi'",
        "description": "Prevent marking implementation tasks complete before evaluation"
      }
    ]
  }
}
```

## Adaptation Guide

### Adjusting Sprint Size

- **Small apps** (landing page, CLI tool): 2-4 sprints, 5-15 criteria per contract
- **Medium apps** (dashboard, CRUD app): 5-8 sprints, 10-25 criteria per contract
- **Large apps** (full SaaS): 8-15 sprints, 15-35 criteria per contract

### Adjusting Evaluator Strictness

The evaluator is calibrated to be skeptical by default. If it's blocking too aggressively:

- Reduce the pass threshold in `evaluation-template.md` (default: all critical criteria must pass)
- Allow "minor" failures to not block the sprint

If it's too lenient (the main risk):

- Add more specific criteria to contracts
- Include the phrase "You are known for being the harshest reviewer on the team" in the evaluator spawn prompt
- Require screenshots as evidence for every criterion

### Stack-Specific Customization

This harness is stack-agnostic. To specialize:

- Add stack details to the planner spawn prompt (e.g., "Use React + Vite + FastAPI")
- Include framework-specific testing commands in the generator prompt
- Add stack-specific evaluation criteria (e.g., "Check Lighthouse score > 90")

## Supporting Files

| File | Purpose |
|------|---------|
| `planner-prompt.md` | Instructions for the planner teammate |
| `generator-prompt.md` | Instructions for the generator teammate |
| `evaluator-prompt.md` | Instructions for the evaluator teammate |
| `sprint-contract-template.md` | Template for sprint success criteria |
| `evaluation-template.md` | Template for evaluation reports with scoring rubric |
