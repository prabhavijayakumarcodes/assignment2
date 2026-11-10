# Q8. Executive-Ready Output

**Goal:** Output for senior leadership that is crisp and action-oriented. Force structured output (tables, bullets), avoid verbosity, keep insights sharp.

---

## 1. Full Prompt

```text
ROLE
You are a chief-of-staff writing for a CEO who has 90 seconds to read this.
Your value is judgement and compression, not completeness.

INPUT
<source_material>
{{REPORT / DATA / MEETING NOTES / ANALYSIS}}
</source_material>

<decision_context>
{{Optional: what decision leadership must make, by when. If blank, infer the most
likely decision from the material and state it as an assumption.}}
</decision_context>

WRITING RULES
1. Lead with the answer (BLUF – bottom line up front). No preamble, no restating the task.
2. Every insight must be a "so-what", not a "what". Bad: "Revenue fell 8%".
   Good: "Revenue fell 8%, driven by SMB churn → retention, not acquisition, is the lever."
3. Every number must come from <source_material>. If a needed number is missing,
   write "[data gap]" — never estimate.
4. Use active voice and concrete verbs (cut, launch, hire, stop, approve).
5. Hard limits:
   - Entire output ≤ 250 words (tables count).
   - Bullets ≤ 20 words each.
   - Max 3 key insights, max 3 recommended actions, max 3 risks.
6. Banned: "it is important to note", "in conclusion", "leverage synergies",
   "various", "a number of", "going forward", hedging stacks ("may potentially").
7. If the material does not support a clear recommendation, say so in the headline
   and make the actions about resolving the uncertainty.

OUTPUT FORMAT (exactly this, nothing before or after)

**Headline:** <one sentence, ≤ 25 words: the answer + the single most important reason>

**Decision needed:** <what we are asking leadership to decide, by when>

**Key insights**
| # | Insight (so-what) | Evidence (number/source) | Impact |
|---|---|---|---|
| 1 | | | High/Med/Low |
| 2 | | | |
| 3 | | | |

**Recommended actions**
| Action (verb-first) | Owner | By when | Expected outcome |
|---|---|---|---|

**Risks & mitigations**
- <Risk> → <mitigation>  (max 3)

**Data gaps:** <comma-separated list, or "None">

BEFORE RETURNING (silent check, do not print):
- Word count ≤ 250? If not, cut the weakest insight's detail first, never the headline.
- Does each insight answer "so what?"?
- Is every action verb-first and owned?
- Is any number not traceable to the source? Remove or mark [data gap].
```

---

## 2. Explanation

### (a) Why this structure?

- **Persona with a time constraint** ("CEO with 90 seconds"): gives the model a concrete reason to compress. "Be concise" is vague; "90 seconds" is a target.
- **BLUF headline first**: executives often read only the first line. The single most important sentence goes where it will be read.
- **"Decision needed" line**: leadership output exists to drive decisions. Naming the decision explicitly prevents informative-but-unactionable summaries.
- **Insight table with a "So-what / Evidence / Impact" structure**: separates the insight from its proof, which supports both skimming (read column 1) and drilling (read evidence). The Impact column enables prioritisation.
- **Action table with Owner + By when**: an action without an owner or deadline is a wish. The columns force it to be accountable.
- **Numeric hard limits** (≤250 words, max 3 items, ≤20 words per bullet): models follow numeric limits much better than adjectives like "brief". Limiting to 3 forces *prioritisation*, which is the actual skill being asked for.
- **Banned phrase list**: removes corporate filler, a common source of verbosity, concretely.
- **Silent self-check with a cut-order rule**: when over the limit, the model knows *what* to cut (detail before headline), so it doesn't truncate randomly.

### (b) Why this technique? Zero-shot with a strict template

- A **template is itself a structural demonstration**, so few-shot examples add little and would cost tokens. Worse, an example executive summary would leak its *content style* and domain into unrelated reports.
- **No visible CoT**: reasoning text is exactly the verbosity we are trying to remove. The model does its reasoning implicitly, plus a silent checklist. (With a reasoning model, let it think internally and output only the template.)

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Wall of text / verbosity | Word cap, bullet cap, item caps, banned phrases |
| Descriptive rather than insightful ("what" vs "so-what") | Rule 2 with a bad/good example, table column name |
| Burying the lead | BLUF headline required first |
| Vague actions ("explore options") | Verb-first rule, Owner/By-when columns |
| Hallucinated figures to look sharp | Rule 3 plus "[data gap]" |
| False certainty when the data is weak | Rule 7 |
| Preamble/meta commentary ("Here is your summary…") | "Exactly this, nothing before or after" |

---

## 3. Alternative Prompt Designs

### Alternative A: Pyramid Principle (Minto) prompt

```text
Structure the output as a Minto pyramid:
Level 1 – Governing thought (1 sentence).
Level 2 – Exactly 3 supporting arguments (MECE), each 1 sentence.
Level 3 – Under each, max 2 bullets of evidence with numbers from the source.
Then: "Ask:" one line stating the decision required.
≤ 180 words total.
```
**Pro:** a well-known consulting format, logically rigorous, and good for slides. **Con:** less room for risks and owners, so it suits a "storyline" slide more than an action memo.

### Alternative B: JSON-then-render

Ask the model for strict JSON (`headline`, `insights[]`, `actions[]`, `risks[]`, each with length limits enforced by a schema validator), then render the tables in code. **Pro:** guaranteed structure and length, since violations are caught programmatically and retried, and the output can be reused in decks or dashboards. **Con:** an extra rendering step, and less natural phrasing.
