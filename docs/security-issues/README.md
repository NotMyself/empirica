# Security Issues Index

This directory contains detailed security issue reports identified during the codebase analysis.
Each issue is documented in a format ready to be submitted as a GitHub issue.

## Summary

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| 001 | **CRITICAL** | [Command Injection via shell=True](./001-CRITICAL-command-injection.md) | Action Required |
| 002 | **HIGH** | [SQL Injection in Migration Code](./002-HIGH-sql-injection-migrations.md) | Action Required |
| 003 | ~~HIGH~~ | [SQL Injection in utilities.py](./003-HIGH-sql-injection-utilities-FALSE-POSITIVE.md) | **FALSE POSITIVE** |
| 004 | **MEDIUM** | [Path Traversal Vulnerabilities](./004-MEDIUM-path-traversal.md) | Action Required |
| 005 | **MEDIUM** | [Inconsistent Session ID Validation](./005-MEDIUM-session-id-validation.md) | Recommended |
| 006 | **MEDIUM** | [Weak Command Whitelisting](./006-MEDIUM-weak-command-whitelisting.md) | Action Required |
| 007 | **MEDIUM** | [os.system() in Tests](./007-MEDIUM-os-system-tests.md) | Recommended |
| 008 | **LOW** | [Environment Variable Injection](./008-LOW-env-variable-injection.md) | Optional |
| 009 | **LOW** | [JSON Parsing Validation](./009-LOW-json-parsing-validation.md) | Optional |

## Priority Order

### Immediate (Fix within 24-48 hours)
1. **001-CRITICAL-command-injection** - Remote code execution via shell=True

### High Priority (Fix within 1-2 weeks)
2. **002-HIGH-sql-injection-migrations** - SQL injection in migration helpers
3. **004-MEDIUM-path-traversal** - Arbitrary file read/write

### Medium Priority (Fix within 30 days)
4. **005-MEDIUM-session-id-validation** - Input validation consistency
5. **006-MEDIUM-weak-command-whitelisting** - Improve command filtering
6. **007-MEDIUM-os-system-tests** - Clean up test code

### Low Priority (Address in next release)
7. **008-LOW-env-variable-injection** - Environment variable validation
8. **009-LOW-json-parsing-validation** - Schema validation

## How to Create GitHub Issues

Since the `gh` CLI is not available, copy the content of each markdown file and:

1. Go to https://github.com/NotMyself/empirica/issues/new
2. Copy the **Title** from the markdown file
3. Add the **Labels** specified
4. Copy the **Body** section content
5. Submit the issue

## Notes

- Issue **003** was identified as a false positive after investigation
- The total confirmed vulnerabilities: 8 (1 CRITICAL, 1 HIGH, 4 MEDIUM, 2 LOW)
- All issues in `simple_session_server.py` are related and can be addressed together
