# SDD v2 Implementation Guide

**Companion to**: `terraform-sdd-v2.md` (the proposal)
**Purpose**: Everything a fresh Claude Code session needs to build the v2 workflow from scratch
**Audience**: An AI agent bootstrapping the workflow in a new repository

---

## Document Set

To implement SDD v2, you need exactly 3 documents:

| # | Document | Purpose | You are here |
|---|----------|---------|:---:|
| 1 | `terraform-sdd-v2.md` | The proposal — WHAT and WHY | |
| 2 | `terraform-sdd-v2-implementation-guide.md` | The blueprint — HOW | **<--** |
| 3 | A constitution (`.foundations/memory/constitution.md`) | The rules — MUST and MUST NOT | |

The constitution defines non-negotiable constraints (encryption always on, no provider blocks in root modules, standard file layout, etc.). It is the authority on code standards. This guide is the authority on workflow mechanics.

---

## Part 1: The Design Document Template

This is the single artifact that replaces spec, plan, contracts, data model, and tasks. Every section is mandatory. The design agent produces this.

```markdown
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

**Scope boundary**: {What is explicitly OUT of scope — prevents scope creep.}

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

{If a security control is hardcoded (not configurable), document WHY.
If it is configurable, the default MUST be the secure option.}

---

## 5. Test Scenarios

### Scenario: {name} ({example directory})

**Purpose**: {what this scenario validates}
**Example**: `examples/{name}/`
**Command**: `plan` or `apply`

**Inputs**:
```hcl
{variable assignments for this scenario}
```

**Assertions**:
- {plain-language assertion that maps to a tftest assert block}
- {another assertion}
- ...

### Scenario: Validation Errors

**Purpose**: Verify input validation rejects bad inputs

**Expect error cases**:
- {input}: {value} -> {expected error message substring}
- ...

---

## 6. Implementation Checklist

- [ ] **A: Scaffold** — Create file structure, versions.tf, all variables, locals, base resource, core outputs
- [ ] **B: Security core** — {encryption, access controls, policy — whatever is security-critical}
- [ ] **C: Feature set** — {remaining resources, conditional creation}
- [ ] **D: Examples** — examples/basic/ and examples/complete/
- [ ] **E: Tests** — .tftest.hcl files from scenarios above
- [ ] **F: Polish** — README (terraform-docs), formatting, validation, security scan

{Keep this to 4-8 items. Each item = one implementation pass.
NOT a 34-task breakdown. Each item should be completable in one agent turn.}

---

## 7. Open Questions

{Any deferred decisions marked [DEFERRED] with context.
Empty section if all questions resolved during clarification.}
```

### Template Rules

1. **No section may reference another section by line number** — sections are self-contained
2. **Variable names appear exactly once** — in the Interface Contract table
3. **Resource names appear exactly once** — in the Resource Inventory table
4. **Each test assertion maps 1:1 to a .tftest.hcl assert block** — no ambiguity
5. **Implementation checklist items are coarse-grained** — one per logical unit of work, not per resource

---

## Part 2: Terraform Test Patterns

The TDD approach requires converting design document test scenarios into `.tftest.hcl` files BEFORE writing module code. This section provides the patterns.

### Pattern 1: Plan-Only Test Against Example

This is the most common pattern. Tests plan against an example directory without deploying real infrastructure.

```hcl
# tests/basic.tftest.hcl

# Test secure defaults with minimal inputs
run "secure_defaults" {
  command = plan

  # Point to the example directory
  module {
    source = "./examples/basic"
  }

  # Override variables for testing
  variables {
    bucket_name = "test-basic-bucket"
    environment = "dev"
    owner       = "test-team"
    cost_center = "CC-0000"
  }

  # Each assertion maps to a line in design.md Section 5
  assert {
    condition     = aws_s3_bucket.this.bucket == "test-basic-bucket"
    error_message = "Bucket name must match input variable"
  }

  assert {
    condition     = aws_s3_bucket.this.tags["Environment"] == "dev"
    error_message = "Environment tag must be set from variable"
  }
}
```

### Pattern 2: Conditional Resource Assertions

Test that resources are created or not based on feature flags.

```hcl
run "website_disabled" {
  command = plan

  variables {
    bucket_name    = "test-no-website"
    environment    = "dev"
    owner          = "test"
    cost_center    = "CC-0000"
    enable_website = false
  }

  # Assert the conditional resource is NOT created
  assert {
    condition     = length(aws_s3_bucket_website_configuration.this) == 0
    error_message = "Website configuration must not be created when disabled"
  }
}

run "website_enabled" {
  command = plan

  module {
    source = "./examples/complete"
  }

  variables {
    # ... complete example variables
  }

  # Assert the conditional resource IS created
  assert {
    condition     = aws_s3_bucket_website_configuration.this[0].index_document[0].suffix == "index.html"
    error_message = "Index document must default to index.html"
  }
}
```

### Pattern 3: Variable Validation Tests

Test that invalid inputs are rejected with clear error messages.

```hcl
run "invalid_bucket_name_uppercase" {
  command = plan

  variables {
    bucket_name = "INVALID-UPPERCASE"
    environment = "dev"
    owner       = "test"
    cost_center = "CC-0000"
  }

  # Expect the plan to fail with a validation error
  expect_failures = [
    var.bucket_name,
  ]
}

run "invalid_environment" {
  command = plan

  variables {
    bucket_name = "test-valid-name"
    environment = "production"  # not in allowed list
    owner       = "test"
    cost_center = "CC-0000"
  }

  expect_failures = [
    var.environment,
  ]
}
```

### Pattern 4: Cross-Variable Validation

Test validation rules that span multiple variables.

```hcl
run "oac_without_arn" {
  command = plan

  variables {
    bucket_name                  = "test-oac"
    environment                  = "dev"
    owner                        = "test"
    cost_center                  = "CC-0000"
    cloudfront_oac_access_enabled = true
    # cloudfront_distribution_arn intentionally omitted
  }

  expect_failures = [
    var.cloudfront_distribution_arn,
  ]
}
```

### Pattern 5: Security Assertion Patterns

Common security checks to include in every module's test suite.

```hcl
# Encryption is always on (no way to disable)
assert {
  condition     = aws_s3_bucket_server_side_encryption_configuration.this.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "AES256"
  error_message = "Encryption must be enabled by default"
}

# Public access is blocked
assert {
  condition     = aws_s3_bucket_public_access_block.this.block_public_acls == true
  error_message = "Public ACLs must be blocked"
}

# Tags include required metadata
assert {
  condition     = aws_s3_bucket.this.tags["ManagedBy"] == "terraform"
  error_message = "ManagedBy tag must be set to terraform"
}
```

### Pattern 6: Mock Providers (for unit tests without AWS access)

```hcl
mock_provider "aws" {
  alias = "mock"
}

run "unit_test_with_mock" {
  command = plan

  providers = {
    aws = aws.mock
  }

  variables {
    bucket_name = "test-mock"
    environment = "dev"
    owner       = "test"
    cost_center = "CC-0000"
  }

  # Assertions work the same way against mocked provider
  assert {
    condition     = output.bucket_id != null
    error_message = "bucket_id output must not be null"
  }
}
```

### Test File Organization

```
tests/
  basic.tftest.hcl       # Scenarios: secure defaults, features disabled, outputs
  complete.tftest.hcl     # Scenarios: all features enabled, security assertions
  validation.tftest.hcl   # Scenarios: all invalid input cases
```

Each test file maps to a scenario group in design.md Section 5. The mapping is explicit:
- Design scenario "Secure Defaults" -> `tests/basic.tftest.hcl` run "secure_defaults"
- Design scenario "Full Features" -> `tests/complete.tftest.hcl` run "full_features"
- Design scenario "Validation Errors" -> `tests/validation.tftest.hcl` (multiple runs)

---

## Part 3: MCP Tool Playbook

The workflow relies on two MCP servers for research. This playbook defines which tools to use, when, and in what order.

### Terraform MCP Server

**When**: Phase 1 (Understand) — researching provider resources and registry patterns

```
Discovery flow:
1. get_latest_provider_version(namespace, name)     → Get current provider version
2. get_provider_capabilities(namespace, name)        → See what resource types exist
3. search_providers(provider_name, service_slug,     → Find specific resource docs
     provider_document_type="resources")
4. get_provider_details(provider_doc_id)             → Read full resource documentation

Registry pattern flow:
1. search_modules(module_query)                      → Find public modules for patterns
2. get_module_details(module_id)                     → Study interface conventions

Private registry (if TFE token available):
1. search_private_modules(terraform_org_name)        → Check org's existing modules
2. get_private_module_details(...)                   → Study internal conventions
```

**Key rule**: Always check the private registry first when a TFE token is available. Fall back to public.

### AWS Knowledge MCP Server

**When**: Phase 1 (Understand) — researching AWS service behavior and best practices

```
Documentation flow:
1. search_documentation(search_phrase, topics)        → Find relevant docs
   Topics: "reference_documentation" for API/SDK,
           "general" for best practices,
           "troubleshooting" for edge cases
2. read_documentation(url)                           → Read full doc page

Security baseline flow:
1. search_documentation("S3 security best practices",
     topics=["general"])                             → Security guidance
2. search_documentation("CIS AWS S3 controls",
     topics=["general"])                             → Compliance controls

Regional availability (if needed):
1. get_regional_availability(region, resource_type,
     filters)                                        → Check feature availability
```

### Research Strategy

For a typical module, launch **3-4 parallel research agents** covering:

| Agent | MCP Tools | Question |
|-------|-----------|----------|
| Provider resources | `search_providers` + `get_provider_details` | How do the main resources work? What are the arguments, attributes, gotchas? |
| AWS best practices | `search_documentation` + `read_documentation` | What does AWS recommend for this service? What are the security controls? |
| Registry patterns | `search_modules` + `get_module_details` | How do popular public modules structure their interface for this service? |
| Edge cases | `search_documentation` (troubleshooting topic) | What breaks? What are common mistakes? |

**Critical insight from our S3 run**: Research agents discovered architecture-shaping facts NOT available from general knowledge:
- CloudFront OAC cannot use the S3 website endpoint (must use REST endpoint)
- Public access block stays fully enabled with OAC (service principal is non-public)
- `filter {}` block is required in modern lifecycle configurations
- `bucket_key_enabled` reduces KMS API costs by 99%

These findings directly shaped architectural decisions. **Never skip research.**

---

## Part 4: Orchestrator Logic

The v2 orchestrator has 4 phases. This section defines the exact decision tree.

### Phase 1: Understand

```
START
  |
  v
Run validate-env.sh --json
  |
  ├─ gate_passed=false → STOP, show user what's missing
  |
  v
Call MCP list_terraform_orgs → verify TFE token
  |
  v
Read input (prompt.json, issue body, or user message)
  |
  v
Create GitHub issue for tracking
  |
  v
Scan requirements against 8-category taxonomy:
  - Functional Scope         [Clear/Partial/Missing]
  - Domain/Data Model        [Clear/Partial/Missing]
  - Operational Workflows    [Clear/Partial/Missing]
  - Non-Functional Attrs     [Clear/Partial/Missing]
  - Integration/Dependencies [Clear/Partial/Missing]
  - Edge Cases               [Clear/Partial/Missing]
  - Constraints/Tradeoffs    [Clear/Partial/Missing]  ← ALWAYS check security defaults
  - Terminology              [Clear/Partial/Missing]
  |
  v
Rank gaps by Impact × Uncertainty
  |
  v
Ask up to 4 questions via AskUserQuestion
  (MUST include security-defaults question if any security-configurable
   feature exists — "Should X be configurable or hardcoded?")
  |
  v
Launch parallel MCP research agents (3-4)
  |
  v
Wait for all research to complete
  |
  v
Post progress to GitHub issue: "Phase 1 complete"
  |
  v
PROCEED TO PHASE 2
```

### Phase 2: Design

```
Launch design agent with:
  - Clarified requirements (from Phase 1)
  - Research findings (from Phase 1 — passed directly, NOT as files)
  - Constitution reference (for security/code standards)
  - Design.md template (Part 1 of this guide)
  |
  v
Verify specs/{FEATURE}/design.md exists
  |
  v
Quick validation:
  - Every variable in Interface Contract has a type and description?
  - Every resource in Resource Inventory has a logical name?
  - Every security control has a CIS/WA reference?
  - Every test scenario has at least 2 assertions?
  - Implementation checklist has 4-8 items?
  |
  ├─ Validation fails → Fix inline, do not re-run agent
  |
  v
Post progress to GitHub issue: "Design complete, ready for review"
  |
  v
AWAIT USER APPROVAL (or proceed if autonomous mode)
  |
  v
PROCEED TO PHASE 3
```

### Phase 3: Build + Test

```
Step 1: Write test files from design.md Section 5
  |
  v
For each scenario in design.md:
  Convert assertions to .tftest.hcl run blocks
  Write to tests/{scenario_group}.tftest.hcl
  |
  v
Step 2: Implement module code
  |
  v
For each item in design.md Section 6 (Implementation Checklist):
  |
  ├─ Write/edit .tf files
  ├─ Run: terraform validate
  ├─ Run: terraform test (expect some failures early — that's TDD)
  ├─ Mark checklist item complete
  |
  v
All checklist items complete?
  ├─ No → Continue to next item
  |
  v
Run: terraform test (all tests should pass now)
  ├─ Failures → Fix module code, re-run
  |
  v
PROCEED TO PHASE 4
```

### Phase 4: Validate

```
Launch ALL in parallel:
  |
  ├─ terraform test                    → All pass?
  ├─ terraform validate                → Clean?
  ├─ terraform fmt -check -recursive   → Formatted?
  ├─ trivy config .                    → No Critical/High?
  ├─ terraform-docs markdown . > README.md
  |
  v
All green?
  ├─ No → Fix issues, re-validate
  |
  v
Post final progress to GitHub issue: "Implementation complete"
  |
  v
DONE
```

---

## Part 5: Security Baseline Quick Reference

Every AWS module should check these controls during design. Not all apply to every module — mark N/A where not applicable.

| Control | Default | How to Enforce | CIS Reference |
|---------|---------|---------------|---------------|
| Encryption at rest | ON, not disableable | Dedicated encryption resource, no toggle variable | CIS 2.1.1 |
| Encryption in transit | ON (TLS) | Bucket/resource policy with `aws:SecureTransport` deny | CIS 2.1.1 |
| Public access blocked | ON, hardcoded | All four S3 public access block flags `true` (or equivalent) | CIS 2.1.4 |
| Versioning | ON by default, toggleable | Variable with `default = true` | CIS 2.1.1 |
| Logging | Documented guidance if opt-in | Variable or note in README | CIS 2.1.2 |
| IAM least privilege | Always | Specific actions + resource ARNs + conditions | NIST AC-6 |
| Tags | Required tags enforced | Merge with precedence for org tags | AWS Well-Architected |
| Force destroy | OFF by default | `default = false`, examples may use `true` | Constitution 4.3 |
| No provider blocks | Always | Root module never has `provider {}` | Constitution 3.2 |
| No hardcoded creds | Always | Accept ARNs for KMS/secrets, never keys | Constitution 4.1 |

---

## Part 6: Worked Example — S3 Static Website Module

This is a condensed version of what the S3 module design.md would look like under v2. Use as a reference for the design agent.

### Condensed design.md (S3 Static Website)

```markdown
# Module Design: terraform-aws-s3-static-website

**Branch**: feat/s3-static-website
**Date**: 2026-02-18
**Status**: Draft
**Provider**: aws >= 5.0
**Terraform**: >= 1.5

## 1. Purpose

Creates an S3 bucket configured for static website hosting with encryption,
versioning, lifecycle rules, and configurable public access. Supports secure
content delivery via CloudFront OAC while keeping the bucket private.

**Scope boundary**: CloudFront distribution, DNS, ACM certificates, logging
bucket creation, and KMS key management are out of scope.

## 2. Interface Contract

### Inputs

| Variable | Type | Required | Default | Validation | Sensitive | Description |
|----------|------|----------|---------|------------|-----------|-------------|
| bucket_name | string | Yes | -- | 3-63 chars, lowercase+hyphens, no reserved prefixes | No | S3 bucket name |
| environment | string | Yes | -- | dev, staging, prod | No | Deployment environment |
| owner | string | Yes | -- | -- | No | Owner tag value |
| cost_center | string | Yes | -- | -- | No | CostCenter tag value |
| tags | map(string) | No | {} | -- | No | Additional tags (required tags take precedence) |
| versioning_enabled | bool | No | true | -- | No | Enable versioning |
| kms_key_arn | string | No | null | -- | Yes | KMS key ARN (null = AES256) |
| force_destroy | bool | No | false | -- | No | Allow non-empty bucket deletion |
| enable_website | bool | No | true | -- | No | Enable website hosting |
| index_document | string | No | "index.html" | -- | No | Website index document |
| error_document | string | No | "error.html" | -- | No | Website error document |
| cloudfront_oac_access_enabled | bool | No | false | -- | No | Enable CloudFront OAC policy |
| cloudfront_distribution_arn | string | No | null | Required when OAC enabled | No | CloudFront distribution ARN |
| enable_lifecycle_rule | bool | No | true | -- | No | Enable Glacier transition |
| lifecycle_glacier_days | number | No | 90 | > 0 | No | Days before Glacier transition |
| cors_allowed_origins | list(string) | No | [] | http/https or * | No | CORS origins (empty = no CORS) |
| logging_bucket | string | No | null | -- | No | Access log target (null = disabled) |
| logging_prefix | string | No | "" | -- | No | Access log prefix |

### Outputs

| Output | Type | Conditional On | Description |
|--------|------|----------------|-------------|
| bucket_id | string | always | Bucket name |
| bucket_arn | string | always | Bucket ARN |
| bucket_domain_name | string | always | Bucket domain |
| bucket_regional_domain_name | string | always | Regional domain (use for CloudFront origin) |
| website_endpoint | string | enable_website | Website URL (null when disabled) |
| website_domain | string | enable_website | Website domain (null when disabled) |

## 3. Resources & Architecture

| Resource Type | Name | Conditional | Depends On | Key Config |
|---------------|------|-------------|------------|------------|
| aws_s3_bucket | this | always | -- | bucket, force_destroy, tags |
| aws_s3_bucket_server_side_encryption_configuration | this | always | bucket | AES256 or aws:kms, bucket_key_enabled |
| aws_s3_bucket_versioning | this | always | bucket | Enabled/Suspended via variable |
| aws_s3_bucket_public_access_block | this | always | bucket | All four flags hardcoded true |
| aws_s3_bucket_policy | this | always | bucket, public_access_block | TLS deny + optional OAC allow |
| aws_s3_bucket_website_configuration | this | enable_website | bucket | index/error documents |
| aws_s3_bucket_lifecycle_configuration | this | enable_lifecycle_rule | bucket, versioning | filter {}, Glacier transition |
| aws_s3_bucket_cors_configuration | this | cors non-empty | bucket | GET/HEAD, specified origins |
| aws_s3_bucket_logging | this | logging_bucket != null | bucket | target_bucket, target_prefix |

**OAC uses REST endpoint**: CloudFront must use bucket_regional_domain_name
(not website_endpoint). OAC cannot work with S3 website endpoints.

**Public access hardcoded**: All four flags always true. OAC uses a service
principal (non-public). No block_public_access variable exists.

**Bucket policy composition**: Single aws_s3_bucket_policy with
data.aws_iam_policy_document containing TLS deny (always) + OAC allow (conditional).

## 4. Security Controls

| Control | Enforcement | Configurable? | Reference |
|---------|-------------|---------------|-----------|
| Encryption at rest | SSE-S3 default, SSE-KMS optional via kms_key_arn | Key choice only, cannot disable | CIS 2.1.1, S3.17 |
| Encryption in transit | Bucket policy deny on SecureTransport=false | No | CIS 2.1.1, S3.5 |
| Public access | All four flags hardcoded true | No | CIS 2.1.4, S3.8 |
| Bucket keys | Enabled when KMS used | No | Cost optimization |
| OAC least privilege | s3:GetObject only, SourceArn condition | No | S3.6 |
| Versioning | Enabled by default | Yes (toggleable) | S3.14 |
| Logging | Opt-in (documented limitation) | Yes | S3.9 (partial) |

## 5. Test Scenarios

### Scenario: Secure Defaults (examples/basic/)
Purpose: Validate secure baseline with only required inputs
Command: plan

Inputs:
  bucket_name = "test-secure-defaults"
  environment = "dev"
  owner = "test-team"
  cost_center = "CC-0000"
  enable_website = false
  force_destroy = true

Assertions:
- Encryption algorithm is "AES256"
- Versioning status is "Enabled"
- block_public_acls is true
- block_public_policy is true
- ignore_public_acls is true
- restrict_public_buckets is true
- Tags include Environment="dev", Owner="test-team", ManagedBy="terraform"
- website_endpoint output is null
- bucket_id output is not null

### Scenario: Full Features (examples/complete/)
Purpose: Validate all features enabled together
Command: plan

Inputs:
  bucket_name = "test-full-features"
  environment = "prod"
  owner = "platform-team"
  cost_center = "CC-1234"
  kms_key_arn = "arn:aws:kms:us-east-1:123456789012:key/test-key"
  enable_website = true
  cloudfront_oac_access_enabled = true
  cloudfront_distribution_arn = "arn:aws:cloudfront::123456789012:distribution/EXAMPLE"
  cors_allowed_origins = ["https://example.com"]
  logging_bucket = "test-logging-bucket"
  logging_prefix = "s3-access-logs/"
  lifecycle_glacier_days = 30
  force_destroy = true

Assertions:
- Encryption algorithm is "aws:kms"
- bucket_key_enabled is true
- Website index_document suffix is "index.html"
- website_endpoint output is not null
- Lifecycle rule transition days is 30
- Lifecycle rule storage_class is "GLACIER"
- CORS allowed_origins includes "https://example.com"
- Logging target_bucket is "test-logging-bucket"

### Scenario: Validation Errors
Purpose: Verify input validation rejects bad inputs

Expect error cases:
- bucket_name = "AB" -> var.bucket_name (too short, uppercase)
- environment = "production" -> var.environment (not in list)
- lifecycle_glacier_days = 0 -> var.lifecycle_glacier_days (not positive)
- cloudfront_oac_access_enabled = true, cloudfront_distribution_arn = null -> var.cloudfront_distribution_arn

## 6. Implementation Checklist

- [ ] A: Scaffold + versions.tf + all 18 variables with validations + locals (common_tags, is_kms) + aws_s3_bucket.this + 4 core outputs
- [ ] B: Encryption config + versioning + public access block (hardcoded) + bucket policy (TLS deny + conditional OAC)
- [ ] C: Website config + lifecycle + CORS + logging (all conditional) + website outputs
- [ ] D: examples/basic/ (4 required inputs + test overrides) + examples/complete/ (all features)
- [ ] E: tests/basic.tftest.hcl + tests/complete.tftest.hcl + tests/validation.tftest.hcl
- [ ] F: terraform-docs README + terraform fmt + terraform validate + trivy scan
```

---

## Part 7: Agent & Skill Architecture

### What to Build

| Component | Type | Purpose | Tools Needed |
|-----------|------|---------|-------------|
| `tf-plan-v2` | Skill (orchestrator) | 4-phase workflow entry point | All |
| `sdd-design` | Agent | Produces design.md from requirements + research | Read, Write, Glob, Grep, MCP tools |
| `tf-test-writer` | Agent | Converts design.md test scenarios to .tftest.hcl | Read, Write, Edit |
| `tf-implement` (updated) | Skill (orchestrator) | TDD implementation from design.md | All |
| `tf-task-executor` (updated) | Agent | Executes implementation checklist items | Read, Write, Edit, Bash, MCP tools |

### Agent Input/Output Contracts

**sdd-design agent**:
- Input: Clarified requirements text + research findings text + constitution path + design template
- Output: `specs/{FEATURE}/design.md` (single file)
- Model: opus (creative, architectural reasoning)

**tf-test-writer agent**:
- Input: `specs/{FEATURE}/design.md` Section 5 (Test Scenarios)
- Output: `tests/*.tftest.hcl` files
- Model: sonnet (pattern translation, not creative reasoning)

**tf-task-executor agent** (per implementation checklist item):
- Input: `specs/{FEATURE}/design.md` + checklist item letter + existing module code
- Output: Modified .tf files
- Model: sonnet (code generation)

### Skills to Keep, Merge, or Remove

| Current Skill | v2 Action | Rationale |
|---------------|-----------|-----------|
| `tf-plan` | **Replace** with `tf-plan-v2` | New 4-phase orchestrator |
| `tf-implement` | **Update** for TDD | Tests written first, run after each phase |
| `tf-domain-taxonomy` | **Keep** | Used in Phase 1 clarification |
| `tf-research-heuristics` | **Keep** | Used in Phase 1 research |
| `tf-architecture-patterns` | **Keep** | Used by design + implementation agents |
| `tf-implementation-patterns` | **Keep** | Used by implementation agent |
| `terraform-test` | **Elevate** | Central to Phase 3 TDD |
| `terraform-style-guide` | **Keep** | Used during implementation |
| `tf-security-baselines` | **Merge** into design agent prompt | Security controls table in design.md |
| `tf-judge-criteria` | **Keep** | Used in Phase 4 validation |
| `tf-spec-writing` | **Remove** | Replaced by design.md template |
| `tf-task-patterns` | **Remove** | Replaced by implementation checklist |
| `tf-checklist-patterns` | **Remove** | No separate checklists in v2 |
| `tf-consistency-rules` | **Remove** | Single artifact, no cross-checking needed |
| `tf-report-template` | **Simplify** | Validation results only |
| `tf-compound-patterns` | **Keep** | Post-implementation learning |

---

## Part 8: Bootstrap Checklist

When starting a new repo with SDD v2, create these files in order:

```
1. .claude/CLAUDE.md                          ← Project context + pointer to constitution
2. AGENTS.md                                  ← Project documentation
3. .foundations/memory/constitution.md         ← Non-negotiable rules (adapt from template)
4. .foundations/templates/design-template.md   ← Copy from Part 1 of this guide
5. .claude/skills/tf-plan-v2/SKILL.md         ← Orchestrator (from Part 4 logic)
6. .claude/agents/sdd-design.md               ← Design agent (references template)
7. .claude/agents/tf-test-writer.md           ← Test writer agent
8. .claude/skills/tf-implement/SKILL.md       ← Updated TDD implementer
9. .github/ISSUE_TEMPLATE/module-request.yml  ← Requirements intake template
10. .foundations/scripts/bash/validate-env.sh  ← Environment validation
```

### Minimum Viable Setup

If you want the fastest possible start, you need only items 1-5. The design agent can be a general-purpose subagent with a detailed prompt (no custom agent file needed). The test writer can be inline in the orchestrator. This gets you running with:

- `CLAUDE.md` pointing to constitution
- Constitution defining the rules
- Design template defining the output format
- One orchestrator skill with 4 phases

Everything else is optimization.
