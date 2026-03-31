# Sprint Contract Template

Copy this template to `.harness/sprints/sprint-N-contract.md` and fill it in.

---

```markdown
# Sprint [N] Contract

## Sprint Goal
[One sentence: what does this sprint deliver?]

## Spec Reference
[Which section(s) of .harness/spec.md does this sprint implement?]

## Success Criteria

Each criterion must be:
- **Observable**: the evaluator can see/measure the result
- **Specific**: includes numbers, names, or exact behavior
- **Independent**: can be tested without other criteria passing

### Critical Criteria (all must pass for sprint to pass)

| # | Criterion | Verification Method |
|---|-----------|-------------------|
| C1 | [Specific, testable behavior] | [How the evaluator will test this] |
| C2 | ... | ... |

### Standard Criteria (80% must pass for sprint to pass)

| # | Criterion | Verification Method |
|---|-----------|-------------------|
| S1 | [Specific, testable behavior] | [How the evaluator will test this] |
| S2 | ... | ... |

## Out of Scope
[What this sprint explicitly does NOT include — prevents scope creep]

## Dependencies
[What must exist before this sprint can start — previous sprint outputs, data, etc.]

## How to Run
[Commands to start the app and reach the features being tested]
```

---

## Criteria Writing Guidelines

### Good Criteria (specific, testable)

- "Login form displays error message 'Invalid email' when user submits 'notanemail'"
- "Dashboard chart renders with data from the last 7 days, showing one data point per day"
- "Clicking 'Delete' shows a confirmation dialog before removing the item"
- "Page loads in under 3 seconds on a cold start"
- "Mobile layout (375px) stacks the sidebar below the main content"
- "Empty task list displays illustration and 'Create your first task' CTA button"

### Bad Criteria (vague, untestable)

- "Login works" — works how? What inputs? What outputs?
- "Dashboard looks good" — good by what standard?
- "Responsive design" — at what breakpoints? What changes?
- "Error handling" — for what errors? What does the user see?
- "Code is well-structured" — not the evaluator's job; this is about behavior
- "Performance is acceptable" — what's the threshold?

### Scope Guidelines

| Sprint Type | Critical Criteria | Standard Criteria | Total |
|-------------|-------------------|-------------------|-------|
| Foundation (sprint 1-2) | 3-5 | 5-10 | 8-15 |
| Feature sprint | 5-8 | 8-15 | 13-23 |
| Polish sprint | 2-3 | 10-20 | 12-23 |
| Large/complex sprint | 8-12 | 15-25 | 23-37 |

### Verification Methods

Be explicit about HOW the evaluator will test:

- **Playwright**: "Navigate to /dashboard, verify chart element is visible with 7 data points"
- **API test**: "POST /api/login with {email: 'bad'} returns 400 with error message"
- **Visual**: "Take screenshot at 375px width, verify sidebar is below main content"
- **Database**: "After creating 3 items, query shows 3 rows in items table"
- **Console**: "No errors or warnings in browser console during the full user flow"
