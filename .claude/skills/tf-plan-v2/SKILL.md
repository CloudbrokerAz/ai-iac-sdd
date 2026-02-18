---
name: tf-plan-v2
description: SDD v2 Phases 1-2. Clarify requirements, research, produce design.md, and await human approval before any code is written.
user-invocable: true
argument-hint: "[module-name] [provider] - Brief description of what the module should create"
---

# SDD v2 — Plan

Produces `specs/{FEATURE}/design.md` from requirements. Stops for human approval before any code is written.

Post progress at key steps: `bash .foundations/scripts/bash/post-issue-progress.sh $ISSUE_NUMBER "<step>" "<status>" "<summary>"`.

## Phase 1: Understand

1. Run `bash .foundations/scripts/bash/validate-env.sh --json`. Stop if `gate_passed=false`.
2. Parse `$ARGUMENTS` for module name, provider, and description. Ask via `AskUserQuestion` if incomplete.
3. Create GitHub issue via `gh issue create --template module-request.yml`. Capture `$ISSUE_NUMBER`.
4. Create feature branch via `bash .foundations/scripts/bash/create-new-feature.sh`. Capture `$FEATURE`.
5. Scan requirements against the `tf-domain-taxonomy` 8-category taxonomy. Always flag security-configurable features.
6. Ask up to 4 clarification questions via `AskUserQuestion`. Must include a security-defaults question.
7. Launch 3-4 concurrent `sdd-research` subagents for provider docs, AWS best practices, registry patterns, and edge cases. Wait for all to complete.

## Phase 2: Design

8. Launch `sdd-design` agent with FEATURE path, clarified requirements, and research findings summary. The agent reads the constitution and design template itself.
9. Verify `specs/{FEATURE}/design.md` exists via Glob. Re-launch once if missing.
10. Grep to confirm all 7 sections present (`## 1. Purpose` through `## 7. Open Questions`). Fix inline if any missing.
11. Present design summary to user via `AskUserQuestion`: input/output counts, resource count, security controls, test scenarios, checklist items. Options: approve, review file first, request changes.
12. If changes requested, apply and re-present. Repeat until approved.

## Done

Design approved at `specs/{FEATURE}/design.md`. Run `/tf-implement $FEATURE` to build.
