---
name: tf-implement
description: TDD implementation orchestrator for Terraform modules. Writes tests first from design.md, then implements module code phase-by-phase. Can run standalone from an existing design.md or as Phase 3 of tf-plan-v2.
user-invokable: true
argument-hint: "[feature-name] - Implement from existing specs/{feature}/design.md"
---

# TDD Implementation Orchestrator

## Context Management (CRITICAL)

1. **NEVER call TaskOutput** to read subagent results. All agents write artifacts to disk — reading them back into the orchestrator bloats context and triggers compaction.
2. **Verify completion via Glob or Grep** — do NOT read full file contents into the orchestrator.
3. **Task executors**: Run as **parallel foreground Task calls** (NOT `run_in_background`). Launch all independent task executors in a single message.
4. **Quality/report agents**: Verify output file exists via Glob, then Grep for severity keywords (e.g., `Critical`, `FAIL`) — do NOT read full report contents.
5. **Minimal $ARGUMENTS**: Only pass the FEATURE path + task scope. Never inject file contents.

### Expected Output Files

| Phase | Agent/Tool | Output | Verification |
|-------|-----------|--------|--------------|
| 2 | tf-test-writer | `tests/*.tftest.hcl` | Glob |
| 3 | tf-task-executor (xN) | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf` | Glob |
| 4 | (orchestrator) | `examples/basic/*`, `examples/complete/*` | Glob |

## Workflow

Execute phases sequentially. Post progress to GH issue before/after each phase. Checkpoint commit after each phase.

Use the following template for GitHub issue progress updates:

```
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "<step>" "<status>" "<summary>" "$DETAILS"
```

### Example

```
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Implementation" "started"
bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Implementation" "complete" "$DETAILS"
```

---

### Phase 1 — Prerequisites

1. Resolve feature directory:
   - If `$ARGUMENTS` contains a feature name, use it: `FEATURE="$ARGUMENTS"`
   - Otherwise, derive from current branch: `FEATURE=$(git rev-parse --abbrev-ref HEAD)`
2. Run environment validation:
   ```
   bash .foundations/scripts/bash/validate-env.sh --json
   ```
   STOP if any gate fails — report which gates failed and exit.
3. Get issue number:
   - If `$ARGUMENTS` includes an issue number, use it.
   - Otherwise, search GitHub: `gh issue list --search "FEATURE" --json number --jq '.[0].number'`
4. Verify `specs/{FEATURE}/design.md` exists:
   - Use Glob: `specs/{FEATURE}/design.md`
   - If missing: **STOP** — display: `"design.md not found for {FEATURE}. Run /tf-plan-v2 first or create design.md manually."`
5. Post progress:
   ```
   bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Implementation" "started"
   ```

---

### Phase 2 — Write Tests First (TDD)

This is the core TDD step: write all test files BEFORE any module code exists.

1. Launch `tf-test-writer` agent with FEATURE path:
   ```
   Task agent: tf-test-writer
   Arguments: FEATURE path — specs/{FEATURE}/design.md
   ```
   The agent reads design.md Section 5 (Test Scenarios) and writes `.tftest.hcl` files.

2. Verify test files exist via Glob — do NOT read them:
   - `tests/basic.tftest.hcl`
   - `tests/complete.tftest.hcl`
   - `tests/validation.tftest.hcl`

   If any expected test file is missing, report which files are missing and STOP.

3. Run `terraform test` — expect ALL failures (no module code exists yet):
   ```
   terraform test
   ```
   This establishes the "red" TDD baseline. Capture the pass/fail count but do NOT read full output into context. Use Grep on output for the summary line.

4. Checkpoint commit:
   ```
   git add tests/
   git commit -m "feat({FEATURE}): add test files from design"
   ```

5. Post progress:
   ```
   bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Tests" "complete" "Tests written — 0/N passing (TDD baseline)"
   ```

---

### Phase 3 — Implement Module Code

1. Extract checklist items from design.md Section 6 (Implementation Checklist):
   ```
   Grep for lines matching "^- \[ \]" or "^[0-9]+\." in specs/{FEATURE}/design.md
   ```
   Do NOT read the full file — use Grep to extract only the checklist lines.

2. For each checklist item:
   a. Launch `tf-task-executor` agent:
      ```
      Task agent: tf-task-executor
      Arguments: FEATURE path + checklist item description (one-line summary)
      ```
   b. Verify new/modified `.tf` files exist via Glob:
      - `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`
   c. Run `terraform validate` — if errors, fix inline and re-validate.
   d. Run `terraform test` — report pass/fail count (failures expected early in TDD).
   e. Checkpoint commit:
      ```
      git add *.tf modules/ tests/
      git commit -m "feat({FEATURE}): implement {checklist-item-summary}"
      ```

3. **Parallel dispatch**: When multiple checklist items are INDEPENDENT (no shared resource dependencies), launch their `tf-task-executor` agents as **parallel foreground Task calls in a single message**. Do NOT use `run_in_background`.

4. After ALL checklist items complete, run final test suite:
   ```
   terraform test
   ```
   - If all tests pass: proceed to Phase 4.
   - If failures remain: fix module code, re-run `terraform test`. Maximum 3 fix iterations.
   - If still failing after 3 iterations: report remaining failures and STOP.

5. Post progress:
   ```
   bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Implementation" "complete" "Implementation complete — N/N tests passing"
   ```

---

### Phase 4 — Write Examples

1. Create `examples/basic/main.tf`:
   - Minimal usage with only required variables
   - Provider configuration block
   - Module source using relative path: `source = "../../"`
   - Reference design.md Interface Contract (Section 1) for required variables — use Grep, do NOT read the full file.

2. Create `examples/complete/main.tf`:
   - All features enabled, all optional variables set
   - Provider configuration block
   - Module source using relative path: `source = "../../"`
   - Demonstrates every optional variable and feature toggle from design.md Interface Contract.

3. Both examples need `versions.tf` with provider requirements:
   ```hcl
   terraform {
     required_version = ">= 1.0"
     required_providers {
       aws = {
         source  = "hashicorp/aws"
         version = ">= 5.0"
       }
     }
   }
   ```

4. Validate each example:
   ```
   cd examples/basic && terraform init && terraform validate
   cd examples/complete && terraform init && terraform validate
   ```
   Fix any validation errors before proceeding.

5. Checkpoint commit:
   ```
   git add examples/
   git commit -m "feat({FEATURE}): add examples"
   ```

---

### Phase 5 — Final Check + Handoff

1. Run formatting fix (apply, not just check):
   ```
   terraform fmt -recursive
   ```

2. Run validation:
   ```
   terraform validate
   ```

3. Run full test suite — final confirmation:
   ```
   terraform test
   ```
   All tests MUST pass. If any fail, fix and re-run (max 2 iterations).

4. **Standalone mode** (invoked directly via `/tf-implement`, NOT by `tf-plan-v2`):
   - Run remaining validation checks that Phase 4 of tf-plan-v2 would handle:
     ```
     trivy config .
     terraform-docs markdown . > README.md
     ```
   - Run `code-quality-judge` agent with FEATURE path. Verify report via Glob — do NOT read it. Grep for `Critical` — flag if found.
   - Create PR:
     ```
     git push -u origin HEAD
     gh pr create --title "feat({FEATURE}): Terraform module implementation" --body "..."
     ```
   - Post completion to issue with PR link.

5. **Delegated mode** (invoked by `tf-plan-v2` as Phase 3):
   - Do NOT run trivy, terraform-docs, quality review, or create PR.
   - Return control to tf-plan-v2 — Phase 4 validation is handled by the parent orchestrator.

6. Checkpoint commit:
   ```
   git add -A
   git commit -m "feat({FEATURE}): implementation complete"
   ```

7. Post progress:
   ```
   bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "Implementation" "complete"
   ```

---

## Completion Message

```
> Implementation complete. {N}/{N} tests passing.
> Module files: main.tf, variables.tf, outputs.tf, versions.tf
> Tests: tests/basic.tftest.hcl, tests/complete.tftest.hcl, tests/validation.tftest.hcl
> Examples: examples/basic/, examples/complete/
```
