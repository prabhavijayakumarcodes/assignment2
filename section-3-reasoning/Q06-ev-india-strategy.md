# Q6. Strategy Recommendation Under Uncertainty

**Client asks:** *"Should we enter the EV market in India?"*
**Goal:** Break the problem into sub-decisions, evaluate multiple scenarios, and produce a structured recommendation **with counterarguments** and **explicit decision criteria**.

---

## 1. Full Prompt

```text
ROLE
You are a strategy partner at a top-tier consulting firm advising a client's board.
You are rigorous, hypothesis-driven, and comfortable saying "it depends, and here is
exactly on what".

CLIENT CONTEXT
<client_context>
{{CLIENT_PROFILE: industry, current geographies, revenue, capabilities, risk appetite,
  capital available, time horizon. If any are blank, write "NOT PROVIDED".}}
</client_context>

QUESTION
"Should we enter the EV market in India?"

EPISTEMIC RULES
1. Distinguish clearly between (a) facts provided in <client_context>, (b) general
   market knowledge, and (c) assumptions. Tag claims [CLIENT], [MARKET], or [ASSUMPTION].
2. For any [MARKET] figure (market size, growth rate, subsidy amounts, penetration),
   state it as an approximate range with "as of <your knowledge cutoff>; verify" —
   never as a precise current fact. If unsure, say "requires verification" instead of a number.
3. Missing client information must be listed, not assumed. Where an assumption is
   unavoidable to proceed, state it and show how the recommendation changes if it is wrong.

PROCESS (follow in order)

STEP 1 – Define the decision criteria FIRST (before any analysis).
  Produce 5–7 criteria with weights summing to 100%, e.g.: strategic fit,
  market attractiveness, right-to-win, financial return (NPV/IRR vs hurdle),
  risk exposure, time-to-scale, capital intensity. Justify each weight in one line
  based on the client context (or state default weights if context is missing).

STEP 2 – Decompose into sub-decisions.
  At minimum:
  SD1 WHERE to play – segment (2W / 3W / 4W passenger / commercial / buses /
      charging infra / batteries & components / services)
  SD2 HOW to enter – build / partner / JV / acquire / licence
  SD3 WHEN to enter – now / staged pilot / wait-and-watch
  SD4 HOW BIG – capital commitment and stage gates
  For each SD, list 2–4 options.

STEP 3 – Define 3 scenarios for the external environment through ~5 years:
  - BULL: fast adoption, sustained policy support, falling battery costs
  - BASE: steady adoption, moderate policy support
  - BEAR: slow adoption, subsidy withdrawal, infra bottlenecks, price war
  For each, list the 3–4 key drivers and their assumed direction. Do not invent
  probabilities with false precision; use LOW/MED/HIGH likelihood.

STEP 4 – Evaluate the leading options under each scenario against the criteria.
  Score 1–5 per criterion, show the weighted total, per scenario.

STEP 5 – Counterarguments.
  For the option you lean towards, write the strongest case AGAINST it (steelman),
  at least 3 points, and state whether each counterargument is rebuttable,
  mitigable, or a genuine unresolved risk.

STEP 6 – Recommendation.
  Give a conditional recommendation ("Enter via X in segment Y, staged, IF …").
  Include stage gates: specific measurable triggers to scale up, hold, or exit.

OUTPUT FORMAT (use exactly these headings)

## Executive Answer (≤ 4 sentences)
## 1. Decision Criteria
| Criterion | Weight | Why this weight |
## 2. Sub-Decisions & Options
| Sub-decision | Options considered | Leading option | Rationale |
## 3. Scenarios
| Scenario | Key drivers | Likelihood (L/M/H) |
## 4. Option Evaluation
| Option | Criterion scores (1–5) | Weighted score – Bull | Base | Bear |
## 5. Counterarguments (steelman)
| Counterargument | Type (rebuttable / mitigable / unresolved) | Response |
## 6. Recommendation & Stage Gates
- Recommendation:
- Conditions under which this recommendation flips to "Do not enter":
- Stage gates: | Gate | Metric | Threshold | Decision |
## 7. Key Assumptions & Information Gaps
| Item | Tag ([CLIENT]/[MARKET]/[ASSUMPTION]) | Impact if wrong (H/M/L) | How to validate |
```

---

## 2. Explanation

### (a) Why this structure?

- **Criteria defined before analysis (Step 1)**: the key move against post-hoc rationalisation. If criteria come after the options, the model (like humans) picks criteria that justify its preferred answer. Locking them first makes the evaluation honest, and it satisfies "decision criteria explicitly".
- **Sub-decision decomposition (where / how / when / how big)**: "Should we enter EV?" is not one decision. It is a bundle. Decomposition turns a yes/no question into a set of actionable choices, and often reveals the real answer: "yes, but not in 4W, and via partnership".
- **Scenario × option matrix**: shows *robustness*. An option that wins in Bull but collapses in Bear is a risky choice even with the highest base-case score. Laying them side by side makes that visible.
- **Steelman counterarguments with classification**: "include counterarguments" can be satisfied with weak strawmen. Requiring the *strongest* case against, *and* classifying each as rebuttable, mitigable or unresolved, forces real engagement.
- **Conditional recommendation plus flip conditions plus stage gates**: under uncertainty, a good recommendation is a *decision rule*, not a verdict. Stage gates turn strategy into a monitored plan.
- **Epistemic tagging ([CLIENT]/[MARKET]/[ASSUMPTION])**: market figures for India's EV sector move quickly and the model's knowledge is dated. Tags plus the "verify" requirement prevent stale numbers being presented as current fact.

### (b) Why this technique? Structured (controlled) CoT with ToT elements

- The task needs **multi-step reasoning**, so pure zero-shot "just answer" would give a shallow yes/no.
- It borrows from **Tree-of-Thought**: multiple options across multiple scenarios are explored *in parallel* and then compared, instead of following a single chain. It is done in a single prompt via the evaluation matrix rather than multiple model calls, which keeps cost low while capturing the benefit of branch comparison.
- **Zero-shot** rather than few-shot, because a worked example (say, "enter the solar market in Brazil") would leak its specific conclusions and segment logic. The heading schema provides the consistency.

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Superficial yes/no answer | Sub-decision decomposition, conditional recommendation |
| Post-hoc criteria to justify a gut answer | Criteria and weights fixed in Step 1 |
| One-scenario optimism | Bull/Base/Bear evaluation matrix |
| Strawman counterarguments | Steelman requirement, minimum of 3, classification |
| Stale or hallucinated market stats | [MARKET] tag, ranges, "verify" requirement |
| Silent assumptions about the client | "NOT PROVIDED" plus an Assumptions & Gaps table |
| Unactionable recommendation | Stage gates with metric, threshold and decision |
| False-precision probabilities | L/M/H likelihood only |

---

## 3. Alternative Prompt Designs

### Alternative A: Multi-agent debate (3 calls)

1. **Advocate prompt:** "Make the strongest evidence-tagged case FOR entering India EV for this client."
2. **Critic prompt:** "Make the strongest evidence-tagged case AGAINST entering."
3. **Judge prompt:** "Given both cases and these pre-defined weighted criteria, score each argument, identify which disagreements are factual (resolvable with data) versus value-based (risk appetite), and issue a conditional recommendation."

**Pro:** genuinely adversarial counterarguments, since a single model arguing against itself tends to go easy. **Con:** 3× cost, and needs orchestration.

### Alternative B: Pre-mortem framing

```text
Assume it is 2031 and the client's India EV venture has FAILED. Write the 5 most
likely reasons why. Then assume it SUCCEEDED and write the 5 most likely reasons.
Use both lists to derive decision criteria, stage gates, and a recommendation.
```
**Pro:** pre-mortems are effective at surfacing risks people overlook. **Con:** less systematic coverage of sub-decisions, so it is best combined with the main prompt's Step 2.
