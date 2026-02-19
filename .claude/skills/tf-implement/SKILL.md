---
name: tf-implement
description: SDD v2 Phases 3-4. TDD implementation and validation from an existing design.md. Writes tests first, builds module, validates, creates PR.
user-invocable: true
argument-hint: "[feature-name] - Implement from existing specs/{feature}/design.md"
---

# SDD v2 — Implement

Builds and validates a Terraform module from `specs/{FEATURE}/design.md` using TDD.

Post progress at key steps: `bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "<step>" "<status>" "<summary>"`. Valid status values: `started`, `in-progress`, `complete`, `failed`.
Checkpoint after each phase: `bash .foundations/scripts/bash/checkpoint-commit.sh --dir . --prefix feat "$DESCRIPTION"`.

## Prerequisites

1. Resolve `$FEATURE` from `$ARGUMENTS` or current git branch name.
2. Run `bash .foundations/scripts/bash/validate-env.sh --json`. Stop if `gate_passed=false`.
3. Verify `specs/{FEATURE}/design.md` exists via Glob. Stop if missing — tell user to run `/tf-plan-v2` first.
4. Find `$ISSUE_NUMBER` from `$ARGUMENTS` or `gh issue list --search "$FEATURE"`.

## Phase 3: Build + Test

5. Launch `tf-test-writer` agent with FEATURE path. Verify `versions.tf`, `variables.tf`, and `tests/*.tftest.hcl` exist via Glob.
6. Run `terraform init -backend=false`.
7. Run `terraform test` to establish red TDD baseline. Checkpoint commit.
8. Extract checklist items from design.md Section 6 via Grep.
9. For each checklist item, launch `tf-task-executor` agent with FEATURE path and item description. Use concurrent subagents for independent items.
10. After each item: run `terraform validate` and `terraform test`. Checkpoint commit.
11. After all items: run `terraform test`. If failures remain, re-launch `tf-test-writer` agent with the error output and any data sources reported by task executors as context.

## Phase 4: Validate

12. Run all in parallel: `terraform test`, `terraform validate`, `terraform fmt -check -recursive`, `trivy config .`, `terraform-docs markdown . > README.md`.
13. Fix failures iteratively (max 3 rounds). Run `terraform fmt -recursive` for format issues.
14. Write validation report to `specs/{FEATURE}/reports/` by reading the `tf-report-template` skill inline and applying its format (this is not a subagent dispatch — write the report directly).
15. Checkpoint commit, push branch, create PR linking to `$ISSUE_NUMBER`.

## Done

Report: test pass/fail, validation status, PR link.
