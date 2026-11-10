# Q9. Dual Audience Problem

**Same input → 2 outputs:** (1) Technical team (detailed), (2) Business team (simplified).
**Constraint:** ONE prompt that dynamically adjusts output by audience. No separate prompts.

---

## 1. Full Prompt

```text
ROLE
You are a communication specialist who translates the same source material for
different audiences without changing the underlying facts.

INPUTS
<source>
{{SOURCE_MATERIAL: e.g., incident report, model evaluation, architecture proposal}}
</source>

<audience>{{TECHNICAL | BUSINESS | BOTH}}</audience>

SINGLE SOURCE OF TRUTH (do this first, internally, for any audience)
Extract a "core facts" set from <source>: what happened / what is proposed, key
numbers, causes, impact, decisions needed, risks, next steps. Both audience versions
MUST be derived from this same set. A fact may be simplified for BUSINESS but never
changed, rounded misleadingly, or contradicted. Do not add facts not in <source>.

AUDIENCE PROFILES (apply the one(s) selected in <audience>)

| Dimension         | TECHNICAL                                         | BUSINESS                                         |
|-------------------|---------------------------------------------------|--------------------------------------------------|
| Primary question  | "How does it work / what exactly broke / how to fix?" | "What does it mean for customers, cost, risk, and what do you need from me?" |
| Depth             | Full detail: components, root cause chain, metrics, configs, trade-offs | Outcome level: impact, cost, timeline, decision |
| Vocabulary        | Precise technical terms, no simplification         | Plain language; any unavoidable term gets a ≤10-word gloss in brackets |
| Numbers           | Exact values, units, percentiles, error rates     | Rounded, translated to business units (₹, customers, hours, %) |
| Structure         | Sections: Summary → Root cause / Design → Details → Action items with owners → Open technical questions | Sections: Bottom line → Business impact → What we're doing → Decision/ask → Risks |
| Length            | As long as needed, ~300–600 words                 | ≤ 150 words                                       |
| Format            | Headings, numbered steps, code/config blocks allowed, tables | 3–5 bullets + one short table max; no code       |
| Tone              | Neutral, precise, blameless                        | Confident, reassuring where justified, never hiding risk |

ADAPTATION RULES
1. Select the profile(s) from <audience>. If <audience> is BOTH, output TECHNICAL
   first, then BUSINESS, separated by a line "---".
2. If <audience> is empty or not one of the allowed values, default to BOTH and say
   so in one line at the top.
3. Business version must keep every risk and decision that appears in the technical
   version (simplified, not omitted). Leaving out a material risk is a failure.
4. Technical version must not include business-persuasion language.

OUTPUT TEMPLATES

[TECHNICAL]
## Technical Summary
(2–3 sentences)
## Root Cause / Design Detail
## Supporting Data
| Metric | Value | Notes |
## Action Items
| # | Action | Owner (team) | Priority |
## Open Technical Questions

[BUSINESS]
## Bottom Line
(1 sentence)
## What It Means for the Business
- Customer impact:
- Financial impact:
- Timeline:
## What We Are Doing
## What We Need From You
## Risks to Watch

CONSISTENCY CHECK (silent, before returning)
- Does every number in BUSINESS trace to a number in TECHNICAL / source?
- Does BUSINESS mention every risk and decision listed in TECHNICAL?
- Does BUSINESS contain any unexplained jargon? If yes, gloss or remove.
```

### Example usage

Source: an incident report about a payment-gateway outage (API p99 latency 12 s, connection-pool exhaustion after a config change, 47 minutes, ~18,000 failed transactions).

- `<audience>TECHNICAL</audience>` gives root cause (pool size reduced from 200 → 50 in deploy #4821, retries amplified load), metrics table, and action items (pool alerting, config change review, retry jitter).
- `<audience>BUSINESS</audience>` gives "A configuration error caused 47 minutes of failed payments (~18,000 transactions). Fixed; safeguards being added. Ask: approve 2 days of engineering time for preventive monitoring. Risk: some customers may need proactive refunds/communication."

Same facts, different depth, vocabulary and framing, from one prompt.

---

## 2. Explanation

### (a) Why this structure?

- **Audience as a parameter (`<audience>` variable)**: satisfies "one prompt" literally. The same template serves both, and the variable switches behaviour. It also supports `BOTH` in a single call, guaranteeing the two outputs come from one reasoning pass.
- **"Single source of truth" extraction first**: the biggest dual-audience risk is that the two versions *disagree* (e.g., business version says "minor issue" while technical version shows 18k failures). Deriving both from one core-facts set prevents divergence.
- **Side-by-side audience profile table**: makes the differences explicit on 8 dimensions instead of leaving "simplify for business" to interpretation. A table is also easy to extend (e.g., add a LEGAL column).
- **Separate output templates**: each audience gets the structure that matches its *primary question*. Engineers want root cause and actions, while executives want impact and asks.
- **Rule 3 (risk preservation)**: simplification tends to drop bad news. Explicitly requiring every risk to survive translation protects decision-makers.
- **Consistency check**: a cross-version audit that catches number drift and missing risks.

### (b) Why this technique? Zero-shot, parameterised (conditional) prompting

- **Conditional instruction / parameterisation**: the technique for "one prompt, many behaviours" is to declare all behaviours with explicit selectors, rather than separate prompts.
- **Zero-shot**: the profile table carries the style specification. Few-shot pairs (technical + business versions of one example) would double the prompt length and anchor the domain (e.g., everything starts to look like an incident report).
- **Implicit extraction step** (light CoT): extracting core facts first is a minimal reasoning scaffold that buys consistency without verbose output.

### (c) Failure modes prevented

| Failure mode | Guardrail |
|---|---|
| Two versions contradict each other | Single source of truth plus consistency check |
| Business version hides bad news | Rule 3: all risks and decisions preserved |
| Business version full of jargon | Plain-language rule plus gloss requirement |
| Technical version dumbed down | Profile: "no simplification", exact values |
| Hallucinated business impact (₹ figures not in the source) | "Do not add facts not in <source>" |
| Ambiguous or missing audience value | Rule 2 default to BOTH |
| Length creep in the business version | ≤150-word cap |

---

## 3. Alternative Prompt Designs

### Alternative A: Layered (progressive disclosure) single output

Instead of two outputs, produce one document in which detail increases as you read:
```text
Write ONE document in three layers:
Layer 1 "For everyone" (≤ 60 words, plain language, the bottom line and the ask).
Layer 2 "For managers" (≤ 150 words, impact, timeline, risks).
Layer 3 "For engineers" (full technical detail).
Each layer must be self-contained and consistent with the layers below it.
```
**Pro:** one artifact, and each reader stops when they have enough. **Con:** business readers see a long document, and the format doesn't suit separate channels (e.g., a Slack message to execs).

### Alternative B: Audience-profile injection

Keep the prompt generic and pass a *profile object* at runtime: `{"role":"CFO","expertise":"low-technical","cares_about":["cost","risk"],"max_words":120}`. **Pro:** scales to N audiences (legal, ops, customer) without editing the prompt. **Con:** profile quality determines output quality, so it needs a curated profile library.
