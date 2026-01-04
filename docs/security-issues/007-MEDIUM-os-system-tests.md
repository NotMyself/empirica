# GitHub Issue: os.system() Usage in Tests

**Title:** [Security] MEDIUM: Use of deprecated os.system() in test code

**Labels:** `security`, `medium`, `testing`

---

## Summary

Test code uses the deprecated `os.system()` function which is vulnerable to shell injection and should be replaced with `subprocess.run()`.

**Severity:** MEDIUM (test code only, but sets bad precedent)
**Affected File:** `tests/integration/test_session_database_git.py` (line 30)

## Affected Code

```python
os.system("git init > /dev/null 2>&1")
```

## Impact Assessment

- **Direct Impact:** Low (test code only)
- **Risk:** Bad pattern that could be copied to production code
- **Best Practice Violation:** os.system() is deprecated for security reasons

## Recommended Fix

```python
# Before
os.system("git init > /dev/null 2>&1")

# After
subprocess.run(['git', 'init'], capture_output=True, check=True)
```

## References

- [Python subprocess documentation](https://docs.python.org/3/library/subprocess.html)
- [CWE-78: OS Command Injection](https://cwe.mitre.org/data/definitions/78.html)
