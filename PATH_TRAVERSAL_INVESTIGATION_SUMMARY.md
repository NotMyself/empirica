# Path Traversal Vulnerability Investigation Summary

**Date:** 2026-01-04
**Investigator:** Claude Code (Security Audit)
**Severity:** MEDIUM (CVSS 6.5)
**Status:** Documented, Awaiting Fix

---

## Executive Summary

A comprehensive path traversal vulnerability investigation revealed **3 critical file operation functions** in `simple_session_server.py` and **3 environment variable injection points** in `path_resolver.py` that fail to validate user-provided paths. This allows attackers to:

1. **Read arbitrary files** (`/etc/passwd`, SSH keys, source code)
2. **List arbitrary directories** (enumerate server structure)
3. **Move/overwrite files** anywhere on the filesystem
4. **Create directories** outside workspace
5. **Inject malicious paths** via environment variables (Docker/CI/CD scenarios)

---

## Vulnerability Locations

### Critical (Simple Session Server)

| Function | File | Lines | Exploitability | Impact |
|----------|------|-------|----------------|--------|
| `_read_file()` | `empirica/cli/simple_session_server.py` | 127-147 | **CRITICAL** | Read any file |
| `_list_files()` | `empirica/cli/simple_session_server.py` | 98-125 | **HIGH** | List any directory |
| `_move_file()` | `empirica/cli/simple_session_server.py` | 149-186 | **HIGH** | Move/overwrite any file |

### Medium (Path Configuration)

| Function | File | Lines | Exploitability | Impact |
|----------|------|-------|----------------|--------|
| `get_empirica_root()` | `empirica/config/path_resolver.py` | 118-123 | **MEDIUM** | Env var injection |
| `get_empirica_root()` | `empirica/config/path_resolver.py` | 126-129 | **MEDIUM** | Env var injection |
| `get_session_db_path()` | `empirica/config/path_resolver.py` | 164-167 | **MEDIUM** | Env var injection |

---

## Attack Demonstration

### Attack 1: Read `/etc/passwd`
```bash
curl -X POST http://localhost:8000/sessions/{id}/command \
  -H "Content-Type: application/json" \
  -d '{
    "command": "read_file",
    "args": {"path": "../../../etc/passwd"}
  }'
```

**Result:** Returns system password file contents

---

### Attack 2: Steal SSH Keys
```bash
curl -X POST http://localhost:8000/sessions/{id}/command \
  -H "Content-Type: application/json" \
  -d '{
    "command": "move_file",
    "args": {
      "from": "../../../home/user/.ssh/id_rsa",
      "to": "stolen_key.pem"
    }
  }'
```

**Result:** Private SSH key moved into workspace, exposed to attacker

---

### Attack 3: Overwrite System Files
```bash
curl -X POST http://localhost:8000/sessions/{id}/command \
  -H "Content-Type: application/json" \
  -d '{
    "command": "move_file",
    "args": {
      "from": "malicious.sh",
      "to": "../../../etc/cron.d/backdoor"
    }
  }'
```

**Result:** Cron backdoor installed

---

## Root Cause Analysis

### The Problem

All three vulnerable functions use the same unsafe pattern:

```python
# VULNERABLE CODE
full_path = Path(session["cwd"]) / user_provided_path
```

**Why this is vulnerable:**
- Python's `Path` `/` operator performs simple concatenation
- Does NOT validate the resolved path stays within base directory
- `Path("/workspace") / "../../../etc/passwd"` resolves to `/etc/passwd`

### Expected Behavior

Should validate that resolved path is within workspace:

```python
# SAFE CODE
base_dir = Path(session["cwd"]).resolve()
full_path = (base_dir / user_provided_path).resolve()

# Check if within base directory
try:
    full_path.relative_to(base_dir)  # Raises ValueError if outside
except ValueError:
    raise SecurityError("Path traversal detected")
```

---

## Exploitability Assessment

### Simple Session Server: **CRITICAL (Actively Exploitable)**

| Factor | Assessment |
|--------|------------|
| **Network Access** | ✅ HTTP API exposed on `0.0.0.0:8000` |
| **Authentication** | ❌ None implemented |
| **Attack Complexity** | ✅ Trivial (`../../../`) |
| **Impact** | ✅ Read/write any file |
| **Real-world Risk** | **HIGH** - Designed for remote AI access |

**Verdict:** Immediately exploitable in production deployments.

---

### Path Resolver: **MEDIUM (Requires Environment Control)**

| Factor | Assessment |
|--------|------------|
| **Attack Vector** | Requires setting environment variables |
| **Scenarios** | Docker, CI/CD, shared hosting |
| **Attack Complexity** | Medium (need environment access) |
| **Impact** | Database/config corruption, data exposure |
| **Real-world Risk** | **MEDIUM** - Depends on deployment |

**Verdict:** Exploitable in containerized/multi-tenant environments.

---

## Recommended Fixes

### 1. Create Path Security Utility (Highest Priority)

**File:** `empirica/utils/path_security.py` (new file)

**Functions:**
- `is_safe_path(base_dir: Path, user_path: str) -> bool`
- `safe_join(base_dir: Path, user_path: str) -> Optional[Path]`
- `validate_path_components(user_path: str) -> bool`

**Implementation:** See full code in `/home/user/empirica/docs/security-issues/004-MEDIUM-path-traversal.md`

---

### 2. Fix Simple Session Server

**File:** `empirica/cli/simple_session_server.py`

**Changes Required:**
1. Import: `from empirica.utils.path_security import safe_join`
2. Replace all instances of:
   ```python
   full_path = Path(session["cwd"]) / user_path
   ```
   With:
   ```python
   full_path = safe_join(Path(session["cwd"]), user_path)
   if full_path is None:
       return {"error": "Path traversal detected"}
   ```

**Affected Functions:**
- `_list_files()` (line 101)
- `_read_file()` (line 130)
- `_move_file()` (lines 161-162)

---

### 3. Harden Path Resolver (Defense in Depth)

**File:** `empirica/config/path_resolver.py`

**Changes:**
1. Add validation for environment variables
2. Blacklist dangerous paths (`/`, `/etc`, `/var`, `/sys`)
3. Log suspicious configurations
4. Fail securely when invalid paths detected

---

## Security Recommendations Beyond Fixes

### Immediate Actions

1. **Add Authentication** to Simple Session Server
   - API key/token required for all endpoints
   - Session IDs are not secrets

2. **Restrict Network Binding**
   - Change `0.0.0.0` to `127.0.0.1` (localhost only)
   - Use reverse proxy for external access

3. **Add Audit Logging**
   - Log all file operations with resolved paths
   - Alert on path traversal attempts

### Long-term Improvements

4. **Rate Limiting**
   - Prevent rapid exploitation attempts
   - Limit file operations per session

5. **Principle of Least Privilege**
   - Run server with minimal permissions
   - Use dedicated low-privilege user account
   - Consider read-only root filesystem in Docker

6. **Security Testing**
   - Add unit tests for path traversal
   - Integration tests for all file operations
   - Regular penetration testing

---

## Testing Strategy

### Unit Tests
```python
# tests/security/test_path_traversal.py
def test_path_traversal_blocked():
    assert not is_safe_path(Path("/workspace"), "../../../etc/passwd")
    assert safe_join(Path("/workspace"), "../../../etc/passwd") is None

def test_safe_paths_allowed():
    assert is_safe_path(Path("/workspace"), "docs/file.txt")
    assert safe_join(Path("/workspace"), "docs/file.txt") == Path("/workspace/docs/file.txt")
```

### Integration Tests
```python
# tests/integration/test_session_server_security.py
def test_read_file_blocks_traversal(client, session_id):
    response = client.post(f"/sessions/{session_id}/command", json={
        "command": "read_file",
        "args": {"path": "../../../etc/passwd"}
    })
    assert "error" in response.json()["result"]
    assert "traversal" in response.json()["result"]["error"].lower()
```

---

## Files Created

1. **Security Report (GitHub Issue Format):**
   - `/home/user/empirica/docs/security-issues/004-MEDIUM-path-traversal.md`
   - Complete analysis, proof-of-concept attacks, fixes

2. **Investigation Summary:**
   - `/home/user/empirica/PATH_TRAVERSAL_INVESTIGATION_SUMMARY.md`
   - This file (executive overview)

---

## Next Steps

1. **Review** security report with development team
2. **Prioritize** fix implementation (suggest 2-4 hour sprint)
3. **Implement** `path_security.py` utility module
4. **Update** all vulnerable file operations
5. **Add** comprehensive test coverage
6. **Document** secure coding practices for future development
7. **Audit** rest of codebase for similar patterns

---

## Related Security Issues

- [001-CRITICAL-command-injection.md](./docs/security-issues/001-CRITICAL-command-injection.md)
- [002-HIGH-sql-injection-migrations.md](./docs/security-issues/002-HIGH-sql-injection-migrations.md)
- [003-HIGH-sql-injection-utilities-FALSE-POSITIVE.md](./docs/security-issues/003-HIGH-sql-injection-utilities-FALSE-POSITIVE.md)

**Recommendation:** Conduct comprehensive security audit of all user input handling across entire codebase.

---

## Impact Summary

### Without Fix
- ❌ Arbitrary file read (confidentiality breach)
- ❌ Arbitrary file write/move (integrity breach)
- ❌ System enumeration (reconnaissance)
- ❌ Credential theft
- ❌ Potential privilege escalation

### With Fix
- ✅ All file operations constrained to workspace
- ✅ Path traversal attempts blocked and logged
- ✅ Environment variables validated
- ✅ Defense in depth established
- ✅ Security testing in place

---

**Investigation Status:** ✅ Complete
**Documentation Status:** ✅ Complete
**Fix Status:** ⏳ Pending Implementation
**Estimated Fix Time:** 2-4 hours
