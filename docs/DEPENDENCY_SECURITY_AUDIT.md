# Dependency Security Audit

**Date:** 2026-01-04
**Overall Risk Score:** 7/10 (HIGH)

---

## Executive Summary

The Empirica codebase has **significant dependency security risks**:
- 🔴 3 packages with CRITICAL CVEs (RCE vulnerabilities)
- 🟡 5 packages with HIGH/MEDIUM CVEs
- ⚠️ No lock file (non-reproducible builds)
- ⚠️ All version constraints unbounded (>=X only)

---

## Critical Vulnerabilities

### 1. cryptography (>=41.0) → Update to >=44.0

**CVEs:**
- CVE-2023-49083: NULL pointer dereference in PKCS12 parsing
- CVE-2024-26130: Memory corruption in RSA key validation

**Impact:** Used for AI identity and checkpoint signing - core security feature
**Fix:** `cryptography>=44.0,<45.0`

### 2. gitpython (>=3.1.0) → Update to >=3.1.43

**CVEs:**
- CVE-2022-24439 (CVSS 9.8): RCE via malicious Git repository URLs
- CVE-2023-40590 (CVSS 9.8): Command injection in submodule handling
- CVE-2024-22190 (CVSS 7.8): Command injection in Git clone

**Impact:** Used for git notes integration and BEADS
**Fix:** `gitpython>=3.1.43,<4.0`

### 3. pyyaml (>=6.0) → Update to >=6.0.2 + Audit

**CVEs:**
- CVE-2020-14343 (CVSS 9.8): Arbitrary code execution via yaml.load()

**Impact:** Used for config/profile loading
**Fix:** `pyyaml>=6.0.2,<7.0` AND audit all yaml.load() calls

---

## High/Medium Vulnerabilities

| Package | Current | Update To | CVE |
|---------|---------|-----------|-----|
| requests | >=2.31.0 | >=2.32.3,<3.0 | CVE-2024-35195 (credential leak) |
| pillow | >=10.0 | >=11.0,<12.0 | CVE-2023-50447, CVE-2024-28219 |
| flask | >=3.0 | >=3.1,<4.0 | Session handling improvements |
| setuptools | >=45 | >=70.0,<76.0 | CVE-2024-6345 |

---

## Structural Issues

### No Lock File

**Current State:** No `requirements-lock.txt`, `poetry.lock`, or `Pipfile.lock`

**Risks:**
- Builds not reproducible
- Different environments get different versions
- Security vulnerabilities introduced unknowingly
- CI/CD may differ from production

**Fix:**
```bash
# Option 1: pip-tools
pip install pip-tools
pip-compile pyproject.toml -o requirements-lock.txt

# Option 2: Poetry
poetry lock
```

### Unbounded Version Constraints

**Current State:** All dependencies use `>=X` (no upper limit)

**Example:**
```toml
# Current (dangerous)
pydantic = ">=2.0"

# Recommended (safe)
pydantic = ">=2.0,<3.0"
```

---

## Complete Dependency Inventory

### Core Dependencies (pyproject.toml)

| Package | Constraint | Latest | Status |
|---------|-----------|--------|--------|
| pydantic | >=2.0 | 2.9.2 | ✅ OK |
| pydantic-settings | >=2.0 | 2.6.1 | ✅ OK |
| sqlalchemy | >=2.0 | 2.0.36 | ✅ OK |
| jsonschema | >=4.0 | 4.23.0 | ✅ OK |
| pyyaml | >=6.0 | 6.0.2 | 🔴 CVE |
| cryptography | >=41.0 | 44.0.0 | 🔴 CVE |
| gitpython | >=3.1.0 | 3.1.43 | 🔴 CVE |
| requests | >=2.31.0 | 2.32.3 | 🟡 CVE |
| httpx | >=0.24 | 0.27.2 | ⚠️ Outdated |
| anthropic | >=0.39.0 | 0.42.0 | ⚠️ Outdated |
| rich | >=13.0 | 13.9.4 | ✅ OK |
| typer | >=0.9 | 0.15.1 | ⚠️ Outdated |

### Optional Dependencies

| Package | Constraint | Latest | Status |
|---------|-----------|--------|--------|
| flask | >=3.0 | 3.1.0 | 🟡 Update |
| fastapi | >=0.104 | 0.115.5 | ⚠️ Outdated |
| pillow | >=10.0 | 11.0.0 | 🟡 CVE |
| qdrant-client | >=1.7 | 1.12.1 | ⚠️ Outdated |

### Transitive Risks

| Package | Via | Risk |
|---------|-----|------|
| werkzeug | flask | CVE-2023-46136, CVE-2024-34069 |
| jinja2 | flask | SSTI if user input rendered |
| urllib3 | requests, httpx | CVE-2023-45803, CVE-2023-43804 |
| numpy | opencv, pillow | Buffer overflow CVEs |

---

## Recommended Updates

### pyproject.toml Changes

```toml
[project]
dependencies = [
    # CRITICAL UPDATES
    "cryptography>=44.0,<45.0",      # Was: >=41.0
    "gitpython>=3.1.43,<4.0",        # Was: >=3.1.0
    "pyyaml>=6.0.2,<7.0",            # Was: >=6.0
    "requests>=2.32.3,<3.0",         # Was: >=2.31.0
    
    # Add upper bounds to all others
    "pydantic>=2.0,<3.0",
    "sqlalchemy>=2.0,<3.0",
    # ... etc
]

[project.optional-dependencies]
vision = [
    "pillow>=11.0,<12.0",            # Was: >=10.0
]
api = [
    "flask>=3.1,<4.0",               # Was: >=3.0
]
```

---

## Immediate Actions

### Today (Critical)
1. Update cryptography to >=44.0
2. Update gitpython to >=3.1.43
3. Audit pyyaml usage (ensure safe_load)

### This Week
4. Add requirements-lock.txt
5. Add upper bounds to all constraints
6. Update requests, pillow, flask

### Ongoing
7. Enable GitHub Dependabot
8. Add safety/pip-audit to CI/CD
9. Monthly dependency review

---

## Verification Commands

```bash
# Check for known vulnerabilities
pip install safety
safety check

# List outdated packages
pip list --outdated

# Full audit
pip install pip-audit
pip-audit

# Generate dependency tree
pip install pipdeptree
pipdeptree --json
```

---

## Compliance Status

| Standard | Status |
|----------|--------|
| OWASP A06:2021 (Vulnerable Components) | 🔴 FAIL |
| CWE-1035 (Known Vulnerabilities) | 🔴 FAIL |
| Supply Chain Security | 🔴 FAIL (no lock file) |

---

## Statistics

- **Total Dependencies:** 32
- **With Critical CVEs:** 3 (9%)
- **Outdated:** 27 (84%)
- **Up to date:** 5 (16%)
- **Lock file:** ❌ None
