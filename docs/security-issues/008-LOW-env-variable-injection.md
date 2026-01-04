# GitHub Issue: Environment Variable Path Injection

**Title:** [Security] LOW: Environment variables for paths not validated

**Labels:** `security`, `low`, `enhancement`

---

## Summary

Environment variables used for path configuration are not validated, allowing potential path manipulation in containerized environments.

**Severity:** LOW (requires environment control to exploit)
**Affected File:** `empirica/config/path_resolver.py` (lines 118-167)

## Affected Code

```python
if workspace_root := os.getenv('EMPIRICA_WORKSPACE_ROOT'):
    workspace_path = Path(workspace_root).expanduser().resolve()
    # No validation that path is safe
```

## Impact Assessment

| Factor | Rating |
|--------|--------|
| Attack Vector | Local (requires env control) |
| Attack Complexity | Medium |
| Impact | Medium (data exposure/corruption) |

## Recommended Fix

```python
def _validate_workspace_path(path: str) -> Path:
    """Validate workspace path is safe and accessible."""
    resolved = Path(path).expanduser().resolve()

    # Blacklist dangerous system paths
    FORBIDDEN_PREFIXES = ['/etc', '/var', '/usr', '/bin', '/sbin', '/root']
    for prefix in FORBIDDEN_PREFIXES:
        if str(resolved).startswith(prefix):
            raise ValueError(f"Workspace cannot be in system directory: {prefix}")

    return resolved

if workspace_root := os.getenv('EMPIRICA_WORKSPACE_ROOT'):
    workspace_path = _validate_workspace_path(workspace_root)
```

## References

- [CWE-426: Untrusted Search Path](https://cwe.mitre.org/data/definitions/426.html)
