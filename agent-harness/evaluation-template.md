# Evaluation Report Template

Copy this template to `.harness/sprints/sprint-N-evaluation.md` and fill it in.

---

```markdown
# Sprint [N] Evaluation

**Evaluator verdict: [PASS / FAIL]**
**Attempt: [M] of 3**
**Testing method: [Playwright MCP / Test suite / Manual / Code review]**

## Criteria Results

### Critical Criteria

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| C1 | [from contract] | ✅ PASS / ❌ FAIL | [screenshot ref, test output, or observation] |
| C2 | ... | ... | ... |

**Critical pass rate: [X/Y] — required: Y/Y**

### Standard Criteria

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| S1 | [from contract] | ✅ PASS / ❌ FAIL | [evidence] |
| S2 | ... | ... | ... |

**Standard pass rate: [X/Y] ([Z]%) — required: 80%**

## Design Quality Assessment (frontend only)

| Dimension | Score (1-5) | Notes |
|-----------|-------------|-------|
| Design Quality | [score] | [specific observations] |
| Originality | [score] | [specific observations] |
| Craft | [score] | [specific observations] |
| Functionality | [score] | [specific observations] |

**AI slop detected: [yes/no] — [details if yes]**

## Failures Detail

For each failed criterion:

### ❌ [Criterion ID]: [Criterion text]

**Expected**: [what should happen per the contract]
**Actual**: [what actually happened]
**Evidence**: [screenshot path, error message, or test output]
**Severity**: [critical / standard]
**Fix suggestion**: [optional — what the generator should try]

## Overall Notes

[Any patterns in failures, architectural concerns, or recommendations for the next attempt]
```

---

## Scoring Calibration

Use these examples to calibrate your scoring:

### Design Quality (1-5)

- **1**: No styling at all, browser defaults, unstyled HTML
- **2**: Basic CSS applied but generic (default Bootstrap/Tailwind, no customization)
- **3**: Intentional styling with a consistent color scheme and layout, but nothing distinctive
- **4**: Cohesive design system with clear visual hierarchy, intentional whitespace, polished
- **5**: Museum-quality: every pixel is intentional, distinctive visual identity, you'd show it in a portfolio

### Originality (1-5)

- **1**: Exact copy of a known template or tutorial
- **2**: Template with minor color/font changes
- **3**: Custom layout but recognizable patterns (standard card grid, generic dashboard)
- **4**: Distinctive choices that reflect the app's purpose (custom illustrations, unique interactions)
- **5**: You couldn't tell this was AI-generated; feels like a designer's portfolio piece

### Craft (1-5)

- **1**: Broken layout, overlapping text, unreadable colors
- **2**: Layout works but spacing is inconsistent, typography is flat, colors clash
- **3**: Consistent spacing and readable text, meets basic accessibility
- **4**: Strong type hierarchy, harmonious colors, WCAG AA contrast, aligned grid
- **5**: Pixel-perfect: consistent 8px grid, fluid type scale, AAA contrast, motion design

### Functionality (1-5)

- **1**: App doesn't start or immediately crashes
- **2**: Main flow works but secondary flows are broken
- **3**: All specified flows work but edge cases fail
- **4**: All flows work including edge cases, good error handling
- **5**: Bulletproof: handles network failures, works offline if applicable, zero console errors

## Pass/Fail Rules

**Sprint PASSES if ALL of these are true:**

1. Every critical criterion passes
2. ≥80% of standard criteria pass
3. No AI slop detected (for frontend work)
4. App is in a runnable state (no crashes on start)

**Sprint FAILS if ANY of these are true:**

1. Any critical criterion fails
2. <80% of standard criteria pass
3. AI slop detected and not addressed
4. App crashes on start or during primary user flow

**When in doubt, FAIL.** The generator can fix and resubmit. Passing something that shouldn't pass wastes everyone's time downstream.
