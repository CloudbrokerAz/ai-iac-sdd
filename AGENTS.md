# AI-Assisted Terraform Module Development (SDD v2)

AI-assisted development of enterprise-ready Terraform modules via spec-driven development. One design document replaces five artifacts. Tests before code. Security embedded in every phase.

## Core Principles

1. **Security-First**: All decisions prioritize security. No workarounds for security requirements. Constitution MUST rules are non-negotiable.
2. **Module-First**: Author well-structured modules using raw resources with secure defaults. Follow standard module structure (`examples/`, `tests/`, `modules/`).
3. **Single-Artifact Design**: Requirements flow into one `design.md` that replaces spec, plan, contracts, data model, and tasks. One source of truth eliminates cross-artifact contradictions.
4. **MCP-First**: Use MCP tools for AWS documentation and provider docs before general knowledge. Research resource behavior before writing code.
5. **TDD-First**: Tests before code. Write `.tftest.hcl` files from design scenarios, then implement the module to pass them. All tests green = implementation complete.
6. **Parallel Where Safe**: Independent tasks run concurrently. MCP-dependent tasks run sequentially.
7. **Quality Gates**: CRITICAL findings block progression. Reviews use evidence-based findings with citations.

## Workflow Overview

```
 Phase 1              Phase 2           Phase 3              Phase 4
 ────────────        ────────────      ────────────         ────────────
 UNDERSTAND      →   DESIGN       →   BUILD + TEST    →   VALIDATE
 Clarify + Research   Single doc        TDD-style           Security + Quality

 ~8 min               ~8 min            ~15 min              ~5 min
 0 artifacts          1 artifact        Module code          Test results
                      (design.md)       + test files         Security scan
```

**Phase 1: Understand** — Validate environment, intake requirements, clarify ambiguities against the 8-category taxonomy (always ask about security defaults), launch parallel MCP research agents.

**Phase 2: Design** — Produce a single `specs/{FEATURE}/design.md` containing interface contract, resource inventory, security controls, test scenarios, and implementation checklist.

**Phase 3: Build + Test** — Write `.tftest.hcl` files from design scenarios first, then implement module code phase-by-phase. Run `terraform validate` and `terraform test` after each phase.

**Phase 4: Validate** — Run all checks in parallel: `terraform test`, `terraform validate`, `terraform fmt -check`, `trivy config .`, `terraform-docs`. Fix any failures and confirm clean.

## MCP Tools Priority

1. `search_documentation` -> `read_documentation` (AWS best practices and resource behavior)
2. `search_providers` -> `get_provider_details` (provider resource docs -- arguments, attributes, examples)
3. `search_modules` / `search_private_modules` (study public/private registry for design patterns and conventions)
4. `get_regional_availability` (validate resource/feature availability in target regions)

## Directory Map

| Path | Purpose |
|------|---------|
| `main.tf`, `variables.tf`, `outputs.tf` | Root module -- primary resource definitions |
| `specs/{FEATURE}/design.md` | THE single design artifact (replaces spec, plan, contracts, data model, tasks) |
| `tests/` | Terraform test files (`.tftest.hcl`) -- written before module code |
| `examples/basic/` | Minimal usage example with provider config |
| `examples/complete/` | Full-featured usage example |
| `modules/` | Submodules (optional, for complex modules) |
| `.foundations/` | Constitution, templates, and scripts |
| `.foundations/memory/constitution.md` | Non-negotiable rules for all Terraform code generation |
| `.foundations/templates/design-template.md` | Standardized template for design.md |
| `.foundations/scripts/bash/` | Utility scripts (validate-env, checkpoint-commit, post-issue-progress, create-new-feature, common) |
| `.claude/skills/` | Skill definitions (orchestrators and knowledge) |
| `.claude/agents/` | Agent definitions (subagent specifications) |

## Agent Architecture

Agents are subagents dispatched by orchestrator skills. Each agent has a single responsibility, reads its own inputs from disk, and writes its own outputs to disk.

| Agent | Model | Purpose | Input | Output |
|-------|-------|---------|-------|--------|
| `sdd-design` | opus | Produce design.md from clarified requirements and research findings | Requirements + research + constitution + template | `specs/{FEATURE}/design.md` |
| `sdd-research` | opus | Answer one specific research question using MCP tools | Feature path + research question | Research findings (passed to design agent, not persisted) |
| `tf-test-writer` | sonnet | Convert design.md test scenarios into `.tftest.hcl` files | `specs/{FEATURE}/design.md` Section 5 | `tests/*.tftest.hcl` |
| `tf-task-executor` | opus | Implement one checklist item from design.md | `specs/{FEATURE}/design.md` + checklist item + existing code | Modified `.tf` files |
| `tf-deployer` | opus | Run tests and optionally deploy to sandbox | Module code + test files | Test results + deploy status |
| `code-quality-judge` | opus | Score module across 6 quality dimensions | Completed module code + design.md | Quality report with scores and findings |
| `aws-security-advisor` | opus | Assess security posture against CIS/Well-Architected controls | Module code + security controls from design.md | Security assessment with pass/fail per control |

## Skill Architecture

Skills provide domain knowledge and orchestration logic. They are loaded into agent context as needed.

### Orchestrators

| Skill | Purpose |
|-------|---------|
| `tf-plan-v2` | 4-phase workflow entry point: Understand -> Design -> Build+Test -> Validate |
| `tf-implement` | TDD-aware implementation: write tests first, run after each phase |

### Domain Knowledge (kept from v1)

| Skill | Purpose |
|-------|---------|
| `tf-domain-taxonomy` | 8-category taxonomy for scanning requirements and identifying gaps |
| `tf-research-heuristics` | Strategies for MCP research -- what to look for, which tools, in what order |
| `tf-architecture-patterns` | Patterns for module architecture -- resource composition, conditional creation, policy composition |
| `tf-implementation-patterns` | Patterns for Terraform code -- locals, for_each, dynamic blocks, lifecycle |
| `terraform-test` | Terraform test patterns -- plan-only, conditional resources, validation errors, mocks (elevated to central role in v2) |
| `terraform-style-guide` | Code style conventions -- naming, formatting, file organization |
| `tf-judge-criteria` | 6-dimension quality scoring rubric for module evaluation |
| `tf-compound-patterns` | Post-implementation compound learning -- cross-module patterns discovered during builds |

### Simplified (from v1)

| Skill | Purpose |
|-------|---------|
| `tf-report-template` | Validation results summary (simplified from full readiness report) |

## Prerequisites

1. GitHub CLI authenticated: `gh auth status`
2. HCP Terraform token: `$TFE_TOKEN` set (for publishing/testing)
3. MCP servers configured: terraform, aws-knowledge-mcp-server
4. Pre-commit hooks installed: `pre-commit install`

## Testing Strategy

Module testing follows a TDD approach. Tests are written before module code and drive the implementation.

1. **Write tests first** from `design.md` test scenarios. Each scenario becomes a `run` block in a `.tftest.hcl` file. Each assertion in the design maps 1:1 to an `assert` block.
2. **Plan-only tests** (`command = plan`) for fast feedback without cloud access. Validate resource configuration, conditional creation, variable validation, and security defaults.
3. **Run tests after each implementation phase**. Expect failures early -- that is the point of TDD. Track progress by counting passing assertions.
4. **Variable validation tests** use `expect_failures` to verify that invalid inputs are rejected with clear error messages.
5. **Security assertions exist from line 1**. Encryption, public access blocks, TLS enforcement, and least-privilege policies are tested before any feature code is written.
6. **Pre-commit checks**: `terraform fmt`, `terraform validate`, `tflint`, `trivy`, `terraform-docs`.

```
tests/
  basic.tftest.hcl         # Secure defaults, features disabled, core outputs
  complete.tftest.hcl       # All features enabled, security assertions
  validation.tftest.hcl     # All invalid input cases (expect_failures)
```

## Operational Notes

### GitHub Enterprise Authentication

For GHE repositories:
- **Authentication**: `gh auth login --hostname <hostname>` is required. Standard `gh auth login` only authenticates against github.com.
- **Operations**: Most `gh` commands (issue, pr, repo, etc.) do NOT accept `--hostname` flag. Use `GH_HOST` environment variable instead:
  ```bash
  export GH_HOST=github.enterprise.com
  gh issue create --title "Bug report"
  # Or inline:
  GH_HOST=github.enterprise.com gh pr create --title "Feature"
  ```

### Agent Output Persistence

All agents have the Write tool and are responsible for persisting their own output artifacts. The orchestrator verifies that expected output files exist after each agent dispatch.

## Context Management

1. **NEVER call TaskOutput** to read subagent results. All agents write artifacts to disk -- reading them back into the orchestrator bloats context and triggers compaction.
2. **Verify file existence with Glob** after each agent completes -- do NOT read file contents into the orchestrator.
3. **Downstream agents read their own inputs from disk.** The orchestrator passes only the FEATURE path and a brief scope description via `$ARGUMENTS`.
4. **Research agents: parallel foreground Task calls** (NOT `run_in_background`). Launch ALL research agents in a single message with multiple Task tool calls, then wait for all to complete before proceeding.
5. **Minimal $ARGUMENTS**: Only pass the FEATURE path + a specific question or scope. Never inject file contents.

**Remember**: Always verify with MCP tools. Security is non-negotiable.
