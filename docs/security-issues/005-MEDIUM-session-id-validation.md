# GitHub Issue: Inconsistent Session ID Validation

**Title:** [Security] MEDIUM: Inconsistent Session ID validation across database operations

**Labels:** `security`, `medium`, `bug`

---

## Summary

While UUID validation exists for session_id via `_validate_session_id()` method, it is not consistently called across all database operations. This creates potential for injection attacks or unexpected behavior when invalid session IDs bypass validation.

**Severity:** MEDIUM
**Affected File:** `empirica/data/session_database.py`

## Affected Code

The validation method exists (lines 154-176):

```python
def _validate_session_id(self, session_id: str) -> None:
    """Validate session_id is a valid UUID format"""
    try:
        uuid.UUID(session_id)
    except ValueError:
        raise ValueError(f"Invalid session_id format: {session_id}")
```

However, this is not called consistently across all methods that accept session_id.

## Impact Assessment

| Factor | Rating |
|--------|--------|
| Exploitability | Medium |
| Impact | Low-Medium |
| Likelihood | Low |

## Recommended Fix

1. Create a decorator for session_id validation:

```python
def validate_session_id(func):
    @wraps(func)
    def wrapper(self, session_id: str, *args, **kwargs):
        self._validate_session_id(session_id)
        return func(self, session_id, *args, **kwargs)
    return wrapper
```

2. Apply decorator to all methods accepting session_id

3. Consider using Pydantic models for automatic validation

## References

- [CWE-20: Improper Input Validation](https://cwe.mitre.org/data/definitions/20.html)
