# Q10. Self-Improving Prompt

**Goal:** A prompt that generates an answer, critiques it, and improves it, while **controlling the verbosity of the critique** and **preventing infinite loops**.

---

## 1. Full Prompt

```text
ROLE
You are an expert in {{DOMAIN}} who produces a draft, critiques it against a fixed
rubric, and revises it. You work in a strictly bounded process.

TASK
<task>
{{USER_TASK}}
</task>

RUBRIC (the ONLY criteria you may critique against)
C1 Correctness – facts, logic, and calculations are accurate; no unsupported claims.
C2 Completeness – every explicit requirement in <task> is addressed.
C3 Clarity – structure and wording are easy to follow for the intended reader.
C4 Concision – no redundancy or filler.
(Optional task-specific criterion C5: {{CUSTOM_CRITERION or "none"}})

PROCESS – maximum 2 revision rounds. This limit is absolute.

ROUND 0 – DRAFT
Write the best answer you can.

CRITIQUE (after each draft/revision)
- Score each criterion 1–5.
- List AT MOST 3 issues, ranked by impact. Each issue:
  "[C#] <problem in ≤ 20 words> → <specific fix in ≤ 20 words>"
- Only list an issue if fixing it would raise a score by at least 1 point.
  Do not list stylistic preferences or rephrase-for-its-own-sake.
- The critique must be ≤ 80 words total.

STOPPING RULES (check in this order after each critique)
S1. If every criterion scores ≥ 4 and no issue is rated C1 (correctness) → STOP.
S2. If this was revision round 2 → STOP (hard cap).
S3. If the new scores are not higher than the previous round's scores on at least
    one criterion, and not lower on any → STOP, keep the higher-scoring version
    (no-progress rule: prevents oscillation).
S4. Otherwise → REVISE, fixing ONLY the listed issues. Do not rewrite unaffected parts.

OUTPUT FORMAT

### Draft (Round 0)
...
### Critique 0
Scores: C1 _/5 · C2 _/5 · C3 _/5 · C4 _/5
Issues:
1. ...
Decision: REVISE | STOP (rule S#)

### Revision 1          (only if Decision was REVISE)
...
### Critique 1
...
Decision: REVISE | STOP (rule S#)

### Revision 2          (only if Decision was REVISE)
...
### Critique 2
Decision: STOP (S1 or S2 or S3)

### Final Answer
<the final version, reproduced in full, clean, with no critique notes>
### Change Log
- One bullet per round: what changed and which criterion it improved (≤ 15 words each).
```

### Production variant (output only final answer)

For end users who shouldn't see the internals, add: *"Output only the 'Final Answer' section. Perform the draft and critique silently."* Keep the full trace in logs for debugging.

---

## 2. Explanation

### (a) Why this structure?

- **Fixed rubric (C1–C4)**: without a rubric, self-critique drifts into vague "this could be improved" commentary or endless stylistic nitpicking. A rubric bounds *what* may be critiqued, and the scores make progress measurable, which the stopping rules depend on.
- **Verbosity controls on the critique**:
  - Max 3 issues, ranked by impact (forces prioritisation)
  - ≤20 words per problem and per fix, ≤80 words total
  - "Problem → fix" format, so every issue is actionable
  - **Materiality filter**: an issue must raise a score by at least 1 point, which removes trivial nitpicks
- **Loop prevention, four layers**:
  1. **Hard cap** (S2): maximum 2 revisions, absolute.
  2. **Quality-threshold exit** (S1): stop early when it is good enough. Most tasks end after Round 0 or 1.
  3. **No-progress rule** (S3): if a revision doesn't strictly improve, stop and keep the better version. This prevents *oscillation*, a known self-refinement failure where the model alternates between two versions or degrades a good answer.
  4. **Targeted revision** (S4): fix only the listed issues, which avoids full rewrites that introduce new problems and restart the cycle.
- **Explicit "Decision: … (rule S#)" line**: makes the stopping logic visible and verifiable.
- **Clean Final Answer plus Change Log**: separates the deliverable from the process, while keeping the improvements traceable.

### (b) Why this technique? Self-Refine (generate → critique → refine) with bounded iteration

- Based on the **Self-Refine** pattern, where a single model iteratively provides feedback and revises. It works best when the feedback is *specific and actionable*, which the "problem → fix" format enforces.
- **Zero-shot**: the rubric and output template define the behaviour. A few-shot example of a critique would make critiques copy the example's issues.
- **Structured CoT**: the critique *is* the reasoning step, but it is bounded and rubric-driven rather than free-form.

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Infinite / runaway loop | Hard cap of 2, explicit STOP decisions |
| Oscillation (A → B → A) | No-progress rule S3, keep best version |
| Over-editing degrades a good answer | Threshold exit S1, targeted fixes only, materiality filter |
| Verbose, rambling critique | 3-issue max, word limits, problem→fix format |
| Sycophantic self-critique ("Looks great!") | Numeric scoring per criterion; issues required whenever a score is below 4 |
| Vague critique ("could be clearer") | Required specific fix for each issue |
| Final answer polluted with critique notes | Separate clean Final Answer section |

**Known limitation:** a model critiquing itself shares its own blind spots. If it is confidently wrong on a fact, its critique may not catch it. For high-stakes factual tasks, use Alternative A or external verification (see Q12).

---

## 3. Alternative Prompt Designs

### Alternative A: Separate critic (two-model / two-call)

Call 1 generates. Call 2 uses a *different system prompt* (and ideally a different model or temperature) as a strict reviewer: "You did not write this. Find up to 3 material issues against rubric C1–C4." Call 3 revises. Orchestration code enforces the iteration cap. **Pro:** less self-confirmation bias, and the loop is controlled in code, which is more reliable than in-prompt rules. **Con:** 3 or more calls, higher latency.

### Alternative B: Generate-N-then-select (best-of-N with rubric)

```text
Produce 3 materially different candidate answers. Score each on C1–C4.
Select the highest total; if tied, prefer higher C1. Output only the winner and a
one-line justification.
```
**Pro:** no loop at all, so a loop is structurally impossible, and diversity helps on creative tasks. **Con:** 3× generation cost, and it doesn't fix flaws shared by all candidates.
