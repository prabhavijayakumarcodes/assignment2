# Q13. Design a Prompting Strategy (Not Just a Prompt)

**Scenario:** Building an AI assistant for consulting teams.
**Design:** A prompting strategy across **data extraction → reasoning → validation**, not a single prompt.
**Must include:** when to use few-shot vs zero-shot, when to enforce vs suppress reasoning, and how to reduce hallucinations systematically.

---

## 1. Architecture Overview

```
                         ┌────────────────────────────┐
  User request ─────────▶│ 0. ROUTER / INTENT CLASSIFIER│  (few-shot, no visible reasoning)
                         └─────────────┬──────────────┘
                                       │ task_type, risk_level
             ┌─────────────────────────┼──────────────────────────┐
             ▼                         ▼                          ▼
 ┌────────────────────┐   ┌────────────────────────┐   ┌──────────────────────┐
 │ 1. EXTRACTION      │   │ 2. REASONING           │   │ 4. FORMATTING        │
 │ docs, interviews,  │──▶│ hypotheses, sizing,    │──▶│ exec / technical /   │
 │ data → JSON facts  │   │ options, recommendation│   │ slide-ready output   │
 │ (few-shot, T=0,    │   │ (structured CoT,       │   │ (zero-shot template, │
 │  no reasoning)     │   │  facts-only context)   │   │  reasoning suppressed)│
 └────────────────────┘   └───────────┬────────────┘   └──────────▲───────────┘
                                      ▼                           │
                          ┌────────────────────────┐              │
                          │ 3. VALIDATION          │──── pass ────┘
                          │ claim-by-claim check,  │
                          │ numeric recompute,     │──── fail (≤2 retries) ──▶ back to 2
                          │ critic model           │──── still fail ─────────▶ human review
                          └────────────────────────┘
```

**Design principle:** each stage does **one kind of cognitive work** with a prompt specialised for it. Mixing extraction, reasoning and formatting in one prompt is the main source of hallucination, because the model "fills in" facts while reasoning and "smooths over" uncertainty while formatting.

Data passes between stages as **structured JSON with provenance**, so every downstream claim can be traced to a source span.

---

## 2. Stage-by-Stage Prompt Design

### Stage 0: Router

| Aspect | Choice |
|---|---|
| Purpose | Classify request: `extraction`, `analysis`, `market_sizing`, `recommendation`, `summarisation`, `drafting`; assign `risk_level` (client-facing numbers = HIGH) |
| Technique | **Few-shot** (6–10 labelled examples). Intent boundaries are fuzzy and org-specific, and examples teach them best. |
| Reasoning | **Suppressed.** Output is a JSON label only. Reasoning adds latency and no accuracy on short classification. |
| Hallucination control | Closed label set; `UNCLEAR → ask user` path |

```text
Classify the consultant's request. Output JSON only:
{"task_type": "<extraction|analysis|market_sizing|recommendation|summarisation|drafting|unclear>",
 "risk_level": "<LOW|HIGH>", "needs_documents": true|false}
HIGH if the output will contain client-facing numbers or recommendations.
<examples> ... 8 labelled examples incl. 2 'unclear' ... </examples>
<request>{{USER_REQUEST}}</request>
```

### Stage 1: Data Extraction

| Aspect | Choice |
|---|---|
| Purpose | Turn unstructured inputs (interview notes, PDFs, annual reports, Excel exports) into structured facts |
| Technique | **Few-shot** (2–3 examples) when the schema is complex or domain-specific (e.g., extracting KPIs with periods and units from financial statements). **Zero-shot + JSON schema** for simple schemas. |
| Reasoning | **Suppressed.** Extraction should be literal. Reasoning during extraction invites inference ("revenue probably grew"), which is a hallucination vector. |
| Temperature | 0 |
| Hallucination control | Verbatim `source_quote` required per fact; `null` for missing; no derived values |

```text
ROLE: You are a precise data extractor. You do not interpret, infer, or calculate.

Extract every fact matching the schema from <document>. Rules:
1. Each extracted item MUST include "source_quote": the exact verbatim span.
2. If a field is not explicitly stated, set it to null. Never compute or estimate.
3. Keep units and periods exactly as written (do not convert).
4. If the document states conflicting values, extract both and set "conflict": true.

Schema:
{"facts":[{"metric":str,"value":str,"unit":str|null,"period":str|null,
  "entity":str,"source_quote":str,"page_or_section":str|null,"conflict":bool}],
 "stakeholder_opinions":[{"who":str,"opinion":str,"source_quote":str}],
 "explicitly_missing":[str]}

<examples>{{2 worked examples for complex schemas only}}</examples>
<document>{{DOC_CHUNK}}</document>
```

**Post-processing in code (not LLM):** verify that every `source_quote` is an exact substring of the source document. Drop or flag items that fail. This catches fabricated quotes deterministically and cheaply.

### Stage 2: Reasoning

| Aspect | Choice |
|---|---|
| Purpose | Hypothesis trees, market sizing, option evaluation, root-cause analysis, recommendations |
| Technique | **Zero-shot with frameworks** (MECE trees, issue trees, weighted criteria, scenario analysis) given as *structure*, not examples. Few-shot would anchor conclusions to the example's industry and answer. Exception: **few-shot for house-style methods** (e.g., the firm's specific market-sizing template). |
| Reasoning | **Enforced, as controlled CoT.** Named steps, each citing fact IDs from Stage 1. Free-form CoT is avoided (storytelling risk). For high-stakes, multi-option decisions, use **ToT-lite**: evaluate 2–3 options in parallel against fixed criteria (see Q6). |
| Context | The model receives **only the Stage 1 JSON**, not raw documents. It literally cannot see unextracted text, which removes a whole hallucination class. |
| Hallucination control | Every claim tagged `[F#]` (fact ID), `[CALC]` (with formula shown), `[DOMAIN]` (general knowledge, needs review) or `[ASSUMPTION]` (with sensitivity) |

```text
ROLE: Senior consultant. You reason ONLY from <facts> (IDs F1..Fn).

Rules:
1. Every claim must end with a tag: [F#] (cite fact ID), [CALC: formula],
   [DOMAIN] (general industry knowledge – will be reviewed), or [ASSUMPTION: …].
2. Calculations: show formula and inputs with fact IDs; do not round until the end.
3. If facts are insufficient for a conclusion, write "INSUFFICIENT FACTS:
   need <X>" instead of concluding.
4. Separate reasoning from conclusion. Reasoning in numbered steps; conclusion last.
5. State confidence (H/M/L) per conclusion, based on share of [F#] vs [DOMAIN]/[ASSUMPTION] support.

Task: {{TASK_FROM_ROUTER}}
Framework to apply: {{e.g., ARPU driver tree / TAM-SAM-SOM / weighted criteria}}
<facts>{{STAGE1_JSON}}</facts>
```

### Stage 3: Validation

Three validators. Run the cheap ones first, and only run the expensive one for HIGH risk.

| Validator | Type | What it checks |
|---|---|---|
| **3a. Citation check** | Code | Every `[F#]` exists; every claim sentence has a tag; the share of `[DOMAIN]` + `[ASSUMPTION]` tags is below a threshold |
| **3b. Numeric recompute** | Code (or code-interpreter tool) | Re-executes every `[CALC]` formula from fact values; flags mismatches |
| **3c. Critic LLM** | **Separate prompt, ideally a different model**, zero-shot rubric, **reasoning enforced** | Claim-by-claim support check (Chain-of-Verification), logical gaps, overconfidence, missing counterarguments |

```text
ROLE: You are an independent reviewer. You did not write this analysis.

For each numbered claim in <analysis>:
- Check it against <facts>. Status: SUPPORTED / PARTIALLY SUPPORTED / UNSUPPORTED / CONTRADICTED.
- For [DOMAIN] claims: rate plausibility H/M/L and flag any specific numbers as "verify".
- Flag any conclusion whose confidence exceeds its support.
- Flag missing obvious counter-hypotheses (max 3).

Output JSON:
{"claims":[{"id":..,"status":..,"issue":..}],
 "overconfidence_flags":[..], "missing_counterpoints":[..],
 "verdict":"PASS|REVISE|ESCALATE", "revision_instructions":[max 5 items, ≤25 words each]}

PASS only if no CONTRADICTED claims and ≤ 1 UNSUPPORTED non-critical claim.
<facts>{{STAGE1_JSON}}</facts>
<analysis>{{STAGE2_OUTPUT}}</analysis>
```

**Loop control:** REVISE sends the `revision_instructions` to Stage 2 (targeted fixes only). **Maximum 2 retries**, then ESCALATE to a human with the critic's report attached. This is the Q10 loop-prevention logic, enforced in orchestration code rather than in the prompt.

### Stage 4: Formatting

| Aspect | Choice |
|---|---|
| Purpose | Render validated content for the audience: exec summary (Q8), dual-audience (Q9), slide headlines, emails |
| Technique | **Zero-shot strict template.** Few-shot only for the firm's house slide style (1–2 examples of approved headlines). |
| Reasoning | **Suppressed.** Visible reasoning is verbosity. The thinking is already done. |
| Hallucination control | "Use only content in <validated_analysis>; do not add facts, numbers, or claims." Code diff check: any number in the output must appear in the validated input. |

---

## 3. Decision Rules

### When to use few-shot vs zero-shot

| Use **few-shot** when… | Use **zero-shot** when… |
|---|---|
| The output **schema or format** is complex or idiosyncratic (extraction JSON, house slide style) | The task is **reasoning** where examples would anchor *conclusions* (strategy, hypotheses) |
| **Category boundaries are fuzzy** and org-specific (router, complaint classification, Q7) | Clear definitions/rules fully specify the task |
| You need **calibration** of confidence labels or edge-case behaviour (show what LOW looks like) | Inputs vary widely in domain, so any example would be off-topic for most calls |
| The model shows **format drift** in zero-shot testing | Token budget or latency is tight and zero-shot already meets accuracy on evals |
| **Rule:** examples are balanced, contrastive, and include an "unknown/abstain" case; refresh when inputs drift. | **Rule:** compensate with explicit definitions, decision rules, and output templates. |

**Meta-rule:** start zero-shot. Add few-shot only when an eval shows a *specific* failure that examples fix, and prefer **dynamically retrieved** examples (similar past cases) over static ones.

### When to enforce reasoning vs suppress it

| **Enforce** (controlled CoT) | **Suppress** (direct output) |
|---|---|
| Multi-step quantitative work (market sizing, financial modelling, Q5 risk scoring) | Literal extraction (reasoning invites inference) |
| Decisions with trade-offs (Q6 strategy) | Simple classification/routing (adds latency, no accuracy) |
| Validation / verification (Q12, critic) | Formatting for executives (reasoning = verbosity) |
| Diagnosing contradictions in inputs (Q1, Q2) | Translation / tone rewriting where the facts are fixed |
| Any HIGH-risk, client-facing output | Low-risk drafting |

**How reasoning is enforced safely:** named steps, not "think step by step"; each step cites evidence IDs; reasoning is kept *separate* from the final answer so it can be logged but hidden from end users; the length of each step is bounded.

**With native reasoning models:** let the model think internally, and still require the *structured, cited* output. The visible structure is for auditability, not for making the model think.

### How to reduce hallucinations systematically

Hallucination control is applied in **layers**, so no single layer has to be perfect:

| Layer | Mechanism | Stage |
|---|---|---|
| **1. Input isolation** | Reasoning sees only extracted facts, never raw documents; untrusted text is delimited | 1→2 |
| **2. Grounding requirement** | Verbatim `source_quote` for every fact; `[F#]` tag for every claim | 1, 2 |
| **3. Legitimate "unknown" paths** | `null`, `NOT PROVIDED`, `INSUFFICIENT FACTS`, `CANNOT DETERMINE`, so abstaining is a valid answer | All |
| **4. Epistemic tagging** | `[F#]` vs `[CALC]` vs `[DOMAIN]` vs `[ASSUMPTION]`, so the reader sees what is grounded | 2 |
| **5. Deterministic checks in code** | Quote-substring match, fact-ID existence, numeric recompute, numbers-in-output ⊆ numbers-in-input | 1, 3, 4 |
| **6. Independent critic** | Separate prompt or model, claim-by-claim verification (CoVe) | 3 |
| **7. Calibration controls** | Confidence derived from support mix; banned certainty words; separate risk vs confidence | 2, 3 |
| **8. Bounded retries + human escalation** | Max 2 revisions, then a human with the critic's report | 3 |
| **9. Low temperature for factual stages** | T=0 for extraction and validation; moderate T only for ideation | 1, 3 |
| **10. Retrieval over memory** | Market stats come from retrieved sources (RAG/search) with citations, not model recall | 2 |
| **11. Continuous evaluation** | Golden test set of past engagements; track hallucination rate, citation accuracy, escalation rate per release | Ops |

---

## 4. Operational Practices

- **Prompt versioning:** every prompt lives in source control with an ID and version; outputs log the prompt version.
- **Eval suite per stage:** extraction (precision/recall against labelled docs), reasoning (expert-graded rubric, using Q11), validation (catch-rate on seeded errors).
- **Red-teaming:** a regular injection and contradiction test set (Q3, Q1 patterns) run before each release.
- **Human-in-the-loop by risk:** LOW risk ships after Stage 3 passes. HIGH risk always gets consultant sign-off, and the UI shows the citations and `[DOMAIN]`/`[ASSUMPTION]` tags highlighted.
- **Feedback capture:** consultant corrections are logged, so error types (Q12 taxonomy) feed back into prompt and few-shot updates.
- **Confidentiality:** client data is processed only in approved environments. Extraction prompts strip PII before downstream stages where possible.

---

## 5. Why this strategy (summary)

| Requirement | How the strategy satisfies it |
|---|---|
| Not just one prompt | 5 specialised stages with typed hand-offs |
| Few-shot vs zero-shot | Explicit decision table plus meta-rule (start zero-shot, add examples on evidence of failure) |
| Enforce vs suppress reasoning | Explicit decision table: enforce for multi-step, high-risk and verification; suppress for extraction, classification and formatting |
| Systematic hallucination reduction | 11 independent layers combining prompt design, code checks, critic model, human review and evals |

---

## 6. Alternative Strategy Designs

### Alternative A: Single agent with tools (ReAct-style)

One agent prompt with tools (`search_docs`, `extract_table`, `calculator`, `web_search`, `verify_claim`) that plans and acts iteratively. **Pro:** flexible and handles open-ended requests; less orchestration code. **Con:** less predictable, harder to audit, and validation happens only if the agent chooses to do it. Mitigation: make `verify_claim` mandatory before `final_answer` via tool-gating.

### Alternative B: Template-first ("playbook") assistant

Pre-build prompt chains for the 10–15 most common consulting tasks (market sizing, interview synthesis, competitor benchmark, issue tree, exec summary), each with its own tuned few-shot examples and validators, and let the router pick a playbook. **Pro:** highest quality and consistency on common tasks, easy to evaluate. **Con:** poor coverage of novel requests, so it needs a fallback to the general pipeline above.
