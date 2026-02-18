---
name: tf-plan-v2
description: 4-phase SDD v2 workflow orchestrator. Entry point for Terraform module development. Phases: Understand (clarify + research), Design (single design.md), Build+Test (TDD), Validate (security + quality). Invoked via /tf-plan-v2.
user-invokable: true
argument-hint: "[module-name] [provider] - Brief description of what the module should create"
---

# SDD v2 Planning Orchestrator

Execute phases sequentially. Complete each before proceeding. All artifacts land in `specs/{FEATURE}/`.

Post progress to the GitHub issue before and after each subagent run or major step:

```
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "<step>" "<status>" "<summary>" "$DETAILS"
```

---

## Context Management (CRITICAL)

Follow the 5 rules in AGENTS.md `## Context Management`. Key: never call TaskOutput, verify via Glob, minimal `$ARGUMENTS`.

---

## Phase 1: Understand

**Goal**: Validate environment, intake requirements, clarify ambiguities, research provider/AWS/registry patterns.

### Step 1.1 — Environment Validation

```bash
bash .foundations/scripts/bash/validate-env.sh --json
```

Parse the JSON output. If `gate_passed=false`, STOP. Show the user what is missing and how to fix it.

### Step 1.2 — TFE Token Connectivity

Call MCP `list_terraform_orgs` to verify TFE_TOKEN connectivity. If this fails, STOP and instruct the user to set `TFE_TOKEN`.

### Step 1.3 — Requirements Intake

Read input from `$ARGUMENTS`. Extract module name, provider, and description. If incomplete, use `AskUserQuestion` to gather: what infrastructure, what provider(s), key features, and key inputs/outputs.

### Step 1.4 — GitHub Issue Creation

```bash
gh issue create \
  --template module-request.yml \
  --title "feat: $MODULE_NAME" \
  --body "$REQUIREMENTS_SUMMARY"
```

Capture `$ISSUE_NUMBER` from the output. If this fails, ask the user whether to continue without issue tracking. If yes, skip all subsequent `post-issue-progress.sh` calls.

### Step 1.5 — Feature Branch Creation

Call `create-new-feature.sh` with the module name and issue number (if available). Capture `$BRANCH_NAME` and `$FEATURE_DIR`.

```bash
bash .foundations/scripts/bash/create-new-feature.sh --issue $ISSUE_NUMBER --short-name "$MODULE_SHORT_NAME" "$MODULE_NAME"
```

The `{FEATURE}` path used throughout this workflow corresponds to the `$BRANCH_NAME` value (e.g., `specs/042-s3-static-website/`).

### Step 1.6 — Taxonomy Scan

Load the `tf-domain-taxonomy` skill and scan requirements against all 8 categories (Functional Scope, Domain/Data Model, Operational Workflows, Non-Functional Attrs, Integration/Dependencies, Edge Cases, Constraints/Tradeoffs, Terminology). Rate each as Clear / Partial / Missing.

**ALWAYS check "Constraints/Tradeoffs" for security defaults.** If any feature is security-configurable, note it for clarification.

### Step 1.7 — Gap Ranking

Rank Partial/Missing gaps by Impact x Uncertainty. High/High = must ask user. High/Low = resolve via research. Low = defer or use defaults. Select up to 4 questions from highest-ranked gaps.

### Step 1.8 — User Clarification

Ask up to 4 questions via `AskUserQuestion`:
1. MUST include a security-defaults question if any security-configurable feature exists.
2. Be specific and actionable, not open-ended.
3. Provide context and options where possible.

### Step 1.9 — Parallel MCP Research

Launch 3-4 `sdd-research` agents in parallel. Each answers ONE specific research question. **Launch ALL agents in a single message with multiple Task tool calls.**

| Agent | MCP Tools | Research Question |
|-------|-----------|-------------------|
| Provider resources | `search_providers` + `get_provider_details` | How do the main resources work? Arguments, attributes, gotchas? |
| AWS best practices | `search_documentation` + `read_documentation` | What does AWS recommend? Security controls? |
| Registry patterns | `search_modules` + `get_module_details` | How do popular modules structure their interface? |
| Edge cases | `search_documentation` (troubleshooting) | What breaks? Common mistakes? Behavioral quirks? |

Pass via `$ARGUMENTS`: FEATURE path, specific research question, provider name and resource types.

**Wait for ALL agents to complete before proceeding.** Research findings stay in the orchestrator's context. Pass a summary of key findings to the design agent via `$ARGUMENTS`.

**Critical**: Research agents frequently discover architecture-shaping facts not available from general knowledge. **Never skip research.**

### Step 1.10 — Post Progress

Post progress: Phase 1 complete. Include taxonomy scan results, clarification count, and research agent count.

**Quality gate**: All requirements categories must be Clear or resolved. No unresolved `[NEEDS CLARIFICATION]` markers.

---

## Phase 2: Design

**Goal**: Produce `specs/{FEATURE}/design.md` containing interface contract, resource inventory, security controls, test scenarios, and implementation checklist.

### Step 2.1 — Launch Design Agent

Post progress: Phase 2 started.

Launch the `sdd-design` agent with `$ARGUMENTS` containing:
- The FEATURE path (e.g., `specs/042-s3-static-website/`)
- Clarified requirements summary (from Phase 1 user answers)
- Research findings summary (key facts only, not raw MCP output)

The agent reads the constitution and design template on its own. Do NOT inject those contents.

### Step 2.2 — Verify Design Artifact

Verify `specs/{FEATURE}/design.md` exists via Glob. If missing, re-launch once. If it fails again, STOP and flag to user.

### Step 2.3 — Quick Validation

Lightweight validation using Grep -- do NOT read the full file into context.

**Section presence checks** (Grep for each heading):
- `## 1. Purpose`
- `## 2. Interface Contract`
- `## 3. Resources & Architecture`
- `## 4. Security Controls`
- `## 5. Test Scenarios`
- `## 6. Implementation Checklist`
- `## 7. Open Questions`

All 7 must be present. If any are missing, fix inline.

**Content quality checks** (Grep for patterns):
- `| ` in Section 2 -- Interface Contract table exists
- `CIS\|Well-Architected\|NIST` in Section 4 -- security references exist
- `- [ ]` in Section 6 -- implementation checklist items exist
- `Assertions:` or `assert` in Section 5 -- test assertions defined

If any check fails, edit the file directly. Do NOT re-run the design agent for validation failures.

### Step 2.4 — Post Progress

Post progress: Phase 2 complete. Include section count and key contents.

### Step 2.5 — Await User Approval

**Mandatory gate.** Present the design summary to the user via `AskUserQuestion`: interface contract stats (inputs/outputs), resource count, security control count, test scenario count, checklist item count. Offer: approve and proceed, review first, or request changes.

If changes requested, apply them, re-validate (Step 2.3), and ask again.

**Quality gate**: Explicit user approval required. No unresolved CRITICAL findings.

---

## Phase 3: Build + Test

**Goal**: Implement the module using TDD -- tests first from design scenarios, then module code phase-by-phase.

### Step 3.1 — Launch Implementation

Post progress: Phase 3 started.

Invoke the `tf-implement` skill with `$ARGUMENTS` containing the FEATURE path and `$ISSUE_NUMBER`.

`tf-implement` handles the full TDD cycle internally: test writing, task execution, validate/test after each phase.

### Step 3.2 — Verify Completion

Verify all expected artifacts exist via Glob:
- Module files: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`
- Test files: `tests/*.tftest.hcl`
- Examples: `examples/basic/main.tf`, `examples/complete/main.tf`

If any are missing, flag to user.

### Step 3.3 — Post Progress

Post progress: Phase 3 complete. Include module file list, test file count, example directories.

**Quality gate**: `terraform validate` must pass. All Section 6 checklist items complete.

---

## Phase 4: Validate

**Goal**: Run all validation checks, fix failures, generate report, create PR.

### Step 4.0 — Initialize

```bash
terraform init -backend=false
```

Fix any init failures before proceeding.

### Step 4.1 — Parallel Validation

Post progress: Phase 4 started.

Launch ALL checks in parallel:

| Check | Command | Pass Criteria |
|-------|---------|---------------|
| Tests | `terraform test` | All tests pass |
| Validate | `terraform validate` | No errors |
| Format | `terraform fmt -check -recursive` | No unformatted files |
| Security | `trivy config .` | No CRITICAL or HIGH |
| Docs | `terraform-docs markdown . > README.md` | README generated |

### Step 4.2 — Fix Failures

Fix failures iteratively. After fixing, re-run only failed checks. **Max 3 iterations.** If issues persist, flag to user with details.

| Failure | Fix Action |
|---------|------------|
| `terraform fmt` | Run `terraform fmt -recursive` |
| `terraform validate` | Fix reported code errors |
| `terraform test` | Fix module code to pass assertions |
| `trivy` CRITICAL/HIGH | Fix security misconfigurations |
| `terraform-docs` | Ensure all variables/outputs have descriptions |

### Step 4.3 — Generate Validation Report

Use the `tf-report-template` skill format. Capture: test results per file, validate result, fmt result, trivy results by severity, security checklist from design.md Section 4, overall status.

### Step 4.4 — Write Report

```
specs/{FEATURE}/reports/validation_$(date +%Y%m%d-%H%M%S).md
```

### Step 4.5 — Post Final Progress

Post progress: Phase 4 complete. Include pass/fail for each check.

### Step 4.6 — Commit and Create PR

```bash
bash .foundations/scripts/bash/checkpoint-commit.sh --dir . --prefix feat "implementation-complete"
```

```bash
gh pr create \
  --title "feat: $MODULE_NAME" \
  --body "## Summary

Implements $MODULE_NAME module via SDD v2 workflow.

Closes #$ISSUE_NUMBER

## Artifacts
- Design: \`specs/{FEATURE}/design.md\`
- Tests: \`tests/*.tftest.hcl\`
- Examples: \`examples/basic/\`, \`examples/complete/\`
- Validation report: \`specs/{FEATURE}/reports/validation_*.md\`

## Validation
- [x] terraform test -- all pass
- [x] terraform validate -- clean
- [x] terraform fmt -check -- formatted
- [x] trivy config -- no CRITICAL/HIGH
- [x] terraform-docs -- README generated"
```

---

## Expected Output Files

| Phase | Agent/Tool | Output | Verification |
|-------|-----------|--------|--------------|
| 1 | sdd-research (x3-4) | Research findings (in orchestrator context) | Agent completion |
| 2 | sdd-design | `specs/{FEATURE}/design.md` | Glob + Grep for 7 sections |
| 3 | tf-test-writer | `tests/*.tftest.hcl` | Glob for test files |
| 3 | tf-task-executor (xN) | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf` | Glob for .tf files |
| 3 | tf-task-executor (xN) | `examples/basic/`, `examples/complete/` | Glob for example dirs |
| 4 | (tools) | `specs/{FEATURE}/reports/validation_*.md` | Glob for report |
| 4 | (tools) | `README.md` | Glob for README |

---

## Completion Message

```
> SDD v2 workflow complete. PR created and linked to issue #{ISSUE_NUMBER}.
> Design: specs/{FEATURE}/design.md
> Report: specs/{FEATURE}/reports/validation_*.md
> Run /tf-implement to re-run implementation from an existing design.
```
