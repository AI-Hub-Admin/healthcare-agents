# Healthcare Agent Exam Architect & Self-Improvement Loop

**Date**: 2026-03-20
**Status**: Approved
**Approach**: B — Agent + Schema + Harness Pipeline

## Goal

Build a generic system that tests any of the 51 healthcare administration agents against psychometrically valid exam items and recursively improves their system prompts using DSPy GEPA optimization. Modeled on Karpathy's autoresearch loop but adapted for non-deterministic evaluation via LLM-as-judge with structured rubrics.

First target: `revenue-medical-coding-specialist.md` (451 lines, 132 testable claims across ICD-10-CM/PCS, CPT, MS-DRG, HCC, E/M, NCCI).

## Architecture Overview

Three cleanly separated components:

1. **Exam Architect Agent** (`agents/eval-exam-architect.md`) — LLM agent that reads any healthcare agent's `.md`, extracts testable claims, and generates structured exam items (JSON) following NBME/AHIMA psychometric standards.

2. **Question Schema + Validation** (`eval/schema/`) — JSON schema defining valid exam items with psychometric guardrails. Validation script rejects items failing quality checks.

3. **DSPy Eval Harness** (`eval/harness/`) — Python pipeline: loads exam items, runs target agent, scores via two-tier evaluation, feeds scores + textual feedback to GEPA, optimizes agent `.md`, git commits if improved.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Agent location | In this repo (`agents/`) | Co-located with what it tests |
| Source material access | Web fetch at generation time | Always current, uses agent's `services:` URLs |
| Eval strategy | Two-tier: MCQ + rubric-judged scenarios | Knowledge claims need binary scoring; reasoning claims need graduated rubrics |
| Loop autonomy | Full auto (Karpathy style) | Score improved → commit. Score worse → reset. No human gate. |
| Scope | Generic from day one | Exam architect reads any agent `.md` to discover claims |
| Optimizer | DSPy + GEPA (ICLR 2026) | Outperforms MIPROv2 by 10%+, accepts textual feedback, Pareto frontier avoids local optima |

## Component 1: Exam Architect Agent

An LLM agent (`.md` system prompt) that produces structured exam items for any healthcare agent.

### Claim Extraction Algorithm

1. Read target agent's `.md` frontmatter for `name`, `description`, `services` (source URLs)
2. Parse body for assertion patterns:
   - "You know/understand/have expertise in X" → knowledge claim
   - "You can/will/should do X" → reasoning claim
   - Cited guideline sections (e.g., "Section I.C.9.a.1") → knowledge claim with source
   - Cross-references between domains → cross-domain claim
   - "however/unless/except" language, specific edge cases → edge case claim
3. Fetch source URLs from `services:` block to ground questions in current regulatory text
4. Generate items following the schema, targeting Bloom's/DOK distribution below

### Bloom's Taxonomy Distribution Targets

| Claim Type | Bloom's Floor | Distribution |
|-----------|--------------|-------------|
| Knowledge | Understand | 30% understand, 50% apply, 20% analyze |
| Reasoning | Apply | 60% apply, 40% analyze |
| Cross-domain | Apply | 40% apply, 40% analyze, 20% evaluate |
| Edge case | Analyze | 50% analyze, 30% evaluate, 20% create |

### Psychometric Standards (Baked Into Agent Prompt)

- NBME 6th edition item-writing principles: vignette-first structure, focused lead-in, homogeneous distractors
- AHIMA cognitive levels (RE/AP/AN) and CCS multi-select format
- Reject recall-only items even for knowledge claims (googleable = useless)
- Distractors must be same-family codes representing documented coding error patterns
- Every distractor classified: `similar_code | sequencing_error | guideline_misapplication | scope_error | obsolete_code`
- Prohibited: "all of the above," "none of the above," negative stems without emphasis, "which of the following is true"

### What the Agent Does NOT Do

- Does not run or score exams (separation of concerns)
- Does not modify target agent `.md` files
- Does not define rubrics (those are frozen, stored in `eval/rubrics/`)

## Component 2: Question Schema

### Shared Base Fields (Both Tiers)

```json
{
  "id": "uuid-v4",
  "item_code": "MC-K17-001",
  "version": 1,
  "status": "draft | field_test | operational | retired | flagged",
  "target_agent": "revenue-medical-coding-specialist",
  "claim_id": "K17",
  "claim_text": "Hypertension + heart disease = assumed causal relationship (I11.-)",
  "claim_type": "knowledge | reasoning | cross_domain | edge_case",
  "cognitive_level": "understand | apply | analyze | evaluate | create",
  "depth_of_knowledge": 2,
  "tier": "mcq | scenario",
  "source_citation": "ICD-10-CM Official Guidelines Section I.C.9.a.1",
  "source_url": "https://www.cms.gov/medicare/coding-billing/icd-10-codes",
  "content_classification": {
    "domain": "ICD-10-CM coding",
    "subdomain": "Circulatory system - Chapter 9",
    "competency_standard": "CPC: 10000 Series",
    "content_tag_primary": "assumed_causal_relationship",
    "content_tags_secondary": ["hypertension", "heart_failure", "combination_codes"]
  },
  "ctt_targets": {
    "p_value": [0.30, 0.85],
    "point_biserial_min": 0.20,
    "distractor_selection_min": 0.05
  },
  "irt_targets": {
    "b_difficulty": [-2.0, 2.0],
    "a_discrimination": {"min": 0.5, "ideal": [0.8, 2.5]},
    "c_guessing": 0.25
  },
  "information_target": {
    "theta_range": [-1.0, 1.0],
    "min_information_at_theta_0": 0.5
  },
  "item_metadata": {
    "created_at": "2026-03-20T10:00:00Z",
    "last_calibrated_at": null,
    "exposure_count": 0,
    "exposure_cap": 50,
    "content_hash": "sha256:...",
    "parent_item_id": null,
    "enemy_item_ids": [],
    "retirement_reason": null
  },
  "optimization_lineage": {
    "generation_method": "auto_generated | human_authored | hybrid",
    "generator_model": "claude-opus-4-20250514",
    "generator_prompt_version": "exam_architect_v1",
    "optimization_run_id": null,
    "iteration": 0,
    "change_type": "new",
    "change_justification": null
  },
  "calibration_history": [],
  "dif_analysis": {
    "groups_tested": [],
    "dif_flag": false,
    "dif_method": "Mantel-Haenszel",
    "dif_magnitude": null
  }
}
```

### Tier 1 MCQ Extension

```json
{
  "vignette": "A 68-year-old male is admitted with...",
  "lead_in": "What is the correct principal diagnosis code?",
  "response_format": {
    "type": "single_best_answer | multi_select | ordered_response",
    "option_count": 4,
    "select_count": null,
    "select_count_revealed": false
  },
  "options": [
    {
      "key": "A",
      "text": "I11.0 - Hypertensive heart disease with heart failure",
      "is_correct": true,
      "rationale": "Correct: assumed causal relationship per guideline I.C.9.a.1",
      "distractor_error_type": null
    },
    {
      "key": "B",
      "text": "I50.9 - Heart failure, unspecified",
      "is_correct": false,
      "rationale": "Misses the hypertensive relationship assumption",
      "distractor_error_type": "guideline_misapplication"
    },
    {
      "key": "C",
      "text": "I10 - Essential hypertension",
      "is_correct": false,
      "rationale": "Does not capture the heart disease component",
      "distractor_error_type": "scope_error"
    },
    {
      "key": "D",
      "text": "I13.0 - Hypertensive heart and CKD",
      "is_correct": false,
      "rationale": "No CKD documented",
      "distractor_error_type": "similar_code"
    }
  ],
  "answer_position_tracking": {
    "correct_position": "A",
    "note": "Randomize across bank; target uniform distribution"
  }
}
```

### Tier 2 Scenario Extension

```json
{
  "prompt": "Given this discharge summary, assign all ICD-10-CM codes...",
  "rubric": [
    {
      "criterion": "Principal diagnosis selection",
      "weight": 0.30,
      "max_points": 4,
      "scoring_levels": [
        {"score": 4, "label": "exemplary", "description": "Selects I13.10, cites I.C.9.a.3, explains combination code logic"},
        {"score": 3, "label": "proficient", "description": "Selects I13.10 with correct reasoning, incomplete citation"},
        {"score": 2, "label": "developing", "description": "Selects I13 family but wrong specificity"},
        {"score": 1, "label": "novice", "description": "Selects related but wrong category (e.g., I11.0)"},
        {"score": 0, "label": "incorrect", "description": "Unrelated code or no code"}
      ],
      "anchor_response_exemplary": null,
      "anchor_response_borderline": null,
      "common_errors": ["Selecting I11.0 (misses CKD)", "Coding separately instead of combination"]
    }
  ],
  "judging_config": {
    "judge_model": "claude-opus-4-20250514",
    "judge_prompt_template": "tier2_coding_rubric_v1",
    "num_judges": 3,
    "agreement_threshold": 0.70,
    "agreement_metric": "weighted_kappa",
    "adjudication": "majority_vote",
    "calibration_responses": []
  },
  "scoring_rule": {
    "type": "modified_conjunctive",
    "minimum_per_criterion": 2,
    "weighted_total_minimum": 0.70,
    "critical_criteria": ["principal_diagnosis_selection"],
    "critical_criterion_minimum": 3
  }
}
```

### Validation Rules (validate.py)

Reject items that:

1. Have Bloom's level "remember" for any claim type
2. Have Bloom's level below "apply" for reasoning/cross-domain/edge claims
3. Have fewer than 4 options for MCQ
4. Have options with >2x length variance (longest-answer-correct cue)
5. Lack source citation
6. Have vignette shorter than 50 words for scenario items
7. Have Jaccard similarity > 0.8 against existing bank items
8. Use "all of the above," "none of the above," or "which of the following is true"
9. Have negative stems without emphasis
10. Contain absolute terms ("always," "never") in distractors (test-wiseness cue)
11. Have non-functional distractors (< 5% selection rate in calibration data)
12. Have positive point-biserial on any distractor (high-ability agents selecting wrong answer)
13. Have item-total correlation below 0.15
14. Have IRT outfit > 1.5 or infit > 1.3 (misfitting items)
15. Have fewer than 3 or more than 7 rubric criteria for Tier 2 items
16. Have non-randomized correct answer positions across the bank

## Component 3: DSPy Eval Harness

### Pipeline Stages

```
1. EXTRACT   →  Parse target agent .md, extract testable claims
2. GENERATE  →  Dispatch exam architect agent to produce exam items (JSON)
3. VALIDATE  →  Run items through schema validation + psychometric checks
4. EXECUTE   →  Run target agent against each exam item, capture responses
5. SCORE     →  Tier 1: auto-score MCQ. Tier 2: multi-judge rubric scoring
6. ANALYZE   →  Compute CTT stats, IRT calibration (3PL), distractor analysis, item fit
7. OPTIMIZE  →  Feed scores + textual feedback to GEPA optimizer
8. APPLY     →  GEPA produces revised .md sections
9. VERIFY    →  Re-run exam against revised agent, confirm improvement
10. COMMIT   →  Score improved: git commit. Score worse: git reset. Repeat.
```

### Section-Aware Optimization

The harness parses the agent `.md` into semantic sections (ICD-10-CM guidelines, PCS root operations, DRG logic, E/M, HCC, modifiers, NCCI, etc.). GEPA optimizes failing sections independently, then reassembles and runs an integration eval to catch regressions.

### GEPA Integration

GEPA returns `dspy.Prediction(score=..., feedback=...)`. The textual feedback ("agent missed CC exclusion logic in sepsis scenario - Section I.C.1.d not referenced") targets mutations at specific weaknesses. GEPA maintains a Pareto frontier of candidates that each achieve highest score on at least one evaluation instance.

### Tier 2 Scoring Pipeline

1. Three LLM judge instances score each scenario response independently
2. Weighted kappa computed across judges
3. If agreement < 0.70, flag rubric as ambiguous (problem is the test, not the agent)
4. Calibration anchor responses fed to judges before real scoring to verify alignment
5. Judge prompt itself is optimizable via Evidently pattern (meta-loop)

### Frozen Metric Principle

The rubrics and MCQ answer keys are immutable. The agent `.md` can change; the definition of "correct" cannot. This is the analog to Karpathy's immutable `prepare.py`. Constitutional principles for healthcare coding serve as the fixed reference point.

### Commit Cycle

- Score improved → `git commit -m "opt_047: medical-coding +3.2% (sepsis, CC exclusion)"`
- Score equal or worse → `git reset --hard`
- No human gate
- Scores embedded in commit messages for traceability

### Cost Model

- GEPA "medium" (40 trials) per agent: ~$3-15
- 51 agents full pass: $150-750
- Run overnight in batches

## Directory Structure

```
healthcare-agents/
├── agents/
│   ├── revenue-medical-coding-specialist.md     # target (51 total)
│   └── eval-exam-architect.md                   # exam builder agent
├── eval/
│   ├── schema/
│   │   ├── item_schema.json                     # JSON Schema (both tiers)
│   │   └── validate.py                          # psychometric validation
│   ├── harness/
│   │   ├── pipeline.py                          # main orchestrator
│   │   ├── claim_extractor.py                   # parse .md → claims
│   │   ├── exam_runner.py                       # execute agent against items
│   │   ├── scorer.py                            # Tier 1 auto + Tier 2 judge
│   │   ├── analyzer.py                          # IRT calibration, item stats
│   │   ├── optimizer.py                         # GEPA wrapper, section-aware
│   │   └── config.py                            # model configs, thresholds
│   ├── items/
│   │   └── {agent-name}/                        # per-agent item banks
│   │       ├── bank.jsonl                       # item bank (append-only)
│   │       └── calibration/                     # IRT calibration data
│   ├── results/
│   │   └── {agent-name}/                        # per-agent run results
│   │       └── opt_{NNN}/                       # per-optimization-run
│   ├── rubrics/
│   │   ├── coding_scenario_v1.json              # shared rubric templates
│   │   └── calibration_anchors.json             # pre-scored anchors
│   └── requirements.txt                         # dspy, numpy, scipy
├── scripts/
│   ├── lint-agents.sh
│   └── run-eval.sh                              # kick off the loop
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-03-20-exam-architect-eval-loop-design.md
```

## Success Criteria

### Medical Coding Agent (First Target)

- Exam bank of ~130 items covering all 132 testable claims
- Distribution: ~40% Tier 1 MCQ, ~40% Tier 2 scenario, ~20% multi-select
- All items pass schema validation and psychometric screening
- Baseline score established before optimization
- At least one complete optimization loop executed end-to-end
- Score improvement demonstrated on at least one claim category

### Generic System (Works Across 51 Agents)

- Exam architect extracts claims from at least 3 agents across different divisions
- Schema validates items regardless of healthcare domain
- Harness runs end-to-end on any agent without code changes
- Item banking tracks versions across optimization iterations

## Key References

- **GEPA**: arxiv.org/abs/2507.19457 (ICLR 2026), github.com/gepa-ai/gepa
- **DSPy**: github.com/stanfordnlp/dspy (v3.1.3+)
- **auto-rag-eval**: github.com/amazon-science/auto-rag-eval (ICML 2024)
- **NBME Item Writing Guide**: nbme.org/educators/item-writing-guide (6th ed)
- **Karpathy autoresearch**: github.com/karpathy/autoresearch
- **LLM exam quality**: PMC 10981304 (systematic review, 22% error rate)
- **IRT 3PL model**: auto-rag-eval arXiv 2405.13622
- **SPO**: arxiv.org/abs/2502.06855 (pairwise optimization without ground truth)
- **OpenAI Self-Evolving Agents**: developers.openai.com/cookbook
- **Evidently judge optimization**: evidentlyai.com/blog/llm-judge-prompt-optimization
