---
name: make-check
description: Runs a pod of three agents (Maker, Checker, Decider) through repeated build → adversarial review → arbitration rounds until the work is independently declared done. Use when the user invokes "make-check" or asks for a make-check pod, or when high-stakes engineering work (significant features, risky refactors, production migrations, deep investigations, plans about to be executed) warrants iterative multi-agent adversarial review before being considered done. The review loop runs once on the completed work, not per phase of a plan.
---

# Make-Check

## Overview

Make-check is a multi-agent engineering pattern: a **pod** of three agents takes one unit of work through repeated rounds of build → adversarial review → arbitration until the third agent declares the work done.

**Core principle:** Separation of duties beats self-review. The agent producing the work cannot also judge it; an independent reviewer hunts for problems; a third agent arbitrates and owns the final call on what gets fixed.

The unit of work can be anything: a plan, an investigation writeup, a refactor, a feature implementation, a bug fix, a piece of documentation. Make-check is agnostic to what's being made — it only structures how it's reviewed.

**Timing:** the review loop runs on the **completed** unit of work, not on each phase of building it. See [When the Loop Runs](#when-the-loop-runs).

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
- **A single phase or stage of a larger multi-phase build** — wait until the whole thing is done (see [When the Loop Runs](#when-the-loop-runs))

## When the Loop Runs

**One pod covers one complete piece of work, reviewed once it is complete.** The Maker still builds in phases; the Checker/Decider loop does not run per phase.

- **Maker:** works through the plan's stages in order, keeping each stage compiling and tested, committing working code as it goes. Phased building is still the right way to build.
- **Checker/Decider:** do not engage until the Maker reports the whole unit of work functionally complete — all stages done, tests green. Then the loop runs over the finished body of work and iterates to `DONE`.

So a five-stage feature is **one** pod with one review loop at the end, not five pods or five review rounds — the Maker's internal stages are invisible to the loop.

**Why:** per-phase review critiques half-finished work, generates fix lists for code a later stage was going to change anyway, and — because each reviewer only ever sees one slice — systematically misses the cross-phase integration bugs that are the expensive ones. A Checker looking at the completed work sees the whole shape of it, including how the phases fit together.

**Review before completion only when:**

- A phase is independently shippable and will be committed/deployed before the rest lands
- A phase is irreversible (schema migrations, data backfills, production deploys, published API changes)
- The overall work is large enough that a flaw found at the end would be prohibitively expensive to unwind — in that case, split it into genuinely separate units of work with their own pods, rather than adding review rounds inside one pod

When in doubt, prefer one pod at the end. Splitting is a decomposition decision made up front, not a mid-build reflex.

## The Three Roles

Each role is a fresh subagent dispatch via the `Agent` tool with `subagent_type: "general-purpose"` (or a more specific agent type when one is available and fits the role — e.g. a code-reviewer agent for the Checker on a code change). Isolated context per role is load-bearing — it's what makes the Checker's review adversarial rather than self-confirming.

### Maker

- Produces the initial **artifact** in round 1; in later rounds, implements the Decider's fix list to produce the next artifact
- Round 1 covers the **whole** unit of work. Build it in phases if the work has phases — stage by stage, each stage compiling and tested — but do not hand anything to the Checker until every phase is done
- The artifact must be concrete, reviewable, and **complete** — actual code/diffs for implementation work, a written plan for planning work, a findings doc for investigations. "I'd do X next", "phase 1 of 4 done", or a partial implementation is NOT a valid artifact
- May NOT self-critique or anticipate the Checker — trust the role separation
- Has full edit/bash tools by default for implementation work

### Checker

- Reviews the current artifact adversarially but collaboratively, as a **finished whole** — including how its parts fit together, not just each part in isolation
- Must be given the **same context** as the Maker (codebase access, conventions, project docs) so it can flag drift from conventions, not just internal logic flaws
- Hunts for: logic errors, edge cases, missing tests, security gaps, performance pitfalls, unclear naming, fragile assumptions, missing error handling, broken invariants, convention drift, and integration seams between the parts the Maker built at different times
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
1. Maker → builds the COMPLETE unit of work (in phases if needed) → v1
2. Checker reviews v(n) → produces recommendations
3. Decider reviews v(n) + recommendations → outputs fix list OR "DONE"
4. If DONE → exit; the pod's final artifact is v(n)
5. Else Maker applies fix list → produces v(n+1)
6. GOTO 2
```

**Iteration cap:** Stop at 5 rounds even if the Decider hasn't signaled DONE. If you hit the cap, surface remaining issues to the user — don't quietly ship. A pod that won't converge is a signal the work needs human input or a different decomposition.

## Pods and Parallelism

- One pod = one Maker + one Checker + one Decider + one **complete** unit of work
- A multi-phase piece of work is one unit, not one unit per phase — the phases are the Maker's build sequence, not pod boundaries
- Independent units of work → spawn N pods in parallel from the orchestrating session
- Pods do not share state during their loops; the orchestrator only sees each pod's final artifact
- "Independent" means the units don't read or write each other's outputs mid-flight — if they do, run them sequentially or as one larger pod

## Implementation

The orchestrating session drives the loop with real `Agent` tool calls. State (the current artifact, the Checker's recommendations, the Decider's fix list) lives in the orchestrator and is pasted verbatim into each subagent's prompt — subagents have no shared memory.

**Model: all three roles MUST run on the strongest available reasoning model.** Prefer `mythos` if it appears in the `Agent` tool's `model` enum; otherwise use `opus`. Never run a make-check role on `sonnet` or `haiku` — the adversarial review and arbitration quality depends on top-tier reasoning, and weaker models cause the Checker to miss issues and the Decider to rubber-stamp. If you cannot pass `model: "opus"` (or `"mythos"`) to the `Agent` tool in the current environment, surface that to the user before proceeding — do not silently fall back.

**Round-1 Maker dispatch:**

```
Agent({
  subagent_type: "general-purpose",
  model: "opus",   // or "mythos" if available
  description: "Make-check Maker — round 1",
  prompt: `You are the MAKER in a make-check pod. Produce the initial
  artifact for review. Do not self-critique or anticipate a reviewer.

  Build the ENTIRE task below before returning. If it has phases or
  stages, work through them in order and keep each one compiling and
  tested — but do not stop and hand back a partial artifact. Review
  happens once, on the completed work.

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
  model: "opus",   // or "mythos" if available
  description: "Make-check Checker — round N",
  prompt: `You are the CHECKER in a make-check pod. Adversarial but
  collaborative review of the artifact below. It is a COMPLETED unit of
  work — review it as a whole, including how its parts fit together.
  Hunt for logic errors, edge cases, security gaps, missing tests,
  performance pitfalls, fragile assumptions, convention drift, and
  integration seams between parts built at different times.
  Read-only — do not edit.

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
  model: "opus",   // or "mythos" if available
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
  model: "opus",   // or "mythos" if available
  description: "Make-check Maker — round N",
  prompt: `You are the MAKER. Apply this fix list to the current
  artifact. Implement each fix faithfully; do not skip or reinterpret.

  CURRENT ARTIFACT: <paste>
  FIX LIST: <paste Decider output>`
})
```

**Orchestrator loop:**

```
artifact = dispatch(Maker, round=1)   # complete unit of work, all phases done
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
| Running a Checker/Decider round after each phase of a plan | Critiques half-finished work, churns on code a later phase rewrites, and still misses cross-phase integration bugs |
| Handing the Checker a partial artifact ("phase 1 of 4 done") | The loop reviews completed work; a partial artifact makes the review's findings provisional and its fix list wasted effort |
| Maker tries to pre-empt the Checker | Wastes Maker effort on guessing; trust the role separation |
| Running any role on `sonnet` / `haiku` to save tokens | Weak reasoning kills review quality; use `opus` (or `mythos` if available), or surface the constraint to the user |

## Quick Reference

| | Maker | Checker | Decider |
|---|---|---|---|
| Reads | Task / fix list | Current artifact | Artifact + recommendations |
| Produces | Complete artifact (built in phases if needed) | Numbered recommendations | Fix list OR `DONE` |
| Constraints | No self-review | No direct edits | Must justify rejects |
| Context | Fresh subagent | Fresh subagent | Fresh subagent |
| Termination authority | None | None | Owns `DONE` |

## Real-World Fit

Make-check trades latency and tokens for quality. Use it when the cost of a missed issue (production bug, lost data, security incident, bad architectural commitment) outweighs the cost of 2-4 extra subagent rounds. For everything else, single-agent execution is faster and usually fine.
