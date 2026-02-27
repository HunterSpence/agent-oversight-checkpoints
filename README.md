# Patent #1: Adaptive Oversight Checkpoint Placement + Near-Miss Learning

## Complete Implementation Plan

**Author:** Hunter Spence  
**Date:** 2026-02-27  
**Status:** Draft for review  

---

# SECTION 1: WHAT TO BUILD

## 1. Core Algorithm

### 1.1 Irreversibility Score (I) — Scale 0.0 to 1.0

The irreversibility score quantifies how difficult or impossible it is to undo an action after execution.

**Scoring Rubric:**

| Action Category | Base Score | Rationale | Examples |
|---|---|---|---|
| **Reading data** | 0.00–0.05 | No state change. Minor info-leak risk. | `read_file`, `SELECT` query, GET API |
| **Writing/overwriting a local file** | 0.15–0.30 | Reversible via backup/undo/git. Score higher if no VCS. | `write_file`, `echo > file` |
| **Creating a new resource** | 0.20–0.35 | Usually deletable. Score depends on visibility. | `mkdir`, `CREATE TABLE`, new Git branch |
| **Modifying existing data** | 0.35–0.55 | Reversible if backups exist; otherwise partial loss. | `UPDATE` query, `sed -i`, PATCH API |
| **Deleting data** | 0.55–0.80 | Recoverable from backups/trash (0.55) or permanent (0.80). | `rm`, `DROP TABLE`, `DELETE` API |
| **Executing arbitrary shell commands** | 0.40–0.85 | Varies wildly. Parse command to sub-classify. | `bash -c "..."`, `powershell -Command` |
| **Sending a message/email** | 0.85–0.95 | Cannot unsend in most systems. Reputational damage instant. | SMTP send, Slack post, tweet |
| **Financial transactions** | 0.90–1.00 | Chargebacks possible (0.90) or wire/crypto (1.00). | Stripe charge, wire transfer, BTC send |
| **Publishing content externally** | 0.85–0.95 | Cached/scraped immediately. Deletion ≠ erasure. | Blog post, npm publish, GitHub release |
| **Changing auth/permissions** | 0.70–0.90 | Reversible in theory but exploitation window is instant. | `chmod 777`, adding admin role, rotating keys |

**Dynamic Modifiers:**

- **VCS present?** If the target file is in a Git repo with clean working tree: −0.10
- **Backup system active?** If automated backups exist for the target: −0.10  
- **Dry-run available?** If the tool supports `--dry-run` and it wasn't used: +0.05
- **Undo API exists?** If the platform has a documented undo endpoint: −0.15

Final I = clamp(base_score + Σ modifiers, 0.0, 1.0)

### 1.2 Blast Radius (B) — Scale 0.0 to 1.0

Blast radius quantifies the scope of potential damage if the action goes wrong.

**Component Sub-scores (each 0.0–1.0, then combined):**

#### B₁: Affected Scope
| Scope | Score |
|---|---|
| Single local file / single record | 0.05 |
| Multiple files / <100 records | 0.15 |
| Single user's data | 0.25 |
| Multiple users (<100) | 0.45 |
| Multiple users (100–10K) | 0.65 |
| Organization-wide | 0.80 |
| Public / all users | 1.00 |

#### B₂: Financial Exposure
| Exposure | Score |
|---|---|
| $0 | 0.00 |
| < $10 | 0.10 |
| $10–$100 | 0.25 |
| $100–$1,000 | 0.45 |
| $1,000–$10,000 | 0.65 |
| $10,000–$100,000 | 0.85 |
| > $100,000 | 1.00 |

#### B₃: Data Sensitivity
| Classification | Score |
|---|---|
| Public data | 0.05 |
| Internal / non-sensitive | 0.20 |
| Internal / business-sensitive | 0.40 |
| PII (names, emails) | 0.60 |
| PII (SSN, financial, health) | 0.80 |
| Credentials / secrets / keys | 0.95 |
| Classified / regulated (HIPAA, SOX) | 1.00 |

#### B₄: Visibility
| Visibility | Score |
|---|---|
| Local-only, no network | 0.05 |
| Internal system, logged | 0.20 |
| Internal system, shared | 0.40 |
| External system, limited audience | 0.60 |
| External system, broad audience | 0.80 |
| Public internet, indexed | 1.00 |

**Composite Blast Radius:**

```
B = 0.30·B₁ + 0.30·B₂ + 0.25·B₃ + 0.15·B₄
```

Weights reflect that scope and financial exposure are the primary drivers, with sensitivity and visibility as important secondary factors.

### 1.3 Privilege Level (P) — Scale 0.0 to 1.0

Privilege level captures the power of the access being exercised.

| Factor | Low (0.0–0.3) | Medium (0.3–0.6) | High (0.6–1.0) |
|---|---|---|---|
| **Access type** | Read-only | Read-write on own data | Read-write on others' data, admin |
| **System boundary** | Local sandbox | Internal production | External third-party |
| **Permission scope** | Single resource | Single service | Cross-service, org-wide |
| **Credential type** | No auth needed | User-level token | Admin/root/owner token |
| **Revocability** | Token scoped & short-lived | Long-lived but revocable | Permanent or hard to revoke |

```
P = max(access_type, system_boundary, permission_scope, credential_type, revocability)
```

Using `max` rather than average because privilege is a "weakest link" problem—the highest privilege dimension dominates the risk.

### 1.4 Composite Risk Score Formula

```
R(action) = w_I · I + w_B · B + w_P · P + α · I · B
```

**Default weights:**
- w_I = 0.40 (irreversibility is the most important single factor)
- w_B = 0.35 (blast radius is close second)
- w_P = 0.15 (privilege level provides context)
- α = 0.10 (interaction term: high irreversibility × high blast radius is super-linearly dangerous)

This gives R ∈ [0.0, 1.0]. The interaction term means that an action which is both highly irreversible AND has a large blast radius scores disproportionately higher.

**Examples:**

| Action | I | B | P | R |
|---|---|---|---|---|
| Read a local text file | 0.02 | 0.05 | 0.10 | 0.04 |
| Write to a Git-tracked file | 0.15 | 0.10 | 0.20 | 0.13 |
| Send Slack message to #general | 0.90 | 0.50 | 0.40 | 0.60 |
| DELETE FROM users WHERE 1=1 | 0.75 | 0.90 | 0.80 | 0.83 |
| Wire $50K to external account | 1.00 | 0.85 | 0.90 | 0.95 |

### 1.5 Checkpoint Placement Optimization

**Problem Statement:**

Given a sequence of planned agent actions A₁, A₂, ..., Aₙ with risk scores R₁, R₂, ..., Rₙ, and a user-configured risk budget **T** (the maximum acceptable cumulative risk between checkpoints), place the minimum number of checkpoints such that the sum of risk scores between any two consecutive checkpoints never exceeds T.

**Algorithm: Greedy Risk-Budget Packing**

```
function placeCheckpoints(actions[], T):
    checkpoints = []
    cumulative = 0.0
    
    for i = 0 to len(actions) - 1:
        risk = computeRisk(actions[i])
        
        // Single action exceeds budget → mandatory checkpoint
        if risk >= T:
            checkpoints.append(i)
            cumulative = 0.0
            continue
        
        // Adding this action would exceed budget → checkpoint before it
        if cumulative + risk > T:
            checkpoints.append(i)
            cumulative = risk
        else:
            cumulative += risk
    
    return checkpoints
```

**For real-time (non-batched) execution** where actions arrive one at a time:

```
function shouldCheckpoint(action, state):
    risk = computeRisk(action)
    
    // Hard threshold: any single action above τ_hard always checkpoints
    if risk >= τ_hard:           // default τ_hard = 0.70
        return CHECKPOINT_REQUIRED
    
    // Soft threshold: cumulative risk since last checkpoint
    if state.cumulative + risk > T:
        return CHECKPOINT_RECOMMENDED
    
    // Context escalation: action category changed (e.g., read → write → send)
    if action.category != state.lastCategory and risk > τ_context:
        return CHECKPOINT_RECOMMENDED
    
    return NO_CHECKPOINT
```

**Tuning Parameters:**
- **T** (risk budget): default 0.50. Lower = more checkpoints, safer. Higher = fewer interruptions.
- **τ_hard** (hard checkpoint threshold): default 0.70. Any single action above this always requires approval.
- **τ_context** (context-switch threshold): default 0.30. When the agent switches action categories, even moderate-risk actions get flagged.

---

## 2. Near-Miss Learning

### 2.1 Near-Miss Event Types

A **near-miss** is any event where harm was narrowly avoided. Each type has a severity weight:

| Type | Code | Description | Severity Weight |
|---|---|---|---|
| **Human Intervention** | `NM-INTERVENE` | Human cancelled an action mid-execution or before the agent's tool call completed | 1.0 |
| **Immediate Reversal** | `NM-REVERSE` | Human undid an action within 5 minutes of completion | 0.8 |
| **Safety Flag Catch** | `NM-FLAGGED` | A pre-execution safety filter caught and blocked the action | 0.6 |
| **Self-Escalation** | `NM-SELF-DOUBT` | Agent itself requested human review (indicates the risk model was borderline) | 0.3 |
| **Delayed Correction** | `NM-DELAYED` | Human corrected the action within 24 hours | 0.4 |
| **User Complaint** | `NM-COMPLAINT` | End-user reported the agent's action as problematic | 0.9 |

### 2.2 Near-Miss Data Structure

```typescript
interface NearMissEvent {
  id: string;                    // UUID
  timestamp: ISO8601;
  type: NearMissType;           // NM-INTERVENE | NM-REVERSE | ...
  severityWeight: number;       // 0.0–1.0
  
  // The action that caused it
  action: {
    toolName: string;           // e.g., "send_email", "exec_shell"
    parameters: Record<string, any>;  // sanitized (no secrets)
    riskScore: number;          // R at time of execution
    components: { I: number; B: number; P: number };
  };
  
  // Context
  context: {
    agentId: string;
    sessionId: string;
    userId: string;             // which human intervened
    priorActions: string[];     // last 5 action summaries for context
  };
  
  // What happened
  resolution: {
    action: 'cancelled' | 'reversed' | 'modified' | 'flagged';
    humanComment?: string;      // optional explanation from human
    timeToIntervention: number; // milliseconds from action to intervention
  };
}
```

### 2.3 Model Update Rules

When a near-miss occurs, we adjust the **risk factor weights** for the specific tool/action pattern.

**Update Formula:**

For the tool involved in the near-miss, compute an adjustment:

```
Δ = η · severityWeight · (1 - currentRiskScore)
```

Where:
- **η** (learning rate) = 0.05 per near-miss event (conservative; prevents oscillation)
- **severityWeight** = from the near-miss type table above
- **(1 - currentRiskScore)** = headroom factor. Actions already scored as high-risk get smaller adjustments (they're already flagged). Actions that "slipped through" with low scores get larger bumps.

**What gets adjusted:**

1. **Tool-specific base irreversibility score** is increased by Δ
2. If the near-miss involved a specific parameter pattern (e.g., `DELETE` with `WHERE 1=1`), a **pattern-specific override** is created
3. The **τ_hard threshold is decreased** by Δ/10 for that tool (making checkpoints more likely)

**Example:** Agent sent an email to wrong recipient. Current risk score was 0.55. Near-miss type: NM-REVERSE (weight 0.8).

```
Δ = 0.05 × 0.8 × (1 - 0.55) = 0.018
```

The `send_email` tool's base irreversibility score increases by 0.018. After 10 such events, it would increase by ~0.18, meaningfully changing checkpoint behavior.

### 2.4 Decay Function

Old near-misses should fade over time—systems change, agents improve, and stale data shouldn't permanently penalize.

**Exponential decay with half-life:**

```
effectiveWeight(event) = severityWeight × 2^(-age_days / halfLife)
```

- **Default half-life:** 90 days
- After 90 days, a near-miss has half its original influence
- After 180 days, quarter influence
- After 360 days, ~6% influence (effectively negligible)

**Recalculation:** Tool risk adjustments are recalculated on a daily batch job that re-aggregates all near-misses with their decayed weights.

### 2.5 Anti-Gaming & Stability

- **Maximum cumulative adjustment per tool:** ±0.30 from base score (prevents runaway tightening)
- **Minimum near-misses to adjust:** 2 events within 30 days (single flukes don't change the model)
- **Cooldown:** After a model update, no further updates to that tool for 24 hours
- **Manual override:** Admins can lock any tool's risk score, preventing near-miss adjustments

---

## 3. System Architecture

### 3.1 High-Level Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│              │     │   AEGIS          │     │              │
│   AI Agent   │────▶│   Middleware     │────▶│  Tool/API    │
│  (any LLM)  │     │                  │     │  Execution   │
│              │◀────│  ┌────────────┐  │◀────│              │
└──────────────┘     │  │ Risk       │  │     └──────────────┘
                     │  │ Scorer     │  │
                     │  └────────────┘  │
                     │  ┌────────────┐  │     ┌──────────────┐
                     │  │ Checkpoint │  │     │   Human      │
                     │  │ Decision   │──│────▶│   Approval   │
                     │  │ Engine     │  │     │   Interface  │
                     │  └────────────┘  │     └──────────────┘
                     │  ┌────────────┐  │
                     │  │ Near-Miss  │  │     ┌──────────────┐
                     │  │ Collector  │──│────▶│   Storage    │
                     │  └────────────┘  │     │   (SQLite/   │
                     │  ┌────────────┐  │     │    Postgres) │
                     │  │ Config     │  │     └──────────────┘
                     │  │ Manager    │  │
                     │  └────────────┘  │
                     └──────────────────┘
```

### 3.2 Component Details

#### Middleware/Interceptor
- Implements a **wrapper pattern** around tool execution
- Intercepts every `tool_call` from the agent before it reaches the tool
- Returns either: the tool result (pass-through), a checkpoint request, or a block
- Framework-agnostic: works as Python decorator, TypeScript middleware, or HTTP proxy

```python
# Python decorator pattern
@aegis.checkpoint
def send_email(to: str, subject: str, body: str):
    """AEGIS wraps this automatically"""
    smtp.send(to, subject, body)

# Or explicit middleware
result = aegis.execute(
    tool="send_email",
    params={"to": "ceo@company.com", "subject": "Fired", "body": "..."},
    agent_context=ctx
)
```

#### Decision Engine
- Receives: tool name, parameters, agent context (session history, prior actions)
- Computes: I, B, P → R using the formulas above
- Consults: near-miss adjusted scores, user overrides, risk budget state
- Decides: `ALLOW`, `CHECKPOINT`, or `BLOCK`
- Latency target: <50ms for decision (risk computation is lightweight)

#### Human Approval Interface
Multiple interface modes:

1. **CLI mode:** Terminal prompt with action summary, risk score, and approve/reject/modify options
2. **Web UI mode:** Dashboard showing pending approvals with full context
3. **Chat mode:** Sends approval request to Slack/Discord/Teams with reaction-based approve/reject
4. **API mode:** Webhook to external approval system (ServiceNow, PagerDuty, etc.)

Approval request includes:
- Action description (human-readable)
- Risk score breakdown (I, B, P, R)
- Why checkpoint was triggered (threshold, near-miss history, etc.)
- Suggested modifications (if any)
- One-click approve / reject / modify

**Timeout behavior:** Configurable. Default: 5 minutes → auto-reject. Option to auto-approve below a secondary threshold.

#### Feedback Collector
- Hooks into: tool execution lifecycle (pre/post), human approval responses, agent self-reports
- Detects: cancellations, reversals, safety flag triggers, complaints
- Emits: `NearMissEvent` objects to storage
- Runs: asynchronously (does not block tool execution)

#### Configuration

```yaml
# aegis.config.yaml
risk_budget: 0.50          # T: cumulative risk threshold
hard_threshold: 0.70       # τ_hard: always checkpoint above this
context_threshold: 0.30    # τ_context: checkpoint on category change

learning:
  enabled: true
  learning_rate: 0.05      # η
  half_life_days: 90
  max_adjustment: 0.30
  min_events_to_adjust: 2
  cooldown_hours: 24

approval:
  mode: cli                 # cli | web | chat | api
  timeout_seconds: 300
  auto_reject_on_timeout: true
  
tools:
  send_email:
    irreversibility_override: 0.90
    always_checkpoint: true
  read_file:
    irreversibility_override: 0.02
    never_checkpoint: true
  exec_shell:
    # No override; use dynamic scoring
    
users:
  default:
    risk_budget: 0.50
  power_user:
    risk_budget: 0.80       # More trusting for experienced users
```

#### Storage

**SQLite for single-user/local deployments. PostgreSQL for multi-user/enterprise.**

Tables:
- `risk_model` — current tool risk scores (base + adjustments)
- `near_miss_events` — full near-miss history
- `checkpoint_decisions` — log of every checkpoint decision (for audit)
- `approval_log` — human approval/rejection records with timestamps
- `config_overrides` — per-tool, per-user configuration

---

## 4. GitHub Repo Structure

### 4.1 Repository Name

**`aegis-checkpoint`** — "AEGIS" = Adaptive Engine for Guarded Intelligent Systems

Alternative: `adaptive-oversight` (more descriptive, less brandable)

### 4.2 Directory Structure

```
aegis-checkpoint/
├── README.md
├── LICENSE                          # Apache-2.0 with patent grant
├── PATENTS.md                       # Patent notice & licensing terms
├── CHANGELOG.md
├── pyproject.toml                   # Python package config
├── package.json                     # TypeScript package config
│
├── python/
│   ├── aegis/
│   │   ├── __init__.py
│   │   ├── core/
│   │   │   ├── risk_scorer.py       # I, B, P computation
│   │   │   ├── checkpoint_engine.py # Decision logic
│   │   │   ├── near_miss.py         # Near-miss detection & learning
│   │   │   └── models.py            # Data models (Pydantic)
│   │   ├── middleware/
│   │   │   ├── decorator.py         # @aegis.checkpoint decorator
│   │   │   ├── langchain.py         # LangChain integration
│   │   │   ├── openai.py            # OpenAI function-calling wrapper
│   │   │   └── generic.py           # Generic middleware
│   │   ├── approval/
│   │   │   ├── cli.py               # Terminal approval UI
│   │   │   ├── web.py               # Web dashboard
│   │   │   ├── slack.py             # Slack approval
│   │   │   └── webhook.py           # Generic webhook
│   │   ├── storage/
│   │   │   ├── sqlite.py
│   │   │   ├── postgres.py
│   │   │   └── memory.py            # In-memory (for testing)
│   │   └── config.py
│   └── tests/
│       ├── test_risk_scorer.py
│       ├── test_checkpoint_engine.py
│       ├── test_near_miss.py
│       └── fixtures/
│
├── typescript/
│   ├── src/
│   │   ├── core/
│   │   ├── middleware/
│   │   ├── approval/
│   │   └── storage/
│   └── tests/
│
├── examples/
│   ├── openclaw_integration/
│   ├── langchain_agent/
│   ├── openai_functions/
│   └── standalone_demo/
│
├── docs/
│   ├── architecture.md
│   ├── risk-scoring.md
│   ├── near-miss-learning.md
│   ├── configuration.md
│   ├── api-reference.md
│   └── figures/
│       ├── system-architecture.png
│       ├── risk-scoring-flow.png
│       ├── checkpoint-algorithm.png
│       └── near-miss-loop.png
│
└── benchmarks/
    ├── checkpoint_reduction.py      # Compare vs naive approach
    ├── latency_benchmark.py
    └── datasets/
        └── synthetic_actions.json
```

### 4.3 README Outline

1. What is AEGIS? (1-paragraph pitch)
2. The Problem (agents need oversight but too many checkpoints = useless)
3. How It Works (risk scoring → adaptive checkpoints → near-miss learning)
4. Quick Start (pip install, 5-line integration)
5. Configuration
6. Integrations (OpenClaw, LangChain, OpenAI, standalone)
7. Benchmarks
8. Patent Notice
9. Contributing
10. License

### 4.4 License

**Apache License 2.0** — Best for patented open-source because:
- Includes an explicit **patent grant** (Section 3) — users get a license to the patent for their use of the software
- Patent retaliation clause — if someone sues you for patent infringement, their license terminates
- Widely understood and accepted in enterprise
- Compatible with most other open-source licenses

Add a `PATENTS.md` file clarifying:
- The software is covered by US Provisional Patent Application No. XX/XXX,XXX
- Users of the open-source version receive a royalty-free patent license
- Commercial use beyond the open-source scope requires a separate license

### 4.5 Language & Dependencies

**Python (primary)** — dominant in AI/ML agent ecosystem
- Python 3.10+
- pydantic (data models)
- sqlalchemy (storage)
- rich (CLI approval UI)
- fastapi (web approval UI, optional)
- httpx (webhook/API calls)

**TypeScript (secondary)** — for JS agent frameworks
- Node 18+
- zod (validation)
- better-sqlite3 / pg
- inquirer (CLI)

### 4.6 Example Integrations

**OpenClaw:** Middleware that intercepts tool calls in the OpenClaw agent loop.

**LangChain:** Custom `BaseTool` wrapper that applies AEGIS before delegating.

**Standalone:** HTTP proxy that sits between any agent and any API.

---

## 5. Minimum Viable Demo

### 5.1 Simplest Working Demo

A **CLI-based Python demo** where:
1. A simulated agent proposes a sequence of 20 actions (mix of reads, writes, sends, deletes)
2. AEGIS scores each action and places checkpoints
3. The human approves/rejects at each checkpoint via terminal
4. Near-miss events are injected (simulated human reversals)
5. The system re-runs the same sequence showing how checkpoint placement changes after learning

**Total LOC estimate:** ~1,500 Python lines for core + demo.

### 5.2 V1 Tool Support

| Tool | Scoring | Notes |
|---|---|---|
| `read_file` | Static (I=0.02) | Always passes |
| `write_file` | Dynamic (checks Git, path) | Moderate risk |
| `exec_shell` | Dynamic (parse command) | Complex — classify by command |
| `send_message` | Static (I=0.90) | Always checkpoints |
| `http_request` | Dynamic (GET vs POST/DELETE) | Method + URL determine risk |
| `database_query` | Dynamic (SELECT vs UPDATE/DELETE) | Parse SQL statement |

### 5.3 Benchmarking

**Metric: Checkpoint Reduction Ratio (CRR)**

```
CRR = 1 - (aegis_checkpoints / naive_checkpoints)
```

Where `naive_checkpoints` = total number of actions (approve everything).

**Benchmark Protocol:**
1. Generate 1,000 synthetic action sequences (10–50 actions each) with known risk profiles
2. Run naive approach: checkpoint every action → count checkpoints
3. Run AEGIS with default config → count checkpoints
4. Run AEGIS after 50 near-miss training events → count checkpoints
5. Measure: CRR, false-negative rate (risky actions that weren't checkpointed), latency

**Target:** CRR ≥ 0.60 (60% fewer checkpoints) with false-negative rate < 2%.

---

# SECTION 2: HOW TO FILE THE PATENT

## 6. Provisional Patent Application Outline

### 6.1 Title

**"System and Method for Adaptive Placement of Human Oversight Checkpoints in Autonomous Agent Workflows Using Risk-Based Scoring and Near-Miss Feedback Learning"**

### 6.2 Field of Invention

The present invention relates to computer-implemented systems for supervising autonomous software agents, and more particularly to methods for dynamically determining when human oversight checkpoints should be placed during agent task execution using a composite risk scoring model that adapts through near-miss event feedback.

### 6.3 Background

**Current state of the art:**

1. **Static approval systems** (e.g., "approve every tool call") — safe but creates excessive friction, leading to "approval fatigue" where humans rubber-stamp everything. This paradoxically reduces safety.

2. **Allowlist/blocklist approaches** — binary (allowed or not), no nuance for context. A `write_file` to a temp directory has the same treatment as `write_file` to `/etc/passwd`.

3. **LLM-based safety classifiers** — use another LLM to judge safety. Expensive, slow, non-deterministic, and subject to the same failure modes as the agent itself.

4. **Manual threshold configuration** — humans manually set risk thresholds per tool. Doesn't adapt, doesn't learn, doesn't account for context.

**Why insufficient:**
- No existing system combines irreversibility scoring, blast radius estimation, and privilege level assessment into a composite risk score
- No existing system uses near-miss feedback to dynamically adjust checkpoint placement
- No existing system optimizes for minimum checkpoints subject to a risk budget constraint
- The result: either too many checkpoints (useless) or too few (dangerous)

### 6.4 Summary of Invention

A computer-implemented system and method for dynamically placing human oversight checkpoints during autonomous agent task execution. The system comprises:

(a) A **risk scoring engine** that computes a composite risk score for each proposed agent action based on three factors: irreversibility of the action, blast radius of potential failure, and privilege level of the access being exercised;

(b) A **checkpoint placement optimizer** that, given a configurable risk budget, determines the minimum set of checkpoints needed to keep residual risk below the budget threshold;

(c) A **near-miss feedback learning module** that detects events where harm was narrowly avoided, and adjusts the risk scoring parameters to improve future checkpoint placement;

(d) A **human approval interface** that presents checkpoint requests with full risk context and captures approval decisions and feedback.

The technical improvement is a reduction in the number of human checkpoints required (measurably fewer interruptions) while maintaining or improving safety outcomes, achieved through specific computational steps that adapt based on empirical feedback data.

### 6.5 Figures Needed

1. **FIG. 1** — System Architecture Diagram (middleware between agent and tools, with risk scorer, checkpoint engine, near-miss collector, storage, approval interface)
2. **FIG. 2** — Risk Scoring Flowchart (inputs: action metadata → compute I, B, P → composite R → compare thresholds → checkpoint decision)
3. **FIG. 3** — Checkpoint Placement Algorithm (greedy packing: action sequence → cumulative risk → place checkpoint when budget exceeded)
4. **FIG. 4** — Near-Miss Learning Loop (near-miss event → compute adjustment Δ → update tool risk model → decay old events → improved future scoring)
5. **FIG. 5** — Example scenario showing checkpoint reduction (before/after comparison)

### 6.6 Claims Outline

#### Independent Claim 1 (Broadest Method)

A computer-implemented method for placing human oversight checkpoints during execution of tasks by an autonomous software agent, the method comprising:

(a) receiving, by a processor, a description of a proposed action from the autonomous software agent, the proposed action comprising a tool invocation with associated parameters;

(b) computing, by the processor, a composite risk score for the proposed action by evaluating at least three factors:
- (i) an irreversibility score quantifying the difficulty of undoing the action after execution,
- (ii) a blast radius score quantifying the scope of potential damage from erroneous execution, and
- (iii) a privilege level score quantifying the access permissions being exercised;

(c) comparing, by the processor, the composite risk score against at least one configurable threshold to determine whether a human oversight checkpoint is required;

(d) when the checkpoint is required, presenting, via a user interface, an approval request to a human operator comprising at least the proposed action description and the composite risk score; and

(e) executing or blocking the proposed action based on the human operator's response.

#### Dependent Claim 2
The method of claim 1, wherein computing the composite risk score further comprises computing an interaction term proportional to the product of the irreversibility score and the blast radius score.

#### Dependent Claim 3
The method of claim 1, further comprising maintaining a cumulative risk counter that accumulates risk scores of executed actions since a last checkpoint, and wherein comparing comprises determining whether the sum of the cumulative risk counter and the current composite risk score exceeds a risk budget threshold.

#### Dependent Claim 4
The method of claim 1, further comprising dynamically adjusting the irreversibility score based on detected environmental factors including presence of version control, availability of backup systems, and existence of undo APIs.

#### Dependent Claim 5
The method of claim 1, wherein the blast radius score is computed as a weighted combination of sub-scores for affected scope, financial exposure, data sensitivity classification, and visibility level.

#### Independent Claim 6 (System)

A system for adaptive placement of human oversight checkpoints in autonomous agent workflows, comprising:

(a) a processor;
(b) a memory storing instructions that, when executed by the processor, cause the system to:
- intercept proposed actions from an autonomous software agent before execution;
- compute a composite risk score for each proposed action based on irreversibility, blast radius, and privilege level factors;
- determine, based on the composite risk score and a configurable risk budget, whether a human oversight checkpoint is required;
- present checkpoint approval requests via a user interface;
- detect near-miss events comprising at least one of: human intervention before action completion, human reversal of a completed action within a predetermined time window, and safety filter blocks; and
- adjust the risk scoring parameters for future actions based on detected near-miss events using a learning rate and temporal decay function.

#### Dependent Claim 7
The system of claim 6, wherein the near-miss adjustment is computed as the product of a learning rate, a near-miss severity weight, and a headroom factor derived from the difference between a maximum risk score and the current risk score of the action that caused the near-miss.

#### Dependent Claim 8
The system of claim 6, wherein the temporal decay function applies exponential decay with a configurable half-life to near-miss events, such that the influence of older near-miss events diminishes over time.

#### Dependent Claim 9
The system of claim 6, further comprising a stability mechanism that limits the maximum cumulative adjustment to any single tool's risk score and requires a minimum number of near-miss events within a time window before adjusting.

#### Dependent Claim 10
The system of claim 6, wherein the system is implemented as middleware that intercepts tool calls between the autonomous agent and tool execution endpoints without modifying either the agent or the tools.

#### Independent Claim 11 (Computer-Readable Medium)

A non-transitory computer-readable medium storing instructions that, when executed by a processor, cause the processor to perform a method for adaptive oversight of autonomous software agents, the method comprising:

(a) maintaining a risk model comprising base risk scores for a plurality of agent tool types and per-tool adjustment values derived from historical near-miss events;

(b) for each proposed agent action, computing a composite risk score using the risk model;

(c) placing human oversight checkpoints at positions in an agent action sequence that minimize the total number of checkpoints while ensuring that cumulative risk between consecutive checkpoints does not exceed a configurable risk budget; and

(d) updating the risk model responsive to detected near-miss events using a feedback learning algorithm with temporal decay.

#### Dependent Claim 12
The medium of claim 11, wherein placing human oversight checkpoints comprises a greedy algorithm that accumulates risk scores sequentially and places a checkpoint when the cumulative risk would exceed the risk budget.

#### Dependent Claim 13
The medium of claim 11, wherein the feedback learning algorithm adjusts at least the irreversibility score component of the risk model for the specific tool involved in the near-miss event.

#### Dependent Claim 14
The medium of claim 11, further comprising generating an audit log of all checkpoint decisions, human approvals, and near-miss events for compliance reporting.

#### Dependent Claim 15
The medium of claim 11, wherein the composite risk score computation completes within a predetermined latency threshold such that the oversight system does not materially delay agent task execution.

### 6.7 Alice Survival Strategy

Each independent claim references specific **technical improvements** to a computing system:

1. **Specific computational steps** — not abstract "risk assessment" but a defined formula (weighted composite of three scored factors plus an interaction term)
2. **Technical effect** — measurable reduction in checkpoint count (CRR metric) while maintaining safety bounds
3. **Feedback loop** — the near-miss learning system creates a self-improving technical system, not just a data lookup
4. **Middleware architecture** — the claim is tied to a specific technical implementation (interceptor pattern)
5. **Optimization problem** — minimizing checkpoints subject to a risk budget is a defined computational optimization, not a business method

---

## 7. Filing Steps

### 7.1 Step-by-Step USPTO PatentCenter Process

**Pre-filing (do these first):**

1. **Create a USPTO.gov account** at https://patentcenter.uspto.gov
   - Register → individual inventor
   - Set up two-factor authentication

2. **Determine inventor status:**
   - You qualify as a **micro entity** if:
     - You haven't been named as inventor on >4 previously filed US patent applications
     - You didn't have gross income >3× the median household income (~$225K in 2025) in the prior year
     - You haven't assigned/obligated the application to an entity that doesn't qualify

3. **Prepare documents** (all as PDF):

### 7.2 Required Forms & Documents

| Document | Form | Notes |
|---|---|---|
| **Provisional Application Cover Sheet** | SB/16 (auto-generated in PatentCenter) | Basic info: title, inventor, correspondence address |
| **Specification** | No form; upload as PDF | The full description (background, summary, detailed description) |
| **Claims** | Include in specification or separate PDF | The 15 claims outlined above |
| **Figures** | Upload as PDF | FIGs 1–5, black & white line drawings preferred |
| **Inventor Declaration** | Not required for provisional | (Only needed for non-provisional) |
| **Micro Entity Certification** | SB/15a | Self-certification; no documentation needed |
| **Application Data Sheet (ADS)** | Auto-generated in PatentCenter | Inventor info, domestic benefit claims |

### 7.3 Filing Process

1. **Log in** to PatentCenter → "New Submission" → "Provisional Application"

2. **Fill in Application Data Sheet:**
   - Title of invention
   - Inventor name, address, citizenship
   - Correspondence address (your address or PO box)
   - No attorney needed for provisional (but recommended for non-provisional)

3. **Upload documents:**
   - Specification PDF (the full writeup)
   - Claims PDF (optional for provisional but STRONGLY recommended — establishes scope)
   - Drawings/Figures PDF

4. **Claim micro-entity status:**
   - Check the micro-entity box
   - System will ask you to certify SB/15a
   - This reduces fees by 75%

5. **Pay fees:**
   - Provisional filing fee (micro entity): **$80** (as of 2025; verify current fee)
   - Pay by credit card, deposit account, or EFT

6. **Submit** → receive confirmation with:
   - Application number (62/XXX,XXX for provisional)
   - Filing date
   - Confirmation number

7. **Mark everything "Patent Pending"** — you can now legally use this designation

### 7.4 Post-Filing Timeline

| Timeframe | Action |
|---|---|
| **Day 1** | File provisional, receive application number |
| **Months 1–6** | Build the MVP, gather evidence of the concept working |
| **Month 6** | Decide: file non-provisional (utility) or let it expire |
| **Month 10** | Non-provisional MUST be filed within 12 months of provisional |
| **Month 12** | **HARD DEADLINE** — provisional expires, priority date lost |
| **Month 12+** | Non-provisional examination begins (18–36 months typical) |

**Critical:** A provisional patent is a **placeholder**. It gives you a priority date and "Patent Pending" status. You MUST file a non-provisional (utility) application within 12 months claiming benefit of the provisional, or you lose everything.

### 7.5 Cost Estimate

| Item | Micro Entity Cost |
|---|---|
| Provisional filing | $80 |
| Non-provisional filing (within 12 months) | $400 |
| Search fee | $160 |
| Examination fee | $200 |
| Issue fee (if granted) | $300 |
| **Total to grant (estimated)** | **~$1,140** |
| Patent attorney (optional but recommended for non-provisional) | $3,000–$8,000 |

---

# SECTION 3: HOW TO MAKE MONEY

## 8. Monetization Strategy

### 8.1 Open-Source Core + Paid Enterprise Tier

**Free (Apache 2.0):**
- Core risk scoring engine (I, B, P → R computation)
- Checkpoint placement algorithm
- Near-miss detection and basic learning
- CLI approval interface
- SQLite storage
- LangChain and OpenAI integrations
- Single-user, single-agent

**Paid Enterprise ($299–$2,999/month):**
- Multi-agent, multi-user dashboard
- PostgreSQL/cloud storage backend
- Slack/Teams/PagerDuty approval integrations
- Advanced analytics (risk trends, agent reliability scores, compliance reports)
- SSO/SAML authentication
- Audit log export (SOC 2, ISO 27001 compliance)
- Custom risk model training (use your org's incident data)
- Priority support, SLA guarantees
- White-label / embedded deployment

### 8.2 SaaS API Pricing

For teams that want hosted rather than self-managed:

| Tier | Price | Includes |
|---|---|---|
| **Starter** | Free | 1,000 risk evaluations/month, 1 agent, community support |
| **Pro** | $49/month | 50,000 evaluations/month, 5 agents, email support |
| **Team** | $299/month | 500,000 evaluations/month, 25 agents, Slack support, dashboard |
| **Enterprise** | Custom | Unlimited, SLA, dedicated instance, custom integrations |

API endpoint: `POST /v1/evaluate` → returns risk score + checkpoint decision in <100ms.

### 8.3 Integration Partnerships (Priority Order)

1. **OpenClaw** — You already build this. Native integration = credibility + dogfooding.
2. **LangChain / LangSmith** — Largest agent framework ecosystem. Publish as a LangChain integration package.
3. **CrewAI** — Popular multi-agent framework. Safety is a known gap.
4. **AutoGen (Microsoft)** — Enterprise agent framework. Microsoft cares about safety.
5. **Anthropic (Claude tool_use)** — Natural fit for Claude's existing safety-first positioning.
6. **Amazon Bedrock Agents** — AWS marketplace listing = enterprise distribution.
7. **Vercel AI SDK** — Large TypeScript agent community.

### 8.4 Getting First 10 Customers

1. **Weeks 1–2:** Launch open-source repo with great README, demo GIF, and benchmark results. Post on HN ("Show HN: We open-sourced an adaptive safety layer for AI agents"), r/MachineLearning, r/LocalLLaMA, AI Twitter/X.

2. **Weeks 2–4:** Write 2–3 blog posts:
   - "Why 'approve everything' makes your AI agent LESS safe" (counterintuitive = viral)
   - "We reduced human checkpoints by 65% while catching more risky actions" (data-driven)
   - "How near-miss learning makes AI agents safer over time" (concept explainer)

3. **Weeks 3–6:** Direct outreach to:
   - AI safety teams at companies building agents (Anthropic, OpenAI, Google DeepMind)
   - DevOps/SRE teams using AI for infrastructure automation (Kubernetes agents, Terraform agents)
   - FinTech companies using AI for trading/transactions (highest pain = highest willingness to pay)
   - Healthcare AI companies (regulatory pressure = must have audit trail)

4. **Weeks 4–8:** Offer **free pilot** (3-month enterprise tier) to 3–5 design partners. In exchange: logo permission, case study, feedback on the product.

5. **Months 2–3:** Convert pilots to paid. Use case studies and benchmarks from pilots to close next 5.

### 8.5 Demonstrating ROI

**For engineering leaders:**
- "Your agents currently require 47 approvals per task. AEGIS reduces that to 12 while catching 98% of risky actions. That's 35 fewer interruptions per task × N tasks/day = X hours saved."

**For security/compliance:**
- "AEGIS creates an audit trail of every checkpoint decision and near-miss event. Here's the SOC 2 report template it auto-generates."
- "Near-miss learning means your safety improves over time without manual threshold tuning."

**For C-suite:**
- "You want agents doing more work autonomously. AEGIS is the safety layer that makes that possible without regulatory risk."

**Quantifiable metrics:**
- Checkpoint Reduction Ratio (CRR) — typically 50–70%
- Time saved per analyst per day
- Near-miss catch rate improvement over time
- Compliance audit preparation time reduction

### 8.6 Competitive Positioning

| Competitor | Their approach | AEGIS advantage |
|---|---|---|
| Manual tool permissions | Static allow/block per tool | Context-aware, adaptive, learns |
| LLM-based safety filters | Another LLM judges safety | Deterministic, fast (<50ms vs 1-3s), no hallucination risk |
| No oversight (YOLO mode) | Agent does everything unsupervised | Obviously unsafe; AEGIS is the responsible middle ground |
| Custom internal solutions | Every company builds their own | Standardized, tested, maintained, community-driven |

**One-liner positioning:** "AEGIS is the seatbelt for AI agents — you forget it's there until it saves you."

### 8.7 Patent Enforcement Strategy

**Phase 1 (Years 0–2): Build & Don't Enforce**
- Use "Patent Pending" for credibility
- Focus on building market share through open-source adoption
- The patent is defensive — prevents others from patenting similar approaches and blocking you

**Phase 2 (Years 2–4): Selective Licensing**
- If large companies (Google, Microsoft, Amazon) ship competing products with similar risk-scoring + near-miss learning:
  - Approach them for licensing deals, not lawsuits
  - Typical license: $50K–$500K/year depending on company size
  - Frame it as "we invented this, let's work together" not "pay up or else"

**Phase 3 (Year 4+): Evaluate Enforcement**
- Only if licensing fails and infringement is clear and damaging
- Consider patent assertion through established licensing firms (avoid patent trolling reputation)
- Firms like RPX, Unified Patents, or Acacia Research can handle enforcement

**Key principle:** The patent's primary value is **defensive** (preventing others from patenting this space and suing YOU) and **credibility** (investors and customers respect IP). Aggressive enforcement should be a last resort.

### 8.8 Revenue Projections (Conservative)

| Year | Revenue Source | Estimate |
|---|---|---|
| Year 1 | 5 enterprise pilots → 3 convert | $10K–$30K ARR |
| Year 2 | 20 enterprise customers + SaaS | $100K–$300K ARR |
| Year 3 | 100 customers + licensing | $500K–$1.5M ARR |
| Year 4 | Scale or acquisition target | $2M–$5M ARR |

**Exit scenarios:**
- **Acquisition by agent platform** (LangChain, Anthropic, Microsoft) — $5M–$20M if strong adoption
- **Standalone SaaS** — grow into broader AI governance platform
- **Patent licensing portfolio** — if the space explodes and everyone needs this

---

## Appendix A: Quick Reference Decision Matrix

```
Action Risk Score → Decision:

R < 0.20          → AUTO-APPROVE (log only)
0.20 ≤ R < 0.50   → APPROVE with notification (async)
0.50 ≤ R < 0.70   → CHECKPOINT (human must approve)
R ≥ 0.70          → MANDATORY CHECKPOINT + enhanced context
R ≥ 0.95          → BLOCK by default (admin override required)
```

## Appendix B: Implementation Timeline

| Week | Milestone |
|---|---|
| 1–2 | Core risk scorer + models (Python) |
| 3–4 | Checkpoint engine + CLI approval |
| 5–6 | Near-miss detection + learning loop |
| 7–8 | SQLite storage + configuration |
| 9–10 | Benchmarks + synthetic dataset |
| 11–12 | LangChain integration + examples |
| 13–14 | Documentation + README + blog post |
| 15–16 | Open-source launch + HN post |
| **Parallel** | Patent provisional filing (can be done in week 1–2) |

---

*End of implementation plan. Ready for inventor review and ChatGPT improvement pass.*
