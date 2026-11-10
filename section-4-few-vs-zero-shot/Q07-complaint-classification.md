# Q7. Classification with Edge Cases

**Task:** Classify customer complaints into **Billing / Network / Device / Other**. Edge cases are messy and overlapping.
**Deliverables:** a zero-shot prompt, a few-shot prompt, then *why few-shot helps (or doesn't)* and *when it breaks*.

---

## 1A. Zero-Shot Prompt

```text
ROLE
You are a telecom customer-support triage classifier.

TASK
Classify the complaint in <complaint> into exactly ONE primary category, and
optionally ONE secondary category.

CATEGORY DEFINITIONS
- BILLING: charges, invoices, payments, refunds, plan pricing, recharge/top-up
  amounts, unexpected deductions, auto-debit, tariff changes.
- NETWORK: coverage, signal, call drops, data speed, outages, roaming connectivity,
  SMS/call delivery failures caused by the network.
- DEVICE: the handset, SIM card hardware, router/modem hardware, device settings,
  device repair/replacement, device compatibility.
- OTHER: customer service experience, store visits, KYC/documentation, porting
  process, spam calls, account access/login, anything not fitting above.

DECISION RULES FOR OVERLAPS (apply in order)
R1. Primary = the issue the customer wants RESOLVED (the "ask"), not every topic mentioned.
R2. If a money impact is caused by a service failure (e.g., "charged for data that
    didn't work"), primary = BILLING if they ask for refund/credit; NETWORK if they
    ask to fix the service. The other goes in secondary.
R3. If the customer can't tell whether the cause is phone or network
    ("no signal on my phone"), and no device fault is described, primary = NETWORK.
    Choose DEVICE only if a hardware/settings issue is explicitly stated or the
    problem is on one device only while others work.
R4. Complaints about how support handled an issue → OTHER as primary only if the
    underlying issue is resolved or not described; otherwise classify the
    underlying issue and put OTHER as secondary.
R5. If the text is too vague to determine (e.g., "nothing works!!!"), use OTHER with
    confidence LOW and set needs_human_review = true.

OUTPUT (JSON only, no extra text)
{
  "primary_category": "BILLING|NETWORK|DEVICE|OTHER",
  "secondary_category": "BILLING|NETWORK|DEVICE|OTHER|null",
  "confidence": "HIGH|MEDIUM|LOW",
  "rule_applied": "R1|R2|R3|R4|R5|none",
  "evidence": "<short quote from the complaint that drove the decision>",
  "needs_human_review": true|false
}

Set needs_human_review = true whenever confidence is LOW or two categories seem equally primary.

<complaint>
{{COMPLAINT_TEXT}}
</complaint>
```

---

## 1B. Few-Shot Prompt

```text
ROLE
You are a telecom customer-support triage classifier.

CATEGORIES: BILLING, NETWORK, DEVICE, OTHER.
(Use the same definitions and overlap rules R1–R5 as described here in brief:
classify by what the customer wants resolved; money-from-outage → by the ask;
unclear phone-vs-network → NETWORK unless a device fault is explicit;
support-experience → OTHER only if no underlying issue; too vague → OTHER + review.)

Study the examples. They were chosen specifically to show how ambiguous cases are
resolved. Learn the REASONING, not the surface words.

### Example 1 (clear-cut)
Complaint: "My bill this month is ₹400 higher than my plan price. Please explain."
Output: {"primary_category":"BILLING","secondary_category":null,"confidence":"HIGH",
"rule_applied":"none","evidence":"bill this month is ₹400 higher","needs_human_review":false}

### Example 2 (Billing vs Network – ask is refund)
Complaint: "Internet was down for 3 days in Pune and you still charged me full. I want a credit."
Output: {"primary_category":"BILLING","secondary_category":"NETWORK","confidence":"HIGH",
"rule_applied":"R2","evidence":"I want a credit","needs_human_review":false}

### Example 3 (Billing vs Network – ask is fix)
Complaint: "Paying for 5G but getting 2 Mbps everywhere in my area for weeks. Fix this."
Output: {"primary_category":"NETWORK","secondary_category":"BILLING","confidence":"HIGH",
"rule_applied":"R2","evidence":"getting 2 Mbps … Fix this","needs_human_review":false}

### Example 4 (Device vs Network – device-specific)
Complaint: "No signal on my new phone but my old phone with the same SIM works fine at home."
Output: {"primary_category":"DEVICE","secondary_category":null,"confidence":"HIGH",
"rule_applied":"R3","evidence":"old phone with the same SIM works fine","needs_human_review":false}

### Example 5 (Device vs Network – ambiguous)
Complaint: "Calls keep dropping on my phone since yesterday."
Output: {"primary_category":"NETWORK","secondary_category":"DEVICE","confidence":"MEDIUM",
"rule_applied":"R3","evidence":"Calls keep dropping","needs_human_review":false}

### Example 6 (service experience + underlying issue)
Complaint: "Called support 4 times about my SIM not being activated, each agent was rude and useless."
Output: {"primary_category":"OTHER","secondary_category":null,"confidence":"MEDIUM",
"rule_applied":"R4","evidence":"SIM not being activated … agent was rude","needs_human_review":true}
// SIM activation is a provisioning/account issue → OTHER; rudeness also OTHER. Two
// distinct OTHER issues + ambiguity on whether activation is device → human review.

### Example 7 (too vague)
Complaint: "Worst company ever, nothing works, switching to another provider!!"
Output: {"primary_category":"OTHER","secondary_category":null,"confidence":"LOW",
"rule_applied":"R5","evidence":"nothing works","needs_human_review":true}

### Now classify
Return JSON only, in the same schema.
Complaint: "{{COMPLAINT_TEXT}}"
Output:
```

### Example-selection principles used

- **Balanced labels**: every category appears, and no label dominates (to avoid majority-label bias).
- **Contrastive pairs**: Examples 2 vs 3 have the *same topics* but *different labels*, which teaches that the deciding feature is the "ask", not keywords. Examples 4 vs 5 do the same for device vs network.
- **Includes "I don't know"**: Example 7 shows that LOW confidence and human review are acceptable outputs.
- **Clear-cut example first**: anchors the schema before the hard cases.
- **Varied surface form**: different lengths, tone, and regional details, so the model doesn't latch onto phrasing.

---

## 2. Explanation

### (a) Why this structure?

- **Explicit category definitions plus ordered overlap rules**: overlapping categories are a *policy* problem, not a language problem. "Charged for an outage" isn't ambiguous once the business decides the "ask" wins. Writing the rules down makes classifications consistent and explainable.
- **Primary + secondary label**: real complaints are multi-topic. A single label throws away information. The secondary field captures overlap without breaking the 4-class reporting.
- **`rule_applied` and `evidence` fields**: make each decision auditable and support debugging. If R3 misfires often, you can fix R3.
- **`needs_human_review`**: routes the inevitable truly ambiguous cases to a person instead of forcing a guess.
- **JSON-only output**: directly consumable by the ticketing system.

### (b) Zero-shot vs few-shot: why few-shot helps (or doesn't)

**Why few-shot helps here:**
1. **Edge cases are defined by judgement, not vocabulary.** "Ask-based" classification is hard to convey abstractly. Contrastive examples (2 vs 3) demonstrate it far more clearly than a rule.
2. **Calibrates confidence.** Examples show what MEDIUM and LOW look like. Zero-shot models tend to say HIGH for everything.
3. **Locks the output format.** The model sees exact JSON several times, which reduces schema drift.
4. **Encodes domain conventions.** "SIM activation → OTHER, not DEVICE" is an organisational convention that definitions alone may not fully pin down.

**When zero-shot is preferable / few-shot doesn't help:**
1. **Clear-cut complaints** (most traffic). Zero-shot with good definitions is already about as accurate as few-shot, at lower token cost and latency.
2. **Strong models with well-written rules.** Explicit rules can match few-shot quality, and they are easier to maintain (edit a rule rather than re-curating examples).
3. **When categories change often.** Each change means re-curating and re-balancing examples.

### When few-shot breaks

| Breakage | What happens | Mitigation |
|---|---|---|
| **Surface-feature over-fitting** | Model learns "₹ symbol → BILLING", "Pune → NETWORK" | Vary surface features; use contrastive pairs |
| **Label-distribution bias** | If 4 of 7 examples are NETWORK, ambiguous inputs drift toward NETWORK | Balance labels |
| **Recency bias** | The last example's label is over-predicted | Shuffle the order, or end with a neutral/clear example |
| **Examples contradict the rules** | Model follows examples over rules, so behaviour silently diverges | Validate examples against the rules; keep both in sync |
| **Distribution shift** | New complaint types (e.g., eSIM issues, 5G-specific) fall outside the examples, and the model forces them into the nearest example | Monitor `needs_human_review` rates; refresh examples periodically |
| **Context bloat** | Many examples cost latency and money, and can crowd out instructions | Keep 5–8 examples; use dynamic retrieval (see below) |
| **Copying the evidence/reasoning of examples** | Model parrots an example's evidence string | Ask for a quote *from the complaint*; validate in code that the evidence substring exists in the input |

### (c) Failure modes prevented (both prompts)

- Keyword-based misclassification of overlapping complaints (ordered rules, contrastive examples)
- Forced guessing on vague input (R5, `needs_human_review`)
- Overconfidence (confidence field plus calibration examples)
- Format drift (JSON schema, repeated in examples)
- Unexplainable decisions (`rule_applied` + `evidence`)

---

## 3. Alternative Prompt Designs

### Alternative A: Dynamic few-shot (retrieval-augmented examples)

Keep a labelled bank of ~200 tricky complaints. At runtime, embed the new complaint, retrieve the 4 most similar labelled examples plus 1 clear-cut example per category, and insert them into the few-shot template. **Pro:** examples are always relevant to the current edge case, and it handles distribution shift better. **Con:** needs a vector store and a labelled bank, and retrieved examples may all share one label (enforce diversity).

### Alternative B: Multi-label + confidence per class

```text
For each category, output a probability-like score 0–100 that the complaint
substantially involves that category (scores need not sum to 100). Primary = max.
If top two scores are within 15 points, set needs_human_review = true.
```
**Pro:** expresses overlap naturally and supports threshold tuning. **Con:** LLM "scores" are not truly calibrated probabilities. Validate them against labelled data before relying on thresholds.
