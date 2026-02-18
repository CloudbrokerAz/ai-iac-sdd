# Proposal: Terraform Module SDD Workflow v2

**Author**: AI-assisted analysis from S3 Static Website module development run
**Date**: 2026-02-18
**Status**: Draft
**Scope**: Redesign of the spec-driven development workflow for Terraform module authoring

---

## Executive Summary

After completing a full `/tf-plan` workflow run for the `terraform-aws-s3-static-website` module, we conducted a data-driven analysis of the execution. The current 8-phase workflow produced 17 artifacts across 2,221 lines, took ~26 minutes, and required 2 remediation iterations to resolve cross-artifact inconsistencies — the most expensive of which (a CRITICAL constitution violation) was caused by the workflow's own artifact proliferation.

This proposal presents a redesigned 4-phase workflow that eliminates the root causes of those inefficiencies: content duplication across artifacts, sequential phases that could be parallel, and a remediation cycle that exists only because multiple artifacts can contradict each other.

The core changes:
- **One design document** replaces 5 separate artifacts (spec, plan, contracts, data model, tasks)
- **Test-driven implementation** where `.tftest.hcl` files are written before module code
- **Security embedded in every phase** rather than bolted on as a separate review
- **No consistency analysis phase** because there is nothing to get inconsistent

---

## Problem Statement

### Evidence from the S3 Module Run

| Metric | Value | Concern |
|--------|-------|---------|
| Total planning time | ~26 min | Acceptable, but improvable |
| Remediation cycles | 2 iterations, 8.6 min | **22% of total time** fixing cross-artifact contradictions |
| Artifacts produced | 17 files | High maintenance surface area |
| Total artifact lines | 2,221 | ~35% is duplicated content |
| Agent invocations | 12 | Each reads overlapping file sets |
| Content duplication | ~35% | Variable lists appear 3x, resource lists 4x, validation rules 3x |

### Root Cause Analysis

The CRITICAL finding that triggered remediation was a contradiction between `spec.md` (which said public access is configurable) and `plan.md` (which hardcoded it for security). This class of error is **structural** — it exists because the same design decision lives in multiple files that are written by different agents at different times.

| Finding ID | Severity | Root Cause | Time Cost |
|------------|----------|------------|-----------|
| D1 | CRITICAL | Spec said configurable; plan said hardcoded. Two artifacts, two agents, no sync. | 5.8 min |
| E1 | HIGH | Task coverage matrix had 10 wrong cross-references to task IDs | Included above |
| F2 | HIGH | Contracts regex contradicted spec text for bucket naming | Included above |

All three root causes share a pattern: **information exists in multiple places, and the copies diverged**.

### Duplication Map

| Content | spec.md | plan.md | contracts/ | tasks.md | Total |
|---------|---------|---------|------------|----------|-------|
| Variable list (18 vars) | Lines 166-168 | Referenced | Full table (lines 11-32) | T003 description | 3x |
| Resource list (9 resources) | FR-001 to FR-024 | Resource inventory table | data-model.md entities | Phase descriptions | 4x |
| Validation rules (5 rules) | FR-025 to FR-028 | Referenced | Validation table (lines 47-53) | T003, T012, T027, T028 | 3x |
| Security controls | Compliance section | Security checklist table | N/A | Security finding refs | 3x |
| Output definitions (6 outputs) | Lines 167 | Referenced | Output table (lines 34-43) | T006, T015 | 3x |

### Checklist vs. Analysis Overlap

The workflow ran both a checklist phase (120 items) and a consistency analysis phase (16 findings). Overlap was minimal:

| Metric | Value |
|--------|-------|
| Checklist gaps found | 16 `[!]` items |
| Analysis findings (iter 1) | 16 findings |
| Overlap between them | 1 item (6%) |
| Unique to checklists | 15 items (requirement quality gaps) |
| Unique to analysis | 15 items (cross-artifact consistency) |

The analysis phase exists entirely to catch cross-artifact inconsistencies. **If there is only one artifact, the analysis phase is unnecessary.** The checklist function (validating requirement quality) can be embedded as a section within the design document review.

---

## Proposed Workflow: 4 Phases

```
 Phase 1              Phase 2           Phase 3              Phase 4
 ────────────        ────────────      ────────────         ────────────
 UNDERSTAND      →   DESIGN       →   BUILD + TEST    →   VALIDATE
 Clarify + Research   Single doc        TDD-style           Security + Quality

 ~8 min               ~8 min            ~15 min              ~5 min
 0 artifacts          1 artifact        Module code          Test results
                      (design.md)       + test files         Security scan
```

**Total estimated time: ~36 min** (planning + implementation + validation combined)
**Current workflow**: ~26 min planning only, implementation is separate

### Phase 1: Understand

**Goal**: Know exactly what to build before writing anything.

**Duration**: ~8 min (includes user interaction for clarification)

**Steps**:

1. **Environment validation** — Run `validate-env.sh`, verify TFE token via MCP `list_terraform_orgs`
2. **Requirements intake** — Read prompt/issue, extract module name, features, resources, variables, outputs
3. **Clarify ambiguities** — Scan requirements against the 8-category taxonomy. Ask up to 4 questions via `AskUserQuestion`:
   - Always include a **security-defaults question** ("Should X be configurable or hardcoded for security?") — this prevents the most expensive class of downstream contradictions
   - Focus on: security model, feature boundaries, integration points, constraint tradeoffs
4. **Research** — Launch parallel MCP research agents for provider-specific unknowns:
   - `search_providers` + `get_provider_details` for resource behavior
   - `search_documentation` + `read_documentation` for AWS best practices
   - `search_modules` for registry patterns and conventions
5. **GitHub issue creation** — Create tracking issue with requirements summary

**Output**: Clarified requirements + research findings in orchestrator memory (no files written). Research feeds directly into Phase 2.

**What changed from v1**: Research artifacts are not written to disk. They existed only as inputs to the plan agent — writing them created terminology drift (3 MEDIUM findings in our run from variable name mismatches between research files and other artifacts). The design agent receives research findings directly.

### Phase 2: Design

**Goal**: One document that tells an implementer everything needed to build and test the module.

**Duration**: ~8 min (single agent invocation)

**Output**: `specs/{FEATURE}/design.md`

**Document structure**:

```markdown
# Module Design: terraform-aws-{name}

## Purpose
[One paragraph — what the module creates, who consumes it, why it exists]

## Interface Contract

### Inputs
| Variable | Type | Required | Default | Validation | Description |
[Single source of truth for all input variables — NOT repeated elsewhere]

### Outputs
| Output | Type | Conditional On | Description |
[Single source of truth for all outputs]

## Resources & Architecture

### Resource Inventory
| Resource | Logical Name | Conditional | Depends On | Key Configuration |
[Every resource in one table with conditionality and wiring]

### Architectural Decisions
[Inline notes explaining WHY, not just WHAT. Each decision includes:]
- Decision: What was chosen
- Rationale: Why (with MCP research citation where applicable)
- Alternative rejected: What else was considered

## Security Controls
| Control | Enforcement | Configurable? | CIS/WA Reference |
[Every security decision. This IS the security specification.]

## Test Scenarios

### Scenario 1: Secure Defaults (basic example)
- **Inputs**: {only required inputs + test overrides}
- **Assertions**:
  - Encryption algorithm is AES256
  - Versioning status is Enabled
  - All four public access block flags are true
  - Bucket policy contains TLS enforcement deny statement
  - Required tags (Environment, Owner, CostCenter, ManagedBy) present
  - Website outputs are null when website disabled

### Scenario 2: Full Features (complete example)
- **Inputs**: {all features enabled}
- **Assertions**:
  - Website index/error documents configured
  - CloudFront OAC policy grants only s3:GetObject with SourceArn condition
  - KMS encryption active with bucket_key_enabled
  - CORS rules present with specified origins
  - Logging configured with target bucket and prefix

### Scenario 3: Input Validation (error cases)
- **Expect error**: bucket_name with uppercase letters
- **Expect error**: environment = "production" (not in allowed list)
- **Expect error**: lifecycle_glacier_days = 0
- **Expect error**: cloudfront_oac_access_enabled = true without ARN

## Implementation Phases
- [ ] Phase A: Scaffold + versions.tf + all variables + locals + base bucket + core outputs
- [ ] Phase B: Encryption + versioning + public access block + TLS bucket policy
- [ ] Phase C: Website config + CloudFront OAC policy + website outputs
- [ ] Phase D: Lifecycle + CORS + logging
- [ ] Phase E: Examples (basic/ + complete/)
- [ ] Phase F: Tests (.tftest.hcl from scenarios above)
- [ ] Phase G: README + formatting + validation
```

**What changed from v1**:
- Replaces 5 artifacts (spec.md, plan.md, contracts/module-interfaces.md, contracts/data-model.md, tasks.md) with 1
- Variable list exists once (not 3x)
- Resource list exists once (not 4x)
- Validation rules exist once (not 3x)
- Implementation phases are a simple checklist (not a 244-line document with coverage matrices)
- Test scenarios are written in the design phase — they become the implementation target

**Security integration**: The Security Controls table is part of the design, not a separate review. Every security decision is made during design, documented with CIS/Well-Architected references, and translated into test assertions in the Test Scenarios section.

### Phase 3: Build + Test (TDD-Style)

**Goal**: Write tests first from design.md scenarios, then implement the module to pass them.

**Duration**: ~15 min

**This is the fundamental shift**: Terraform test is not a validation step at the end — it drives the implementation.

**Step 1 — Write test files from design.md test scenarios**:

Convert each test scenario from Section 5 of design.md into `.tftest.hcl` assertions:

```hcl
# tests/basic.tftest.hcl — from Design Scenario 1
run "secure_defaults" {
  command = plan

  variables {
    bucket_name = "test-secure-defaults-bucket"
    environment = "dev"
    owner       = "test-team"
    cost_center = "CC-TEST"
  }

  # From design.md: "Encryption algorithm is AES256"
  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.this.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "AES256"
    error_message = "Default encryption must be AES256 (SSE-S3)"
  }

  # From design.md: "All four public access block flags are true"
  assert {
    condition     = aws_s3_bucket_public_access_block.this.block_public_acls == true
    error_message = "block_public_acls must be true"
  }

  # ... remaining assertions from design scenario
}
```

**Step 2 — Implement module code phase-by-phase**:

Follow the implementation phases from design.md Section 6. After each phase:
1. Run `terraform validate` — catch syntax errors immediately
2. Run `terraform test` — see which new assertions now pass
3. Continue to next phase

This creates a tight feedback loop: write code → run tests → see progress.

**Step 3 — Write examples**:

Create `examples/basic/` and `examples/complete/` that exercise the test scenarios. These are the test fixtures — they exist to be tested, not just documented.

**Why TDD for Terraform modules**:

| Benefit | Explanation |
|---------|-------------|
| Interface locked early | Test files reference variable names and resource attributes — any interface change breaks tests |
| Security enforced from line 1 | Encryption, public access, TLS assertions exist before any resource code |
| Clear definition of done | All tests green = implementation complete |
| No separate testing phase | Tests grow with the code, not bolted on after |
| Regression protection | Future changes that break security defaults are caught immediately |
| Documentation by example | Test assertions document expected behavior more precisely than prose |

### Phase 4: Validate

**Goal**: Confirm the module meets security and quality bars.

**Duration**: ~5 min

**All steps run in parallel**:

1. `terraform test` — all scenarios pass (final confirmation)
2. `terraform validate` + `terraform fmt -check -recursive` — clean code
3. `trivy config .` — no Critical/High security findings
4. `terraform-docs markdown . > README.md` — generate documentation
5. **Quick security checklist** — validate against key CIS/Well-Architected controls:
   - Encryption at rest: always on, no disable toggle?
   - Encryption in transit: TLS enforced via bucket policy?
   - Public access: blocked by default?
   - IAM: least-privilege policies only?
   - Logging: documented guidance if opt-in?

**What's NOT here**:
- No consistency analysis (one artifact = nothing to contradict)
- No remediation cycle (no cross-artifact inconsistencies to fix)
- No checklist resolution (requirement quality is validated during design review)
- No separate security review agent (security is embedded in design + test assertions)

**When to escalate**: If `trivy` finds Critical/High, or if `terraform test` fails on security assertions, stop and flag to user. These are genuine issues, not artifact inconsistencies.

---

## Migration Path

### What to Keep from v1

| Component | Keep? | Rationale |
|-----------|-------|-----------|
| Phase 1 clarification questions | Yes | High ROI — prevents most expensive errors |
| MCP research (parallel agents) | Yes | Uniquely valuable — discovers architecture-shaping facts |
| `tf-domain-taxonomy` 8-category scan | Yes | Good framework for identifying ambiguities |
| `aws-security-advisor` | Embed | Security controls become a design.md section, not a separate review |
| `sdd-checklist` | Remove | Requirement quality is validated by design document review |
| `sdd-analyze` | Remove | No cross-artifact consistency to check with single document |
| `sdd-tasks` | Remove | Implementation phases are a simple checklist in design.md |
| Remediation cycle | Remove | Root cause (multi-artifact inconsistency) eliminated |
| GitHub issue tracking | Simplify | 3 comments: start, design complete, implementation complete |

### What to Build

| Component | Description | Effort |
|-----------|-------------|--------|
| `sdd-design` agent | Single agent that produces design.md from requirements + research | Medium — merge logic from sdd-specify + sdd-plan-draft |
| `tf-test-writer` agent | Converts design.md test scenarios into .tftest.hcl files | Medium — new agent |
| Updated `tf-implement` | TDD-aware implementer that runs tests after each phase | Low — modify existing |
| design.md template | Standardized template for the design document | Low |

### Skills Mapping

| Current Skill | v2 Disposition |
|---------------|---------------|
| `tf-plan` | Replace with v2 orchestrator (4 phases) |
| `tf-implement` | Update to TDD-aware (write tests first, run after each phase) |
| `tf-spec-writing` | Merge into design agent guidance |
| `tf-task-patterns` | Remove (implementation checklist in design.md) |
| `tf-checklist-patterns` | Remove (no separate checklists) |
| `tf-consistency-rules` | Remove (single artifact) |
| `tf-domain-taxonomy` | Keep (used in Phase 1 clarification) |
| `tf-research-heuristics` | Keep (used in Phase 1 research) |
| `tf-security-baselines` | Merge into design agent security controls section |
| `tf-architecture-patterns` | Keep (used by design + implementation agents) |
| `tf-implementation-patterns` | Keep (used by implementation agent) |
| `terraform-test` | Elevate (central to Phase 3 TDD workflow) |
| `terraform-style-guide` | Keep (used during implementation) |
| `tf-judge-criteria` | Keep (used in Phase 4 validation) |
| `tf-report-template` | Simplify (validation results, not full readiness report) |

---

## Comparison: v1 vs. v2

| Dimension | v1 (8 phases) | v2 (4 phases) | Delta |
|-----------|---------------|---------------|-------|
| Planning phases | 8 | 2 (Understand + Design) | -75% |
| Total phases (plan + implement) | 8 + implementation | 4 (end to end) | -50% |
| Artifacts produced | 17 files, 2,221 lines | 1 design doc (~200 lines) + code + tests | -94% lines |
| Agent invocations (planning) | 12+ | 4-6 | -50% |
| Content duplication | ~35% | 0% | Eliminated |
| Remediation cycles | 1-3 iterations (up to 8.6 min) | 0 | Eliminated |
| Estimated plan time | ~26 min | ~8 min | -69% |
| Estimated total time (plan + build + validate) | ~26 min + implementation | ~36 min (all-in) | Faster end-to-end |
| Security integration | Separate Phase 5 review | Embedded in design + test assertions | Shift-left |
| Testing integration | Separate Phase 7 in implementation | TDD — tests before code | Shift-left |
| Consistency checking | Separate Phase 7 analysis | Unnecessary | Eliminated |
| Audit trail granularity | High (17 separate files) | Lower (1 design doc + test results) | Trade-off |

---

## When to Use v1 vs. v2

| Scenario | v1 | v2 |
|----------|----|----|
| Module with >20 resources (EKS, VPC+subnets+NAT) | Recommended | Not ideal |
| Module with <15 resources (S3, SQS, SNS, Lambda) | Overkill | Recommended |
| Compliance-heavy org requiring artifact audit trail | Required | Insufficient |
| Rapid iteration / prototyping | Too slow | Ideal |
| Multiple reviewers (security team reviews spec, platform team reviews plan) | Good separation | Single doc may bottleneck |
| Solo developer or small team | Overhead not justified | Right-sized |
| First module in a new domain (unfamiliar AWS service) | Research artifacts valuable for learning | Research still happens, just not persisted |

---

## Open Questions

1. **Design doc review gate**: Should Phase 2 require user approval before Phase 3 begins (like the current `/tf-plan` → user review → `/tf-implement` handoff)? Or should the workflow be fully autonomous?

2. **Research persistence**: The proposal eliminates research files. Should we keep a lightweight `research-notes.md` for institutional memory, even if it's not consumed by downstream agents?

3. **Checklist function**: The checklist phase found 15 unique gaps not caught by analysis. In v2, where does this validation happen? Options: (a) design agent self-validates, (b) a lightweight review step after design.md, (c) accept the gap.

4. **Existing skill ecosystem**: The current repo has 18+ skills. How much of the skill library should be rewritten vs. adapted? Is backward compatibility with v1 worth maintaining?

---

## Next Steps

1. Review and approve this proposal
2. Create the `design.md` template
3. Build the `sdd-design` agent (merge sdd-specify + sdd-plan-draft)
4. Build the `tf-test-writer` agent
5. Update `tf-implement` for TDD workflow
6. Write the v2 orchestrator skill
7. Run a side-by-side comparison on the same S3 module prompt
