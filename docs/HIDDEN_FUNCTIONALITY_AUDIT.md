# Hidden Functionality & Undocumented Features Audit

**Date:** 2026-01-04
**Status:** No malware detected - Legitimate framework

---

## Executive Summary

Empirica is a **legitimate epistemic self-awareness framework** with no malicious code. However, several features are undocumented:
- 6 CLI commands not in README
- 5+ environment variables not documented
- Local telemetry without disclosure
- MCO system entirely undocumented

**Privacy:** All data stays local by default. External services are optional and user-initiated.

---

## 1. Network Requests

### Local Services (Default)

| Service | Default URL | Purpose |
|---------|-------------|---------|
| Ollama | `localhost:11434` | Embeddings generation |
| Qdrant | `localhost:6333` | Vector search |
| Sentinel | `localhost:8765` | Cognitive security |

### External Services (Optional, User-Enabled)

| Service | Requires | Data Sent |
|---------|----------|-----------|
| OpenRouter | `OPENAI_API_KEY` | AI prompts |
| RovoDev | API configuration | AI prompts |
| GitHub | `skill-fetch --url` | Downloads skills |

### No Telemetry
- ✅ No analytics beacons
- ✅ No phone-home functionality
- ✅ No usage reporting to external services

---

## 2. Undocumented CLI Commands

```
goals-list-all     - List ALL goals across all sessions
dashboard          - Launch interactive web dashboard
mco-load           - Load MCO configuration
query              - Unified query interface
edit-with-confidence - Metacognitive edit verification
investigate-multi  - Multi-branch parallel investigation
```

---

## 3. Undocumented Environment Variables

### Database Configuration
```bash
EMPIRICA_DB_TYPE=sqlite|postgresql
EMPIRICA_DB_HOST=localhost
EMPIRICA_DB_PORT=5432
EMPIRICA_DB_NAME=empirica
EMPIRICA_DB_USER=empirica
EMPIRICA_DB_PASSWORD=<password>
```

### Feature Flags
```bash
EMPIRICA_ENFORCE_CASCADE_PHASES=true    # Strict phase ordering
EMPIRICA_ENABLE_MODALITY_SWITCHER=true  # Multi-model routing
SENTINEL_URL=http://localhost:8765      # Cognitive security
```

---

## 4. Undocumented Configuration Files

### MCO (Meta-Cognitive Orchestration)
Location: `/empirica/config/mco/`

- `cascade_styles.yaml` - CASCADE workflow styles
- `confidence_weights.yaml` - Confidence calculations
- `drift_thresholds.yaml` - Drift detection thresholds
- `epistemic_conduct.yaml` - Behavioral rules
- `feedback_loops.yaml` - Feedback mechanisms
- `goal_scopes.yaml` - Goal scoping rules
- `personas.yaml` - AI persona definitions
- `protocols.yaml` - Communication protocols

### Security Configuration
Location: `/empirica/config/mcp_security.yaml`

- Role-based access control (RBAC)
- Per-AI permissions matrix
- Rate limiting rules
- Forbidden commands list
- Sentinel integration

---

## 5. Local Telemetry

**File:** `empirica/cli/cli_core.py:214-226`

### What's Tracked
- Command name
- Execution time (milliseconds)
- Success/failure status
- Error messages

### Storage
- Location: `.empirica/sessions/sessions.db`
- Never transmitted externally
- No documented opt-out

### Purpose
- Legacy command detection
- Performance monitoring
- Local analytics only

---

## 6. Simple Session Server (Experimental)

**File:** `empirica/cli/simple_session_server.py`

### Endpoints
```
POST /sessions              - Create session
POST /sessions/{id}/command - Execute command
GET  /sessions/{id}/dashboard - Get dashboard
GET  /sessions              - List sessions
GET  /                      - API info
```

### Security Concerns
- No authentication
- Command whitelist can be bypassed
- Uses shell=True (CRITICAL vulnerability)

### Status
- Not documented in README
- Appears to be experimental/MVP

---

## 7. Recommendations

### For Documentation
1. Add MCO system documentation
2. Document all environment variables
3. Document local telemetry and add opt-out
4. Document Simple Session Server (or remove)
5. Add SECURITY.md explaining data flow

### For Code
1. Fix shell=True vulnerability
2. Add path validation for env variables
3. Add authentication to Session Server

---

## Verdict

**Security Rating:** 4/5 ⭐⭐⭐⭐☆

**Empirica is safe to use** with awareness of:
- One critical vulnerability (Session Server)
- Local telemetry (not malicious)
- Undocumented features (not hidden, just undocumented)

All external data transmission is optional and user-initiated.
