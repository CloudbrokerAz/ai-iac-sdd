---
name: tf-plan-v2
description: 4-phase SDD v2 workflow orchestrator. Entry point for Terraform module development. Phases: Understand (clarify + research), Design (single design.md), Build+Test (TDD), Validate (security + quality). Invoked via /tf-plan-v2.
user-invokable: true
argument-hint: "[module-name] [provider] - Brief description of what the module should create"
---

# SDD v2 Planning Orchestrator

Execute phases sequentially. Complete each before proceeding. All artifacts land in `specs/{FEATURE}/`.

Before and after each subagent run or major step, post progress to the GitHub issue:

```
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "<step>" "<status>" "<summary>" "$DETAILS"
```

### Examples

```
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Environment Validation" "started"
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Environment Validation" "complete" "All gates passed"
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Research" "started"
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Research" "complete" "4 research agents completed" "- Provider resources documented\n- AWS best practices captured\n- Registry patterns analyzed\n- Edge cases identified"
```

---

## Context Management (CRITICAL)

**Context management**: Follow the 5 rules in AGENTS.md `## Context Management`. Key: never call TaskOutput, verify via Glob, minimal `$ARGUMENTS`.

---

## Phase 1: Understand

**Goal**: Validate environment, intake requirements, clarify ambiguities, research provider/AWS/registry patterns.

### Step 1.1 — Environment Validation

Run the environment validation script:

```bash
bash .foundations/scripts/bash/validate-env.sh --json
```

Parse the JSON output. If `gate_passed=false`, **STOP immediately**. Show the user exactly what is missing and how to fix it. Do NOT proceed until all gates pass.

### Step 1.2 — TFE Token Connectivity

Call MCP `list_terraform_orgs` to verify the TFE_TOKEN is valid and has connectivity to HCP Terraform. If this fails, STOP and instruct the user to set `TFE_TOKEN`.

### Step 1.3 — Requirements Intake

Read input from `$ARGUMENTS`. Extract:
- **Module name** (e.g., `terraform-aws-s3-static-website`)
- **Provider** (e.g., `aws`)
- **Description** of what the module should create

If `$ARGUMENTS` is incomplete or ambiguous, use `AskUserQuestion` to gather the minimum:
- What infrastructure should the module create?
- What provider(s)?
- What are the key features?
- What should the interface look like (key inputs/outputs)?

### Step 1.4 — GitHub Issue Creation

Create a GitHub issue for tracking using the module-request template:

```bash
gh issue create \
  --template module-request.yml \
  --title "feat: $MODULE_NAME" \
  --body "$REQUIREMENTS_SUMMARY"
```

Capture `$ISSUE_NUMBER` from the output. All subsequent `post-issue-progress.sh` calls use this number.

Post progress: Phase 1 started.

### Step 1.5 — Taxonomy Scan

Load the `tf-domain-taxonomy` skill and scan the captured requirements against all 8 categories. Rate each:

| Category | Status |
|----------|--------|
| Functional Scope | Clear / Partial / Missing |
| Domain/Data Model | Clear / Partial / Missing |
| Operational Workflows | Clear / Partial / Missing |
| Non-Functional Attrs | Clear / Partial / Missing |
| Integration/Dependencies | Clear / Partial / Missing |
| Edge Cases | Clear / Partial / Missing |
| Constraints/Tradeoffs | Clear / Partial / Missing |
| Terminology | Clear / Partial / Missing |

**ALWAYS check "Constraints/Tradeoffs" for security defaults.** This is mandatory. If any feature is security-configurable (encryption, public access, logging, IAM), note it for the clarification step.

### Step 1.6 — Gap Ranking

Rank all Partial/Missing gaps by **Impact x Uncertainty**:
- **High Impact, High Uncertainty** = must ask the user
- **High Impact, Low Uncertainty** = can resolve via research
- **Low Impact** = defer or use sensible defaults

Select up to 4 questions from the highest-ranked gaps.

### Step 1.7 — User Clarification

Ask up to 4 questions via `AskUserQuestion`. Rules:

1. **MUST include a security-defaults question** if ANY security-configurable feature exists. Format: "Should [feature X] be configurable or hardcoded to secure default?"
2. Questions should be specific and actionable, not open-ended.
3. Provide context for why you are asking (e.g., "The module creates S3 buckets. S3 supports both AES-256 and KMS encryption...")
4. Offer options where possible (e.g., "Option A: Hardcode encryption on. Option B: Allow toggle with secure default.")

### Step 1.8 — Parallel MCP Research

Launch 3-4 `sdd-research` agents in parallel. Each agent answers ONE specific research question. **Launch ALL agents in a single message with multiple Task tool calls.**

| Agent | MCP Tools | Research Question |
|-------|-----------|-------------------|
| Agent 1: Provider resources | `search_providers` + `get_provider_details` | How do the main resources work? Arguments, attributes, gotchas? |
| Agent 2: AWS best practices | `search_documentation` + `read_documentation` | What does AWS recommend for this service? Security controls? |
| Agent 3: Registry patterns | `search_modules` + `get_module_details` | How do popular public modules structure their interface? |
| Agent 4: Edge cases | `search_documentation` (troubleshooting topic) | What breaks? Common mistakes? Behavioral quirks? |

For each agent, pass via `$ARGUMENTS`:
- The FEATURE path (`specs/{FEATURE}/`)
- The specific research question (one sentence)
- The provider name and resource types to research

**Wait for ALL agents to complete before proceeding.** Research findings remain in-memory with the agents -- do NOT write research files to disk. The findings are passed directly to the design agent in Phase 2.

**Critical**: Research agents frequently discover architecture-shaping facts not available from general knowledge (e.g., CloudFront OAC cannot use S3 website endpoints, `filter {}` is required in modern lifecycle configs). **Never skip research.**

### Step 1.9 — Post Progress

Post progress: Phase 1 complete -- requirements clarified, research complete. Include taxonomy scan results, clarification count, and research agent count in details.

**Quality gate**: All requirements categories must be Clear or have been resolved through clarification/research. No unresolved `[NEEDS CLARIFICATION]` markers may remain.

---

## Phase 2: Design

**Goal**: Produce a single `specs/{FEATURE}/design.md` containing interface contract, resource inventory, security controls, test scenarios, and implementation checklist.

### Step 2.1 — Launch Design Agent

Post progress: Phase 2 started.

Launch the `sdd-design` agent with `$ARGUMENTS` containing:
- The FEATURE path (e.g., `specs/feat/s3-static-website/`)
- A summary of clarified requirements (from Phase 1 user answers -- brief text, NOT file contents)
- A summary of research findings (key discoveries from Phase 1 research agents -- brief text, NOT full research output)

The agent reads the constitution (`.foundations/memory/constitution.md`) and the design template (`.foundations/templates/design-template.md`) on its own. Do NOT inject those contents.

### Step 2.2 — Verify Design Artifact

After the agent completes, verify `specs/{FEATURE}/design.md` exists via Glob. If the file does not exist, the agent failed. Re-launch once. If it fails again, STOP and flag to user.

### Step 2.3 — Quick Validation

The orchestrator performs lightweight validation using Grep -- do NOT read the full file into context.

**Section presence checks** (Grep for each heading):
- `## 1. Purpose`
- `## 2. Interface Contract`
- `## 3. Resources & Architecture`
- `## 4. Security Controls`
- `## 5. Test Scenarios`
- `## 6. Implementation Checklist`
- `## 7. Open Questions`

All 7 sections must be present. If any are missing, fix inline (orchestrator edits the file directly).

**Content quality checks** (Grep for patterns):
- `| ` in Section 2 area -- Interface Contract table exists
- `CIS\|Well-Architected\|NIST` in Section 4 area -- security references exist
- `- [ ]` in Section 6 area -- implementation checklist items exist
- `Assertions:` or `assert` in Section 5 area -- test assertions defined

If any content check fails, the orchestrator edits the file directly to fix. Do NOT re-run the design agent for validation failures.

### Step 2.4 — Post Progress

Post progress: Phase 2 complete -- design document ready for review. Include section count and key contents in details.

### Step 2.5 — Await User Approval

**This is a mandatory gate.** Ask the user via `AskUserQuestion`:

> Design document is ready for review at `specs/{FEATURE}/design.md`.
>
> The design includes:
> - Interface contract with N inputs and N outputs
> - N resources in the architecture
> - N security controls with CIS/WA references
> - N test scenarios with assertions
> - N implementation checklist items
>
> Would you like to proceed to implementation?
>
> Options:
> 1. **Approve and proceed** -- Begin Phase 3 (Build + Test)
> 2. **I need to review first** -- I will review the design and respond when ready
> 3. **Request changes** -- Describe what should change

Handle responses:
- **"Approve and proceed"** -> Proceed to Phase 3
- **"I need to review first"** -> Wait for user to respond (do NOT proceed)
- **"Request changes"** -> Apply the requested changes to `design.md`, re-run Step 2.3 validation, then ask again (Step 2.5)

**Quality gate**: Design -> Build+Test requires explicit user approval. No unresolved CRITICAL findings from security review may exist in the design document.

---

## Phase 3: Build + Test

**Goal**: Implement the module using TDD -- write tests first from design scenarios, then implement module code phase-by-phase.

### Step 3.1 — Launch Implementation

Post progress: Phase 3 started.

Invoke the `tf-implement` skill, which handles the full TDD cycle internally:
- Reads `specs/{FEATURE}/design.md` for all implementation details
- Dispatches `tf-test-writer` agent to convert Section 5 scenarios into `.tftest.hcl` files
- Dispatches `tf-task-executor` agent(s) for each Section 6 checklist item
- Runs `terraform validate` and `terraform test` after each implementation phase
- Tracks TDD progress (expects test failures early, drives toward all-green)

Pass to `tf-implement` via `$ARGUMENTS`:
- The FEATURE path (e.g., `specs/feat/s3-static-website/`)
- The issue number (`$ISSUE_NUMBER`)

### Step 3.2 — Verify Completion

After `tf-implement` completes, verify all expected artifacts exist via Glob:
- Module files: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`
- Test files: `tests/*.tftest.hcl`
- Examples: `examples/basic/main.tf`, `examples/complete/main.tf`

If any expected files are missing, flag to user with specifics on what is missing.

### Step 3.3 — Post Progress

Post progress: Phase 3 complete -- implementation complete. Include module file list, test file count, and example directories in details.

**Quality gate**: `terraform validate` must pass. All Section 6 checklist items must be marked complete.

---

## Phase 4: Validate

**Goal**: Run all validation checks in parallel, fix failures, generate report, create PR.

### Step 4.1 — Parallel Validation

Post progress: Phase 4 started.

Launch ALL validation checks in parallel (multiple Bash calls or a single script):

| Check | Command | Pass Criteria |
|-------|---------|---------------|
| Tests | `terraform test` | All tests pass |
| Validate | `terraform validate` | Clean (no errors) |
| Format | `terraform fmt -check -recursive` | No unformatted files |
| Security | `trivy config .` | No CRITICAL or HIGH findings |
| Docs | `terraform-docs markdown . > README.md` | README generated |

### Step 4.2 — Fix Failures

If any check fails, fix the issue:

| Failure | Fix Action |
|---------|------------|
| `terraform fmt` | Run `terraform fmt -recursive` to auto-fix |
| `terraform validate` | Fix the code errors reported |
| `terraform test` failures | Fix module code to pass assertions |
| `trivy` CRITICAL/HIGH | Fix the security misconfigurations |
| `terraform-docs` | Ensure all variables/outputs have descriptions |

After fixing, **re-run only the failed checks**.

**Max 3 fix iterations.** If issues persist after 3 iterations, flag remaining issues to the user via `AskUserQuestion` with details on what failed and what was attempted.

### Step 4.3 — Generate Validation Report

Use the `tf-report-template` skill format to generate the validation report. The report captures:

- `terraform test` results per test file (pass/fail)
- `terraform validate` result (clean/errors)
- `terraform fmt -check` result (formatted/needs formatting)
- `trivy config` results by severity (CRITICAL/HIGH/MEDIUM/LOW counts)
- Security checklist from design.md Section 4 (pass/fail per control)
- Overall status (PASS/FAIL)

### Step 4.4 — Write Report

Write the report to:

```
specs/{FEATURE}/reports/validation_$(date +%Y%m%d-%H%M%S).md
```

### Step 4.5 — Post Final Progress

Post progress: Phase 4 complete -- validation results. Include pass/fail status for each check (terraform test, validate, fmt, trivy, terraform-docs) in details. If issues remain, include specifics.

### Step 4.6 — Commit and Create PR

Commit all artifacts using the checkpoint commit script:

```bash
bash .foundations/scripts/bash/checkpoint-commit.sh --dir . --prefix feat "implementation-complete"
```

Create a pull request linking to the tracking issue:

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
| 1 | sdd-research (x3-4) | Research findings (in-memory) | Agent completion |
| 2 | sdd-design | `specs/{FEATURE}/design.md` | Glob + Grep for 7 sections |
| 3 | tf-test-writer | `tests/*.tftest.hcl` | Glob for test files |
| 3 | tf-task-executor (xN) | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf` | Glob for .tf files |
| 3 | tf-task-executor (xN) | `examples/basic/`, `examples/complete/` | Glob for example dirs |
| 4 | (tools) | `specs/{FEATURE}/reports/validation_*.md` | Glob for report |
| 4 | (tools) | `README.md` | Glob for README |

---

## Completion Message

Display when the full workflow finishes:

```
> SDD v2 workflow complete. PR created and linked to issue #{ISSUE_NUMBER}.
> Design: specs/{FEATURE}/design.md
> Report: specs/{FEATURE}/reports/validation_*.md
> Run /tf-implement to re-run implementation from an existing design.
```
