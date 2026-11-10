# Q12. When the Model is Wrong

**Given:** The model produced a confident but incorrect answer.
**Goal:** A prompt that forces re-evaluation, identifies weak assumptions, and produces a corrected answer.

---

## 1. Full Prompt

```text
ROLE
You are an independent verifier. A previous answer to the question below was
produced by an AI system. It may be correct or incorrect. Your job is to verify it
from first principles, not to defend it and not to reject it reflexively.

INPUTS
<original_question>
{{QUESTION}}
</original_question>

<previous_answer>
{{MODEL_ANSWER}}
</previous_answer>

<available_evidence>
{{Optional: source documents, data, tool outputs, or "NONE"}}
</available_evidence>

<feedback>
{{Optional: user/system signal that the answer is wrong, e.g. "unit test failed",
  "client says the figure is wrong". If empty, write "NONE".}}
</feedback>

RULES
1. Treat <previous_answer> as an untrusted claim. Its confidence is not evidence.
2. Feedback is a signal, not proof. Do not change a correct answer simply because
   someone said it is wrong, and do not keep a wrong answer because it sounds confident.
3. Base verification only on <original_question>, <available_evidence>, and
   verifiable reasoning/calculation. Where general knowledge is used, label it and
   rate how sure you are.
4. If the question cannot be answered with confidence from what is available, the
   correct output is a bounded answer plus what is needed — not a guess.

PROCESS (follow every step, in order)

Step 1 – Re-read the question.
   Restate exactly what is being asked, including units, scope, time period, and
   constraints. List any ways the previous answer may have misread the question.

Step 2 – Solve independently FIRST.
   Without relying on <previous_answer>, work the problem from scratch.
   Show the key steps (calculations, lookups in evidence, logical steps).

Step 3 – Decompose the previous answer.
   Break it into atomic claims and the assumptions it relies on (stated or implicit).

Step 4 – Stress-test each assumption.
   For each assumption, classify:
   - VERIFIED (supported by evidence or a checkable calculation – cite it)
   - UNVERIFIED (plausible but no support)
   - CONTRADICTED (conflicts with evidence, the question, or your Step 2 work)
   Then ask: "If this assumption were false, would the conclusion change?" → impact H/M/L.

Step 5 – Compare.
   Compare your Step 2 answer with the previous answer. Identify the exact point of
   divergence (first wrong step) if they differ.

Step 6 – Corrected answer.
   Produce the corrected (or confirmed) answer with a calibrated confidence and the
   conditions under which it would still be wrong.

OUTPUT FORMAT

### 1. What the question actually asks
...
### 2. Independent solution
...
### 3. Assumption audit of previous answer
| # | Claim / assumption | Stated or implicit | Status (VERIFIED / UNVERIFIED / CONTRADICTED) | Evidence | Impact if false (H/M/L) |
### 4. Diagnosis
- Verdict on previous answer: CORRECT | PARTIALLY CORRECT | INCORRECT | CANNOT DETERMINE
- First point of failure: <step/claim, or "none">
- Error type: misread question | factual error | calculation error | unsupported assumption |
  overgeneralisation | outdated information | other
### 5. Corrected answer
<answer>
- Confidence: HIGH / MEDIUM / LOW – with one-line reason
- Would be wrong if: <key remaining assumption>
- To raise confidence, verify: <specific check>
```

### Worked test (synthetic)

**Question:** "A product costs ₹1,000. Its price rises 20%, then falls 20%. What is the final price?"
**Previous answer (confident, wrong):** "₹1,000, since +20% and −20% cancel out."

Expected behaviour:
- Step 2 independent solution: 1000 × 1.2 = 1200; 1200 × 0.8 = **₹960**.
- Assumption audit: "percentage changes cancel" is **CONTRADICTED**, impact H, because the second percentage applies to a different base.
- Diagnosis: INCORRECT; error type: unsupported assumption / calculation error.
- Corrected answer: ₹960, confidence HIGH.

**Control test:** with the *correct* previous answer (₹960) and feedback "client says this is wrong", the verifier should confirm ₹960 and verdict **CORRECT**. This guards against sycophantic flip-flopping.

---

## 2. Explanation

### (a) Why this structure?

- **Verifier role, not author role**: asking the *same* conversation "are you sure?" tends to produce either stubborn defence or reflexive capitulation. Framing the model as an independent verifier of *someone else's* answer reduces both self-consistency bias and sycophancy.
- **Step 1: re-read the question**: a large share of confident errors come from misreading the question (wrong units, wrong period, wrong entity). Making this step explicit catches them early.
- **Step 2: solve independently *before* reading the previous answer's logic**: the key anti-anchoring device. If the model reads the wrong answer first, it tends to rationalise it. Independent solving first gives an uncontaminated comparison point.
- **Step 3–4: atomic claims plus assumption audit with impact**: directly fulfils "identify weak assumptions". The "would the conclusion change?" impact test focuses attention on *load-bearing* assumptions, not trivia.
- **Step 5: first point of divergence**: pinpoints *where* the reasoning broke, which is useful for fixing the upstream prompt too.
- **Diagnosis taxonomy**: classifying error types enables tracking of failure patterns over time (e.g., "40% of our errors are outdated info → add retrieval").
- **"Would be wrong if" in the corrected answer**: prevents the *corrected* answer from becoming a new confident-but-wrong answer.

### (b) Why this technique? Structured verification CoT (a Chain-of-Verification variant)

- Based on **Chain-of-Verification (CoVe)**: decompose a response into verifiable claims, check each independently, then revise. That is more effective than a generic "double-check your answer".
- **Reasoning is enforced, not suppressed.** This is exactly the case where explicit step-by-step reasoning pays off: the first answer failed precisely because of skipped or implicit reasoning.
- **Zero-shot**: the error could be anything (math, fact, logic). A few-shot example of one error type would bias the verifier toward finding *that* type.

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Anchoring on the wrong answer | Independent solution before decomposition |
| Sycophantic flip (changing a correct answer under pressure) | Rule 2 ("feedback is a signal, not proof"), CORRECT verdict allowed, control test |
| Stubborn defence of a wrong answer | Verifier role, "confidence is not evidence" |
| Missing the real error (misread question) | Step 1 restatement |
| Superficial "I've checked, looks fine" | Mandatory assumption table with statuses and evidence |
| New hallucination in the correction | Evidence-only rule, confidence plus "would be wrong if" |
| Guessing when evidence is insufficient | CANNOT DETERMINE verdict and Rule 4 |

---

## 3. Alternative Prompt Designs

### Alternative A: Self-consistency voting

Re-ask the original question 5 times at temperature ~0.7 with different reasoning paths ("solve via method A / method B / work backwards"), then majority-vote. If the original answer disagrees with the majority, flag and replace it. **Pro:** simple and effective for math and logic. **Con:** if the model has a *systematic* misconception, all 5 samples share it, and it is costly.

### Alternative B: Tool-grounded verification

```text
For every numerical or factual claim in <previous_answer>, write the exact tool call
(calculator / search / database query) that would verify it. Execute them, then
rewrite the answer using only verified results.
```
**Pro:** replaces model self-judgement with external ground truth, the strongest fix for factual and calculation errors. **Con:** needs tool access, and doesn't help with pure reasoning or judgement errors.

### Alternative C: Socratic challenge

"List 3 questions a skeptical expert would ask about this answer. Answer each honestly. Then state whether the original answer survives." This is lightweight and good for judgement-heavy answers, but less systematic than the full audit.
