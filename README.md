# Assignment 2 – Prompt Engineering on Real-World Scenarios

This repository contains my solutions to **UST Assignment 2: Prompt Engineering**.

For every question I provide:

1. **The full prompt**: a complete, copy-paste-usable prompt, not a bullet outline.
2. **Explanation**
   - (a) Why this structure?
   - (b) Why this prompting technique (zero-shot / few-shot / CoT / controlled reasoning)?
   - (c) Which failure modes it prevents
3. **At least one alternative prompt design**, with its trade-offs.

## Contents

| # | Section | Question | File |
|---|---------|----------|------|
| 1 | Ambiguity + Incomplete Context | Q1. Consulting Case: Messy Client Problem (Telecom ARPU) | [Q01](section-1-ambiguity/Q01-consulting-telecom-arpu.md) |
| 2 | Ambiguity + Incomplete Context | Q2. Healthcare Risk Scenario | [Q02](section-1-ambiguity/Q02-healthcare-risk.md) |
| 3 | Robustness + Adversarial Inputs | Q3. Prompt Injection Attack | [Q03](section-2-robustness/Q03-prompt-injection.md) |
| 4 | Robustness + Adversarial Inputs | Q4. Toxic + Biased Input Handling | [Q04](section-2-robustness/Q04-toxic-biased-input.md) |
| 5 | Multi-Step Reasoning Design | Q5. Financial Fraud Detection | [Q05](section-3-reasoning/Q05-fraud-detection.md) |
| 6 | Multi-Step Reasoning Design | Q6. Strategy Under Uncertainty (EV in India) | [Q06](section-3-reasoning/Q06-ev-india-strategy.md) |
| 7 | Few-Shot vs Zero-Shot | Q7. Complaint Classification with Edge Cases | [Q07](section-4-few-vs-zero-shot/Q07-complaint-classification.md) |
| 8 | Output Control + Format | Q8. Executive-Ready Output | [Q08](section-5-output-control/Q08-executive-output.md) |
| 9 | Output Control + Format | Q9. Dual Audience Problem | [Q09](section-5-output-control/Q09-dual-audience.md) |
| 10 | Meta Prompting + Self-Critique | Q10. Self-Improving Prompt | [Q10](section-6-meta-prompting/Q10-self-improving.md) |
| 11 | Meta Prompting + Self-Critique | Q11. Prompt Evaluation Framework | [Q11](section-6-meta-prompting/Q11-prompt-evaluator.md) |
| 12 | Real Failure Simulation | Q12. When the Model is Wrong | [Q12](section-7-failure-simulation/Q12-model-is-wrong.md) |
| 13 | Final Challenge | Q13. Prompting Strategy for a Consulting AI Assistant | [Q13](final-challenge/Q13-consulting-assistant-strategy.md) |

## Conventions used across all prompts

- **XML-style tags** (`<input>`, `<rules>`, `<output_format>`) separate instructions from untrusted data, so the model never confuses the two.
- **`{{placeholders}}`** mark where runtime input is injected.
- **Explicit "unknown / insufficient evidence" escape hatches** give the model a legitimate alternative to guessing, which is the most effective single hallucination control.
- **Evidence grounding**: every claim must cite a quote or field from the input, or be labelled as an assumption.
- **Bounded reasoning**: reasoning is requested in a fixed number of named steps, not open-ended "think step by step", so it stays auditable and short.

## Repository layout

```
assignment2/
├── README.md
├── section-1-ambiguity/
├── section-2-robustness/
├── section-3-reasoning/
├── section-4-few-vs-zero-shot/
├── section-5-output-control/
├── section-6-meta-prompting/
├── section-7-failure-simulation/
└── final-challenge/
```

Synthetic sample inputs used to sanity-check each prompt are included inline in the relevant question file.
