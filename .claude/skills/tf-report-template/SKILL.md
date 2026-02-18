---
name: tf-report-template
description: Validation results summary template for Phase 4 output. Provides the format for reporting terraform test, validate, fmt, trivy, and security checklist results.
---

# Validation Results Report Template

Phase 4 output format. Report validation results only — no resource tracking, token usage, or workaround logs.

## Report Location

`specs/{FEATURE}/reports/validation_$(date +%Y%m%d-%H%M%S).md`

## Template

```markdown
# Validation Report: {{MODULE_NAME}}

| Field    | Value              |
|----------|--------------------|
| Branch   | {{BRANCH}}         |
| Date     | {{DATE}}           |
| Provider | {{PROVIDER_VERSION}} |

## terraform test

| Test File            | Result |
|----------------------|--------|
| basic.tftest.hcl     | {{PASS/FAIL}} |
| complete.tftest.hcl  | {{PASS/FAIL}} |
| validation.tftest.hcl| {{PASS/FAIL}} |

**Summary**: {{PASSED}}/{{TOTAL}} passed

## terraform validate

**Result**: {{CLEAN / ERRORS}}

{{If errors, list each as a bullet: file:line — message}}

## terraform fmt -check

**Result**: {{FORMATTED / NEEDS FORMATTING}}

{{If unformatted, list files as bullets}}

## trivy config

| Severity | Count |
|----------|-------|
| CRITICAL | {{N}} |
| HIGH     | {{N}} |
| MEDIUM   | {{N}} |
| LOW      | {{N}} |

{{If CRITICAL or HIGH > 0, list each finding: file:line — description}}

## Security Checklist

Controls from design.md Section 4. Each control is pass or fail.

| # | Control            | Result |
|---|--------------------|--------|
| 1 | {{CONTROL_NAME}}   | {{PASS/FAIL}} |
| 2 | {{CONTROL_NAME}}   | {{PASS/FAIL}} |
| ...| ...               | ...    |

## Overall Status

**{{PASS / FAIL}}**

{{If FAIL, list each failing category as a bullet}}
```

## Rules

1. Replace all `{{PLACEHOLDERS}}` — use "N/A" if data is unavailable
2. Verify no `{{` remains before writing the final file
3. Keep the report under 80 lines — tables over prose
4. CRITICAL or HIGH trivy findings force overall FAIL
5. Any terraform test failure forces overall FAIL
6. Any security checklist failure forces overall FAIL

## PASS Criteria

All of the following must be true for overall PASS:

- All `terraform test` files pass
- `terraform validate` is clean
- `terraform fmt -check` reports no changes needed
- trivy reports 0 CRITICAL and 0 HIGH findings
- All security checklist controls pass
