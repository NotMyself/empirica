# GitHub Issue: Path Traversal Vulnerabilities

**Title:** [Security] MEDIUM: Path Traversal vulnerabilities in simple_session_server.py

**Labels:** `security`, `medium`, `bug`

---

## Summary

Multiple path traversal vulnerabilities exist in the Simple Session Server that allow reading, listing, and moving arbitrary files on the system. The file operations do not validate that user-provided paths stay within the allowed workspace directory.

**Severity:** MEDIUM (CRITICAL when server is network-accessible)
**Affected File:** `empirica/cli/simple_session_server.py`

## Affected Code

### 1. Arbitrary File Read - `_read_file()` (lines 127-147)

```python
def _read_file(self, session: dict, path: str) -> dict:
    """Read a file's contents"""
    full_path = Path(session["cwd"]) / path  # No validation!
    content = full_path.read_text()  # Reads ANY file
```

### 2. Directory Enumeration - `_list_files()` (lines 98-125)

```python
def _list_files(self, session: dict, path: str = ".") -> dict:
    """List files in a directory"""
    full_path = Path(session["cwd"]) / path  # No validation!
    for item in full_path.iterdir():  # Lists ANY directory
```

### 3. Arbitrary File Move - `_move_file()` (lines 149-186)

```python
def _move_file(self, session: dict, from_path: str, to_path: str) -> dict:
    """Move or rename a file"""
    from_full = Path(session["cwd"]) / from_path  # No validation!
    to_full = Path(session["cwd"]) / to_path
    to_full.parent.mkdir(parents=True, exist_ok=True)  # Creates dirs ANYWHERE
    from_full.rename(to_full)  # Moves/overwrites ANY file
```

## Attack Vectors

### Read /etc/passwd
```bash
curl -X POST http://localhost:8000/sessions/{sid}/command \
  -H "Content-Type: application/json" \
  -d '{"command": "read_file", "args": {"path": "../../../etc/passwd"}}'
```

### List Root Directory
```bash
curl -X POST http://localhost:8000/sessions/{sid}/command \
  -d '{"command": "list_files", "args": {"path": "../../../"}}'
```

### Steal SSH Keys
```bash
curl -X POST http://localhost:8000/sessions/{sid}/command \
  -d '{"command": "move_file", "args": {"from": "../../../home/user/.ssh/id_rsa", "to": "stolen.pem"}}'
```

## Impact Assessment

| Factor | Rating |
|--------|--------|
| Attack Vector | Network (HTTP API) |
| Attack Complexity | Low (trivial `../` pattern) |
| Privileges Required | None (no authentication) |
| Confidentiality | High (read any file) |
| Integrity | High (modify/delete files) |

## Recommended Fix

### Create Path Security Utility

```python
# empirica/utils/path_security.py
from pathlib import Path
from typing import Optional

def is_safe_path(base: Path, target: Path) -> bool:
    """Check if target path is within base directory."""
    try:
        base_resolved = base.resolve()
        target_resolved = target.resolve()
        return str(target_resolved).startswith(str(base_resolved))
    except (OSError, ValueError):
        return False

def safe_join(base: Path, user_path: str) -> Optional[Path]:
    """Safely join user path with base, returning None if traversal detected."""
    if '..' in user_path or user_path.startswith('/'):
        return None

    full_path = (base / user_path).resolve()
    if not is_safe_path(base, full_path):
        return None

    return full_path
```

### Update File Operations

```python
from empirica.utils.path_security import safe_join

def _read_file(self, session: dict, path: str) -> dict:
    """Read a file's contents"""
    workspace = Path(session.get("workspace", session["cwd"]))
    full_path = safe_join(workspace, path)

    if full_path is None:
        return {"error": "Path traversal detected", "path": path}

    if not full_path.exists():
        return {"error": "File not found"}

    content = full_path.read_text()
    return {"content": content, "path": str(full_path)}
```

## References

- [CWE-22: Path Traversal](https://cwe.mitre.org/data/definitions/22.html)
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
