---
name: make-check
description: Use when the user invokes "make-check" or asks for a make-check pod, or when high-stakes engineering work (significant features, risky refactors, deep investigations, plans about to be executed) warrants iterative multi-agent adversarial review before being considered done
---

# Make-Check

## Overview

Make-check is a multi-agent engineering pattern: a **pod** of three agents takes one unit of work through repeated rounds of build → adversarial review → arbitration until the third agent declares the work done.

**Core principle:** Separation of duties beats self-review. The agent producing the work cannot also judge it; an independent reviewer hunts for problems; a third agent arbitrates and owns the final call on what gets fixed.

The unit of work can be anything: a plan, an investigation writeup, a refactor, a feature implementation, a bug fix, a piece of documentation. Make-check is agnostic to what's being made — it only structures how it's reviewed.

## When to Use

- The user explicitly invokes "make-check", "make-check pod", or "make-check this"
- A change is high-stakes (production migrations, security-sensitive code, irreversible refactors)
- A plan about to be executed needs a sanity pass before commitment
- An investigation needs independent confirmation before findings are acted on
- The work has several plausible designs and the first agent's choice should be challenged
- Parallel independent units of work warrant separate pods running concurrently

**Do NOT use for:**

- Trivial edits (typos, single-line tweaks, formatting) — the overhead dwarfs the work
- Tasks with hard time pressure where the user wants the boring fastest path
- Work already protected by strong automated checks (comprehensive tests, type system, linting) doing the same job
- Exploratory throwaway code where iteration cost matters more than quality

## The Three Roles

Each role is a fresh subagent dispatch via the `Agent` tool with `subagent_type: "general-purpose"` (unless the work matches a more specific agent type, e.g. `feature-dev:code-reviewer` for the Checker on a code change). Isolated context per role is load-bearing — it's what makes the Checker's review adversarial rather than self-confirming.

### Maker

- Produces the initial **artifact** in round 1; in later rounds, implements the Decider's fix list to produce the next artifact
- The artifact must be concrete and reviewable — actual code/diffs for implementation work, a written plan for planning work, a findings doc for investigations. "I'd do X next" is NOT a valid artifact
- May NOT self-critique or anticipate the Checker — trust the role separation
- Has full edit/bash tools by default for implementation work

### Checker

- Reviews the current artifact adversarially but collaboratively
- Must be given the **same context** as the Maker (codebase access, conventions, project docs) so it can flag drift from conventions, not just internal logic flaws
- Hunts for: logic errors, edge cases, missing tests, security gaps, performance pitfalls, unclear naming, fragile assumptions, missing error handling, broken invariants, convention drift
- Produces a **numbered list** of specific, actionable recommendations — file paths, line numbers, suggested change where possible, severity (blocker / important / nice-to-have), and one-line rationale per item
- Read-only: no Edit/Write tools — keeps its hands off the artifact
- Does NOT prioritize across recommendations; that's the Decider's job
- If genuinely nothing is wrong: explicitly output "no issues found" rather than padding the list

### Decider

- Receives the artifact AND the Checker's full recommendation list
- For each recommendation, decides:
  - **accept** → goes on the fix list unchanged
  - **modify** → the adjusted version goes on the fix list (Decider writes the modified instruction)
  - **reject** → does NOT go on the fix list (with one-line reason for the audit trail)
- May **add fixes** the Checker missed — these go on the fix list too
- The fix list is what the Maker will implement next round; if it's empty, output the literal token `DONE`
- A Checker round with "no issues found" plus a Decider with no additions = automatic `DONE`
- Read-only tools; owns the iteration-stop decision

## The Loop

```
1. Maker → produces v1
2. Checker reviews v(n) → produces recommendations
3. Decider reviews v(n) + recommendations → outputs fix list OR "DONE"
4. If DONE → exit; the pod's final artifact is v(n)
5. Else Maker applies fix list → produces v(n+1)
6. GOTO 2
```

**Iteration cap:** Stop at 5 rounds even if the Decider hasn't signaled DONE. If you hit the cap, surface remaining issues to the user — don't quietly ship. A pod that won't converge is a signal the work needs human input or a different decomposition.

## Pods and Parallelism

- One pod = one Maker + one Checker + one Decider + one unit of work
- Independent units of work → spawn N pods in parallel from the orchestrating session
- Pods do not share state during their loops; the orchestrator only sees each pod's final artifact
- "Independent" means the units don't read or write each other's outputs mid-flight — if they do, run them sequentially or as one larger pod

## Implementation

The orchestrating session drives the loop with real `Agent` tool calls. State (the current artifact, the Checker's recommendations, the Decider's fix list) lives in the orchestrator and is pasted verbatim into each subagent's prompt — subagents have no shared memory.

**Round-1 Maker dispatch:**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Maker — round 1",
  prompt: `You are the MAKER in a make-check pod. Produce the initial
  artifact for review. Do not self-critique or anticipate a reviewer.

  TASK: <original task>

  Output a single response containing the complete artifact:
    - For implementation work: list of files changed with diffs, plus tests
    - For a plan: the full plan doc
    - For an investigation: the findings + evidence`
})
```

**Checker dispatch (each round):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Checker — round N",
  prompt: `You are the CHECKER in a make-check pod. Adversarial but
  collaborative review of the artifact below. Hunt for logic errors,
  edge cases, security gaps, missing tests, performance pitfalls,
  fragile assumptions, convention drift. Read-only — do not edit.

  Output a NUMBERED list. Each item: file:line, suggested change,
  severity (blocker | important | nice-to-have), one-line rationale.
  Do NOT prioritize across items. If nothing is wrong, output exactly:
  "no issues found".

  ARTIFACT:
  <paste artifact verbatim>`
})
```

**Decider dispatch (each round):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Decider — round N",
  prompt: `You are the DECIDER. For each recommendation: accept (goes on
  fix list unchanged), modify (write the adjusted instruction that goes
  on the fix list), or reject (with one-line reason; does NOT go on fix
  list). You may add fixes the Checker missed.

  Output the ordered fix list for the Maker. If the fix list would be
  empty, output exactly: DONE.

  ARTIFACT: <paste artifact>
  RECOMMENDATIONS: <paste Checker output>`
})
```

**Round-N Maker dispatch (applying fixes):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Maker — round N",
  prompt: `You are the MAKER. Apply this fix list to the current
  artifact. Implement each fix faithfully; do not skip or reinterpret.

  CURRENT ARTIFACT: <paste>
  FIX LIST: <paste Decider output>`
})
```

**Orchestrator loop:**

```
artifact = dispatch(Maker, round=1)
for round in 2..5:
    recs = dispatch(Checker, artifact)
    decision = dispatch(Decider, artifact, recs)
    if decision == "DONE": return artifact
    artifact = dispatch(Maker, artifact, decision)
surface_to_user(artifact, "iteration cap reached; remaining concerns: ...")
```

**Large artifacts:** if the artifact won't fit in a prompt, write it to a file (e.g. `/tmp/make-check-pod-N/v3.md`) and pass the path. Each role reads from disk.

**Merging back:** when the pod finishes, the final artifact is the result. For implementation work, the Maker's edits are already on disk; the orchestrator just reports completion. For plans / investigations, paste the final doc into the orchestrator's response or write it to the location the user asked for.

**Parallel pods:** dispatch multiple pod orchestrators concurrently for independent units of work — they don't share state.

**Sequential pods:** for chained work (plan-pod → implementation-pod), the implementation pod's round-1 Maker is given the plan pod's final artifact as input.

## Common Mistakes

| Mistake | Why it breaks make-check |
|---|---|
| Same agent plays Maker and Checker | Eliminates the adversarial review that gives the pattern its value |
| Checker softens findings to be "nice" | Useful issues stay hidden; the pod converges on something mediocre |
| Decider rubber-stamps everything | Becomes a pass-through; no real arbitration happens |
| No iteration cap | Loop can churn indefinitely on aesthetic disagreements |
| Sharing chat context across roles | Defeats independence; Checker absorbs Maker's framing |
| Using make-check on trivial work | Burns tokens and time for no quality gain |
| Skipping the loop after one round | The pattern's value is in iteration, not just one review |
| Maker tries to pre-empt the Checker | Wastes Maker effort on guessing; trust the role separation |

## Quick Reference

| | Maker | Checker | Decider |
|---|---|---|---|
| Reads | Task / fix list | Current artifact | Artifact + recommendations |
| Produces | Artifact | Numbered recommendations | Fix list OR `DONE` |
| Constraints | No self-review | No direct edits | Must justify rejects |
| Context | Fresh subagent | Fresh subagent | Fresh subagent |
| Termination authority | None | None | Owns `DONE` |

## Real-World Fit

Make-check trades latency and tokens for quality. Use it when the cost of a missed issue (production bug, lost data, security incident, bad architectural commitment) outweighs the cost of 2-4 extra subagent rounds. For everything else, single-agent execution is faster and usually fine.
