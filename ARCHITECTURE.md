# Empirica Architecture Analysis

**Version:** 1.0.5
**Total Code:** ~20,500 lines of Python
**Analysis Date:** 2026-01-04

## Executive Summary

Empirica is an epistemic self-awareness framework for AI agents that provides metacognitive capabilities through a 13-dimensional vector assessment system. The architecture consists of a CLI interface, MCP (Model Context Protocol) server, database layer, and multiple specialized subsystems for session management, goal tracking, git integration, and multi-agent orchestration.

---

## 1. Directory Structure

### Main Modules (`/home/user/empirica/empirica/`)

```
empirica/
├── cli/                    # Command-line interface and handlers
│   ├── cli_core.py        # Main entry point, argument parsing
│   ├── command_handlers/  # 40+ command handler modules
│   ├── parsers/           # Argument parser definitions
│   └── utils/             # CLI utilities
│
├── core/                   # Core business logic (21 subdirectories)
│   ├── canonical/         # Canonical data structures and git integration
│   ├── goals/             # Goal management system
│   ├── sentinel/          # Safety gates and orchestration
│   ├── agents/            # Multi-agent system
│   ├── persona/           # Persona-based priors
│   ├── git_ops/           # Git operations (signed commits, notes)
│   ├── schemas/           # Pydantic schemas for epistemic assessment
│   └── [16 more...]       # Various specialized modules
│
├── data/                   # Data persistence layer
│   ├── session_database.py # Main database class (facade)
│   ├── repositories/      # Domain repositories (10 modules)
│   ├── schema/            # Database table schemas (5 modules)
│   ├── formatters/        # Data formatters
│   └── migrations/        # Database migrations
│
├── config/                 # Configuration system
│   ├── path_resolver.py   # Path resolution (git-aware)
│   ├── threshold_loader.py # Dynamic threshold configuration
│   ├── mco/               # Meta-Agent Configuration Objects
│   └── [10 more...]       # Various config loaders
│
├── api/                    # REST API (future)
│   └── routes/
│
├── integrations/           # External integrations
│   └── beads/             # BEADS issue tracking
│
├── plugins/                # Plugin system
│   └── modality_switcher/ # Modality switching
│
└── [10 more...]           # tests, utils, metrics, etc.
```

### MCP Server (`/home/user/empirica/empirica-mcp/`)

```
empirica-mcp/
├── empirica_mcp/
│   ├── server.py          # MCP v2 server (thin CLI wrapper)
│   ├── epistemic_middleware.py # Optional epistemic mode
│   └── epistemic/         # Epistemic middleware modules
└── test_epistemic.py      # Integration tests
```

---

## 2. Core Components

### 2.1 CLI Entry Point (`cli/cli_core.py`)

**Purpose:** Main entry point for all Empirica commands

**Architecture:**
- **Argument Parser:** Modular parser system with 15+ parser modules
- **Command Routing:** 60+ commands organized into 19 categories
- **Handler Delegation:** Routes to specialized command handler modules
- **Telemetry:** Logs command usage to database
- **Error Handling:** Centralized error handling with CLI-friendly output

**Key Functions:**
```python
create_argument_parser() -> ArgumentParser
    # Creates main parser with all subcommands

main(args=None)
    # Main entry point, routes to handlers
    # Logs telemetry and execution time
```

**Command Categories:**
1. Session Management (7 commands)
2. CASCADE Workflow (4 commands)
3. Goals & Tasks (12 commands)
4. Project Management (8 commands)
5. Checkpoints (7 commands)
6. Identity (4 commands)
7. Monitoring (4 commands)
8. Investigation (5 commands)
9. Agents (6 commands)
10. Sentinel (4 commands)
11. [9 more categories...]

### 2.2 MCP Server (`empirica-mcp/server.py`)

**Purpose:** Thin wrapper around CLI for AI agent integration

**Architecture:**
- **Tool Count:** 40 tools (3 stateless, 37 stateful)
- **Routing Strategy:** Stateful tools route to CLI via subprocess
- **Output Format:** JSON for machine consumption
- **Token Efficiency:** 75% reduction vs direct implementation

**Key Design Decisions:**
```python
# Stateless tools (handled in MCP)
- get_empirica_introduction
- get_workflow_guidance
- cli_help

# Stateful tools (route to CLI)
- All session, goal, cascade, checkpoint operations
- Execution: subprocess.run(['empirica', cmd, ...])
```

**Benefits:**
- Single source of truth (CLI)
- No async bugs (subprocess in executor)
- Easy testing (CLI commands directly)
- 90% code reduction (500 vs 5000 lines)

### 2.3 Session Database (`data/session_database.py`)

**Purpose:** Central database facade for all data operations

**Architecture Pattern:** Repository Pattern + Facade

```python
class SessionDatabase:
    def __init__(self, db_path=None, db_type=None):
        # Pluggable backend (SQLite or PostgreSQL)
        self.adapter = DatabaseAdapter.create(db_type, ...)
        self.conn = self.adapter.conn

        # Domain repositories (shared connection)
        self.sessions = SessionRepository(self.conn)
        self.cascades = CascadeRepository(self.conn)
        self.goals = GoalRepository(self.conn)
        self.branches = BranchRepository(self.conn)
        self.breadcrumbs = BreadcrumbRepository(self.conn)
        self.projects = ProjectRepository(self.conn)
        self.tokens = TokenRepository(self.conn)
        self.commands = CommandRepository(self.conn)
        self.workspace = WorkspaceRepository(self.conn)
        self.vectors = VectorRepository(self.conn)
```

**Database Location:**
- **Default:** `./.empirica/sessions/sessions.db` (project-local)
- **Configurable:** Via `EMPIRICA_SESSION_DB` env var or `.empirica/config.yaml`
- **Why project-local:** Data is scoped to repository, not global

**Initialization:**
1. Load database config (type, path)
2. Create adapter (SQLite or PostgreSQL)
3. Create tables from schema modules
4. Run migrations (tracked in `migrations` table)
5. Create indexes for performance
6. Initialize domain repositories

### 2.4 Core Business Logic Classes

#### Canonical Data Structures (`core/canonical/reflex_frame.py`)

**Key Types:**
```python
class Action(Enum):
    PROCEED = "proceed"
    INVESTIGATE = "investigate"
    CLARIFY = "clarify"
    RESET = "reset"
    STOP = "stop"

@dataclass
class VectorState:
    score: float  # 0.0-1.0
    rationale: str  # Genuine AI reasoning
    evidence: Optional[str]
    warrants_investigation: bool
    investigation_priority: Optional[str]  # low/medium/high/critical
    investigation_reason: Optional[str]
```

**Canonical Weights:**
```python
CANONICAL_WEIGHTS = {
    'foundation': 0.35,      # know, do, context
    'comprehension': 0.25,   # clarity, coherence, signal, density
    'execution': 0.25,       # state, change, completion, impact
    'engagement': 0.15       # engagement (gate + weight)
}
```

#### Git-Enhanced Reflex Logger (`core/canonical/git_enhanced_reflex_logger.py`)

**Purpose:** Epistemic checkpoint storage with 3-layer architecture

**Storage Layers:**
1. **SQLite:** Queryable index with reflexes table
2. **Git Notes:** Compressed checkpoints (~450 tokens vs ~6,500)
3. **JSON Logs:** Full audit trail (optional)

**Key Innovation:**
```python
# Traditional: Load full session history from SQLite (~6,500 tokens)
# Git-Enhanced: Load compressed checkpoint from git notes (~450 tokens)
# Token Reduction: 80-90%

checkpoint = logger.get_last_checkpoint()
# Returns: phase, round, vectors, metadata, git_commit_sha
```

**Architecture:**
```python
class GitEnhancedReflexLogger:
    def __init__(self, session_id, enable_git_notes=True):
        self.git_state_capture = GitStateCapture(repo_path)
        self.git_notes_storage = GitNotesStorage(session_id, repo_path)
        self.checkpoint_storage = CheckpointStorage(session_id, log_dir)

    def add_checkpoint(self, phase, vectors, metadata):
        # 1. Capture git state (commit SHA, branch, diff stats)
        git_state = self.git_state_capture.capture_state()

        # 2. Create compressed checkpoint
        checkpoint = {
            'phase': phase,
            'round': auto_increment(),
            'vectors': vectors,
            'metadata': metadata,
            'git_commit': git_state['commit']
        }

        # 3. Store in all 3 layers
        self.git_notes_storage.add_note(checkpoint)  # Git notes
        self.checkpoint_storage.save_sqlite(checkpoint)  # SQLite
        self.checkpoint_storage.save_json(checkpoint)  # JSON audit
```

---

## 3. Data Flow

### 3.1 CLI Command → Handler → Database

**Example: `empirica preflight-submit`**

```
1. CLI Entry Point (cli_core.py)
   ↓
   main() parses args
   ↓
   Routes to handle_preflight_submit_command()

2. Command Handler (workflow_commands.py)
   ↓
   handle_preflight_submit_command(args)
   ↓
   - Parse JSON config or CLI args
   - Extract vectors (13D)
   - Validate session_id

3. Storage Layer
   ↓
   GitEnhancedReflexLogger.add_checkpoint()
   ↓
   - SQLite: INSERT INTO reflexes (session_id, phase, vectors, ...)
   - Git Notes: git notes add refs/empirica/session/{session_id}
   - JSON: Write to .empirica_reflex_logs/{session_id}/checkpoints.json

4. CASCADE Tracking
   ↓
   SessionDatabase.conn.execute(
       "INSERT INTO cascades (cascade_id, session_id, task, ...)"
   )

5. Bayesian Calibration (optional)
   ↓
   BayesianBeliefManager.get_calibration_adjustments(ai_id)
   ↓
   Returns historical bias adjustments

6. Sentinel Hook (optional)
   ↓
   SentinelHooks.post_checkpoint_hook(session_id, phase, checkpoint_data)
   ↓
   Returns routing decision (PROCEED, INVESTIGATE, etc.)

7. Response
   ↓
   Return JSON to stdout
   {
       "ok": true,
       "checkpoint_id": "abc123",
       "vectors_submitted": 13,
       "storage_layers": {"sqlite": true, "git_notes": true, "json": true},
       "calibration": {...},
       "sentinel": {...}
   }
```

### 3.2 Data Structures

#### Sessions
```python
{
    'session_id': 'uuid',
    'ai_id': 'claude-3.5-sonnet',
    'start_time': timestamp,
    'end_time': timestamp,
    'components_loaded': 0,
    'total_turns': 0,
    'total_cascades': 0,
    'avg_confidence': 0.75,
    'drift_detected': False,
    'session_notes': 'User feedback',
    'bootstrap_level': 1
}
```

#### Cascades
```python
{
    'cascade_id': 'uuid',
    'session_id': 'uuid',
    'task': 'Review codebase for security issues',
    'goal_id': 'uuid',

    # Phase tracking (boolean flags)
    'preflight_completed': True,
    'check_completed': True,
    'postflight_completed': True,

    # Results
    'final_action': 'proceed',
    'final_confidence': 0.82,
    'investigation_rounds': 2,
    'duration_ms': 45000,

    # Gates
    'engagement_gate_passed': True,
    'bayesian_active': True,
    'drift_monitored': False
}
```

#### Epistemic Vectors (Reflexes)
```python
{
    'id': auto_increment,
    'session_id': 'uuid',
    'cascade_id': 'uuid',
    'phase': 'PREFLIGHT',  # or CHECK, POSTFLIGHT
    'round': 1,
    'timestamp': timestamp,

    # 13 epistemic vectors (0.0-1.0)
    'engagement': 0.85,
    'know': 0.75,
    'do': 0.80,
    'context': 0.70,
    'clarity': 0.65,
    'coherence': 0.75,
    'signal': 0.80,
    'density': 0.60,
    'state': 0.70,
    'change': 0.75,
    'completion': 0.20,
    'impact': 0.60,
    'uncertainty': 0.40,

    # Metadata
    'reflex_data': '{"reasoning": "...", "evidence": "..."}',
    'reasoning': 'AI self-assessment rationale',
    'evidence': 'Supporting facts'
}
```

#### Goals
```python
{
    'id': 'uuid',
    'session_id': 'uuid',
    'objective': 'Fix authentication bug in login flow',
    'scope': {
        'breadth': 0.3,      # 0=single function, 1=entire codebase
        'duration': 0.2,     # 0=minutes, 1=months
        'coordination': 0.1  # 0=solo, 1=multi-agent
    },
    'success_criteria': [
        {
            'id': 'uuid',
            'description': 'All tests pass',
            'validation_method': 'quality_gate',
            'is_required': True,
            'is_met': False
        }
    ],
    'estimated_complexity': 0.65,
    'created_timestamp': timestamp,
    'completed_timestamp': None,
    'is_completed': False,
    'status': 'in_progress'  # in_progress | complete | blocked
}
```

### 3.3 Session → Cascade → Goals Relationships

```
Session (1) ───────── (N) Cascades
   │                         │
   │                         │
   └───── (N) Goals          └── (1) Goal (optional)
              │
              │
              └───── (N) SubTasks
```

**Cardinality:**
- 1 Session → N Cascades (reasoning workflows)
- 1 Session → N Goals (objectives)
- 1 Cascade → 0-1 Goal (optional link)
- 1 Goal → N SubTasks (task breakdown)

**Example:**
```
Session: "Code review session - 2026-01-04"
├── Cascade 1: "Review authentication module"
│   └── Goal 1: "Fix auth bug" (linked)
│       ├── SubTask 1: "Investigate token validation"
│       ├── SubTask 2: "Fix CSRF handling"
│       └── SubTask 3: "Add tests"
├── Cascade 2: "Review database module"
│   └── Goal 2: "Optimize queries" (linked)
└── Cascade 3: "General exploration" (no goal)
```

---

## 4. Key Subsystems

### 4.1 CASCADE Workflow

**Phases:**
1. **PREFLIGHT:** Baseline epistemic assessment before task
2. **CHECK:** Mid-workflow decision gate (0-N times)
3. **POSTFLIGHT:** Final calibration after task

**Key Files:**
- `cli/command_handlers/workflow_commands.py` (handlers)
- `core/canonical/git_enhanced_reflex_logger.py` (storage)
- `core/schemas/epistemic_assessment.py` (schema)

**Flow:**
```python
# 1. PREFLIGHT
empirica preflight-submit --session-id {id} --vectors {...}
→ Creates baseline checkpoint
→ Returns calibration adjustments (historical bias)
→ Sentinel evaluates routing decision

# 2. THINK/INVESTIGATE (implicit guidance, not tracked)
# AI performs investigation based on PREFLIGHT vectors

# 3. CHECK (0-N times)
empirica check --session-id {id} --findings [...] --unknowns [...] --confidence 0.7
→ Loads PREFLIGHT baseline
→ Loads current checkpoint
→ Calculates drift from baseline
→ Evaluates engagement gate (≥0.60)
→ Returns decision: PROCEED or INVESTIGATE

# 4. POSTFLIGHT
empirica postflight-submit --session-id {id} --vectors {...}
→ Creates final checkpoint
→ Calculates epistemic deltas
→ Logs calibration data for future sessions
```

**Engagement Gate:**
```python
ENGAGEMENT_THRESHOLD = 0.60

if vectors['engagement'] < ENGAGEMENT_THRESHOLD:
    return Action.CLARIFY  # Task unclear, need user input
```

**CHECK Decision Logic:**
```python
def evaluate_check(baseline, current, findings, unknowns, confidence):
    # Calculate drift
    drift = calculate_drift(baseline, current)

    # Gate 1: Engagement
    if current['engagement'] < 0.60:
        return "CLARIFY"

    # Gate 2: Critical thresholds
    if current['coherence'] < 0.50:
        return "RESET"
    if current['density'] > 0.90:
        return "RESET"
    if current['change'] < 0.50:
        return "STOP"

    # Gate 3: Confidence
    if confidence < 0.70:
        return "INVESTIGATE"

    # All gates passed
    return "PROCEED"
```

### 4.2 Epistemic Vectors (13 Dimensions)

**Structure:**
```
├── Gate
│   └── engagement (0.60 threshold)
│
├── Foundation (Tier 0) - 35% weight
│   ├── know (Knowledge of domain)
│   ├── do (Ability to execute)
│   └── context (Situational awareness)
│
├── Comprehension (Tier 1) - 25% weight
│   ├── clarity (Task clarity)
│   ├── coherence (Internal consistency)
│   ├── signal (Signal-to-noise ratio)
│   └── density (Information density)
│
├── Execution (Tier 2) - 25% weight
│   ├── state (Current state mapping)
│   ├── change (Change detection)
│   ├── completion (Progress toward goal)
│   └── impact (Expected impact)
│
└── Meta - 15% weight
    └── uncertainty (Epistemic uncertainty)
```

**Schema Definition (`core/schemas/epistemic_assessment.py`):**
```python
@dataclass
class VectorAssessment:
    score: float  # 0.0-1.0
    rationale: str  # GENUINE reasoning (not heuristics)
    evidence: Optional[str]
    warrants_investigation: bool
    investigation_priority: int  # 0-10

@dataclass
class EpistemicAssessmentSchema:
    # Gate
    engagement: VectorAssessment

    # Foundation (Tier 0)
    foundation_know: VectorAssessment
    foundation_do: VectorAssessment
    foundation_context: VectorAssessment

    # Comprehension (Tier 1)
    comprehension_clarity: VectorAssessment
    comprehension_coherence: VectorAssessment
    comprehension_signal: VectorAssessment
    comprehension_density: VectorAssessment

    # Execution (Tier 2)
    execution_state: VectorAssessment
    execution_change: VectorAssessment
    execution_completion: VectorAssessment
    execution_impact: VectorAssessment

    # Meta
    uncertainty: VectorAssessment
```

**Key Principle:**
> Epistemic weights ≠ internal model weights
> We measure knowledge state, we don't modify model parameters.

### 4.3 Git Integration

**Components:**
1. **Git Notes Storage** (`core/canonical/git_notes_storage.py`)
2. **Git State Capture** (`core/canonical/git_state_capture.py`)
3. **Checkpoint Storage** (`core/canonical/checkpoint_storage.py`)
4. **Signed Operations** (`core/git_ops/signed_operations.py`)

**Git Notes Architecture:**
```
refs/empirica/session/{session_id}
├── checkpoint-1-preflight
├── checkpoint-2-check
├── checkpoint-3-check
└── checkpoint-4-postflight

Each note contains:
{
    "phase": "PREFLIGHT",
    "round": 1,
    "vectors": {13 dimensions},
    "git_commit": "abc123",
    "git_branch": "main",
    "files_changed": 5,
    "metadata": {...}
}
```

**Signed Checkpoints:**
```python
# Using SigningPersona for cryptographic signing
logger = GitEnhancedReflexLogger(
    session_id=session_id,
    signing_persona=persona  # Optional
)

# Add signed checkpoint
checkpoint_id = logger.add_checkpoint(phase, vectors, metadata)
# → Creates git commit with checkpoint data
# → Signs commit with persona's private key
# → Stores pointer in SQLite

# Verify checkpoint
is_valid = logger.verify_checkpoint(checkpoint_id, persona.public_key)
```

**Benefits:**
- Distributed: Git notes sync across clones
- Immutable: Can't modify without breaking signatures
- Token-efficient: Compressed format
- Verifiable: Cryptographic signing
- Temporal: Linked to git commits (code state correlation)

### 4.4 Multi-Agent System

**Components:**
- **EpistemicAgent** (`core/agents/epistemic_agent.py`)
- **Sentinel Orchestrator** (`core/sentinel/orchestrator.py`)
- **Persona System** (`core/persona/`)

**Architecture:**
```python
# 1. Spawn parallel agents with different personas
sentinel = Sentinel(session_id)
result = sentinel.orchestrate(
    task="Review code for security issues",
    max_agents=3,  # Spawn 3 parallel agents
    loop_mode=LoopMode.SENTINEL  # Sentinel governs convergence
)

# 2. Sentinel selects personas based on task
personas = sentinel.select_personas(task)
# → Returns: ["security_expert", "code_quality", "performance_optimizer"]

# 3. Spawn agents with persona-seeded priors
agents = []
for persona in personas:
    agent = EpistemicAgent.spawn(
        task=task,
        persona=persona,  # Seeds epistemic priors
        session_id=session_id
    )
    agents.append(agent)

# 4. Agents run CASCADE workflow in parallel
results = await asyncio.gather(*[agent.run() for agent in agents])

# 5. Merge results
merged = sentinel.merge_results(
    results,
    strategy=MergeStrategy.WEIGHTED  # Weight by merge_score
)

# 6. Store merged assessment
db.store_vectors(session_id, "CHECK", merged.vectors)
```

**Persona Priors:**
```python
# Each persona has epistemic biases
personas = {
    "security_expert": {
        "know": 0.85,  # High domain knowledge
        "uncertainty": 0.20  # Low uncertainty
    },
    "novice": {
        "know": 0.40,  # Low domain knowledge
        "uncertainty": 0.70  # High uncertainty
    }
}

# Agent applies persona priors to assessments
agent = EpistemicAgent(persona="security_expert")
assessment = agent.assess(task)
# → Vectors are biased by persona priors
```

**Merge Strategies:**
```python
class MergeStrategy(Enum):
    CONSENSUS = "consensus"       # All agents must agree
    BEST_SCORE = "best_score"     # Take highest merge_score
    WEIGHTED = "weighted"         # Weight by merge_score
    UNION = "union"               # Combine all findings
    INTERSECTION = "intersection" # Only common findings
```

### 4.5 Sentinel Safety Gates

**Dual Defense Layers:**

**1. Noetic Filter (Cognition-level)**
```python
@dataclass
class NoeticFilter:
    """Filters what investigation paths are allowed"""
    filter_id: str
    blocked_patterns: List[str]  # Regex patterns to block
    blocked_domains: List[str]   # Domain areas to block
    action_on_match: GateAction  # INVESTIGATE, HALT_AND_AUDIT, etc.

# Example: Block exploit development
filter = NoeticFilter(
    filter_id="security-001",
    blocked_patterns=[r"exploit.*development", r"bypass.*authentication"],
    action_on_match=GateAction.HALT_AND_AUDIT
)

# Evaluate during investigation
if filter.evaluate(investigation_context):
    # Block and log
```

**2. Axiologic Gate (Action-level)**
```python
@dataclass
class AxiologicGate:
    """Validates actions against value constraints"""
    gate_id: str
    action_patterns: List[str]   # Action patterns to gate
    required_vectors: Dict[str, float]  # Min vectors required
    action_on_violation: GateAction

# Example: Prevent deletion without confirmation
gate = AxiologicGate(
    gate_id="safety-001",
    action_patterns=[r"delete.*production", r"drop.*table"],
    required_vectors={"know": 0.90, "impact": 0.80},
    action_on_violation=GateAction.REQUIRE_HUMAN
)

# Evaluate before action
if gate.evaluate(action_context):
    # Require human approval
```

**Gate Actions:**
```python
class GateAction(Enum):
    PROCEED = "proceed"
    INVESTIGATE = "investigate"
    HALT_AND_AUDIT = "halt_and_audit"
    REQUIRE_HUMAN = "require_human_review"
    ESCALATE = "escalate"
    LOG_AND_CONTINUE = "log_and_continue"
```

**Sentinel Orchestration:**
```python
# 1. Load domain profile (e.g., healthcare, finance)
sentinel.load_domain_profile("healthcare")
# → Loads HIPAA compliance gates

# 2. Task analysis
personas = sentinel.analyze_task(task)
gates = sentinel.get_applicable_gates(task, domain="healthcare")

# 3. Spawn agents with gates
agents = sentinel.spawn_agents(task, personas, gates)

# 4. Monitor compliance during execution
for checkpoint in agent_checkpoints:
    decision = sentinel.check_compliance(
        vectors=checkpoint.vectors,
        findings=checkpoint.findings,
        unknowns=checkpoint.unknowns
    )

    if decision == GateAction.HALT_AND_AUDIT:
        # Stop execution, log to audit trail
        sentinel.log_audit(checkpoint, reason="Compliance violation")
        break
```

---

## 5. Database Schema

### 5.1 Core Tables

#### sessions
```sql
CREATE TABLE sessions (
    session_id TEXT PRIMARY KEY,
    ai_id TEXT NOT NULL,
    user_id TEXT,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    components_loaded INTEGER NOT NULL,
    total_turns INTEGER DEFAULT 0,
    total_cascades INTEGER DEFAULT 0,
    avg_confidence REAL,
    drift_detected BOOLEAN DEFAULT 0,
    session_notes TEXT,
    bootstrap_level INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### cascades
```sql
CREATE TABLE cascades (
    cascade_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    task TEXT NOT NULL,
    context_json TEXT,
    goal_id TEXT,
    goal_json TEXT,

    -- Phase tracking
    preflight_completed BOOLEAN DEFAULT 0,
    check_completed BOOLEAN DEFAULT 0,
    postflight_completed BOOLEAN DEFAULT 0,

    -- Results
    final_action TEXT,
    final_confidence REAL,
    investigation_rounds INTEGER DEFAULT 0,
    duration_ms INTEGER,

    -- Timestamps
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,

    -- Gates
    engagement_gate_passed BOOLEAN,
    bayesian_active BOOLEAN DEFAULT 0,
    drift_monitored BOOLEAN DEFAULT 0,

    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

#### reflexes (Epistemic Vectors)
```sql
CREATE TABLE reflexes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    cascade_id TEXT,
    phase TEXT NOT NULL,  -- PREFLIGHT, CHECK, POSTFLIGHT
    round INTEGER DEFAULT 1,
    timestamp REAL NOT NULL,

    -- 13 epistemic vectors
    engagement REAL,
    know REAL,
    do REAL,
    context REAL,
    clarity REAL,
    coherence REAL,
    signal REAL,
    density REAL,
    state REAL,
    change REAL,
    completion REAL,
    impact REAL,
    uncertainty REAL,

    -- Metadata
    reflex_data TEXT,  -- JSON: full VectorAssessment objects
    reasoning TEXT,
    evidence TEXT,

    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

#### goals
```sql
CREATE TABLE goals (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    objective TEXT NOT NULL,
    scope TEXT NOT NULL,  -- JSON: {breadth, duration, coordination}
    estimated_complexity REAL,
    created_timestamp REAL NOT NULL,
    completed_timestamp REAL,
    is_completed BOOLEAN DEFAULT 0,
    goal_data TEXT NOT NULL,  -- JSON: full Goal object
    status TEXT DEFAULT 'in_progress',  -- in_progress | complete | blocked
    beads_issue_id TEXT,  -- Optional BEADS integration

    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

#### subtasks
```sql
CREATE TABLE subtasks (
    id TEXT PRIMARY KEY,
    goal_id TEXT NOT NULL,
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    epistemic_importance TEXT NOT NULL DEFAULT 'medium',
    estimated_tokens INTEGER,
    actual_tokens INTEGER,
    completion_evidence TEXT,
    notes TEXT,
    created_timestamp REAL NOT NULL,
    completed_timestamp REAL,
    subtask_data TEXT NOT NULL,  -- JSON: full SubTask object

    FOREIGN KEY (goal_id) REFERENCES goals(id)
);
```

### 5.2 Project Tables

#### projects
```sql
CREATE TABLE projects (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    repos TEXT,  -- JSON array
    created_timestamp REAL NOT NULL,
    last_activity_timestamp REAL,
    status TEXT DEFAULT 'active',
    metadata TEXT,

    total_sessions INTEGER DEFAULT 0,
    total_goals INTEGER DEFAULT 0,
    total_epistemic_deltas TEXT,  -- JSON

    project_data TEXT NOT NULL
);
```

#### project_findings
```sql
CREATE TABLE project_findings (
    id TEXT PRIMARY KEY,
    project_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    goal_id TEXT,
    subtask_id TEXT,
    finding TEXT NOT NULL,
    created_timestamp REAL NOT NULL,
    finding_data TEXT NOT NULL,
    subject TEXT,
    impact REAL DEFAULT 0.5,

    FOREIGN KEY (project_id) REFERENCES projects(id),
    FOREIGN KEY (session_id) REFERENCES sessions(session_id),
    FOREIGN KEY (goal_id) REFERENCES goals(id),
    FOREIGN KEY (subtask_id) REFERENCES subtasks(id)
);
```

#### project_unknowns
```sql
CREATE TABLE project_unknowns (
    id TEXT PRIMARY KEY,
    project_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    goal_id TEXT,
    unknown TEXT NOT NULL,
    is_resolved BOOLEAN DEFAULT FALSE,
    resolved_by TEXT,
    created_timestamp REAL NOT NULL,
    resolved_timestamp REAL,
    unknown_data TEXT NOT NULL,
    subject TEXT,
    impact REAL DEFAULT 0.5,

    FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

### 5.3 Investigation Tables

#### investigation_branches
```sql
CREATE TABLE investigation_branches (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    branch_name TEXT NOT NULL,
    investigation_path TEXT NOT NULL,
    git_branch_name TEXT NOT NULL,

    -- Epistemic state
    preflight_vectors TEXT NOT NULL,  -- JSON
    postflight_vectors TEXT,  -- JSON

    -- Cost tracking
    tokens_spent INTEGER DEFAULT 0,
    time_spent_minutes INTEGER DEFAULT 0,

    -- Merge metadata
    merge_score REAL,
    epistemic_quality REAL,
    is_winner BOOLEAN DEFAULT FALSE,

    -- Timestamps
    created_timestamp REAL NOT NULL,
    checkpoint_timestamp REAL,
    merged_timestamp REAL,
    status TEXT DEFAULT 'active',

    branch_metadata TEXT,

    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

### 5.4 Tracking Tables

#### mistakes_made
```sql
CREATE TABLE mistakes_made (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    goal_id TEXT,
    project_id TEXT,
    mistake TEXT NOT NULL,
    why_wrong TEXT NOT NULL,
    cost_estimate TEXT,
    root_cause_vector TEXT,  -- Which epistemic vector failed
    prevention TEXT,
    created_timestamp REAL NOT NULL,
    mistake_data TEXT NOT NULL,

    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

#### token_savings
```sql
CREATE TABLE token_savings (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    saving_type TEXT NOT NULL,
    tokens_saved INTEGER NOT NULL,
    evidence TEXT,
    logged_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);
```

### 5.5 Schema Organization

**Schema Modules (`data/schema/`):**
```python
# __init__.py
ALL_SCHEMAS = (
    SESSIONS_SCHEMAS +      # sessions, cascades
    EPISTEMIC_SCHEMAS +     # reflexes, bayesian_beliefs, epistemic_snapshots
    GOALS_SCHEMAS +         # goals, subtasks
    PROJECTS_SCHEMAS +      # projects, findings, unknowns, dead_ends
    TRACKING_SCHEMAS        # mistakes, branches, token_savings
)
```

**Indexes:**
```sql
-- Performance indexes
CREATE INDEX idx_sessions_ai ON sessions(ai_id);
CREATE INDEX idx_cascades_session ON cascades(session_id);
CREATE INDEX idx_reflexes_session ON reflexes(session_id);
CREATE INDEX idx_reflexes_phase ON reflexes(phase);
CREATE INDEX idx_goals_session ON goals(session_id);
CREATE INDEX idx_goals_status ON goals(status);
CREATE INDEX idx_subtasks_goal ON subtasks(goal_id);
```

---

## 6. Configuration System

### 6.1 Path Resolution (`config/path_resolver.py`)

**Priority Order:**
```python
def get_empirica_root() -> Path:
    # 1. EMPIRICA_WORKSPACE_ROOT env var (Docker/multi-AI)
    if workspace_root := os.getenv('EMPIRICA_WORKSPACE_ROOT'):
        return Path(workspace_root) / '.empirica'

    # 2. EMPIRICA_DATA_DIR env var (explicit)
    if data_dir := os.getenv('EMPIRICA_DATA_DIR'):
        return Path(data_dir)

    # 3. .empirica/config.yaml -> root
    config = load_empirica_config()
    if config and 'root' in config:
        return Path(config['root'])

    # 4. Git root (if in git repo)
    git_root = get_git_root()
    if git_root:
        return git_root / '.empirica'

    # 5. Fallback to CWD
    return Path.cwd() / '.empirica'
```

**Directory Structure:**
```
.empirica/
├── config.yaml          # Configuration file
├── sessions/
│   └── sessions.db      # SQLite database
├── identity/
│   ├── {ai_id}.key     # Private keys
│   └── {ai_id}.pub     # Public keys
├── metrics/
│   └── performance.json
├── messages/
│   └── {session_id}.json
└── personas/
    └── {persona_id}.json
```

### 6.2 Configuration Files

#### `.empirica/config.yaml`
```yaml
version: '2.0'
root: /path/to/project/.empirica

paths:
  sessions: sessions/sessions.db
  identity: identity/
  messages: messages/
  metrics: metrics/
  personas: personas/

settings:
  auto_checkpoint: true
  git_integration: true
  log_level: info

database:
  type: sqlite  # or postgresql
  # For PostgreSQL:
  # host: localhost
  # port: 5432
  # database: empirica
  # user: empirica
  # password: secret

env_overrides:
  - EMPIRICA_DATA_DIR
  - EMPIRICA_SESSION_DB
```

### 6.3 Threshold Configuration (`config/mco/cascade_styles.yaml`)

**Dynamic Thresholds (MCO Architecture):**
```yaml
default:
  engagement_threshold: 0.60

  critical:
    coherence_min: 0.50
    density_max: 0.90
    change_min: 0.50

  cascade:
    max_investigation_rounds: 7
    check_confidence_to_proceed: 0.70

  uncertainty:
    low: 0.70
    moderate: 0.30

  comprehension:
    high: 0.80
    moderate: 0.50

  execution:
    high: 0.80
    moderate: 0.60

# Alternate profiles
exploratory:
  engagement_threshold: 0.50
  cascade:
    max_investigation_rounds: 10

rigorous:
  engagement_threshold: 0.75
  cascade:
    check_confidence_to_proceed: 0.85
```

**Usage:**
```python
from empirica.config import get_threshold_config

config = get_threshold_config(profile="rigorous")
engagement = config.get('engagement_threshold')  # 0.75
```

---

## 7. Key Design Patterns

### 7.1 Repository Pattern

**Implementation:**
```python
class BaseRepository:
    """Base class for all repositories"""
    def __init__(self, conn: sqlite3.Connection):
        self.conn = conn  # Shared connection

class SessionRepository(BaseRepository):
    def create_session(self, ai_id, ...):
        self._execute("INSERT INTO sessions ...")
        self.commit()

    def get_session(self, session_id):
        return self._execute("SELECT * FROM sessions WHERE ...").fetchone()

# Usage
db = SessionDatabase()
session = db.sessions.get_session(session_id)
goal = db.goals.get_goal(goal_id)
```

**Benefits:**
- Single connection (no connection pool overhead)
- Transaction management in one place
- Clean separation of concerns
- Easy to test (mock connection)

### 7.2 Facade Pattern

**SessionDatabase as Facade:**
```python
class SessionDatabase:
    """Facade over all repositories"""
    def __init__(self):
        self.conn = create_connection()

        # Initialize all repositories
        self.sessions = SessionRepository(self.conn)
        self.cascades = CascadeRepository(self.conn)
        self.goals = GoalRepository(self.conn)
        # ... 10 more repositories

    # Convenience methods delegate to repositories
    def create_session(self, ai_id):
        return self.sessions.create_session(ai_id)
```

### 7.3 Strategy Pattern

**Merge Strategies:**
```python
class MergeStrategy(ABC):
    @abstractmethod
    def merge(self, results: List[AgentResult]) -> MergedResult:
        pass

class ConsensusMerge(MergeStrategy):
    def merge(self, results):
        # All agents must agree
        ...

class WeightedMerge(MergeStrategy):
    def merge(self, results):
        # Weight by merge_score
        ...

# Usage
strategy = WeightedMerge()
merged = strategy.merge(agent_results)
```

### 7.4 Adapter Pattern

**Database Adapter:**
```python
class DatabaseAdapter:
    @staticmethod
    def create(db_type: str, **kwargs):
        if db_type == "sqlite":
            return SQLiteAdapter(**kwargs)
        elif db_type == "postgresql":
            return PostgreSQLAdapter(**kwargs)

# Usage
adapter = DatabaseAdapter.create("postgresql", host="localhost", ...)
conn = adapter.conn
```

---

## 8. Token Efficiency Innovations

### 8.1 Git Notes Storage

**Problem:** Loading full session history from SQLite uses ~6,500 tokens

**Solution:** Compressed checkpoints in git notes use ~450 tokens (93% reduction)

**Implementation:**
```python
# Traditional approach
session_history = db.get_session_history(session_id)
# Returns: All cascades, all reflexes, all findings
# Token count: ~6,500

# Git notes approach
checkpoint = logger.get_last_checkpoint()
# Returns: Single compressed checkpoint
# Token count: ~450

# Checkpoint format (compressed)
{
    "phase": "CHECK",
    "round": 3,
    "vectors": {13 dimensions},
    "git_commit": "abc123",
    "summary": "After fixing auth bug, confidence high"
}
```

### 8.2 MCP Thin Wrapper

**Problem:** Implementing MCP tools directly duplicates CLI logic (~5,000 lines)

**Solution:** Route MCP tools to CLI via subprocess (500 lines, 90% reduction)

**Implementation:**
```python
# MCP tool definition
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    # Route to CLI
    cmd = ["empirica", name.replace("_", "-")]
    for key, value in arguments.items():
        cmd.extend([f"--{key}", str(value)])

    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)
```

**Benefits:**
- Single source of truth (CLI)
- 75% token reduction (CLI docs vs MCP schemas)
- No async bugs
- Easy testing

### 8.3 Epistemic Snapshots

**Problem:** Storing full assessments with rationales is verbose

**Solution:** Store vectors as floats, metadata as JSON

**Implementation:**
```sql
-- Traditional (verbose)
CREATE TABLE epistemic_assessments (
    ...
    engagement_score REAL,
    engagement_rationale TEXT,
    engagement_evidence TEXT,
    know_score REAL,
    know_rationale TEXT,
    know_evidence TEXT,
    ... (39 columns)
);

-- Compressed (efficient)
CREATE TABLE reflexes (
    ...
    engagement REAL,
    know REAL,
    ... (13 columns for scores)
    reflex_data TEXT  -- JSON with rationales
);
```

---

## 9. Data Access Patterns

### 9.1 Session Lifecycle

```python
# 1. Create session
db = SessionDatabase()
session_id = db.create_session(ai_id="claude-3.5-sonnet")

# 2. Add baseline checkpoint (PREFLIGHT)
logger = GitEnhancedReflexLogger(session_id)
logger.add_checkpoint(phase="PREFLIGHT", vectors={...})

# 3. Create goal
goal = Goal.create(
    objective="Fix authentication bug",
    success_criteria=[...],
    scope=ScopeVector(breadth=0.3, duration=0.2, coordination=0.1)
)
db.goals.save_goal(goal, session_id)

# 4. Create cascade
cascade_id = db.create_cascade(
    session_id=session_id,
    task="Review auth module",
    context={...},
    goal_id=goal.id
)

# 5. Add CHECK checkpoint
logger.add_checkpoint(phase="CHECK", vectors={...})

# 6. Complete cascade
db.complete_cascade(
    cascade_id=cascade_id,
    final_action="proceed",
    final_confidence=0.82,
    investigation_rounds=2,
    duration_ms=45000,
    engagement_gate_passed=True
)

# 7. Add final checkpoint (POSTFLIGHT)
logger.add_checkpoint(phase="POSTFLIGHT", vectors={...})

# 8. Complete goal
db.goals.update_goal_completion(goal.id, is_completed=True)

# 9. End session
db.end_session(session_id, avg_confidence=0.82, notes="Successful review")
```

### 9.2 Query Patterns

```python
# Get active sessions
active = db.sessions.list_sessions(
    ai_id="claude-3.5-sonnet",
    ended=False
)

# Get session cascades
cascades = db.cascades.get_session_cascades(session_id)

# Get session goals
goals = db.goals.get_session_goals(session_id)

# Get goal progress
goal = db.goals.get_goal(goal_id)
progress = goal.calculate_progress()
# Returns: {total_subtasks, completed, in_progress, ...}

# Get epistemic trajectory
vectors = db.vectors.get_session_vectors(session_id)
# Returns: List of reflexes ordered by timestamp

# Get project findings
findings = db.projects.get_project_findings(project_id)
unknowns = db.projects.get_project_unknowns(project_id, is_resolved=False)
```

### 9.3 Checkpoint Queries

```python
# Load last checkpoint
checkpoint = logger.get_last_checkpoint()

# Load specific phase
preflight = logger.get_checkpoint(phase="PREFLIGHT", round=1)

# List all checkpoints
checkpoints = logger.list_checkpoints(limit=10)

# Load from git notes (cross-session)
notes = GitNotesStorage(session_id)
checkpoint = notes.get_latest_note()

# Verify signed checkpoint
is_valid = logger.verify_checkpoint(checkpoint_id, public_key)
```

---

## 10. Error Handling & Resilience

### 10.1 Retry Policy

```python
class RetryPolicy:
    def __init__(self, max_retries=5, base_delay=0.1, max_delay=10.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay

    def execute(self, func):
        for attempt in range(self.max_retries):
            try:
                return func()
            except sqlite3.OperationalError as e:
                if "database is locked" in str(e):
                    delay = min(self.base_delay * (2 ** attempt), self.max_delay)
                    time.sleep(delay)
                else:
                    raise
        raise Exception("Max retries exceeded")
```

### 10.2 Session ID Validation

```python
@staticmethod
def _validate_session_id(session_id: str):
    """Validate session_id is a proper UUID"""
    try:
        uuid.UUID(session_id)
    except (ValueError, AttributeError, TypeError):
        raise ValueError(
            f"Invalid session_id: '{session_id}'. "
            f"Session IDs must be valid UUIDs"
        )
```

### 10.3 Git Fallback

```python
def __init__(self, session_id, enable_git_notes=True):
    self.git_available = self._check_git_available()

    if not self.git_available:
        logger.warning("Git not available. Falling back to SQLite only.")
        enable_git_notes = False

    self.enable_git_notes = enable_git_notes
```

---

## 11. Testing Strategy

### 11.1 Unit Tests

**Location:** `/home/user/empirica/tests/unit/`

**Coverage:**
- Canonical data structures
- Reflex logger
- Repository operations
- Schema validation
- Path resolution

### 11.2 Integration Tests

**Location:** `/home/user/empirica/tests/integration/`

**Coverage:**
- CLI commands
- MCP tools
- Database operations
- Git integration
- CASCADE workflow

### 11.3 Epistemic Checking Tests

**Location:** `/home/user/empirica/tests/epistemic_checking/`

**Purpose:** Test epistemic assessment accuracy

---

## 12. Performance Characteristics

### 12.1 Database Operations

**Session Creation:** < 10ms
**Checkpoint Storage:** < 50ms (SQLite + Git Notes)
**Query Session:** < 5ms (indexed)
**List Cascades:** < 10ms (typical session has 10-50)

### 12.2 Token Efficiency

**Git Notes vs SQLite:** 93% reduction (~450 vs ~6,500 tokens)
**MCP Thin Wrapper:** 75% reduction (CLI docs vs schemas)
**Compressed Checkpoints:** 80% reduction vs full assessments

### 12.3 Scalability

**SQLite Limits:**
- Max database size: 281 TB
- Max sessions: Limited by disk space
- Concurrent reads: Unlimited
- Concurrent writes: Serialized (lock)

**PostgreSQL (optional):**
- No practical limits
- Concurrent writes supported
- Network overhead

---

## 13. Security Considerations

### 13.1 Cryptographic Signing

**Purpose:** Verify epistemic state integrity

**Implementation:**
```python
class SigningPersona:
    def __init__(self, persona_id, private_key):
        self.persona_id = persona_id
        self.private_key = private_key  # Ed25519

    def sign(self, message: bytes) -> bytes:
        return self.private_key.sign(message)

# Usage
persona = SigningPersona.load(persona_id)
logger = GitEnhancedReflexLogger(session_id, signing_persona=persona)
checkpoint_id = logger.add_checkpoint(phase, vectors, metadata)
# → Creates signed git commit
```

### 13.2 Sentinel Gates

**Noetic Filters:** Prevent harmful investigation paths
**Axiologic Gates:** Validate actions against value constraints

**Example:**
```python
# Block exploit development
noetic_filter = NoeticFilter(
    filter_id="security-001",
    blocked_patterns=[r"exploit.*development"],
    action_on_match=GateAction.HALT_AND_AUDIT
)

# Require high confidence for production changes
axiologic_gate = AxiologicGate(
    gate_id="prod-safety",
    action_patterns=[r"deploy.*production"],
    required_vectors={"know": 0.90, "impact": 0.80},
    action_on_violation=GateAction.REQUIRE_HUMAN
)
```

### 13.3 Session Isolation

**Project-local databases:** Each git repo has its own `.empirica/sessions/sessions.db`
**No cross-contamination:** Sessions in different repos are isolated
**Workspace mode:** Shared database for multi-AI coordination (Docker)

---

## 14. Future Directions

### 14.1 Planned Enhancements

1. **Vector Embeddings:** Semantic search over checkpoints
2. **Real-time Dashboard:** Live session monitoring
3. **API Layer:** REST API for external tools
4. **Qdrant Integration:** Vector database for epistemic search
5. **Multi-modal Support:** Image/audio epistemic assessment

### 14.2 Experimental Features

1. **Bayesian Belief Tracking:** Evidence-based belief updates
2. **Drift Monitoring:** Long-term behavioral pattern detection
3. **Persona Emergence:** Automatic persona discovery from agent behavior
4. **Memory Gap Detection:** Identify knowledge gaps from conversation

---

## 15. Glossary

**CASCADE:** Metacognitive workflow (PREFLIGHT → THINK → INVESTIGATE → CHECK → ACT → POSTFLIGHT)
**Epistemic Vector:** Single dimension of self-awareness (0.0-1.0 scale)
**Reflex:** Epistemic assessment snapshot (13 vectors + metadata)
**Checkpoint:** Compressed epistemic state stored in git notes
**Goal:** Structured objective with success criteria and scope
**Persona:** Epistemic profile that seeds agent priors
**Sentinel:** Safety orchestrator with noetic filters and axiologic gates
**MCO:** Meta-Agent Configuration Object (dynamic config)
**BEADS:** Build-Eval-Adapt-Deploy-Ship (issue tracking integration)

---

## 16. Quick Reference

### 16.1 Essential CLI Commands

```bash
# Session management
empirica session-create --ai-id claude-3.5-sonnet
empirica sessions-list
empirica sessions-show --session-id {id}

# CASCADE workflow
empirica preflight-submit --session-id {id} --vectors {...}
empirica check --session-id {id} --findings [...] --unknowns [...] --confidence 0.7
empirica postflight-submit --session-id {id} --vectors {...}

# Goals
empirica goals-create --session-id {id} --objective "..." --scope {...}
empirica goals-list --session-id {id}
empirica goals-complete --goal-id {id}

# Checkpoints
empirica checkpoint-list --session-id {id}
empirica checkpoint-load --session-id {id} --phase PREFLIGHT
```

### 16.2 Essential Imports

```python
# Database
from empirica.data.session_database import SessionDatabase

# Canonical structures
from empirica.core.canonical.reflex_frame import VectorState, Action
from empirica.core.schemas.epistemic_assessment import EpistemicAssessmentSchema

# Git integration
from empirica.core.canonical.git_enhanced_reflex_logger import GitEnhancedReflexLogger

# Goals
from empirica.core.goals.types import Goal, ScopeVector

# Configuration
from empirica.config.path_resolver import get_empirica_root, get_session_db_path
from empirica.config.threshold_loader import get_threshold_config
```

### 16.3 Environment Variables

```bash
# Path overrides
export EMPIRICA_DATA_DIR=/path/to/.empirica
export EMPIRICA_SESSION_DB=/path/to/sessions.db
export EMPIRICA_WORKSPACE_ROOT=/workspace  # Docker mode

# MCP settings
export EMPIRICA_EPISTEMIC_MODE=true
export EMPIRICA_PERSONALITY=balanced_architect
```

---

## Conclusion

Empirica is a sophisticated metacognitive framework that combines:
- **Epistemic Self-Awareness:** 13-dimensional vector assessment
- **Token Efficiency:** Git notes, thin MCP wrapper, compressed checkpoints
- **Multi-Agent Orchestration:** Persona-based priors, merge strategies, Sentinel gates
- **Distributed Architecture:** Git-backed storage, cryptographic signing, cross-session capabilities
- **Clean Code Design:** Repository pattern, facade pattern, modular architecture

The system is designed for AI agents to track their own knowledge state, coordinate with other agents, and make metacognitive decisions about investigation and action.

**Total Lines of Code:** ~20,500 Python
**Total Commands:** 60+
**Total Database Tables:** 25+
**Epistemic Dimensions:** 13

For detailed implementation examples, see the source code in `/home/user/empirica/empirica/`.
