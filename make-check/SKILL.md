---
name: make-check
description: Use when the user invokes "make-check" or asks for a make-check pod, or when high-stakes engineering work (significant features, risky refactors, production migrations, deep investigations, plans about to be executed) warrants independent multi-agent adversarial review (Maker / Checker / Decider) before being considered done.
---

# Make-Check

## Overview

Make-check is a multi-agent engineering pattern: a **pod** of three roles (Maker, Checker, Decider) takes one unit of work through build → adversarial review → arbitration until the third role independently declares the work done.

**Core principle:** Separation of duties provides independent verification. While the Maker is free (and encouraged) to self-check, test, and validate its own output to catch obvious issues early, the agent producing the work cannot be the sole judge; an independent reviewer hunts for blind spots and edge cases, and a third agent arbitrates and owns the final call on what gets fixed.

**Batched review cadence.** Review is batched so the Maker builds chunks of the plan between reviews: early checkpoints catch direction errors before they propagate, and the full review loop runs once on the completed whole. Review effort concentrates where independent review matters most — systemic direction errors and cross-phase integration seams — not on per-phase rework.

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
- **A single phase of a larger build as its own pod** — phases are the Maker's build sequence; review points are set by the [Review Cadence](#review-cadence)

## Review Cadence

The Maker builds in **batches** of phases. Review points are scheduled up front by plan size:

| Plan size | Review points |
|---|---|
| 1–3 phases | Full review loop at completion only |
| 4–9 phases | Checkpoint after ~the first 2 phases, then full loop at completion |
| 10+ phases | Checkpoints after ~20% and ~60% of the phases, then full loop at completion |

Worked example: a 10-phase plan (5 steps per phase) → Maker builds phases 1–2 → **checkpoint** → phases 3–6 → **checkpoint** → phases 7–10 → **completion loop** to `DONE`.

There are two kinds of review point:

- **Checkpoint — single pass.** One Checker pass + one Decider pass, scoped to the phases built so far. The fix list is applied by the Maker at the start of the next batch; if nothing needs fixing the Decider outputs `CONTINUE`. Checkpoints never iterate.
- **Completion loop — iterates.** The full Checker → Decider → Maker loop over the finished whole, repeating until the Decider outputs `DONE` (cap 5 rounds).

**Why an early checkpoint:** cheap insurance against systemic errors — wrong architecture, misread requirements, convention drift — being propagated across every remaining phase. **Why not per-phase:** per-phase review critiques half-finished work, generates fixes for code a later phase rewrites anyway, and still misses the cross-phase integration bugs that matter most.

**Pull a review earlier than the cadence only when** a phase is independently shippable and will be committed/deployed before the rest lands, or a phase is irreversible (schema migrations, data backfills, production deploys, published API changes).

## The Three Roles

Each role is a fresh subagent dispatch via the `Agent` tool with `subagent_type: "general-purpose"` (or a more specific agent type when one is available and fits the role — e.g. a code-reviewer agent for the Checker on a code change). Isolated context per role is load-bearing — it's what makes the Checker's review adversarial rather than self-confirming.

### Maker

- Builds the unit of work in batches per the cadence — within a batch, works phase by phase, keeping each phase compiling and tested, committing working code as it goes
- At the start of each batch after the first, applies the previous checkpoint's fix list **before** building new phases
- In the completion loop, implements the Decider's fix list (equipped with diagnostic context from the Checker and out-of-scope boundaries from the Decider) to produce the next version
- Every hand-off to a review point must be concrete and reviewable: actual code/diffs with test and build output pasted verbatim for implementation work, a written plan for planning work, a findings doc for investigations — plus, at checkpoints, one line stating which phases are built and which are not
- Self-checking, running automated tests, and validating work before handoff is encouraged to catch obvious issues early
- May be **several Makers running concurrently** when phases are independent (see [Parallel Makers](#parallel-makers))
- Has full edit/bash tools by default for implementation work

### Checker

- Reviews adversarially but collaboratively, and must be given the **same context** as the Maker (codebase access, conventions, project docs) so it can flag drift from conventions, not just internal logic flaws
- At a **checkpoint**: reviews the phases built so far as a coherent partial whole — direction, architecture, conventions, and integration of what exists. Unbuilt phases are NOT findings; never flag planned-but-unbuilt work as missing
- At **completion (Round 1)**: reviews the finished whole — including how parts built at different times, or by different parallel Makers, fit together
- At **completion (Round 2+)**: operates in **verification and regression mode**. It receives the prior round's decisions (accepted fixes + rejected items) to:
  1. Verify whether accepted fixes were applied properly and resolved the root problem
  2. Check for regressions or new bugs introduced by the fixes
  3. Avoid re-flagging items already rejected by the Decider unless new evidence or broken invariants emerged
- Hunts for: logic errors, edge cases, missing tests, security gaps, performance pitfalls, unclear naming, fragile assumptions, missing error handling, broken invariants, convention drift, and integration seams
- Produces a **numbered list** of specific, actionable recommendations — file paths, line numbers, suggested change where possible, severity (blocker / important / nice-to-have), and one-line rationale per item
- Read-only: no Edit/Write tools — keeps its hands off the artifact
- Does NOT prioritize across recommendations; that's the Decider's job
- If genuinely nothing is wrong: explicitly output "no issues found" rather than padding the list

### Decider

- Receives the artifact AND the Checker's full recommendation list (and in round 2+, prior round decisions)
- For each recommendation, decides:
  - **accept** → goes on the fix list unchanged, preserving the Checker's diagnostic rationale
  - **modify** → the adjusted version goes on the fix list (Decider writes the modified instruction and rationale)
  - **reject** → does NOT go on the fix list (recorded with a clear one-line reason so the orchestrator can pass it to round 2+ agents as out-of-scope guardrails)
- May **add fixes** the Checker missed — these go on the fix list too
- **At a checkpoint:** outputs the fix list (Maker applies it at the start of the next batch), or exactly `CONTINUE` if it's empty. Never `DONE` at a checkpoint
- **At completion (Round 1):** outputs the fix list with attached rationales, plus the list of rejected items, or exactly `DONE` if empty. If every surviving item is nice-to-have, output `DONE` plus an "optional polish" list for the user — don't send the Maker back for aesthetics
- **At completion (Round 2+):** applies strict convergence thresholding. Only send the Maker back for genuine blockers or correctness regressions. Minor or cosmetic suggestions must be routed to the user's "optional polish" list alongside `DONE` to prevent endless review churn
- Read-only tools; owns the stop decision

## Parallel Makers

When the plan contains independent phases, run multiple Makers concurrently instead of one Maker serially:

- **Derive the dependency map first.** Phases are independent when none reads or writes another's outputs and they touch disjoint files/modules. When in doubt, treat them as dependent.
- **One Maker per independent group,** dispatched in a single message so they run concurrently. Dependent phases stay serial — with one Maker, or scheduled after the batch they depend on.
- **Overlapping files → isolate.** If groups might touch the same files, give each Maker `isolation: "worktree"` and merge the worktrees when the batch completes; otherwise partition by file ownership.
- **Review is always unified.** The Checker reviews the merged, integrated state — never one Maker's slice alone; the seams between parallel slices are exactly where the bugs live. Checkpoint cadence counts phases completed across all Makers.

## The Loop

```
batches = partition(plan)                 # cadence table + dependency map
work = ∅
for each batch except the last:
    work += Makers build batch            # parallel where independent; apply pending fixes first
    recs  = Checker(work, scope = phases built so far)    # checkpoint: single pass
    fixes = Decider(work, recs)                           # fix list or CONTINUE
work += Makers build final batch          # apply pending fixes first

# Completion Loop (rapid convergence in 1–2 rounds)
prior_decisions = null
for round in 1..5:
    if round == 1:
        recs = Checker(work, scope = "full completion")
    else:
        recs = Checker(work, scope = "verification & regressions", prior = prior_decisions)
    
    decision = Decider(work, recs, prior = prior_decisions, round = round)
    if decision.status == DONE: return work
    
    prior_decisions = decision
    work = Maker(
        work,
        fixes = decision.fixes_with_rationales,
        rejected_boundaries = decision.rejected_items
    )
surface_to_user(work, "iteration cap reached; remaining concerns: ...")
```

**Convergence expectation:** with targeted round 2+ context (rationales + reject boundaries + verification-mode checking), the completion loop typically reaches `DONE` in 1–2 rounds. Three or more rounds of churn is a signal the work needs human input or a different decomposition — surface it rather than grinding to the cap.

**Iteration cap:** stop the completion loop at 5 rounds even if the Decider hasn't signaled `DONE`. If you hit the cap, surface remaining issues to the user — don't quietly ship.

## Pods and Parallelism

- One pod = one **complete** unit of work + its Makers + one Checker + one Decider
- Parallel **Makers** live inside one pod (independent phases of one unit); parallel **pods** are for independent units of work
- Independent units of work → spawn N pods in parallel from the orchestrating session
- Pods do not share state during their loops; the orchestrator only sees each pod's final artifact
- "Independent" means the units don't read or write each other's outputs mid-flight — if they do, run them sequentially or as one larger pod

## Implementation

The orchestrating session drives the loop with real `Agent` tool calls. State (the current artifact, the Checker's recommendations, the Decider's fix list, which phases are built) lives in the orchestrator and is pasted verbatim into each subagent's prompt — subagents have no shared memory.

**Model selection:** All three roles should run on capable reasoning models configured in your environment to ensure thorough review and unbiased arbitration.

**Maker dispatch (per batch):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Maker — batch 1 (phases 1–2 of 10)",
  prompt: `You are the MAKER in a make-check pod.

  Build phases <a>–<b> of the task below. Work phase by phase, keeping
  each phase compiling and tested. If a pending fix list is included,
  apply it BEFORE building new phases.

  TASK: <original task / full plan>
  YOUR PHASES: <a>–<b> of <N>
  PENDING FIX LIST: <paste Decider output, or "none">

  Return the concrete artifact for your phases — files changed with
  diffs, tests, and test/build output pasted verbatim — and state
  which phases are now built.`
})
```

For **parallel Makers**, dispatch several of these in ONE message, one per independent phase group; add `isolation: "worktree"` when file ownership overlaps, and merge before any review.

**Maker dispatch (completion loop — remediation round N):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Maker — completion round N remediation",
  prompt: `You are the MAKER in a make-check pod applying completion fixes.

  Review and implement the Decider's fix list below. Each item includes
  the Checker's diagnostic rationale to help you address the root cause
  directly. Respect the OUT-OF-SCOPE / REJECTED boundaries — do NOT alter
  or refactor areas that the Decider explicitly designated to leave as-is.

  Ensure all changes compile, pass tests, and don't introduce regressions.

  TASK / ARTIFACT: <original task or path to current artifact>
  ACCEPTED FIXES & RATIONALE:
  <paste Decider fix list with diagnostic notes>

  OUT-OF-SCOPE / REJECTED (DO NOT TOUCH):
  <paste Decider reject list with reasons, or "none">

  Return the concrete artifact or summary of changed files with diffs
  and verbatim test/build output.`
})
```

**Checker dispatch (checkpoint or completion):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Checker — checkpoint 1 | completion round N",
  prompt: `You are the CHECKER in a make-check pod. Adversarial but
  collaborative review. Read-only — do not edit anything.

  SCOPE — one of:
    CHECKPOINT: phases <1>–<k> of <N> are built; the rest are not.
    Review what exists as a coherent partial whole — direction,
    architecture, conventions, integration. Unbuilt phases are NOT
    findings.

    COMPLETION (Round 1): the work is finished. Review it as a whole,
    including how parts built at different times or by parallel Makers
    fit together.

    COMPLETION (Round 2+ — VERIFICATION & REGRESSION):
    Focus strictly on:
    1. Verifying whether the previous round's accepted fixes were applied
       properly and resolved the root problem.
    2. Detecting regressions or unintended side-effects from the fixes.
    Do NOT re-raise items listed under PRIOR REJECTED DECISIONS unless
    new evidence or broken invariants have surfaced.

  Hunt for logic errors, edge cases, security gaps, missing tests,
  performance pitfalls, fragile assumptions, convention drift, and
  integration seams.

  Output a NUMBERED list. Each item: file:line, suggested change,
  severity (blocker | important | nice-to-have), one-line rationale.
  Do NOT prioritize across items. If nothing is wrong, output exactly:
  "no issues found".

  ARTIFACT: <paste artifact or file path>
  PRIOR ACCEPTED FIXES (Round 2+ only): <paste prior round fixes, or "N/A">
  PRIOR REJECTED DECISIONS (Round 2+ only): <paste prior round rejects, or "N/A">`
})
```

**Decider dispatch (each review point):**

```
Agent({
  subagent_type: "general-purpose",
  description: "Make-check Decider — checkpoint 1 | completion round N",
  prompt: `You are the DECIDER in a make-check pod. For each
  recommendation: accept (goes on fix list, preserving Checker's rationale),
  modify (write adjusted instruction and reason), or reject (one-line reason;
  stays off the fix list). You may add fixes the Checker missed.

  This is a <CHECKPOINT | COMPLETION ROUND 1 | COMPLETION ROUND 2+> review.
  - CHECKPOINT: output the ordered fix list (applied at the start of
    the Maker's next batch), or exactly CONTINUE if it is empty.
  - COMPLETION ROUND 1: output the ordered fix list (with Checker rationales)
    plus the list of rejected items, or exactly DONE if empty. If every
    surviving item is nice-to-have, output DONE plus an "optional polish" list.
  - COMPLETION ROUND 2+: apply strict convergence thresholding. Only output
    a fix list for genuine blockers or correctness regressions. Route
    all nice-to-have/polish items to the "optional polish" list and output DONE.

  OUTPUT FORMAT:
  STATUS: <DONE | CONTINUE | FIXES_REQUIRED>
  ACCEPTED FIXES (with rationale):
  - [item 1] ...
  REJECTED ITEMS (with reason):
  - [item 1] ...
  OPTIONAL POLISH (for user, non-blocking):
  - [item 1] ...

  ARTIFACT: <paste artifact>
  RECOMMENDATIONS: <paste Checker output>
  PRIOR ROUND DECISIONS (Round 2+ only): <paste prior decisions, or "N/A">`
})
```

**Dual-lens Checker (optional, for large or high-stakes completion reviews):** dispatch two Checkers in parallel with different lenses — one on correctness/security/tests, one on design/simplicity/conventions — and give the single Decider both lists to arbitrate.

**Large artifacts:** if the artifact won't fit in a prompt, write it to a file (e.g. `/tmp/make-check-pod-N/v3.md`) and pass the path. Each role reads from disk.

**Merging back:** when the pod finishes, the final artifact is the result. For implementation work, the Maker's edits are already on disk; the orchestrator just reports completion. For plans / investigations, paste the final doc into the orchestrator's response or write it to the location the user asked for.

**Parallel pods:** dispatch multiple pod orchestrators concurrently for independent units of work — they don't share state.

**Sequential pods:** for chained work (plan-pod → implementation-pod), the implementation pod's first Maker is given the plan pod's final artifact as input.

## Common Mistakes

| Mistake | Why it breaks make-check |
|---|---|
| Same agent plays Maker and Checker | Eliminates the adversarial review that gives the pattern its value |
| Checker softens findings to be "nice" | Useful issues stay hidden; the pod converges on something mediocre |
| Decider rubber-stamps everything | Becomes a pass-through; no real arbitration happens |
| Reviewing after every phase | Critiques half-finished work, churns on code later phases rewrite, still misses cross-phase bugs — batch per the cadence table |
| Zero checkpoints on a 10+ phase build | A systemic error made in phase 1 propagates through nine more phases before anyone looks |
| Iterating at a checkpoint | Checkpoints are single-pass; iteration belongs to the completion loop |
| Checker flags unbuilt phases at a checkpoint | Turns a scoped review into noise; state the scope explicitly in the Checker prompt |
| Parallel Makers on overlapping files without worktree isolation | Clobbered edits and silent merge conflicts |
| Reviewing one parallel Maker's slice in isolation | Misses exactly the parallel-seam bugs unified review exists to catch |
| Sharing chat context across roles | Defeats independence; Checker absorbs Maker's framing |
| Using make-check on trivial work | Burns tokens and time for no quality gain |
| Sending round 2+ Maker a raw fix list without diagnostic rationale or rejected boundaries | Maker applies superficial fixes or re-touches rejected areas, delaying convergence |
| Checker in round 2+ re-flagging items already rejected by the Decider | Causes circular review loops; supply prior decisions so Checker focuses on verification and regressions |
| Decider looping on nice-to-haves in round 2+ | Churns on polish; route nice-to-haves to optional user notes and mark DONE |
| Skipping the Checker because the Maker seems confident | Any model can make systemic and integration errors; independent review is the pattern's whole value |
| Downgrading Checker or Decider to an underpowered model | Weak reasoning undermines review and arbitration quality; maintain consistent model capability across all roles |

## Quick Reference

| | Maker | Checker | Decider |
|---|---|---|---|
| Reads | Task + assigned phases + pending fix/remediation list (with rationales & reject boundaries) | Artifact + scope (checkpoint, round 1, or round 2+ verification with prior decisions) | Artifact + recommendations + scope + round context |
| Produces | Batch / remediation artifact with verbatim test output | Numbered recommendations | Fix list, `CONTINUE` (checkpoint), or `DONE` (completion) |
| Constraints | Self-check & testing encouraged; may run in parallel | No edits; respect scope (verification mode in round 2+) | Must justify rejects; enforce strict thresholding in round 2+ |
| Context | Fresh subagent | Fresh subagent | Fresh subagent |
| Termination authority | None | None | Owns `CONTINUE` / `DONE` |

## Real-World Fit

Make-check trades latency and tokens for quality. Use it when the cost of a missed issue (production bug, lost data, security incident, bad architectural commitment) outweighs the cost of a few extra subagent rounds. The overhead is modest — typically one checkpoint, one or two completion rounds — but for everything trivial, single-agent execution is still faster and usually fine.
