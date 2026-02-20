# tf-plan-v2 Flow Diagram

Mapping of the `tf-plan-v2` orchestrator skill and its interaction with the `sdd-research` and `sdd-design` agents.

## Full Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      tf-plan-v2 (Orchestrator Skill)                     │
│                           Phases 1 + 2                                   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PHASE 1: UNDERSTAND                                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  Step 1: Run validate-env.sh --json                                │  │
│  │          gate_passed=false? ──Yes──▶ STOP                          │  │
│  │                │ OK                                                │  │
│  │                ▼                                                   │  │
│  │  Step 2: Parse $ARGUMENTS (module name, provider, description)     │  │
│  │          Incomplete? ──▶ AskUserQuestion                           │  │
│  │                │                                                   │  │
│  │                ▼                                                   │  │
│  │  Step 3: Create GitHub issue                                       │  │
│  │          - Read issue-body-template.md                             │  │
│  │          - Fill placeholders                                       │  │
│  │          - gh issue create → capture $ISSUE_NUMBER                 │  │
│  │          (issue body updated again after Step 6)                   │  │
│  │                │                                                   │  │
│  │                ▼                                                   │  │
│  │  Step 4: create-new-feature.sh → capture $FEATURE branch          │  │
│  │                │                                                   │  │
│  │                ▼                                                   │  │
│  │  Step 5: Scan requirements against tf-domain-taxonomy              │  │
│  │          (8-category ambiguity scan)                               │  │
│  │          Always flag security-configurable features                │  │
│  │                │                                                   │  │
│  │                ▼                                                   │  │
│  │  Step 6: AskUserQuestion (up to 4 questions)                       │  │
│  │          MUST include security-defaults question                   │  │
│  │          ┌──────────────────────────────────┐                      │  │
│  │          │ User answers clarifications      │                      │  │
│  │          └──────────────┬───────────────────┘                      │  │
│  │                         │                                          │  │
│  │                         ▼                                          │  │
│  │  Step 7: Launch 3-4 CONCURRENT sdd-research agents                 │  │
│  │                                                                    │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────┐ │  │
│  │  │ sdd-research │ │ sdd-research │ │ sdd-research │ │sdd-resrch│ │  │
│  │  │  (Agent 1)   │ │  (Agent 2)   │ │  (Agent 3)   │ │(Agent 4) │ │  │
│  │  │              │ │              │ │              │ │ optional  │ │  │
│  │  │ Provider     │ │ AWS best     │ │ Registry     │ │ Edge     │ │  │
│  │  │ docs         │ │ practices    │ │ patterns     │ │ cases    │ │  │
│  │  │              │ │              │ │              │ │          │ │  │
│  │  │ INPUT:       │ │ INPUT:       │ │ INPUT:       │ │ INPUT:   │ │  │
│  │  │ 1 question   │ │ 1 question   │ │ 1 question   │ │1 question│ │  │
│  │  │              │ │              │ │              │ │          │ │  │
│  │  │ MCP calls:   │ │ MCP calls:   │ │ MCP calls:   │ │MCP calls:│ │  │
│  │  │ -get_provider│ │ -aws_search  │ │ -search      │ │-aws_read │ │  │
│  │  │ -search_provs│ │ -aws_read    │ │  _modules    │ │-get_provs│ │  │
│  │  │              │ │ -aws_recomm  │ │ -get_module  │ │          │ │  │
│  │  │ OUTPUT:      │ │ OUTPUT:      │ │ OUTPUT:      │ │ OUTPUT:  │ │  │
│  │  │ Findings     │ │ Findings     │ │ Findings     │ │ Findings │ │  │
│  │  │ (<500 tok)   │ │ (<500 tok)   │ │ (<500 tok)   │ │(<500 tok)│ │  │
│  │  │ IN-MEMORY    │ │ IN-MEMORY    │ │ IN-MEMORY    │ │IN-MEMORY │ │  │
│  │  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └────┬─────┘ │  │
│  │         │                │                │              │        │  │
│  │         └────────────────┴────────┬───────┴──────────────┘        │  │
│  │                                   │                               │  │
│  │                    All findings returned in-memory                 │  │
│  │                    (P4: NO files written to disk)                  │  │
│  └───────────────────────────────────┬───────────────────────────────┘  │
│                                      │                                  │
│            Orchestrator holds:                                           │
│            - Clarified requirements (from Step 6)                        │
│            - Research findings (from Step 7)                             │
│            - $FEATURE path                                               │
│                                      │                                  │
│                                      ▼                                  │
│  PHASE 2: DESIGN                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  Step 8: Launch sdd-design agent                                   │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │               sdd-design (Agent)                              │  │  │
│  │  │                                                              │  │  │
│  │  │  INPUT (via $ARGUMENTS):                                     │  │  │
│  │  │  - FEATURE path                                              │  │  │
│  │  │  - Clarified requirements                                    │  │  │
│  │  │  - Research findings summary                                 │  │  │
│  │  │                                                              │  │  │
│  │  │  READS ITSELF:                                               │  │  │
│  │  │  - .foundations/memory/constitution.md                        │  │  │
│  │  │  - .foundations/templates/design-template.md                  │  │  │
│  │  │                                                              │  │  │
│  │  │  PRODUCES 7 SECTIONS:                                        │  │  │
│  │  │  ┌────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │ § 1. Purpose & Requirements                            │  │  │  │
│  │  │  │ § 2. Resources & Architecture (resource inventory)    │  │  │  │
│  │  │  │ § 3. Interface Contract (variables + outputs)         │  │  │  │
│  │  │  │ § 4. Security Controls (6 domains)                    │  │  │  │
│  │  │  │ § 5. Test Scenarios (5 scenario groups)               │  │  │  │
│  │  │  │ § 6. Implementation Checklist (4-8 items)             │  │  │  │
│  │  │  │ § 7. Open Questions                                   │  │  │  │
│  │  │  └────────────────────────────────────────────────────────┘  │  │  │
│  │  │                                                              │  │  │
│  │  │  VALIDATES before writing:                                   │  │  │
│  │  │  - Every variable has Type + Description                     │  │  │
│  │  │  - Every resource has Logical Name + Key Config              │  │  │
│  │  │  - Every security control has CIS/WA reference               │  │  │
│  │  │  - Security controls map to test assertions                  │  │  │
│  │  │  - All 5 scenario groups present                             │  │  │
│  │  │  - Every scenario has >= 2 assertions                        │  │  │
│  │  │  - Checklist has 4-8 items                                   │  │  │
│  │  │  - No cross-section line references                          │  │  │
│  │  │  - Variable/resource names appear exactly once               │  │  │
│  │  │                                                              │  │  │
│  │  │  OUTPUT: specs/{FEATURE}/design.md                           │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                         │                                          │  │
│  │                         ▼                                          │  │
│  │  Step 9:  Glob — specs/{FEATURE}/design.md exists?                 │  │
│  │           No? → Re-launch sdd-design once                          │  │
│  │                         │ Yes                                      │  │
│  │                         ▼                                          │  │
│  │  Step 10: Grep — all 7 sections present?                           │  │
│  │           (## 1. Purpose through ## 7. Open Questions)             │  │
│  │           Missing? → Fix inline                                    │  │
│  │                         │ All present                              │  │
│  │                         ▼                                          │  │
│  │  Step 11: AskUserQuestion — present design summary                 │  │
│  │           ┌─────────────────────────────────────────────┐          │  │
│  │           │ Summary: input/output counts, resource      │          │  │
│  │           │ count, security controls, test scenarios,   │          │  │
│  │           │ checklist items                             │          │  │
│  │           │                                             │          │  │
│  │           │ Options:                                    │          │  │
│  │           │   [Approve]  [Review file first]  [Changes] │          │  │
│  │           └──────────────────┬──────────────────────────┘          │  │
│  │                              │                                     │  │
│  │                   ┌──────────┼──────────┐                          │  │
│  │                   ▼          ▼          ▼                          │  │
│  │              Approve    Review     Request Changes                  │  │
│  │                 │       file first       │                          │  │
│  │                 │          │              │                          │  │
│  │                 │          │    Step 12: Apply changes,             │  │
│  │                 │          │    re-present (loop until approved)    │  │
│  │                 │          │              │                          │  │
│  │                 │          └──────────────┘                          │  │
│  │                 │                │                                  │  │
│  │                 ▼                ▼                                  │  │
│  │                 └────────┬───────┘                                  │  │
│  │                          │ APPROVED                                │  │
│  └──────────────────────────┼─────────────────────────────────────────┘  │
│                             │                                            │
│                             ▼                                            │
│  DONE                                                                    │
│  Design approved at specs/{FEATURE}/design.md                            │
│  Run /tf-implement $FEATURE to build.                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Summary

```
User prompt
    │
    ▼
tf-plan-v2 orchestrator
    │
    ├──▶ Parse arguments + AskUserQuestion (clarifications)
    │         │
    │         ▼
    │    Clarified requirements ─────────────────────────────────┐
    │                                                            │
    ├──▶ 3-4x sdd-research agents (concurrent, in-memory)       │
    │    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
    │    │ Provider  │ │ AWS best │ │ Registry │ │ Edge     │   │
    │    │ docs Q    │ │ practice │ │ patterns │ │ cases    │   │
    │    └─────┬────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘   │
    │          └───────────┴────────────┴─────────────┘         │
    │                      │                                     │
    │              Research findings (in-memory) ────────────────┤
    │                                                            │
    │                                                            ▼
    ├──▶ sdd-design agent ◀──── requirements + findings + $FEATURE
    │         │
    │         │  Also reads (itself):
    │         │  - constitution.md
    │         │  - design-template.md
    │         │
    │         ▼
    │    specs/{FEATURE}/design.md   ◀── SINGLE OUTPUT ARTIFACT
    │
    ├──▶ Orchestrator verifies (Glob + Grep, never reads content)
    │
    └──▶ User approval gate (AskUserQuestion)
              │
              ▼
         /tf-implement picks up from here
```

## Handoff to tf-implement

```
┌─────────────┐                              ┌──────────────┐
│ tf-plan-v2  │  produces                    │ tf-implement  │
│ (Phases 1-2)│ ──────▶ design.md ──────▶    │ (Phases 3-4)  │
│             │         (approved)           │               │
└─────────────┘                              └──────────────┘

The ONLY artifact passed between the two skills is:
    specs/{FEATURE}/design.md

No other files, no shared state, no intermediate research artifacts.
```

## Analysis: Does the Flow Make Sense?

**Yes, the flow is well-structured.** It faithfully implements AGENTS.md principles P1, P3, P4, P6, and P8.

### What's Right

1. **Single artifact output (P1)**: The entire planning phase produces exactly one file: `specs/{FEATURE}/design.md`. No research files, no separate specs, no intermediate artifacts.

2. **Research feeds design, not files (P4)**: The sdd-research agents return findings in-memory (<500 tokens each). The orchestrator passes these to sdd-design via `$ARGUMENTS`. Nothing is written to disk. This directly prevents the v1 terminology drift problem.

3. **Security embedded in design (P3)**: Security is woven through at three points:
   - Step 5: Ambiguity scan flags security-configurable features
   - Step 6: Mandatory security-defaults clarification question
   - sdd-design agent: Mandatory Section 4 (Security Controls) with CIS/WA references, plus security assertions required in Section 5 tests

4. **Orchestrator directs, doesn't accumulate (P6)**: The orchestrator passes short context (requirements, findings summary, file paths) to agents. It verifies design.md exists via Glob and checks section presence via Grep. It never reads the full design content itself.

5. **Phase order is fixed (P8)**: Understand must complete before Design starts. Research agents must all return before sdd-design launches. User must approve before /tf-implement can run.

6. **Agents have one job (P5)**: Each sdd-research agent answers exactly ONE question. The sdd-design agent takes requirements + findings and produces exactly ONE file.

### One Thing to Watch

The GitHub issue is created at Step 3 (before clarification) and updated after Step 6 (after clarification). This means there's a window where the issue exists with incomplete information. This is intentional — the issue serves as a tracking anchor from the start — but if the workflow fails between Steps 3 and 6, there's an orphaned issue with placeholder content. Not a design flaw, just an operational edge case worth being aware of.
