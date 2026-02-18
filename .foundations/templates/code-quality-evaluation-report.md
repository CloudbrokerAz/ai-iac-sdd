# Code Quality Evaluation: {{MODULE_NAME}}

**Date**: {{EVALUATION_DATE}}
**Feature**: {{FEATURE_PATH}}
**Evaluator**: code-quality-judge
**Iteration**: {{ITERATION}}

---

## 1. Executive Summary

**Overall Score**: {{OVERALL_SCORE}} / 10.0
**Production Readiness**: {{READINESS_BADGE}}

**Top 3 Strengths**:
1. {{STRENGTH_1}}
2. {{STRENGTH_2}}
3. {{STRENGTH_3}}

**Top 3 Priority Issues**:
1. {{ISSUE_1}}
2. {{ISSUE_2}}
3. {{ISSUE_3}}

---

## 2. Score Breakdown

| # | Dimension | Weight | Score (1-10) | Weighted |
|---|-----------|--------|--------------|----------|
| D1 | Module Structure | 25% | {{D1_SCORE}} | {{D1_WEIGHTED}} |
| D2 | Security & Compliance | 30% | {{D2_SCORE}} | {{D2_WEIGHTED}} |
| D3 | Code Quality | 15% | {{D3_SCORE}} | {{D3_WEIGHTED}} |
| D4 | Variables & Outputs | 10% | {{D4_SCORE}} | {{D4_WEIGHTED}} |
| D5 | Testing | 10% | {{D5_SCORE}} | {{D5_WEIGHTED}} |
| D6 | Constitution Alignment | 10% | {{D6_SCORE}} | {{D6_WEIGHTED}} |
| | **Overall** | **100%** | | **{{OVERALL_SCORE}}** |

{{SECURITY_OVERRIDE_NOTE}}

---

## 3. Dimension Analysis

{{DIMENSION_ANALYSIS}}

---

## 4. Findings

| ID | Severity | Category | Location | Description | Recommendation |
|----|----------|----------|----------|-------------|----------------|
{{FINDINGS_ROWS}}

---

## 5. Improvement Roadmap

### P0 -- Critical (must fix before merge)
{{P0_ITEMS}}

### P1 -- High (fix before release)
{{P1_ITEMS}}

### P2 -- Medium (address in next iteration)
{{P2_ITEMS}}

### P3 -- Low (optional refinement)
{{P3_ITEMS}}

---

## 6. Constitution Compliance

| Rule | Status | Evidence |
|------|--------|----------|
{{CONSTITUTION_ROWS}}

---

## 7. Next Steps

{{NEXT_STEPS}}

{{REFINEMENT_OPTIONS}}
