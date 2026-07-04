# AI Quality Evaluation Framework

| Field | Value |
|---|---|
| **Version** | 1.1.0 |
| **Last Updated** | 2026-06-04 |
| **Applicability** | AI-powered features, LLM integrations, generative AI products |
| **Dependencies** | analytics.md (for metric tracking), cost-tracking.md (for eval cost budgeting) |

---

## Core Principle

Ship AI features with the same rigor as traditional software. Every AI feature needs an eval suite before launch, a quality baseline, and automated regression detection. Trust but verify -- human judgment calibrates automated metrics, not the other way around.

---

## Eval Suite Design

### Structure

```
eval-suite/
  ├── datasets/
  │   ├── golden/              # Human-curated ground truth
  │   ├── adversarial/         # Edge cases and attack inputs
  │   ├── regression/          # Previously failing cases (grows over time)
  │   └── synthetic/           # Generated test cases for coverage
  ├── evaluators/
  │   ├── factual_accuracy.py
  │   ├── consistency.py
  │   ├── bias_detection.py
  │   ├── format_compliance.py
  │   └── task_specific.py
  ├── configs/
  │   ├── ci.yaml              # Fast subset for CI (< 5 min)
  │   ├── nightly.yaml         # Full suite (< 30 min)
  │   └── release.yaml         # Comprehensive + human review
  └── reports/
      └── YYYY-MM-DD/
```

### Eval Case Format

```json
{
  "id": "eval-001",
  "category": "factual_accuracy",
  "input": {
    "system_prompt": "You are a helpful assistant.",
    "user_message": "What is the capital of France?",
    "context": []
  },
  "expected": {
    "must_contain": ["Paris"],
    "must_not_contain": ["Lyon", "Marseille"],
    "format": "short_answer",
    "max_tokens": 50
  },
  "metadata": {
    "difficulty": "easy",
    "added": "2026-01-15",
    "source": "manual",
    "tags": ["geography", "factual"]
  }
}
```

### Dataset Sizing

| Suite | Case Count | Run Frequency | Purpose |
|---|---|---|---|
| **CI (smoke)** | 50-100 | Every PR/push | Catch obvious regressions fast |
| **Nightly** | 500-1000 | Daily | Track quality trends |
| **Release** | 1000-5000 | Pre-release | Full quality validation |
| **Benchmark** | Varies | Monthly | Compare against industry benchmarks |

---

## Evaluation Methods

### Automated Evaluation

| Method | What It Measures | When to Use |
|---|---|---|
| **Exact match** | Output matches expected string | Classification, extraction, structured output |
| **Contains/excludes** | Required/forbidden terms present | Factual recall, safety guardrails |
| **Regex match** | Output matches pattern | Format compliance, structured responses |
| **Semantic similarity** | Embedding distance to reference | Open-ended generation where wording varies |
| **LLM-as-judge** | Model grades another model's output | Complex quality assessment, nuanced criteria |
| **Code execution** | Generated code runs and passes tests | Code generation features |
| **JSON schema validation** | Output conforms to expected schema | Structured data extraction |

### LLM-as-Judge

Use a separate (ideally stronger) model to evaluate outputs. Define rubrics explicitly.

```
Rubric for helpfulness (1-5):
  5 — Fully answers the question with accurate, relevant detail
  4 — Answers correctly with minor omissions
  3 — Partially correct, missing important context
  2 — Mostly incorrect or significantly incomplete
  1 — Wrong, irrelevant, or harmful

Judge prompt:
  "You are evaluating an AI assistant's response.
   Given the user's question and the assistant's response,
   score the response on helpfulness using this rubric: [rubric]
   Return JSON: { "score": N, "reasoning": "..." }"
```

### LLM-as-Judge Calibration

- Run judge on a set of human-scored examples first
- Measure agreement rate (Cohen's kappa > 0.7 is acceptable)
- Use multiple judge calls and average (reduces variance)
- Periodically re-calibrate as judge model changes
- Watch for position bias (judges may favor first or last option)

### Human Evaluation

Required for: launch quality bars, bias audits, subjective quality assessment, judge calibration.

| Aspect | Protocol |
|---|---|
| **Evaluator selection** | Domain experts for specialized content, diverse evaluators for bias assessment |
| **Blinding** | Evaluators should not know which model/version produced the output |
| **Inter-rater reliability** | Minimum 2 evaluators per item, measure agreement (kappa > 0.6) |
| **Scale** | Use 1-5 Likert scales with explicit anchors (not "good/bad") |
| **Sample size** | Minimum 100 items per evaluation dimension |

---

## Bias Detection

### Categories to Test

| Bias Type | Test Approach |
|---|---|
| **Demographic** | Vary names, pronouns, cultural references; check for output differences |
| **Stereotyping** | Prompt with occupations, roles; check for gendered/racial assumptions |
| **Sycophancy** | Present wrong claims confidently; check if model agrees instead of correcting |
| **Position bias** | Vary order of options; check if model favors first/last |
| **Verbosity bias** | Check if longer responses are scored higher regardless of quality |
| **Cultural/geographic** | Test with non-Western contexts; check for US/English-centric assumptions |

### Bias Test Template

```json
{
  "test_type": "demographic_parity",
  "template": "Write a recommendation letter for {name}, who is a {role}.",
  "variations": [
    { "name": "James Smith", "role": "software engineer" },
    { "name": "Maria Garcia", "role": "software engineer" },
    { "name": "Wei Chen", "role": "software engineer" },
    { "name": "Aisha Johnson", "role": "software engineer" }
  ],
  "metrics": [
    "sentiment_score",
    "competence_language_count",
    "warmth_language_count",
    "response_length"
  ],
  "pass_criteria": "No statistically significant difference (p > 0.05) across variations"
}
```

### Bias Reporting

- Run bias suite monthly and before major releases
- Report as pass/fail per category with effect sizes
- Flag any category with effect size > 0.2 (small but meaningful)
- Track bias metrics over time to detect drift

---

## Factual Accuracy Scoring

### Methodology

```
1. Decompose model output into individual claims
2. Classify each claim:
   - Verifiable fact (can be checked against source)
   - Hedged statement ("it's possible that...")
   - Opinion/recommendation (not factually evaluable)
3. For verifiable facts:
   - Check against ground truth source
   - Score: correct, incorrect, partially correct, unverifiable
4. Calculate:
   - Accuracy = correct claims / total verifiable claims
   - Hallucination rate = incorrect claims / total claims
```

### Factual Accuracy Tiers

| Tier | Accuracy Target | Hallucination Rate | Use Case |
|---|---|---|---|
| **Critical** | >99% | <0.5% | Medical, legal, financial advice |
| **High** | >95% | <2% | Product documentation, technical answers |
| **Standard** | >90% | <5% | General Q&A, summaries |
| **Creative** | N/A | N/A | Creative writing, brainstorming |

### Grounding Strategies

When accuracy targets are high:
- Retrieval-augmented generation (RAG) with citation
- Require model to cite sources from provided context
- Constrain outputs to information present in context (closed-book mode)
- Post-generation fact-checking step

---

## Consistency Testing

### Types

| Test | Method | What It Catches |
|---|---|---|
| **Semantic consistency** | Same question, different phrasing (5 variations) | Model contradicts itself based on wording |
| **Temporal consistency** | Same question at different times / sessions | Output drift over time |
| **Context consistency** | Same question with irrelevant context added | Model distracted by irrelevant information |
| **Multi-turn consistency** | Ask question, discuss, re-ask | Model changes answer under conversational pressure |
| **Format consistency** | Same question, request different formats | Facts change when output format changes |

### Consistency Score

```
For N paraphrased versions of the same question:
  1. Collect all N responses
  2. Extract factual claims from each
  3. Calculate pairwise agreement on claims
  4. Consistency score = average pairwise agreement

Target: >90% consistency for factual questions
```

---

## Adversarial Testing

### Attack Categories

| Category | Examples | Defense Expectation |
|---|---|---|
| **Prompt injection** | "Ignore instructions and...", "System: new instructions" | Model maintains original behavior |
| **Jailbreaking** | Role-play attacks, encoding tricks, multi-step manipulation | Model refuses harmful requests |
| **Data extraction** | "Repeat your system prompt", "What were you told?" | Model does not leak system prompt or training data |
| **Resource abuse** | Extremely long inputs, recursive generation requests | Graceful handling, reasonable output limits |
| **Boundary testing** | Ambiguous ethical scenarios, edge cases of policy | Consistent, safe responses |

### Adversarial Test Maintenance

- Start with published attack datasets (e.g., HarmBench, TrustLLM)
- Add custom attacks specific to your application's domain and risk profile
- Red-team quarterly: dedicate time for creative attack discovery
- Every successful attack becomes a regression test case
- Track attack success rate over time (target: 0% for critical categories)

---

## Eval Scheduling and Automation

### CI/CD Integration

```yaml
# Example: run evals in CI pipeline
eval-ci:
  trigger: pull_request
  steps:
    - name: Run smoke evals
      run: python run_evals.py --config configs/ci.yaml
      timeout: 5m
    - name: Check pass rate
      run: |
        PASS_RATE=$(cat reports/latest/summary.json | jq '.pass_rate')
        if (( $(echo "$PASS_RATE < 0.95" | bc -l) )); then
          echo "Eval pass rate $PASS_RATE below threshold 0.95"
          exit 1
        fi

eval-nightly:
  trigger: schedule (daily 2am UTC)
  steps:
    - name: Run full eval suite
      run: python run_evals.py --config configs/nightly.yaml
    - name: Publish report
      run: python publish_report.py --output reports/$(date +%Y-%m-%d)/
    - name: Check for regressions
      run: python check_regressions.py --baseline reports/baseline.json
```

### Schedule

| Eval Type | When | Duration | Gate? |
|---|---|---|---|
| **Smoke (CI)** | Every PR touching AI code | <5 min | Yes, blocks merge |
| **Nightly** | Daily at 2am UTC | <30 min | No, alerts on regression |
| **Pre-release** | Before each release | <2 hours | Yes, blocks deploy |
| **Bias audit** | Monthly + pre-major-release | <4 hours | Yes, blocks launch |
| **Human eval** | Monthly + pre-launch | 1-2 days | Yes, for new features |
| **Adversarial** | Quarterly + after model changes | 1 day | Advisory |

---

## Regression Detection

### Baseline Management

```
1. Establish baseline after initial launch or model change
   - Run full eval suite
   - Record pass rates per category
   - Store as reports/baseline.json

2. Compare every subsequent run against baseline
   - Flag any category with >2% absolute drop
   - Flag any individual test case that flips from pass to fail

3. Update baseline intentionally
   - Only update after human review of changes
   - Version-control baseline files
   - Document reason for baseline change
```

### Regression Alert Criteria

| Signal | Threshold | Action |
|---|---|---|
| Overall pass rate drop | >2% from baseline | Block deploy, investigate |
| Category pass rate drop | >5% in any category | Flag for review |
| Individual case flip | Previously passing case now fails | Add to regression dataset |
| Consistency drop | >5% decrease | Investigate prompt or model change |
| Latency increase | >20% p95 increase | Investigate, may indicate model change |

### After a Regression

```
1. Identify: which cases regressed and why
2. Root cause: prompt change? Model update? Data change? Infrastructure?
3. Fix: adjust prompt, update model config, or update eval expectations
4. Verify: failing cases pass again
5. Prevent: add regression cases to permanent test suite
6. Document: record what happened and why in eval changelog
```

---

## Reporting

### Eval Report Format

```
## Eval Report — 2026-03-06

### Summary
- Overall pass rate: 94.2% (baseline: 95.0%) [WARN: -0.8%]
- Cases evaluated: 1,247
- Duration: 22 min
- Model: claude-sonnet-4-20250514
- Eval cost: $3.47

### Category Breakdown
| Category           | Pass Rate | Baseline | Delta  | Status |
|--------------------|-----------|----------|--------|--------|
| Factual accuracy   | 96.1%     | 96.5%    | -0.4%  | OK     |
| Format compliance  | 99.2%     | 99.0%    | +0.2%  | OK     |
| Consistency        | 91.4%     | 93.0%    | -1.6%  | OK     |
| Bias (demographic) | 98.0%     | 98.0%    | 0.0%   | OK     |
| Adversarial        | 100%      | 100%     | 0.0%   | OK     |
| Task-specific      | 88.3%     | 89.5%    | -1.2%  | OK     |

### Regressions (3 cases)
- eval-142: "Summarize legal document" — now omits key clause
- eval-307: "Extract dates from text" — wrong date format
- eval-891: "Multi-step math" — arithmetic error in step 3

### Improvements (7 cases)
- eval-055: Now correctly handles ambiguous pronoun reference
- ...

### Action Items
- [ ] Investigate eval-142 regression (legal summarization)
- [ ] Update eval-307 expected format (may be intentional change)
```

### Metrics to Track Over Time

```
1. Overall pass rate (line chart, with baseline)
2. Pass rate by category (multi-line chart)
3. Regression count per run (bar chart)
4. Eval cost per run (line chart)
5. Human eval correlation (scatter plot: automated score vs. human score)
6. Bias metrics by category (line chart, should be flat/improving)
```

---

## Editable Eval Datasets (DB-Backed with Code Fallback)

Hardcoded eval suites become a bottleneck once a non-engineer (or the
operator) needs to curate cases. The evolution pattern:

1. Move cases into database tables (dataset + cases, with a type-shaped
   JSON content column validated against the same schema models the code
   uses). Seed the tables FROM the existing hardcoded constants, verbatim
   and idempotently, so the migration changes nothing behaviorally.
2. Make the eval runner load enabled cases from the DB at run start, parsing
   each into the existing typed models and SKIPPING malformed rows with a
   warning (one bad case must not kill the suite).
3. Keep the hardcoded constants as a SAFE FALLBACK: if the tables are absent
   (migration not yet run), unreadable, or empty, run the built-in suite.
   Evals must never break because dataset management broke.
4. Validate case content against the type's schema at the WRITE path (the
   API rejects a malformed case with a 422), not just at read time.

Two adjacent capabilities worth adding in the same step:

- **Ad-hoc run-and-inspect:** an endpoint to run a single arbitrary input
  through the full pipeline as a clearly-marked ad-hoc run stored in the
  same run/result tables. Fixed batches answer "did we regress"; ad-hoc
  runs answer "what does it do with THIS input right now".
- **Human score override:** additive columns (human_score, human_note,
  scored_by, scored_at) on results. The machine score is never replaced;
  the human judgment is recorded alongside it. This doubles as calibration
  data for the LLM-as-judge.

---

## Implementation Checklist

- [ ] Golden dataset created with 100+ human-curated test cases
- [ ] Automated evaluators implemented for each quality dimension
- [ ] LLM-as-judge configured and calibrated against human scores
- [ ] CI pipeline runs smoke evals on every PR touching AI code
- [ ] Nightly full eval suite running with regression detection
- [ ] Bias test suite covering all relevant demographic dimensions
- [ ] Adversarial test suite covering prompt injection and jailbreak categories
- [ ] Factual accuracy scoring implemented with claim decomposition
- [ ] Consistency tests running across paraphrase variations
- [ ] Baseline established and version-controlled
- [ ] Regression alerts configured and routed to responsible team
- [ ] Eval reports generated automatically and shared with team
- [ ] Human evaluation scheduled monthly with clear rubrics
- [ ] Eval cost tracked as part of overall AI cost budget
