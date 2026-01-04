# GitHub Issue: JSON Parsing Without Schema Validation

**Title:** [Security] LOW: JSON inputs lack schema validation

**Labels:** `security`, `low`, `enhancement`

---

## Summary

Many `json.loads()` calls do not validate the structure of parsed JSON before use, potentially leading to unexpected behavior with malformed or malicious data.

**Severity:** LOW
**Affected:** Multiple files (30+ instances)

## Example

```python
data = json.loads(user_input)
# Assumes data has specific structure without validation
session_id = data['session_id']  # Could KeyError or be wrong type
```

## Impact Assessment

- **Exploitability:** Low
- **Impact:** Application errors, potential DoS via malformed input

## Recommended Fix

Use Pydantic models for automatic validation:

```python
from pydantic import BaseModel, validator

class SessionCreateInput(BaseModel):
    ai_id: str
    session_type: str = "development"

    @validator('ai_id')
    def validate_ai_id(cls, v):
        if not v or len(v) > 100:
            raise ValueError('Invalid ai_id')
        return v

# Usage
try:
    validated = SessionCreateInput(**json.loads(user_input))
except ValidationError as e:
    return {"error": str(e)}
```

## References

- [CWE-20: Improper Input Validation](https://cwe.mitre.org/data/definitions/20.html)
