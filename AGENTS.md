# SDD v2 System Design Principles

**Scope**: Rules for building the v2 workflow system — agents, skills, orchestrator, templates, and repo structure. This governs the developer creating the tooling, not the user running it.

**Authority**: If a design choice during development conflicts with a principle here, the principle wins. No exceptions without removing the principle first.

---

## P1. Single Artifact Output

The entire planning phase produces one file: `specs/{FEATURE}/design.md`.

- Do not build agents that produce separate specification, plan, contract, data model, or task files.
- Do not build templates with sections that restate information from other sections. Every variable name, resource entry, validation rule, and output appears in exactly one table.
- Do not build a consistency analysis agent or skill. It exists in v1 only because multiple artifacts can contradict each other. One file cannot contradict itself.
- Do not build a remediation cycle. There is nothing to remediate.

**Test**: If you can grep `design.md` for a variable name and get more than one row across all tables, the template is wrong.

---

## P2. Tests Before Code

The system writes `.tftest.hcl` files before the module `.tf` files they validate.

- Build the test-writer agent (or step) to run before the implementation agent. Not after. Not in parallel.
- Build the orchestrator to enforce this order: design.md exists -> tests exist -> implementation starts.
- Do not build a separate "testing phase" that runs after implementation. Tests grow alongside code. The final validation phase confirms they all pass — it does not create them.
- Build test scenarios as a mandatory section in the design.md template. If the section is empty, the design is incomplete and implementation cannot start.

**Test**: Remove all `.tf` files except `versions.tf` and `variables.tf`. `terraform test` should still parse the test files without syntax errors.

---

## P3. Security Embedded in Design

Do not build a separate security review agent or phase.

- Build the design agent to include security research (MCP calls to AWS docs, CIS benchmarks, provider security attributes) as part of its own execution.
- Build the design.md template with a mandatory Security Controls table. This table is the security specification — no separate review document feeds into it.
- Build each security control row to require a mapping to at least one test assertion. The orchestrator validates this mapping before approving the design.
- Build the Phase 1 clarification step to always include a security-defaults question. This is not optional. The v1 CRITICAL finding (hardcoded vs. configurable public access) came from skipping this question.

**Test**: Search the entire repo for an agent or skill file containing "security-review" or "security-advisor" as a standalone phase. Finding one means the system violates this principle.

---

## P4. Research Feeds Design, Not Files

Do not build research agents that write output to disk.

- Build the orchestrator to run MCP research (provider docs, AWS docs, registry patterns) during Phase 1 and pass results directly to the design agent as input context.
- Do not create a `research/` directory, `research-*.md` files, or any intermediate research artifacts. Persisted research files caused terminology drift in v1 — variable names in research files diverged from variable names in the spec.
- Build the design.md template to capture research outcomes as citations in the Architectural Decisions section. The citation (provider doc version, AWS URL) is the audit trail.

**Test**: After a complete workflow run, `ls specs/{FEATURE}/` should contain exactly one file: `design.md`. Nothing else.

---

## P5. Agents Have One Job

Each agent takes one defined input and produces one defined output.

- Do not build agents that read outputs from other agents' files. The orchestrator passes context — agents do not cross-reference each other.
- Do not build agents that produce multiple files. One agent, one output path.
- Do not build the orchestrator to read agent output files back into its own context. The orchestrator verifies file existence (glob), not content. Downstream agents read their own inputs.

| Agent         | Input                                                     | Output                      |
| ------------- | --------------------------------------------------------- | --------------------------- |
| Design agent  | Requirements text + research findings + constitution path | `specs/{FEATURE}/design.md` |
| Test writer   | `specs/{FEATURE}/design.md` Section 5                     | `tests/*.tftest.hcl`        |
| Task executor | `specs/{FEATURE}/design.md` + checklist item              | Modified `.tf` files        |

- If you find yourself building an agent that needs to read two other agents' outputs and merge them, the upstream agents are wrong — merge them into one.

**Test**: Draw the data flow between agents. If any arrow goes agent-to-agent without passing through the orchestrator or through `design.md`, the architecture is wrong.

---

## P6. The Orchestrator Directs, Does Not Accumulate

The orchestrator skill manages phase sequencing, file verification, and user interaction. It does not accumulate content.

- Do not build the orchestrator to read file contents from agent outputs. It checks file existence. That is all.
- Do not build the orchestrator to pass large blocks of text between phases. It passes file paths and short scope descriptions.
- Build the orchestrator with exactly 4 phases: Understand, Design, Build+Test, Validate. Do not add phases. If something feels like it needs a new phase, it belongs inside an existing one.
- Build the orchestrator to stop on exactly 3 conditions: missing prerequisites (Phase 1), user rejects design (Phase 2), security test failures (Phase 3). Everything else is recoverable without stopping.

**Test**: Count the number of `Read` tool calls the orchestrator makes against `specs/` files. The target is zero. File existence checks use `Glob`.

---

## P7. The Design Template Is the Contract

The design.md template defines what the design agent must produce. It is the single enforced schema.

- Build the template with mandatory sections. No section is optional. Empty tables are permitted (e.g., no CORS resources); missing sections are not.
- Build the template so each section is self-contained. No section references another section by line number or says "see above."
- Do not build the template with more than 7 sections: Purpose, Interface Contract, Resources & Architecture, Security Controls, Test Scenarios, Implementation Checklist, Open Questions.
- Build the Implementation Checklist section to contain 4-8 items. Not 34 tasks. Each item is one logical unit of work — "Scaffold + versions.tf + variables + base resource" is one item.

**Test**: A developer unfamiliar with the system should be able to read `design.md` alone and know everything needed to build and test the module. If they need to open a second file for context, the template is incomplete.

---

## P8. Phase Order Is Fixed

The 4 phases execute in strict sequence. No phase is skippable. No phase runs out of order.

```
Understand -> Design -> Build+Test -> Validate
```

- Do not build shortcuts that skip Design and jump to Build. Even for "simple" modules.
- Do not build the Validate phase to run before Build+Test completes. Validation confirms the finished module, not a partial one.
- Do not build parallel execution across phases. Parallelism is permitted within a phase (multiple MCP research calls in Phase 1, multiple validation tools in Phase 4). Never across phases.

**Test**: Remove any single phase from the orchestrator. The system should fail clearly, not silently produce an incomplete result.

---

## P9. The Constitution Is Separate

Do not merge the module constitution into the workflow system.

- The constitution defines code standards: file structure, naming, security defaults, provider constraints, testing coverage. It governs what correct module code looks like.
- These principles define system architecture: single artifact, test-first, embedded security, agent boundaries. They govern how the tooling is built.
- Build the design agent to receive the constitution as a reference input (file path). The agent applies it during design. The orchestrator does not interpret it.
- Do not duplicate constitution rules into agent prompts, skill files, or templates. Point to the constitution. One source.

**Test**: Change a constitution rule (e.g., the minimum Terraform version). Exactly one file should need updating. If agent prompts or templates also need updating, the rule is duplicated.

---

## P10. Build for Deletion

Every component in the system should be removable without breaking unrelated components.

- Build agents as independent units. Removing the test-writer agent means tests are written inline by the implementer — the orchestrator and design agent still function.
- Build skills with no hard dependencies on other skills. Each skill file is self-contained.
- Do not build shared state between agents beyond `design.md` and the module's `.tf` files.
- Do not build helper utilities, shared libraries, or abstraction layers that multiple agents depend on. If two agents need the same logic, each agent contains its own copy. Duplication in agent prompts is acceptable. Coupling between agents is not.

**Test**: Delete any single agent file from `.claude/agents/`. The orchestrator should degrade gracefully (skip that step or inline the work), not crash.
