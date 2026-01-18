# GitHub Issue: SQL Injection in Migration Code

**Title:** [Security] HIGH: SQL Injection via f-string interpolation in migration_runner.py

**Labels:** `security`, `high`, `bug`

---

## Summary

Two helper functions in the migration system use f-string interpolation to construct SQL queries with table and column names, creating a SQL injection vulnerability. While current usage only passes hardcoded values, the functions are designed as reusable utilities that could be called with untrusted input.

**Severity:** HIGH
**Affected File:** `empirica/data/migrations/migration_runner.py` (lines 77, 85)

## Affected Code

### Vulnerable Function 1: `column_exists()`

```python
# Line 75-78
def column_exists(cursor: sqlite3.Cursor, table: str, column: str) -> bool:
    """Check if a column exists in a table"""
    cursor.execute(f"SELECT COUNT(*) FROM pragma_table_info('{table}') WHERE name='{column}'")
    return cursor.fetchone()[0] > 0
```

### Vulnerable Function 2: `add_column_if_missing()`

```python
# Line 81-85
def add_column_if_missing(cursor: sqlite3.Cursor, table: str, column: str, column_type: str, default: str = ""):
    """Add a column to a table if it doesn't already exist"""
    if not column_exists(cursor, table, column):
        default_clause = f" DEFAULT {default}" if default else ""
        cursor.execute(f"ALTER TABLE {table} ADD COLUMN {column} {column_type}{default_clause}")
```

## Attack Vectors

**Current Risk Level: MEDIUM-LOW** (currently only called with hardcoded values)

However, the vulnerability becomes exploitable if:
1. Future code changes pass user-controlled input
2. Dynamic migration generation uses external data
3. Migration configuration files are introduced

### Example Attack

```python
# If ever called with user input:
default = "0); DROP TABLE users; --"
add_column_if_missing(cursor, "sessions", "active", "BOOLEAN", default)
# Executes: ALTER TABLE sessions ADD COLUMN active BOOLEAN DEFAULT 0); DROP TABLE users; --
```

## Impact Assessment

| Factor | Current Risk | Future Risk |
|--------|--------------|-------------|
| Exploitability | Low (hardcoded values only) | High (if user input accepted) |
| Data Loss | Potential table drops | Critical |
| Data Corruption | Schema manipulation | Critical |

### Mitigating Factors
- Functions not directly exposed in public API
- All current call sites use hardcoded string literals
- Migrations run during initialization, not runtime

## Recommended Fix

Add identifier validation following existing patterns in the codebase:

```python
import re

def _validate_sql_identifier(identifier: str, identifier_type: str = "identifier") -> None:
    """Validate SQL identifier to prevent SQL injection."""
    if not identifier:
        raise ValueError(f"{identifier_type} cannot be empty")

    if len(identifier) > 64:
        raise ValueError(f"{identifier_type} '{identifier}' is too long")

    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', identifier):
        raise ValueError(f"Invalid {identifier_type} '{identifier}'")


def _validate_default_value(default: str) -> None:
    """Validate default value to prevent SQL injection."""
    if not default:
        return

    # Allow numeric literals
    if re.match(r'^-?\d+(\.\d+)?$', default):
        return

    # Allow quoted strings
    if (default.startswith("'") and default.endswith("'")) or \
       (default.startswith('"') and default.endswith('"')):
        return

    # Allow SQL keywords
    ALLOWED_KEYWORDS = {'NULL', 'TRUE', 'FALSE', 'CURRENT_TIMESTAMP', '0', '1'}
    if default.upper() in ALLOWED_KEYWORDS:
        return

    raise ValueError(f"Invalid default value '{default}'")


def column_exists(cursor: sqlite3.Cursor, table: str, column: str) -> bool:
    """Check if a column exists in a table."""
    _validate_sql_identifier(table, "table name")
    _validate_sql_identifier(column, "column name")

    cursor.execute(f"SELECT COUNT(*) FROM pragma_table_info('{table}') WHERE name='{column}'")
    return cursor.fetchone()[0] > 0


def add_column_if_missing(cursor: sqlite3.Cursor, table: str, column: str, column_type: str, default: str = ""):
    """Add a column to a table if it doesn't already exist."""
    _validate_sql_identifier(table, "table name")
    _validate_sql_identifier(column, "column name")
    _validate_default_value(default)

    if not column_exists(cursor, table, column):
        default_clause = f" DEFAULT {default}" if default else ""
        cursor.execute(f"ALTER TABLE {table} ADD COLUMN {column} {column_type}{default_clause}")
```

## Testing Recommendations

```python
def test_column_exists_rejects_malicious_table_name(test_db):
    with pytest.raises(ValueError):
        column_exists(test_db, "test'; DROP TABLE test_table; --", "id")

def test_add_column_rejects_malicious_default(test_db):
    with pytest.raises(ValueError):
        add_column_if_missing(test_db, "test", "col", "TEXT", "0); DROP")
```

## References

- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
- Similar pattern in codebase: `empirica/data/repositories/cascades.py:60-63`
