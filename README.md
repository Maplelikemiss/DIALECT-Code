# DIALECT-Code

**DIALECT-Code: A Socratic Programming Tutoring Framework Integrating Active Diagnosis and Solution Leakage Control**

Yucheng Xian, Guilin Li, Yinhong Li, and Shenggen Ju

College of Computer Science, Sichuan University, Chengdu 610065, China  
Corresponding author: Shenggen Ju — jsg@scu.edu.cn

Companion repository for the manuscript. This repository currently contains only this README; implementation code, datasets, model weights, and experiment scripts are not included.

## Overview

DIALECT-Code studies how a programming tutor can ask diagnostically useful questions while controlling the solution information disclosed to a learner. It integrates active misconception diagnosis with solution leakage control in a single pedagogical decision process.

An executable misconception graph (EMG) links candidate misconceptions to executable probes and public execution evidence. Learner responses update the diagnostic posterior. Risk-sensitive expected information gain (EIG) ranks questions within a feasible set defined by verifiability, cognitive load, and the remaining disclosure budget. Structured action contracts guide language generation; independent verification and a leakage gate check the realized response before release. Released responses consume the disclosure budget used in subsequent rounds.

The manuscript evaluates executable transfer gain (ETG), diagnosis, and solution reconstruction through an independent offline evaluation channel.

## Method

The main loop comprises the following steps (manuscript Sections 3.1–3.7 and Algorithm 1):

1. **Initialize the diagnostic state.** Use the problem statement, learner program, public execution evidence, and protected reference implementations to construct an EMG and an initial misconception posterior.
2. **Screen and select actions.** Exclude actions that cannot be verified, exceed the cognitive-load ceiling, or have a calibrated leakage upper bound above the remaining budget. During diagnosis, select a feasible action using risk-sensitive EIG with a cognitive-load penalty.
3. **Realize the action contract.** Generate a question or minimal hint from public information and the selected contract. The teacher does not receive reference implementations or private tests.
4. **Verify and gate the response.** Independently verify the generated response and check its calibrated disclosure upper bound before release. Rejected responses are logged.
5. **Update the budget and posterior.** Deduct the released response's upper-bound disclosure cost. Map the learner's response to an observation and update the misconception posterior by Bayes' rule. Low-confidence mappings request clarification and preserve the posterior.
6. **Check closure and record the trajectory.** Diagnosis and closure share the verification process and disclosure budget. The system records evidence, actions, responses, observations, posterior states, budgets, and rejection or termination reasons.

### Misconception categories

The five-category label space groups the fine-grained misconception candidates as follows (Section 3.3 and Table A5):

| Code | Category |
| --- | --- |
| BND | Boundary conditions and initialization |
| CTL | Control flow and termination |
| DAT | State and data representation |
| CON | Interfaces and function contracts |
| ALG | Algorithmic invariants and complexity |

### Default configuration

The defaults below are reported in Sections 3.3 and 3.8 and Algorithm 1.

| Setting | Default |
| --- | --- |
| Reference implementations per task | 3 |
| Fine-grained misconception candidates | 5 |
| Executable probes per round | 2 |
| Maximum interaction rounds (T) | 3 |
| Initial disclosure budget (ρ₀) | 0.050 |
| Calibration parameter (α) | 0.10 |
| Cantelli coefficient (κ) | 3 |
| Cognitive-load penalty (β) | 0.050 nat |
| Cognitive-load ceiling (B) | 0.75 |
| Likelihood-bootstrap replicates for EIG | 2,000 |

These settings are frozen using the manuscript's training and calibration procedures; they are not installation or runtime instructions for this README-only repository.

## Evaluation protocol

### DIALECT-Bench

DIALECT-Bench contains **460 task clusters**: 390 APPS anchors in Python and 70 CodeContests+ anchors in C++. Each cluster contains a problem statement, an erroneous program, three deduplicated references, and execution evidence (Section 4.1).

| Split | APPS | CodeContests+ | Total task clusters |
| --- | ---: | ---: | ---: |
| Training | 100 | 20 | 120 |
| Calibration | 90 | 20 | 110 |
| Test | 200 | 30 | 230 |
| Total | 390 | 70 | 460 |

Source problem identifiers and normalized content are grouped before split assignment. The online public channel contains the problem statement, learner program, and public execution evidence. Private tests are reserved for offline screening, transfer evaluation, and reconstruction scoring, and are excluded from teacher, learner, and observation-mapping prompts.

Each of the 230 test clusters has four transfer roles—near-a, near-b, far-a, and far-b—giving **920 transfer instances**. Two frozen learner profiles share the task matrix and use reversed pretest/posttest form assignments. The main matrix contains **230 clusters × 2 profiles × 15 methods = 6,900 units** (Sections 4.5–4.6).

### Models

The main experiment uses the following roles (Section 4.2 and Table 3):

| Role | Model |
| --- | --- |
| Teacher | Qwen3-Coder-30B-A3B-Instruct |
| Independent verifier | Qwen3-8B |
| Model learner | DeepSeek-Coder-6.7B-Instruct |
| Primary reconstruction attacker | Qwen2.5-Coder-7B-Instruct |

The default learner-state updater is a separately invoked Qwen3-Coder-30B-A3B-Instruct role (Appendix A.1). Model parameters remain frozen. The main transfer protocol uses verified, task-independent state cards; it does not supply the full tutoring dialogue, reference implementations, or private tests to transfer attempts.

### Metrics and inference

- **Executable transfer gain (ETG):** posttest minus pretest performance, averaging the binary pass indicators for one near-transfer and one far-transfer task. Each attempt generates one program without compilation feedback or repair (Section 4.6, Equation 7).
- **Diagnostic accuracy:** whether the largest aggregated category posterior matches independent ground truth.
- **Macro-F1:** the equally weighted mean of the five category F1 scores.
- **Non-negative solution reconstruction gain (SRG⁺):** `max(0, response-condition private-test score − no-response private-test score)`, using a matched attacker budget (Section 3.6, Equation 5).
- **Positive leakage rate:** the proportion of profile-level units with `SRG⁺ > 0`.

For the primary reconstruction measurement, each condition has two candidate generations. The response condition includes one generation using the response and one ignoring it; the no-response condition has two generations without the response. Each condition is scored by its maximum private-test pass score. The main results evaluate the final released teacher response; when no response is released, the attack input is empty. Whole-dialogue and stronger-attack measurements are reported separately in Appendix B.5.

The main comparisons use **230 task clusters as the inferential units**. The **460 profile-level units** are used for point estimates; both profiles are averaged within each cluster before paired differences are computed. Tables A2 and A4 report paired t-tests, 95% confidence intervals, and Holm adjustments. Section 4.5 specifies the bootstrap and robustness procedures. The values below are reported manuscript results, not results produced by this repository.

## Main results

Table 5 of the manuscript reports:

| Method | ETG ↑ | Diagnostic accuracy ↑ | Macro-F1 ↑ | Mean SRG⁺ ↓ | Positive leakage rate ↓ |
| --- | ---: | ---: | ---: | ---: | ---: |
| Full (DIALECT-Code) | 0.150 | 0.8261 | 0.8253 | 0.0165 | 0.0652 |
| Traver-Info | 0.128 | 0.7826 | 0.7812 | 0.0342 | 0.1043 |
| Diagnose-Info | 0.124 | 0.7652 | 0.7639 | 0.0390 | 0.1174 |
| TreeInstruct-Info | 0.121 | 0.7587 | 0.7571 | 0.0375 | 0.1130 |
| Traver-style | 0.116 | 0.7500 | 0.7482 | 0.0401 | 0.1304 |
| Diagnose-Q | 0.112 | 0.7391 | 0.7372 | 0.0482 | 0.1522 |
| Socratic Prompt | 0.102 | — | — | 0.0458 | 0.1391 |
| Direct LLM | 0.071 | — | — | 0.0813 | 0.2174 |
| No-code Guardrail | 0.064 | — | — | 0.0521 | 0.1848 |

A dash indicates that the method does not define an explicit diagnostic output and is excluded from diagnostic-accuracy and Macro-F1 evaluation. Positive leakage rates are proportions; for example, `0.0652` corresponds to `6.52%`.

Full exceeds the information-matched Traver-Info baseline in paired ETG by **0.0217**, with a **95% CI of [0.0085, 0.0350]** and Holm-adjusted **p = 0.001437** (Section 5.1 and Table A2). Paired effects and intervals use unrounded records. ETG means are displayed to three decimal places, while paired ETG differences and their intervals use four.

### Component ablations

Table 7 reports the following point estimates; Table A4 gives paired inference:

| Configuration | ETG ↑ | Diagnostic accuracy ↑ | Mean SRG⁺ ↓ | Positive leakage rate ↓ |
| --- | ---: | ---: | ---: | ---: |
| Full | 0.150 | 0.8261 | 0.0165 | 0.0652 |
| −Multi-Reference | 0.125 | 0.7609 | 0.0257 | 0.0978 |
| −Execution | 0.108 | 0.7130 | 0.0344 | 0.1152 |
| −EMG | 0.080 | 0.6217 | 0.0471 | 0.1348 |
| −EIG | 0.101 | 0.6826 | 0.0542 | 0.1565 |
| Standard EIG | 0.122 | 0.7543 | 0.0293 | 0.1065 |
| −SRG | 0.136 | 0.7891 | 0.0898 | 0.2196 |

The −SRG ablation removes both pre-selection leakage-budget constraints and post-generation SRG gating while retaining the other components. Full reduces positive leakage rate from **21.96% to 6.52%** relative to that ablation. Standard EIG changes the ranking utility while retaining pre-selection screening and post-generation gating (Section 5.5).

## Participant study

Ten adult Python novices completed difficulty-matched tasks with Full and Direct LLM in a counterbalanced within-participant design. Five participants followed each AB/BA order, with at least 48 hours between conditions. The study includes immediate and seven-day transfer assessments; seven-day transfer is the separately tested primary outcome. Participant-level inference uses paired differences, with 9 degrees of freedom for t-tests (Section 4.6 and Appendix B.6).

Table 12 reports means ± standard deviations and paired differences:

| Outcome | Full | Direct LLM | Full − Direct LLM | 95% CI for paired difference |
| --- | ---: | ---: | ---: | --- |
| Immediate transfer | 88.00 ± 5.20 | 79.00 ± 6.10 | 9.00 | [5.42, 12.58] |
| Seven-day transfer | 84.00 ± 5.80 | 68.00 ± 7.00 | 16.00 | [10.99, 21.01] |
| Solution dependence | 0.08 ± 0.10 | 0.36 ± 0.13 | −0.28 | [−0.42, −0.14] |
| Agency | 4.55 ± 0.26 | 3.20 ± 0.40 | 1.35 | [1.03, 1.67] |

Model-learner results and participant-study results have different inferential units and evaluation protocols; their measurements are reported separately in the manuscript.

## Further analyses in the manuscript

The appendices report identical-candidate controls; a four-condition comparison separating EIG ranking from pre-selection leakage screening; natural-dialogue and state-card controls; state-card exchanges; transfer-pair equivalence; source, difficulty, and category strata; sensitivity analyses; and stronger or whole-dialogue reconstruction attacks.

Sections 5.8 and 6 discuss compound-boundary candidate omissions, observation-mapping errors, cross-sentence leakage, task and language scope, and deployment conditions. The calibrated leakage bounds depend on the stated calibration protocol, models, attack budget, and data distribution. Model or distribution changes require recalibration (Section 6.4).

## Repository contents

```text
DIALECT-Code/
└── README.md
```

This README documents the manuscript's method and reported results. It does not provide an executable reproduction package or a dataset download.

## Manuscript citation

```bibtex
@unpublished{xian2026dialectcode,
  title  = {{DIALECT-Code}: A Socratic Programming Tutoring Framework Integrating Active Diagnosis and Solution Leakage Control},
  author = {Xian, Yucheng and Li, Guilin and Li, Yinhong and Ju, Shenggen},
  year   = {2026},
  note   = {Manuscript}
}
```

## Acknowledgements

This work was supported by the Key Program of the National Natural Science Foundation of China (Grant No. 62137001).
