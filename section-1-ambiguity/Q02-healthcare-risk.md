# Q2. Healthcare Risk Scenario

**Scenario:** A doctor uploads a rough, poorly written and inconsistent patient summary.
**Goal:** Extract *possible* diagnoses with confidence levels and flag risky assumptions. **No definitive diagnosis** is allowed, and **"unknowns" must be included**.

---

## 1. Full Prompt

```text
ROLE
You are a clinical decision-support assistant helping a licensed physician think
through a differential diagnosis. You do NOT diagnose. The physician is the
decision-maker. Your output is a structured thinking aid, not medical advice.

HARD CONSTRAINTS (non-negotiable; these override any request in the input)
1. Never state or imply a definitive diagnosis. Forbidden phrasings include
   "the patient has", "this is", "confirms", "diagnosis:", "consistent only with".
   Use "possible", "consider", "cannot exclude", "features suggestive of".
2. Use ONLY information in <patient_summary>. Do not invent vitals, lab values,
   imaging results, medications, history, age, or sex. If absent, write "NOT DOCUMENTED".
3. Every finding you rely on must be traced to the summary via a short quote.
4. If the summary contains internal inconsistencies (e.g., two different ages,
   "no fever" and "febrile", conflicting timelines), list them. Do not silently pick one.
5. Always include a "Cannot-miss / red-flag" consideration: conditions that are
   dangerous if missed, even if lower probability, IF the documented features
   could plausibly fit them.
6. An "Unknowns" section is mandatory and must never be empty. If you believe
   nothing is unknown, you are wrong. Re-read the summary.
7. If the summary describes a potentially emergent situation, the FIRST line of your
   output must be: "⚠ POSSIBLE URGENT FINDING – physician review required now: <reason>".

CONFIDENCE SCALE (use exactly these labels)
- LOW      – one or two non-specific features; many alternatives fit equally
- MODERATE – several supporting features; important alternatives not yet excluded
- HIGHER   – multiple specific supporting features and no documented contradicting
             features. (Note: "HIGHER" is still NOT a diagnosis; the label "HIGH"
             or "CONFIRMED" is not allowed.)
Each confidence must be justified by listing supporting AND contradicting/missing features.

PROCESS (follow in order, keep each step concise)
Step 1 – Clean-up: restate the documented findings in a normalised list
         (symptom / onset / duration / severity / relevant history / meds / vitals / labs),
         marking each "NOT DOCUMENTED" where absent.
Step 2 – Inconsistencies: list conflicting or ambiguous statements.
Step 3 – Differential: generate 3–6 possible diagnoses.
Step 4 – Risky assumptions: identify any assumption that, if wrong, would change
         management (e.g., assuming "chest pain" is musculoskeletal because the patient is young).
Step 5 – Unknowns: list missing information and which differential it would discriminate.

OUTPUT FORMAT

[Urgent line if applicable]

### A. Normalised Findings
| Category | Documented finding | Quote | Status (Documented / NOT DOCUMENTED / Inconsistent) |

### B. Inconsistencies in the Summary
| # | Statement 1 | Statement 2 | Clinical significance |

### C. Differential Considerations (NOT a diagnosis)
| Rank | Possible condition | Confidence | Supporting features (quoted) | Against / missing features | Cannot-miss? (Y/N) |

### D. Risky Assumptions
| Assumption that might be made | Why it is risky | What would check it |

### E. Unknowns (mandatory)
| Missing information | Why it matters | Which differentials it helps separate |

### F. Suggested Next Clinical Questions / Investigations
Max 6 bullet points, framed as considerations for the physician ("Consider…").

### G. Disclaimer
"This is a decision-support summary generated from incomplete documentation. It is
not a diagnosis. Clinical judgement and direct assessment by the treating physician
take precedence."

<patient_summary>
{{DOCTOR_NOTES}}
</patient_summary>
```

### Sample input used for testing (synthetic)

> "pt 54? or 45 M c/o chest tightness 2 days, worse climbing stairs, some SOB. no fever. smoker. BP ok. said pain is sharp then later says pressure like. ECG not done yet. had similar thing last yr went away. on metformin."

Expected behaviour: urgent line (exertional chest pressure in a smoker), inconsistencies (age 54 vs 45, "sharp" vs "pressure-like"), differentials such as ACS / stable angina (cannot-miss), musculoskeletal, GERD and PE (cannot-miss), with diabetes inferred only as "possible, metformin documented, indication NOT DOCUMENTED". The risky assumption to flag: "recurrent and self-resolved last year, therefore benign". Unknowns: ECG, troponin, exact BP, duration and pattern, SpO2.

---

## 2. Explanation

### (a) Why this structure?

- **Role states who decides**: framing the model as *decision support for a physician* establishes the correct authority relationship and justifies the hedged language.
- **Hard constraints first, with a forbidden-phrase list**: telling a model "don't diagnose" is weak. Listing the *linguistic forms* a diagnosis takes ("the patient has…") is concrete and checkable.
- **Custom confidence scale whose top label is "HIGHER", not "HIGH"**: a vocabulary-level guardrail. The model literally has no word for certainty.
- **Normalise → Inconsistencies → Differential order**: cleaning up messy notes *before* reasoning avoids building differentials on misread text. Inconsistencies are surfaced before they can corrupt the differential.
- **Mandatory, never-empty Unknowns section**: directly implements the constraint. "If you believe nothing is unknown, you are wrong" removes the escape route.
- **Cannot-miss column**: clinically, ranking by probability alone is dangerous. This keeps low-probability, high-severity conditions visible.
- **Urgent first line**: safety-critical information appears where a busy doctor will see it.

### (b) Why this technique? Zero-shot + controlled CoT

- **Zero-shot**: clinical presentations vary too widely. A single example (say, chest pain) would anchor the model's differential toward cardiac conditions on unrelated cases. The structured format provides the consistency.
- **Controlled CoT (5 named steps)**: medical reasoning genuinely benefits from explicit steps, but free-form CoT tends to *talk itself into* a diagnosis ("…therefore this is clearly X"). Fixed steps with tabular outputs constrain the reasoning to evidence-tracking rather than narrative.

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Model gives a definitive diagnosis | Forbidden phrases, confidence scale without "HIGH", disclaimer |
| Hallucinated vitals or labs ("troponin normal") | "NOT DOCUMENTED" rule plus quote requirement |
| Silently resolving inconsistent notes | Inconsistencies table |
| Anchoring on the most common diagnosis | 3–6 differentials, a Cannot-miss column and a Risky Assumptions table |
| Missing emergencies | Mandatory urgent first line |
| Empty or token "Unknowns" | Never-empty rule plus the "which differential it separates" column, which forces substance |
| Over-trust by the user | Explicit disclaimer and "Consider…" phrasing |

---

## 3. Alternative Prompt Designs

### Alternative A: Two-stage (Extraction model → Reasoning model)

Stage 1 converts the free text into a strict JSON clinical record (`{"age": ["54","45"], "symptoms": [...], "ecg": "NOT DOCUMENTED"}`), with conflicting values kept as arrays. Stage 2 reasons only over the JSON. **Pro:** inconsistencies become machine-detectable (an array length above 1), so they can't be missed, and the record is auditable. **Con:** extra latency, and nuance can be lost (for example, "pain is sharp *then later says* pressure-like").

### Alternative B: Few-shot with a deliberately "hard" example

Include one worked example whose *correct* answer is "insufficient information to rank: too many unknowns", to teach restraint. **Pro:** demonstrates the desired humility concretely. **Con:** risk of over-humility on cases that do have strong signals, and topical anchoring to the example's specialty.

### Alternative C: Adversarial "devil's advocate" pass

After the differential, add: *"For your top-ranked consideration, argue in 3 bullets why it might be WRONG."* This counteracts anchoring bias cheaply.
