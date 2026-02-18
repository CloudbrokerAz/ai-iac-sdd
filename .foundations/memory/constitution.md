# Terraform AI-Assisted Module Development Constitution

**Organization**: [Your Organization Name]
**Version**: 4.0.0
**Effective Date**: February 2026
**Purpose**: Governing principles for AI-assisted development of enterprise-ready Terraform modules
**Workflow**: SDD v2 (4-phase)

---

## 1. Foundational Principles

### 1.1 Module-First Architecture

**Principle**: This repository develops well-structured, reusable Terraform modules using raw resources that follow HashiCorp and organizational best practices.

**Rationale**: Enterprise-ready modules encapsulate infrastructure patterns with secure defaults, consistent interfaces, and thorough testing — enabling consumers to provision infrastructure safely without deep resource-level knowledge.

**Implementation**:

- Modules MUST be authored using native Terraform resources and data sources from official providers
- Module design MUST follow the [standard module structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure): root module, `variables.tf`, `outputs.tf`, `examples/`, `tests/`, and optional `modules/` for submodules
- Research AWS documentation and provider docs to understand resource behavior before writing module code
- Study well-regarded public registry modules for design patterns and interface conventions (e.g., `terraform-aws-modules/`)
- Modules MUST expose configurable inputs with secure defaults — consumers should get a secure baseline without overriding anything
- Conditional resource creation MUST be supported via `create_*` boolean variables (e.g., `create_vpc = true`)
- Module versioning MUST follow semantic versioning (major.minor.patch) with clear CHANGELOG entries

### 1.2 Security-First Automation

**Principle**: Generated module code MUST assume zero trust and implement security controls by default.

**Rationale**: Modules define infrastructure patterns used across the organization. Security gaps in modules propagate to every consumer.

**Implementation**:

- Module resources MUST enable encryption, logging, and access controls by default
- Security-sensitive inputs MUST have secure defaults (e.g., `public_access = false`, `encryption_enabled = true`)
- Modules MUST NOT require consumers to pass credentials — providers are configured by the consumer, not the module
- Security toggles MUST be exposed as variables so consumers can adjust within guardrails, but defaults MUST be secure
- You MUST include security context in code comments (e.g., "Encryption enabled by default per organizational security standards")

---

## 2. HCP Terraform Prerequisites

### 2.1 Required Configuration Details

**Standard**: HCP Terraform configuration details MUST be determined from the current remote git repository or provided by user before any Terraform operations (testing, publishing).

**Prerequisites**:

- HCP Terraform Organization Name
- HCP Terraform Project Name
- HCP Terraform Workspace Name for module testing
- Pre-commit hooks configured (run `pre-commit install`)

**Rules**:

- Configuration details MUST be automatically detected from the current git repository using Terraform MCP server tools
- If multiple options exist or automatic detection returns ambiguous results, you MUST prompt the user to select/provide configuration details
- You MUST validate that organization, project, and workspace details are available before invoking any Terraform MCP server tools
- All HCP Terraform API calls for workspace or registry operations MUST use the validated organization, project, and workspace context
- Organization and project context MUST be validated before registry publishing operations
- User-provided configuration details MUST be validated against available HCP Terraform resources before proceeding
- Missing prerequisites MUST be surfaced to the user with clear instructions and options

---

## 3. Code Generation Standards

### 3.1 Git Branch Strategy

**Branch Protection Rules**:

- Direct commits to `main` branch are PROHIBITED
- All changes MUST be made via feature branches off the current branch
- Pull requests with human review REQUIRED for all merges

**Rules**:

- Feature branches contain module development work
- Each feature branch MAY have an associated HCP Terraform workspace for example deployment testing
- Test values for examples MUST be managed through `examples/*/terraform.tfvars` or `examples/*/sandbox.auto.tfvars`, NOT hardcoded in module code

### 3.2 File Organization

**Standard**: Terraform module files MUST follow the standard HashiCorp module structure.

**Root Module**:

```
/
├── main.tf              # Primary resource definitions
├── variables.tf         # Input variable declarations
├── outputs.tf           # Output value declarations
├── locals.tf            # Local value computations
├── versions.tf          # Terraform and provider version constraints (required_version + required_providers)
├── data.tf              # Data source definitions (optional, if needed)
├── README.md            # Module documentation (auto-generated header)
├── CHANGELOG.md         # Version history
├── examples/
│   ├── basic/           # Minimal usage example
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── providers.tf # Provider configuration lives HERE (in examples)
│   │   └── README.md
│   └── complete/        # Full-featured usage example
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── providers.tf
│       └── README.md
├── modules/             # Submodules (optional)
│   └── <submodule>/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── tests/               # Terraform test files
    ├── basic.tftest.hcl
    └── complete.tftest.hcl
```

**Rules**:

- Root module MUST NOT contain provider configuration blocks — modules inherit providers from consumers
- Provider requirements (`required_providers`) and version constraints (`required_version`) MUST be declared in the root module's `versions.tf`
- Provider configuration (region, credentials, etc.) belongs ONLY in `examples/` directories
- `examples/basic/` MUST demonstrate minimum viable usage of the module
- `examples/complete/` MUST demonstrate all features and optional configurations
- Test files (`.tftest.hcl`) MUST live in the `tests/` directory

**Prohibitions**:

- You MUST NOT create monolithic single-file configurations exceeding 500 lines
- You MUST NOT intermingle resource types without logical grouping
- You MUST NOT use default values for security-sensitive variables
- You MUST NOT include provider configuration in the root module

### 3.3 Naming Conventions

**Standard**: Names MUST be predictable, consistent, and follow HashiCorp naming standards.

**Format**:

- Resources: Use `this` as the primary resource name when there is a single instance (e.g., `aws_vpc.this`). Use descriptive names for multiple resources of the same type (e.g., `aws_subnet.public`, `aws_subnet.private`)
- Variables: `snake_case` with descriptive names
- Outputs: `snake_case`, mirroring the resource attribute name where possible
- Submodules: `<purpose>` (e.g., `modules/encryption`, `modules/logging`)

**Rules**:

- You MUST follow HashiCorp naming standards (https://developer.hashicorp.com/terraform/plugin/best-practices/naming)
- You MUST infer naming from specification or request clarification
- Names MUST NOT include sensitive information (account IDs, secrets, PII)
- Names MUST be idempotent and not include timestamps or random values unless functionally required
- Prefer `for_each` over `count` for resource iteration — it produces stable resource addresses

### 3.4 Variable Management

**Standard**: Variables MUST be explicitly declared with comprehensive metadata.

**Template**:

```hcl
variable "name" {
  description = "Name to use for all resources created by this module"
  type        = string

  validation {
    condition     = length(var.name) > 0 && length(var.name) <= 64
    error_message = "Name must be between 1 and 64 characters."
  }
}

variable "create" {
  description = "Controls whether resources are created by this module"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Map of tags to apply to all taggable resources"
  type        = map(string)
  default     = {}
}
```

**Rules**:

- ALL variables MUST include `description` explaining purpose and valid values
- Variables MUST include `type` constraints (never use implicit `any`)
- Security-sensitive variables MUST be marked as `sensitive = true`
- Variables SHOULD include `validation` blocks for business logic constraints
- Module variables MUST have sensible defaults where possible — required variables should be the minimum needed for a working deployment
- Boolean feature toggles MUST follow `create_<resource>` or `enable_<feature>` naming

### 3.5 Module Authoring Patterns

**Standard**: Module code MUST follow established resource authoring patterns.

**Example**:

```hcl
resource "aws_vpc" "this" {
  count = var.create ? 1 : 0

  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true  # Secure default
  enable_dns_support   = true  # Secure default

  tags = merge(
    var.tags,
    { Name = var.name }
  )
}

resource "aws_flow_log" "this" {
  count = var.create && var.enable_flow_logs ? 1 : 0

  vpc_id          = aws_vpc.this[0].id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_log[0].arn
  log_destination = aws_cloudwatch_log_group.flow_log[0].arn

  tags = var.tags
}
```

**Rules**:

- Use conditional creation (`count` or `for_each` with enable variables) to allow consumers to selectively enable features
- Use `merge()` for tags to combine module defaults with consumer-provided tags
- Use `try()` and `lookup()` for safely accessing optional nested values
- Dynamic blocks MUST be used for repeatable nested configurations
- Outputs MUST use `try()` to handle conditional resources gracefully:
  ```hcl
  output "vpc_id" {
    description = "The ID of the VPC"
    value       = try(aws_vpc.this[0].id, null)
  }
  ```
- Module MUST NOT hardcode values that consumers should control — expose as variables with defaults

---

## 4. Security and Compliance

### 4.1 Secrets and Credentials

**Policy**: Modules MUST NOT manage or require credentials directly, and MUST support secure secrets handling patterns.

**Rules**:

- Modules inherit provider configuration from consumers — NEVER include provider blocks in the root module
- Examples MAY include provider configuration for testing purposes
- For testing with HCP Terraform, credential configuration is handled through workspace variable sets
- You MUST NOT generate `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` or similar static credential variables in module code
- Variables accepting secrets MUST be marked as `sensitive = true`
- Outputs exposing secrets MUST be marked as `sensitive = true`
- Modules SHOULD use ephemeral resources for handling sensitive values where appropriate (see https://developer.hashicorp.com/terraform/language/manage-sensitive-data/ephemeral)
- Modules SHOULD provide integration points for secrets managers (e.g., accept ARNs for KMS keys, Secrets Manager secrets) rather than managing secrets directly

### 4.2 Security Best Practices

**Policy**: Module code must embed security best practices as defaults.

**Implementation**:

- Resources MUST enable encryption, restrict public access, and enable logging by default
- Security features MUST be enabled by default and exposed as toggleable variables:
  ```hcl
  variable "enable_encryption" {
    description = "Enable encryption at rest for all supported resources"
    type        = bool
    default     = true  # Secure by default
  }
  ```
- Security patterns MUST be implemented proactively, not reactively
- Non-compliant patterns MUST be avoided even if technically functional

### 4.3 Least Privilege by Default

**Policy**: Module resources MUST implement principle of least privilege.

The following cloud-specific rules supplement sections 1.2 and 4.2.

**AWS-Specific Rules**:

- Security Groups MUST deny all traffic by default, only allowing specific required ports and sources
- S3 buckets MUST block public access unless explicitly required for public hosting
- When creating aws_s3_bucket ensure force_destroy is configurable (default `false` for safety, examples MAY set `true` for testing)
- IAM roles MUST use specific resource ARNs instead of wildcards (`*`) when possible
- RDS instances MUST NOT be publicly accessible unless explicitly justified
- EC2 instances MUST use IAM instance profiles instead of embedded credentials
- Lambda functions MUST use least privilege execution roles with specific service permissions
- You must use the aws-security-advisor agent to research and review the required AWS resources

**GCP/Azure-Specific Rules**: Future placeholder -- when GCP or Azure modules are developed, add provider-specific least-privilege rules here following the same pattern as the AWS rules above.

---

## 5. Workspace and Environment Management

### 5.1 HCP Terraform Workspace Management

**Standard**: HCP Terraform workspaces are used for testing module examples against real infrastructure.

**Testing Workspace Rules**:

- A sandbox workspace MAY be used for deploying examples to validate module behavior against real cloud APIs
- This workspace pattern: `sandbox_<module-name>_<example-name>`
- Testing workspaces are for temporary validation — not long-lived environments

**Ephemeral Workspace Rules**:

- You MUST create ephemeral HCP Terraform workspaces ONLY for testing module examples
- Ephemeral workspaces MUST be connected to the current `feature/*` branch of the remote Git repository and will use Terraform CLI
- Before running terraform init you must configure credentials, TFE_TOKEN is already set as an environment variable. See example.

```
  mkdir -p ~/.terraform.d && cat > ~/.terraform.d/credentials.tfrc.json << EOF
    {
      "credentials": {
        "app.terraform.io": {
          "token": "$TFE_TOKEN"
        }
      }
    }
    EOF
```

- The current feature branch MUST be committed and pushed to the remote Git repository BEFORE creating the ephemeral workspace
- Ensure the terraform variables are validated by the user before proceeding, including region values and other required inputs
- Ephemeral workspaces MUST be used to validate that examples deploy successfully
- Use Terraform CLI for all runs (`terraform init/plan/apply`)
- Ephemeral workspaces MUST be deleted after successful testing to avoid unnecessary costs

### 5.2 Release and Publishing

**Standard**: Module releases follow semantic versioning and are published to the private registry.

**Versioning Rules**:

- Major version: Breaking changes to the module interface (removed/renamed variables, changed output types)
- Minor version: New features, new optional variables, new outputs
- Patch version: Bug fixes, documentation updates, security patches
- Git tags MUST use `v` prefix (e.g., `v1.0.0`, `v1.1.0`)

**Publishing Workflow**:

1. All tests pass (`terraform test`)
2. Examples deploy and destroy cleanly
3. Documentation is up-to-date (`terraform-docs`)
4. CHANGELOG.md updated
5. Git tag created
6. Module published to private registry (via CI/CD or manual)

---

## 6. Code Quality and Maintainability

### 6.1 Documentation Requirements

**Standard**: Module code MUST be self-documenting and include external documentation with automated generation.

**Requirements**:

- Every module MUST include `README.md` auto-generated via `terraform-docs` pre-commit hooks
- Complex logic MUST include inline comments explaining rationale
- Resource configurations MUST be justified in comments where non-obvious
- All variables and outputs MUST have proper descriptions for `terraform-docs` automatic documentation generation

### 6.2 Code Style

**Standard**: Generated code MUST follow HashiCorp Style Guide.

**Rules**:

- Alphabetize arguments within blocks for consistency
- Use consistent argument ordering: required args first, optional args second, meta-args last

### 6.3 Testing and Validation

**Standard**: Module code MUST be tested using `terraform test` with `.tftest.hcl` files.

**Testing Requirements**:

- Every module MUST have test files in the `tests/` directory
- Tests MUST cover:
  - Basic creation with defaults (validates secure defaults work)
  - Complete creation with all options (validates full feature set)
  - Conditional creation disabled (validates `create = false` works)
  - Input validation (validates variable constraints)
- Test files MUST use the `terraform test` framework (`.tftest.hcl` format)
- Tests SHOULD use mocks for provider calls where possible to enable fast iteration
- Integration tests (deploying real resources) SHOULD run in CI against a sandbox workspace

**Validation Steps**:

- `terraform fmt -check` — formatting compliance
- `terraform validate` — syntax and configuration validity
- `terraform test` — unit and integration test execution
- `tflint` — linting for best practices
- `trivy` — security scanning for misconfigurations
- Pre-commit hooks MUST enforce these checks before commits

### 6.4 Version Control

**Standard**: Generated code MUST be version controlled with meaningful commits.

- You SHOULD suggest atomic commits per logical change
- You MUST NOT commit secrets, credentials, or sensitive data

---

## 7. Operational Excellence

### 7.1 State Management

**Standard**: Module code MUST NOT manage state — state is the consumer's responsibility.

**Rules**:

- Root modules MUST NOT include backend or cloud configuration blocks
- Examples MAY include backend configuration for testing purposes
- State MUST never be committed to version control

### 7.2 Dependency Management

**Standard**: Provider and Terraform versions MUST be explicitly constrained.

**Template**:

```hcl
terraform {
  required_version = ">= 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

**Rules**:

- Modules MUST declare `required_version` with a minimum Terraform version
- Provider versions MUST use `>=` constraints in modules (not `~>`) to maximize consumer compatibility
- Modules MUST NOT over-constrain provider versions — use the minimum version that supports required features
- You MUST NOT use `latest` or unconstrained versions

### 7.3 Cost Optimization

**Standard**: Module resources SHOULD support cost-conscious configuration.

**Rules**:

- Modules SHOULD expose variables for instance sizing, storage configuration, and scaling parameters
- Default values SHOULD be cost-effective where possible (consumers can override for production)
- Modules SHOULD document cost-impacting configuration choices
- Examples SHOULD use minimal resource sizes for testing

### 7.4 Monitoring and Observability

**Standard**: Module resources SHOULD be observable by default.

**Rules**:

- Modules SHOULD enable CloudWatch/monitoring by default where applicable
- Tags MUST include monitoring metadata (`Name`, `ManagedBy = "terraform"`)
- Modules SHOULD output critical resource identifiers for integration with monitoring systems
- Logging resources (CloudWatch Log Groups, S3 access logs) SHOULD be created by default with opt-out variables

---

## 8. AI Agent Behavior and Constraints

### 8.1 Prerequisites Validation

**Constraint**: You MUST validate prerequisites before any operations.

**Requirements**:

- You MUST NOT proceed with Terraform operations without complete prerequisites
- Missing configuration details MUST be surfaced to the user with clear instructions
- All Terraform MCP server tool calls MUST use the validated configuration values

**Mandatory Prerequisites**:

- HCP Terraform Organization Name (for publishing/testing)
- HCP Terraform Project Name
- Pre-commit hooks installed

### 8.2 Scope Boundaries

**Constraint**: You MUST operate within defined module development patterns.

**In Scope**:

- Writing Terraform resources, data sources, and module code using official providers
- Researching AWS documentation and provider docs for resource behavior and best practices
- Studying public registry modules for design patterns and conventions
- Creating comprehensive test files (`.tftest.hcl`) for the module
- Writing examples that demonstrate module usage
- Generating documentation and README files
- Creating and maintaining `specs/{FEATURE}/design.md` design documents
- Suggesting CI/CD workflows for testing and publishing
- Explaining Terraform concepts to less experienced users

**Out of Scope**:

- Deploying modules to production environments (consumer responsibility)
- Managing consumer workspace configuration
- Bypassing policy controls or suggesting workarounds
- Workspace RBAC configuration (security team responsibility)

### 8.3 Error Handling and Transparency

**Standard**: You MUST acknowledge limitations and uncertainties.

**Rules**:

- Research AWS/provider docs before writing resources
- Explain tradeoffs when multiple patterns exist
- Request clarification for ambiguous requirements in the design document
- Document unsupported features and warn about policy violations

### 8.4 Learning and Adaptation

**Standard**: You MUST learn from organizational patterns and feedback.

**Implementation**:

- You SHOULD reference successful prior module implementations as patterns
- You MUST respect organizational customizations to this constitution
- You SHOULD incorporate policy feedback to avoid repeated violations
- Before executing any operations, you MUST validate that required environment variables are set using the `validate-env.sh` script.

  ```bash
  .foundations/scripts/bash/validate-env.sh
  ```

---

## 9. Governance and Evolution

### 9.1 Constitution Updates

**Process**: This constitution evolves with organizational needs.

**Update Authority**:

- Platform team maintains constitution in version control
- Major changes require review by security and governance teams
- Module developers MAY propose amendments via pull request
- Constitution version MUST be referenced in AI agent prompts

### 9.2 Exception Process

**Policy**: Deviations require explicit approval and documentation.

**Process**:

1. Document specific requirement driving exception
2. Propose alternative approach with risk assessment
3. Obtain platform team approval
4. Document exception in code and centralized exceptions register
5. Review exception during next policy update cycle

### 9.3 Audit and Compliance

**Standard**: AI-generated module code is subject to same audits as human-authored code.

**Requirements**:

- All generated code MUST pass through policy enforcement
- Periodic audits verify constitution compliance
- Non-compliant patterns trigger constitution updates or module improvements
- Metrics track module quality, test coverage, and security posture

### 9.4 Feedback Loop

**Standard**: Continuous improvement through systematic feedback.

**Mechanisms**:

- Module consumers provide feedback on interface usability
- Security reviews inform module design improvements
- AI agent error patterns drive documentation enhancements
- Test coverage metrics guide development priorities

---

## 10. Implementation Checklist

### For Module Developers Using AI Agents:

The implementation checklist for each feature is maintained in **`specs/{FEATURE}/design.md` Section 6 (Implementation Checklist)**. That checklist is the authoritative source of implementation tasks and is generated during the Design phase.

**General workflow:**

- [ ] Clone this module template repository
- [ ] Review this constitution with your team
- [ ] Run the SDD v2 workflow: Understand, Design, Build + Test, Validate
- [ ] Research existing patterns from AWS docs and public registry
- [ ] Configure IDE with AI assistant (Copilot, Claude Code, etc.)
- [ ] Generate module code following this constitution and the design document
- [ ] Write tests in `tests/` using `.tftest.hcl` format (TDD: tests before code)
- [ ] Create `examples/basic/` and `examples/complete/` usage examples
- [ ] Run `pre-commit install` to enable hooks, then commit and push code
- [ ] Run `terraform test` to validate module behavior
- [ ] Deploy examples to sandbox workspace for integration testing
- [ ] Create PR for human review
- [ ] Tag release and publish to private registry

### For Platform Teams:

- [ ] Publish this constitution to organization knowledge base
- [ ] Create starter module templates embodying these principles
- [ ] Configure CI/CD pipelines for testing and publishing
- [ ] Configure workspace-level security policies and controls
- [ ] Establish module publishing workflow to private registry
- [ ] Create variable sets for common testing config
- [ ] Monitor module quality and test coverage
- [ ] Iterate on templates based on developer feedback
- [ ] Verify file structure follows the standard module structure defined in Section 3.2

---

## 11. Workflow Governance

### 11.1 SDD v2 Mandatory Phases

**Rule**: The SDD v2 workflow consists of 4 mandatory phases. All phases MUST be completed in order.

**Phase 1: Understand**

- Environment validation (`.foundations/scripts/bash/validate-env.sh`)
- Requirements intake (GitHub issue as audit trail)
- Clarification of ambiguous requirements
- MCP research (provider docs, public registry patterns, AWS documentation)

**Phase 2: Design**

- Produce a single `specs/{FEATURE}/design.md` artifact that consolidates:
  - Module interface (variables, outputs, resource mapping)
  - Architecture and resource dependency planning
  - Security review findings and mitigations
  - Implementation checklist (replaces standalone task tracking)
- Security review via `aws-security-advisor` MUST be incorporated into the design
- The design document is the single source of truth for implementation

**Phase 3: Build + Test**

- TDD approach: write tests BEFORE implementation code
- `terraform test` MUST pass after each logical implementation phase
- Implementation follows the checklist in `specs/{FEATURE}/design.md` Section 6
- Tests in `tests/` using `.tftest.hcl` format
- Examples in `examples/basic/` and `examples/complete/`

**Phase 4: Validate**

- Run validation checks in parallel:
  - `terraform test` — full test suite execution
  - `terraform validate` — configuration validity
  - `terraform fmt -check` — formatting compliance
  - `trivy` — security scanning for misconfigurations
  - `terraform-docs` — documentation generation and verification
- All checks MUST pass before the feature is considered complete

**Rationale**: The 4-phase workflow reduces overhead while maintaining rigor. The single design document eliminates artifact fragmentation and keeps all design decisions co-located.

### 11.2 Optional Steps Within Phases

The following steps within phases MAY be skipped with documented justification:

- Clarification in Phase 1 (if requirements have zero ambiguity)
- MCP research in Phase 1 (if all resources and patterns are well-known)
- Security review in Phase 2 (for trivial changes with no security impact)
- Compound learning after Phase 4 (for hotfix-only changes)

### 11.3 Quality Gates Between Phases

- **Understand -> Design**: Requirements are clear; no unresolved `[NEEDS CLARIFICATION]` markers
- **Design -> Build + Test**: No unresolved CRITICAL findings from security review in design document; design document approved
- **Build + Test -> Validate**: `terraform validate` passes, all implementation checklist items in `specs/{FEATURE}/design.md` Section 6 marked complete
- **Validate -> Complete**: All parallel validation checks pass; examples deploy/destroy cleanly

### 11.4 Approval Gates

- **Issue-driven workflow**: MUST pause between Design and Build + Test phases for human review of the design document
- Gate signal: "approved" or "proceed" comment on the GitHub issue
- Autonomous mode: MAY skip approval if `agent_autonomy` is "Fully Autonomous" AND no CRITICAL findings

---

## 12. Compound Learning

### 12.1 Capture Requirements

**Rule**: Every successful Build + Test completion SHOULD trigger compound learning (best-effort; failures MUST NOT block workflows).

**Mandatory captures**:

- Pattern extraction -> `.foundations/memory/patterns/`
- Pitfall recording -> `.foundations/memory/pitfalls/`
- AGENTS.md updates -> relevant directory AGENTS.md files

**Conditional captures**:

- Constitution review suggestions (if review flagged constitution issues)
- Template improvement suggestions (if significant template deviations)
- Design document retrospective notes (if `specs/{FEATURE}/design.md` required significant mid-build revisions)

### 12.2 Failure Learning

**Rule**: Failed runs MUST capture pitfalls via `compound-pitfall-recorder` with `failure: true` metadata.

**Rationale**: Failures are the most valuable learning opportunities. Recording what went wrong prevents repeating mistakes.

### 12.3 Auto-Commit

Compound learning artifacts MUST be auto-committed: `compound: capture learnings from <feature-name>`

### 12.4 Best-Effort

Compound agent failures SHOULD be logged but MUST NOT block the overall workflow. Compound learning is best-effort.

---

## 13. Naming Conventions

### 13.1 Skill Naming

- `tf-` prefix: Terraform domain knowledge (e.g., `tf-spec-writing`, `tf-security-baselines`)
- `terraform-` prefix: HashiCorp official patterns (e.g., `terraform-style-guide`, `terraform-test`)
- No other prefixes for skills

### 13.2 Agent Naming

- `sdd-` prefix: SDD pipeline agents (e.g., `sdd-design`, `sdd-research`)
- `tf-` prefix: Implementation agents (e.g., `tf-task-executor`, `tf-module-tester`, `tf-test-writer`)
- `compound-` prefix: Learning phase agents (e.g., `compound-pattern-extractor`)
- Hyphenated names, not dotted (e.g., `sdd-design` not `sdd.foundations`)

### 13.3 File Naming

- Agent files: `.claude/agents/<agent-name>.md`
- Skill dirs: `.claude/skills/<skill-name>/SKILL.md`
- Memory files: `.foundations/memory/<category>/<descriptive-name>.md`
- Design documents: `specs/{FEATURE}/design.md`

---

## 14. References and Resources

### Internal Resources

- Private Module Registry: `app.terraform.io/<org-name>/modules`
- Policy Repository: `<policy-repo-url>`
- Platform Team Contact: `<platform-team-contact>`

### External Resources

- [Terraform Module Development](https://developer.hashicorp.com/terraform/language/modules/develop)
- [Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- [Terraform Best Practices](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices)
- [HashiCorp Style Guide](https://developer.hashicorp.com/terraform/language/style)
- [Terraform Test Framework](https://developer.hashicorp.com/terraform/language/tests)
- [GitHub Spec-Kit](https://github.com/github/spec-kit)
- [AWS Terraform Provider Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/)
- [Azure Terraform Best Practices](https://docs.microsoft.com/en-us/azure/developer/terraform/best-practices)
- [Google Cloud Terraform Best Practices](https://cloud.google.com/docs/terraform/best-practices-for-terraform)
