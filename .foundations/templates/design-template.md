# Module Design: terraform-{provider}-{name}

**Branch**: feat/{name}
**Date**: {YYYY-MM-DD}
**Status**: Draft | Approved | Implementing | Complete
**Provider**: {provider} >= {version}
**Terraform**: >= {version}

---

## 1. Purpose

{One paragraph. What infrastructure this module creates, who consumes it,
what problem it solves. No implementation details.}

**Scope boundary**: {What is explicitly OUT of scope -- prevents scope creep.}

---

## 2. Interface Contract

### Inputs

| Variable | Type | Required | Default | Validation | Sensitive | Description |
|----------|------|----------|---------|------------|-----------|-------------|
| {name} | {type} | {Yes/No} | {value or --} | {rule or --} | {Yes/No} | {description} |

{This table is the SINGLE SOURCE OF TRUTH for the module's input interface.
It is not repeated anywhere else in any artifact.}

### Outputs

| Output | Type | Conditional On | Description |
|--------|------|----------------|-------------|
| {name} | {type} | {variable or "always"} | {description} |

---

## 3. Resources & Architecture

### Resource Inventory

| Resource Type | Logical Name | Conditional | Depends On | Key Configuration |
|---------------|-------------|-------------|------------|-------------------|
| {aws_resource_type} | {this/name} | {variable or "always"} | {resource.name} | {notable settings} |

### Architectural Decisions

{Each decision as a paragraph with this structure:}

**{Decision title}**: {What was chosen}.
*Rationale*: {Why, with MCP research citation if applicable}.
*Rejected*: {What was considered and why it was rejected}.

---

## 4. Security Controls

| Control | Enforcement | Configurable? | Reference |
|---------|-------------|---------------|-----------|
| Encryption at rest | {how} | {Yes: variable / No: hardcoded} | {CIS/WA control} |
| Encryption in transit | {how} | {Yes/No} | {CIS/WA control} |
| Public access | {how} | {Yes/No} | {CIS/WA control} |
| IAM least privilege | {how} | {Yes/No} | {CIS/WA control} |
| Logging | {how} | {Yes/No} | {CIS/WA control} |
| Tagging | {how} | {Yes/No} | {CIS/WA control} |

{Rules:
- If a control is hardcoded (not configurable), document WHY.
- If it is configurable, the default MUST be the secure option.
- Mark N/A where a domain does not apply, with justification.
- Reference column must cite a CIS AWS Benchmark or AWS Well-Architected control.}

---

## 5. Test Scenarios

{Three scenario groups are required: Secure Defaults, Full Features, and Validation Errors.}

### Scenario: Secure Defaults (basic)

**Purpose**: Verify the module works with minimal inputs and security is enabled by default
**Example**: `examples/basic/`
**Command**: `plan`

**Inputs**:
```hcl
{only required variables -- minimal configuration}
```

**Assertions**:
- {security assertion: encryption enabled by default}
- {security assertion: public access blocked by default}
- {functional assertion: core resource created}
- ...

### Scenario: Full Features (complete)

**Purpose**: Verify all features enabled, all optional resources created, all outputs populated
**Example**: `examples/complete/`
**Command**: `plan`

**Inputs**:
```hcl
{all features enabled, all optional variables set}
```

**Assertions**:
- {all optional resources created}
- {all outputs populated}
- {security assertions still hold}
- ...

### Scenario: Validation Errors

**Purpose**: Verify input validation rejects bad inputs

**Expect error cases**:
- {input}: {value} -> {expected error message substring}
- ...

---

## 6. Implementation Checklist

- [ ] **A: Scaffold** -- Create file structure, versions.tf, all variables, locals, base resource, core outputs
- [ ] **B: Security core** -- {encryption, access controls, policy -- whatever is security-critical}
- [ ] **C: Feature set** -- {remaining resources, conditional creation}
- [ ] **D: Examples** -- examples/basic/ and examples/complete/
- [ ] **E: Tests** -- .tftest.hcl files from scenarios above
- [ ] **F: Polish** -- README (terraform-docs), formatting, validation, security scan

{Keep this to 4-8 items. Each item = one implementation pass.
NOT a 34-task breakdown. Each item should be completable in one agent turn.}

---

## 7. Open Questions

{Any deferred decisions marked [DEFERRED] with context.
Empty section if all questions resolved during clarification.}

---

## Template Rules

1. No section may reference another section by line number
2. Variable names appear exactly once -- in Interface Contract
3. Resource names appear exactly once -- in Resource Inventory
4. Each test assertion maps 1:1 to a .tftest.hcl assert block
5. Implementation checklist items are coarse-grained -- one per logical unit
