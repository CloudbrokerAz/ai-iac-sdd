# Constitution Refactoring Notes

**Date**: 2026-02-18
**File**: `.foundations/memory/constitution.md`
**Before**: v4.0.0 — 737 lines, 14 sections
**After**: v5.0.0 — 337 lines, 8 sections
**Reduction**: 54%

---

## Goal

Reduce the constitution to its core enterprise principles — what a financial-industry organization would mandate for Terraform module development. Remove operational procedures, agent behavior rules, and workflow mechanics that belong in other documents. Be concise without opening rules to interpretation.

---

## What Was Removed

| Removed Section | Lines | Reason | Where It Belongs |
|-----------------|-------|--------|------------------|
| Section 2: HCP Terraform Prerequisites | ~65 | Operational procedure for token validation, org/project/workspace detection | `tf-plan-v2` orchestrator skill |
| Section 5: Workspace and Environment Management | ~55 | Ephemeral workspace creation, credential setup, publishing workflow detail | `tf-plan-v2` and `tf-implement` skills |
| Section 8: AI Agent Behavior and Constraints | ~55 | Scope boundaries, error handling guidance, learning/adaptation rules for agents | `AGENTS.md` |
| Section 10: Implementation Checklist | ~30 | Step-by-step how-to for developers and platform teams | `README.md` |
| Section 12: Compound Learning | ~25 | Pattern extraction, pitfall recording, auto-commit rules | `tf-compound-patterns` skill |
| Section 13: Naming Conventions (agents/skills) | ~20 | Agent prefixes (`sdd-`, `tf-`, `compound-`), skill prefixes, file naming for agents | `AGENTS.md` |
| Section 14: References and Resources | ~20 | External links to HashiCorp docs, AWS guides, etc. | `README.md` |
| Verbose code examples | ~50 | Full VPC pattern, variable template with 3 examples, conditional creation tutorial | Kept only `versions.tf` template and one `try()` output example |
| Duplicated content | ~30 | `validate-env.sh` usage (appeared 4x), workflow phase descriptions (duplicated from orchestrator), prerequisite rules (restated from Section 2 in Section 8) | Single-sourced in their canonical owners |

---

## What Was Restructured

### Old Structure (14 sections)

1. Foundational Principles
2. HCP Terraform Prerequisites
3. Code Generation Standards
4. Security and Compliance
5. Workspace and Environment Management
6. Code Quality and Maintainability
7. Operational Excellence
8. AI Agent Behavior and Constraints
9. Governance and Evolution
10. Implementation Checklist
11. Workflow Governance
12. Compound Learning
13. Naming Conventions
14. References and Resources

### New Structure (8 sections)

1. **Core Principles** — Module-first, security-first, TDD, single design document
2. **Code Standards** — File organization, naming, variables, outputs, resource patterns, style
3. **Security and Compliance** — Secrets, AWS baselines (table), tagging
4. **Version and Dependency Management** — Provider constraints, state, releases
5. **Testing and Validation** — Coverage requirements (table), validation pipeline (table), test organization
6. **Change Management** — Git workflow, design approval, quality gates between phases (table)
7. **Operational Standards** — Cost, observability, HCP Terraform essentials
8. **Governance** — Maintenance, exception process, audit, documentation

---

## Key Design Decisions

### Tables over prose for scannable rules

AWS security baselines, validation pipeline, quality gates, and test coverage requirements were converted from paragraphs to tables. Easier to audit, harder to miss.

### Terraform naming stayed, agent naming removed

Terraform naming conventions (resources, variables, outputs, boolean toggles) are code standards — they belong in the constitution. Agent/skill naming (`sdd-` prefix, `compound-` prefix) is workflow system architecture — moved to AGENTS.md.

### HCP Terraform reduced to 4 lines

The full prerequisite validation flow (token detection, org selection, workspace patterns, credential file creation) is operational detail for the orchestrator. The constitution keeps only the principle: validate before operations, delete workspaces after testing, push before creating workspaces.

### Compound learning removed entirely

The compound learning subsystem (pattern extraction, pitfall recording, auto-commit) is a workflow feature, not a code quality standard. It governs what the tooling does after a build, not what correct module code looks like.

### Implementation checklist removed entirely

The step-by-step developer workflow ("clone repo, review constitution, run workflow, configure IDE...") and platform team checklist ("publish constitution, create templates, configure CI...") are onboarding guides, not principles. Moved to README.

---

## What Was Preserved

All MUST/MUST NOT language retained. No rules were softened or made ambiguous. Specifically:

- All security baselines (encryption, public access, IAM, secrets handling)
- File organization rules and 500-line limit
- Provider constraint rules (`>=` not `~>`, no `latest`)
- Git branch protection (no direct commits to main, PR review required)
- Quality gate conditions between phases
- Design approval gate requirement
- Exception process (5-step documented deviation)
- Audit and compliance requirements
- Pre-commit hook enforcement

---

## Ownership Model After Refactoring

| Content Domain | Canonical Owner |
|----------------|-----------------|
| Module code standards | `constitution.md` (this file) |
| Workflow mechanics | `tf-plan-v2/SKILL.md` |
| Implementation mechanics | `tf-implement/SKILL.md` |
| Agent behavior and context rules | `AGENTS.md` |
| Agent/skill naming conventions | `AGENTS.md` |
| Compound learning rules | `tf-compound-patterns/SKILL.md` |
| Project onboarding | `README.md` |
| External references | `README.md` |
