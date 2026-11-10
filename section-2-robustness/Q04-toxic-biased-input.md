# Q4. Toxic + Biased Input Handling

**Input contains:** gender bias and emotionally charged language.
**Goal:** Neutralise bias and produce objective output, **without ignoring the input entirely**. The legitimate signal inside the input must be preserved.

Running example: a manager's performance-review note to be turned into an objective review summary.

---

## 1. Full Prompt

```text
ROLE
You are an HR documentation specialist. You convert raw manager notes into fair,
objective, evidence-based performance summaries that would withstand an
equal-opportunity audit.

CORE PRINCIPLE
Biased or emotional language usually CONTAINS a real, legitimate concern wrapped in
an unfair or inflammatory frame. Your job is to SEPARATE the two:
keep the verifiable, job-relevant substance; remove or neutralise the bias and emotion.
Do NOT discard the input, and do NOT lecture the author.

DEFINITIONS
- Job-relevant observation: a specific, observable behaviour or outcome tied to the
  role (deadlines, quality, communication, collaboration, results).
- Biased content: statements that attribute behaviour to gender (or any protected
  characteristic), use gendered stereotypes ("emotional", "abrasive", "bossy",
  "aggressive for a woman", "not a natural leader like the guys"), or apply different
  standards to people based on gender.
- Emotionally charged language: insults, exaggerations ("always", "never", "useless",
  "disaster"), sarcasm, or loaded adjectives that express the author's feeling
  rather than an observable fact.

PROCESS
Step 1 – Segment: split the input into individual claims.
Step 2 – Classify each claim:
         (A) Objective & job-relevant → keep (light rewording only)
         (B) Job-relevant but emotionally/bias-framed → rewrite to neutral, observable form
         (C) Bias with no job-relevant content → remove from summary, log in audit table
         (D) Vague/unsupported → keep as "needs evidence", do not upgrade into fact
Step 3 – Rewrite: produce the neutral summary using only A, B (rewritten), and D (flagged).
Step 4 – Consistency check: ask "Would this sentence be written identically if the
         employee were of a different gender?" If not, revise.

RULES
- Preserve meaning: never soften a legitimate performance concern into nothing,
  and never invent positive or negative facts.
- Use gender-neutral references to the employee's behaviour (use their name or role);
  do not comment on personality traits, only behaviours and outcomes.
- Replace absolutes with what is supported ("always late" → "missed deadlines;
  frequency not specified").
- Do not include the original biased phrasing in the final summary.
- Tone: professional, calm, specific.

OUTPUT FORMAT

### 1. Objective Summary
3–6 sentences, neutral, behaviour- and outcome-focused.

### 2. Claim-by-Claim Transformation (audit trail)
| # | Original fragment | Category (A/B/C/D) | Issue (bias / emotion / vague / none) | Neutral version or "Removed" |

### 3. Evidence Needed
Bullet list of specifics the manager should provide (dates, examples, metrics)
for any category-D claims.

### 4. Reviewer Note (1–2 sentences, neutral)
Flag, without accusation, that some original wording was removed for
relying on gendered generalisations, and suggest focusing on observable behaviour.

<manager_notes>
{{RAW_INPUT}}
</manager_notes>
```

### Sample input used for testing

> "Priya is honestly a nightmare in meetings, way too emotional, typical woman stuff. She missed the Q3 dashboard deadline by two weeks which screwed the whole team. She's also bossy with the junior devs. To be fair her code quality is solid. She'd never make lead like Rahul, he's a natural."

Expected transformation:

| Original | Cat. | Neutral version |
|---|---|---|
| "nightmare in meetings, way too emotional" | B/D | "Concerns raised about meeting conduct; specific examples needed." |
| "typical woman stuff" | C | Removed (gender stereotype, no job content) |
| "missed the Q3 dashboard deadline by two weeks which screwed the whole team" | B | "Q3 dashboard was delivered two weeks after the deadline, affecting dependent team work." |
| "bossy with the junior devs" | B/D | "Feedback on communication style with junior developers; examples needed." ("bossy" is a gender-coded term) |
| "code quality is solid" | A | "Code quality meets or exceeds expectations." |
| "never make lead like Rahul, he's a natural" | C/D | Removed comparison; "Leadership readiness not assessed against defined criteria." |

The legitimate concerns (deadline slip, meeting conduct, communication with juniors) are **kept**, and the bias is removed. That satisfies "does NOT ignore the input".

---

## 2. Explanation

### (a) Why this structure?

- **"Core principle" up front**: the main risk in bias-handling prompts is *over-correction*, where the model refuses or sanitises the input into nothing. Stating that biased text usually contains a real concern sets the right mental model.
- **Explicit definitions**: "bias" is subjective. Defining it with concrete examples, including gender-coded words like "bossy" and "abrasive", makes the model's classification consistent and explainable.
- **4-way classification (A/B/C/D)**: a binary keep/remove would either lose legitimate signal or keep bias. Category B ("legit but badly framed") is where most of the value lies. Category D stops vague claims from being laundered into facts through neutral rewording.
- **Counterfactual check (Step 4)**: "Would this be written identically for a different gender?" is a well-known, practical debiasing test, and it is mechanical enough for a model to apply.
- **Audit trail table**: transparency. HR can see exactly what was changed and why, which builds trust and supports accountability.
- **Non-accusatory reviewer note**: addresses the bias at the source without moralising, and keeps the output professional.

### (b) Why this technique? Zero-shot with a classification-guided rewrite

- **Zero-shot**: the definitions and categories carry the knowledge. A few-shot example of a biased note could *itself* prime the model with stereotypes, or teach it to look only for the specific bias shown (for example, only "emotional").
- **Light, structured reasoning** (segment → classify → rewrite → check) rather than free CoT: the task is transformation, not deep inference. Visible classification is the useful "reasoning".

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Ignoring/refusing the whole input | Core principle, Category B, and "do NOT discard" |
| Bias passing through in softened form ("can be emotional at times") | Explicit stereotype list plus counterfactual check |
| Over-sanitising real problems into nothing | "Never soften a legitimate concern into nothing" rule |
| Vague claims laundered into facts | Category D plus an Evidence Needed section |
| Hallucinated positive/negative facts | "Never invent" rule |
| Moralising or preachy output | "Do NOT lecture" and a neutral reviewer note |
| Repeating slurs or bias in the final text | "Do not include original biased phrasing in final summary" |

---

## 3. Alternative Prompt Designs

### Alternative A: Two-pass detect-then-rewrite

**Pass 1:** "List every phrase in the input that contains gender bias or emotional language, with a one-line reason. Return JSON." **Pass 2:** "Rewrite the input, neutralising exactly the phrases listed, preserving all other content." **Pro:** detection is auditable and testable on its own, so precision and recall can be measured. **Con:** two calls, and Pass 2 may miss subtle bias that Pass 1 also missed.

### Alternative B: Rubric-scored output

Ask the model to produce the summary and then score it from 1 to 5 on *Neutrality*, *Fidelity to original concerns* and *Specificity*, revising once if any score is below 4. **Pro:** explicitly balances neutrality against fidelity, the two competing goals. **Con:** self-scores can be lenient (see Q10/Q11 for mitigations).

### Alternative C: Persona-free "fact-only extraction"

"Extract only verifiable, job-relevant events (who did what, when, with what outcome). Ignore all adjectives." This is simple and robust, but it loses legitimate soft-skill concerns that need evidence rather than deletion.
