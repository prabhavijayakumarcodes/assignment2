# Q5. Financial Fraud Detection

**Given:** Transaction summaries (synthetic data).
**Goal:** Identify suspicious patterns, explain reasoning step by step, output structured risk scores. **Avoid overconfidence** and **avoid "storytelling hallucination"**, meaning invented narratives such as "the user was probably hacked while on vacation".

---

## Decision: CoT vs ToT vs Controlled Reasoning

**Choice: controlled (rule-anchored) reasoning**, a constrained form of Chain-of-Thought in which every reasoning step must reference a predefined indicator and specific transaction IDs.

| Option | Assessment for fraud detection |
|---|---|
| **Free Chain-of-Thought** | Good for transparency, but free-form CoT is exactly what produces *storytelling hallucination*: the model fills gaps with plausible narratives ("the customer likely travelled…"). Its confidence also tends to grow as the narrative becomes more coherent, regardless of the evidence. |
| **Tree-of-Thought** | Explores multiple branches (fraud vs. legitimate explanations). It is useful for open-ended investigation, but it is expensive (many calls), hard to make deterministic, and its output is hard to audit for compliance. It is overkill for scoring a batch of transactions. |
| **Controlled reasoning ✅** | Reasoning is restricted to (1) a fixed checklist of fraud indicators, (2) evidence that must cite transaction IDs and fields, and (3) a mandatory legitimate alternative explanation for each flag. That gives step-by-step explainability *without* narrative freedom. It is auditable, repeatable, and the best fit for regulated financial workflows. |

We borrow one idea from ToT, **considering the benign explanation for every flag**, without the cost of full tree search.

---

## 1. Full Prompt

```text
ROLE
You are a transaction-monitoring analyst assistant. You produce preliminary risk
assessments for human fraud investigators. You do not make final fraud determinations.

EVIDENCE RULES (strict)
1. Reason ONLY from fields present in <transactions> and <customer_profile>.
2. Every reasoning step must cite transaction IDs (e.g., T004, T007) and the exact
   field values involved.
3. Do NOT speculate about motives, life events, identities, devices, locations, or
   relationships that are not in the data. Forbidden: "probably on holiday",
   "likely a stolen card", "the fraudster then…". Say "data does not indicate why".
4. For every suspicious pattern, you MUST state at least one plausible legitimate
   explanation consistent with the same data.
5. If the data is insufficient to evaluate an indicator, mark it "NOT EVALUABLE".

INDICATOR CHECKLIST (evaluate each, in this order)
I1  Velocity – unusually many transactions in a short window
I2  Amount anomaly – amounts far outside the customer's typical range
I3  Structuring – repeated amounts just below a reporting/approval threshold
I4  Geographic inconsistency – locations impossible/unlikely given timing
I5  New payee / merchant risk – first-time payees receiving large or rapid funds
I6  Time-of-day anomaly – activity at hours atypical for this customer
I7  Round-tripping – funds sent out and returned via related paths
I8  Channel change – sudden switch in channel/device (only if field present)

PROCESS
Step 1 – Baseline: from <customer_profile> (or, if absent, from the earliest
         transactions, stating so), describe typical amount range, frequency,
         locations, and channels. Quote values.
Step 2 – Indicator evaluation: for each of I1–I8, give
         Status (TRIGGERED / NOT TRIGGERED / NOT EVALUABLE), Evidence (IDs + values),
         Legitimate alternative explanation (if TRIGGERED).
Step 3 – Scoring: compute per-transaction and overall risk using the rubric below.
Step 4 – Calibration check: if your overall score ≥ 70, verify that at least two
         independent indicators are TRIGGERED. If not, cap the score at 69.

SCORING RUBRIC
- Each TRIGGERED indicator adds points: I1 15, I2 15, I3 20, I4 20, I5 10, I6 5,
  I7 20, I8 10. Max contribution per indicator applies once per transaction.
- Per-transaction score = sum of points of indicators it participates in, capped at 100.
- Overall score = highest per-transaction score, adjusted ±10 only with a stated reason.
- Risk band: 0–29 LOW, 30–59 MEDIUM, 60–79 HIGH, 80–100 CRITICAL.
- Confidence in the assessment (separate from risk): LOW / MEDIUM / HIGH, based on
  data completeness (how many indicators were NOT EVALUABLE).

OUTPUT FORMAT
### Baseline
(bullets with quoted values)

### Indicator Evaluation
| ID | Indicator | Status | Evidence (txn IDs + values) | Legitimate alternative |

### Transaction Risk Scores
| Txn ID | Indicators triggered | Score | Band |

### Overall Assessment
```json
{
  "overall_risk_score": <int>,
  "risk_band": "<LOW|MEDIUM|HIGH|CRITICAL>",
  "assessment_confidence": "<LOW|MEDIUM|HIGH>",
  "top_flagged_transactions": ["T..."],
  "not_evaluable_indicators": ["I.."],
  "recommended_action": "<NO_ACTION|MONITOR|INVESTIGATE|ESCALATE_IMMEDIATELY>",
  "limitations": "<1–2 sentences on what the data cannot tell us>"
}
```

LANGUAGE CONSTRAINTS
Use "consistent with", "pattern warrants review", "elevated risk". Never use
"fraud confirmed", "definitely", "clearly fraudulent".

<customer_profile>
{{PROFILE_OR_"NOT PROVIDED"}}
</customer_profile>

<transactions>
{{TRANSACTION_TABLE}}
</transactions>
```

### Synthetic test data

```
customer_profile: avg monthly spend INR 25,000; typical txn INR 500–3,000; home city Chennai; channel: mobile app

ID   | datetime          | amount  | type     | payee/merchant       | city      | channel
T001 | 2026-10-01 10:12  | 1,200   | card     | Grocery Mart         | Chennai   | app
T002 | 2026-10-03 19:40  | 2,300   | card     | Fuel Station         | Chennai   | app
T003 | 2026-10-05 02:14  | 49,500  | transfer | NEW: R. Kumar        | Chennai   | web
T004 | 2026-10-05 02:21  | 49,800  | transfer | NEW: R. Kumar        | Chennai   | web
T005 | 2026-10-05 02:29  | 49,700  | transfer | NEW: S. Traders      | Chennai   | web
T006 | 2026-10-05 03:05  | 3,400   | card     | Electronics Store    | Delhi     | card-present
T007 | 2026-10-06 12:00  | 900     | card     | Café                 | Chennai   | app
```

(Assuming a 50,000 approval threshold.) Expected: I1, I2, I3, I5, I6 and I8 TRIGGERED on T003–T005, and I4 TRIGGERED on T006 (Chennai at 02:29 web → Delhi card-present at 03:05). The legitimate alternatives mention a planned large payment split by limit, a family member using the card, and so on. Overall band: CRITICAL, action INVESTIGATE/ESCALATE. **No** invented story about who "R. Kumar" is.

---

## 2. Explanation

### (a) Why this structure?

- **Fixed indicator checklist**: turns open-ended "find fraud" into a bounded set of checks. Each check is a known AML/fraud typology, so findings map to industry language that investigators trust.
- **Baseline before anomalies**: "anomalous" only means something relative to normal. Forcing the baseline first prevents the model from flagging every large transaction.
- **Mandatory legitimate alternative**: the core anti-storytelling device. The model must hold two explanations, which breaks the single-narrative momentum that produces confident fiction.
- **Deterministic scoring rubric**: scores come from rules, not the model's gut feel, so they are reproducible and explainable. The model only decides TRIGGERED or not, which is a much easier judgement.
- **Separate risk vs. confidence**: a high-risk pattern seen on incomplete data should say "HIGH risk, LOW confidence". Collapsing the two is a classic source of overconfidence.
- **Calibration check (≥70 needs 2 independent indicators)**: a hard brake on single-signal escalations.
- **JSON summary**: machine-readable for downstream case-management systems.

### (b) Why this technique?

Controlled reasoning, justified above. In addition, **zero-shot** is chosen over few-shot because a worked example would anchor the model to that example's specific typology, such as always finding structuring. The rubric conveys the "how" without anchoring the "what".

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Storytelling hallucination | Rule 3 (forbidden speculation), mandatory transaction-ID citations |
| Overconfidence | Rubric-based scores, calibration cap, separate confidence field, banned certainty words |
| Single-indicator false positives | Two-indicator requirement for ≥70 |
| Ignoring missing data | NOT EVALUABLE status feeds the confidence rating and the limitations field |
| Inconsistent scores across runs | Deterministic point table (also run at temperature 0) |
| Unauditable reasoning | Every step tied to an indicator ID and transaction IDs |

---

## 3. Alternative Prompt Designs

### Alternative A: Code-first hybrid (LLM explains, code scores)

Compute I1–I8 deterministically in Python/SQL (velocity windows, threshold proximity, etc.) and pass the *computed flags* to the LLM:

```text
Below are rule-engine outputs for a customer. Write an investigator-facing explanation
of each triggered rule, citing transaction IDs, and propose one legitimate explanation
per rule. Do not change any score. Do not add new indicators.
<rule_engine_output>{{JSON}}</rule_engine_output>
```
**Pro:** maximum reliability. The LLM can't miscalculate or over-score. **Con:** it only catches patterns the rules encode, and loses the LLM's ability to notice novel patterns.

### Alternative B: Lightweight Tree-of-Thought for escalated cases only

For cases scoring HIGH or above, run a second prompt: *"Generate three competing hypotheses (fraud, account takeover, legitimate unusual activity). For each, list which transactions support it, which contradict it, and what additional data would discriminate. Do not choose."* **Pro:** deeper analysis where it matters. **Con:** higher cost, so it is reserved for the small fraction of escalated cases.
