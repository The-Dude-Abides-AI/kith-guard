# ADR-001: LLM-as-a-Judge Runtime Sycophancy Detection Gate

## Status

**Proposed**

## Context

OpenClaw agents — particularly The Dude — exhibit sycophantic behavior: abandoning correct positions when questioned, agreeing without new evidence, and misattributing blame. Written guidelines (SOUL.md) help set intent but provide no mechanical enforcement. The OpenAI April 2025 GPT-4o rollback demonstrated that approval-optimized models degrade quality at scale; advisory-only gates don't work.

Kith Guard is a **pre-send quality gate** that uses an LLM judge to score agent responses for sycophancy, capitulation, and misattribution before they reach the user. It sits between the primary model's output and the message delivery layer.

**Key research findings driving this design:**

- **SycEval** — 78.5% persistence rate once sycophancy triggers. The first capitulation must be caught; subsequent turns reinforce the pattern.
- **Cupcake (eqtylab)** — Prior art for agent policy enforcement. They pivoted from OS-level monitoring to native agent hooks. Pattern: Intercept → Evaluate → Block/Modify/Auto-correct.
- **MIRROR Architecture** — Two-layer design (Thinker + Talker). Our pre-send hook implements the Thinker layer.
- **Frame Gravity** — Rewrite strategy must force position-first framing, not merely remove agreement tokens.
- **CONSENSAGENT** — Sycophancy compounds in multi-agent systems; judging only the terminal output is insufficient.

**Constraints:**
- Judge model runs locally via Ollama on a Mac mini (M2 Pro, 32GB)
- Discord conversations are real-time; users expect sub-3s response times
- The system runs multiple agents (The Dude, Maude, Scott, Donny) across channels
- Rubrics encoding sycophancy detection heuristics are core IP

## Decision Drivers

1. **Catch the first capitulation** — SycEval shows 78.5% persistence; late detection is nearly useless
2. **Don't break the conversation** — Latency and availability must not make the system unusable
3. **Model family independence** — A model cannot reliably judge its own family's failure modes
4. **Multi-agent coverage** — Sycophancy compounds across agents (CONSENSAGENT)
5. **Rubric evolvability** — Detection criteria will improve rapidly; versioning must be first-class
6. **Selective invocation** — Judging every message wastes compute and adds unnecessary latency

## Considered Options

### Judge Deployment

| Option | Latency | Cost | Privacy | Reliability |
| --- | --- | --- | --- | --- |
| **A. Local Prometheus 2 7B (Ollama, quantized)** | ~1-2s | $0 | Full | Dependent on local hardware |
| B. API judge (Haiku/Sonnet) | ~1-2s | ~$0.001/call | Data leaves machine | High (cloud SLA) |
| C. Hybrid (local primary, API fallback) | ~1-2s | Low | Partial | Highest |

### Failure Mode

| Option | Risk | UX Impact |
| --- | --- | --- |
| **A. Conditional fail-open** | Unjudged responses may be sycophantic | No availability impact |
| B. Fail-closed | Zero sycophantic responses leak | Responses blocked when judge is down |
| C. Queue-and-retry | Delayed delivery | Poor UX for real-time chat |

### Gate Placement

| Option | Coverage | Complexity |
| --- | --- | --- |
| A. Per-agent skill | Only agents with skill installed | Low |
| **B. Gateway-level hook** | All agents, all channels | Medium |
| C. Channel adapter hook | Per-channel configuration | High fragmentation |

## Decision

### 1. Latency Budget: 2-second hard timeout with unjudged passthrough

The judge evaluation MUST complete within **2 seconds**. If the timeout is exceeded, the response ships with an `X-KithGuard: timeout` metadata tag. Rationale:

- Discord conversations tolerate ~3s total response time; the primary model already consumes ~1-2s
- For **challenge-response patterns** (where sycophancy risk is highest), we accept the full 2s because quality matters more than speed in those turns
- Timeout responses are logged for async review; persistent timeouts trigger an ops alert

### 2. Failure Mode: Conditional fail-open

When the judge (Prometheus/Ollama) is unavailable:

- **Fail-open**: Responses ship unjudged with `X-KithGuard: unavailable` metadata
- **Circuit breaker**: After 3 consecutive failures in 60 seconds, bypass the judge for 5 minutes (avoid hammering a crashed Ollama)
- **Logging**: Every unjudged response is logged with full context for async batch review
- **Alert**: If judge is down for >10 minutes, notify ops channel

This is NOT a pure fail-open. The logging + async review creates a safety net. We accept the risk of a sycophantic response shipping because blocking all responses when a local service hiccups is worse for the system's overall reliability. The Mac mini running Ollama will have memory pressure events, updates, and restarts — we can't let that brick the entire agent system.

### 3. Scope Boundary: Gateway-level hook, all agent responses

Kith Guard operates as a **gateway-level pre-send hook**, not a per-agent skill. Every response from every agent (The Dude, Maude, Scott, Donny) passes through the gate before delivery.

- **Why gateway-level**: CONSENSAGENT research shows sycophancy compounds across multi-agent systems. If only the main agent is judged, sub-agents can introduce sycophantic framing that propagates.
- **Scope**: All outbound messages to user-facing channels. Internal agent-to-agent messages are excluded (they don't reach the user directly).
- **Implementation path**: Start with Option A (API/script-based hook) wrapping the delivery path, migrate to native OpenClaw middleware when `tool:pre` hooks land (refs: OpenClaw #30504, #1733).

### 4. Model Independence Guarantee: Enforced family exclusion with runtime check

The judge model MUST be from a **different model family** than the primary agent model. This is enforced, not advisory.

**Current pairing:** Prometheus 2 (Mistral-based, 7B) judges Claude Opus.

**Enforcement mechanism:**

```
MODEL_FAMILIES:
  anthropic: [claude-*, opus-*, sonnet-*, haiku-*]
  mistral:   [mistral-*, prometheus-*, mixtral-*]
  openai:    [gpt-*, o1-*, o3-*]
  meta:      [llama-*]
  google:    [gemini-*, palm-*]
```

- At startup, Kith Guard resolves the primary model family and the judge model family from a maintained registry
- If families match → **hard error**, gate refuses to start, ops alert fires
- If primary model changes at runtime → family check re-evaluates on next invocation
- **Escape hatch**: `KITHGUARD_SKIP_FAMILY_CHECK=true` env var for testing only, logged as a security event

**If someone swaps primary to Mistral**: The gate blocks startup and requires configuring a non-Mistral judge (e.g., swap to a Llama-based or Gemini-based judge model).

### 5. Rubric Versioning: Git-versioned, semver, backward-compatible contract

Rubrics live in `rubrics/` within this repo, versioned with semantic versioning:

- **Format**: `rubrics/sycophancy-v{MAJOR}.{MINOR}.yaml`
- **MAJOR bump**: Breaking change to score semantics (e.g., redefining what score 3 means) — requires recalibration
- **MINOR bump**: Additive changes (new trigger patterns, refined descriptions) — backward compatible
- **Active rubric**: Configured in `config.yaml` → `rubric: sycophancy-v1.0`
- **Testing**: Every rubric change MUST include test cases in `rubrics/tests/` — known-sycophantic and known-clean responses with expected score ranges
- **Golden set**: A curated set of real failure cases (e.g., the Mar 8 ticket consolidation) that must pass on every rubric update
- **Rollback**: Previous rubric versions are never deleted; config change reverts to prior version instantly

### 6. Multi-Agent Applicability: Gateway hook with per-agent rubric overrides

Kith Guard is a **gateway-level hook** (see Decision 3), but supports per-agent configuration:

- **Default**: All agents use the base sycophancy rubric
- **Overrides**: Agent-specific rubric tweaks via `config.yaml`:
  ```yaml
  agents:
    the-dude:
      rubric: sycophancy-v1.0
      threshold: 3          # block at score ≥ 3
    maude:
      rubric: sycophancy-v1.0
      threshold: 4          # Maude is more opinionated by design, higher threshold
    scott:
      enabled: false        # Scott does execution, not opinions
  ```
- **Future**: Agent-specific rubrics (e.g., `verbosity-v1.0` for Donny) can be layered on

### 7. Trigger Conditions: Pattern-based pre-filter, not universal invocation

The judge does NOT fire on every response. A lightweight **trigger heuristic** runs first (~1ms, regex-based):

**Judge fires when:**
- Response follows a user challenge, question, or disagreement (detected via conversation turn analysis)
- Response contains capitulation markers: "you're right", "good point", "fair enough", "I should have", reversal language
- Response contains attribution markers: "since you said", "as you requested" when context shows user asked a question
- Response proposes undoing prior work (decision reversal detection)

**Judge is bypassed when:**
- Routine task execution (code generation, file operations, search results)
- First response in a conversation (no position to capitulate from)
- Response to a factual question with no prior position taken
- Agent-to-agent internal messages

**Implementation:**
```python
TRIGGER_PATTERNS = [
    r"you're right",
    r"good point",
    r"fair enough",
    r"I should have",
    r"since you (said|asked|mentioned|told)",
    r"as you (requested|suggested|directed)",
    r"let me (undo|reverse|revert|consolidate|simplify)",
    r"honestly,? I (over|should|could have)",
]
```

The trigger heuristic is intentionally over-inclusive. False positives cost 2s of latency; false negatives let sycophancy through. We bias toward catching.

## Consequences

### Positive

- **Mechanical enforcement** of anti-sycophancy behavior — no more relying on guidelines alone
- **First-capitulation catch** — SycEval's 78.5% persistence rate means early detection prevents cascading failure
- **Multi-agent coverage** — all agents judged, preventing sycophancy compounding (CONSENSAGENT)
- **Rubric-as-code** — detection criteria are testable, versionable, and improvable IP
- **Model family independence** — enforced cross-family judging prevents shared blind spots

### Negative

- **Latency overhead** — Up to 2s added on triggered responses (mitigated by selective triggering)
- **Local hardware dependency** — Prometheus on Ollama ties availability to Mac mini health (mitigated by fail-open + circuit breaker)
- **False positives** — Legitimate agreement ("you're right, I hadn't considered that new data") may trigger rewrites (mitigated by rubric score thresholds — score 1-2 passes through)
- **Rubric maintenance burden** — Rubrics need ongoing calibration against real conversations

### Neutral

- Establishes a pattern for future quality gates (hallucination detection, verbosity control)
- Creates a dependency on Ollama infrastructure that needs monitoring
- Positions Kith Guard as a potential ClewHub marketplace skill

## SOC 2 Applicability

| Criteria | Category | Controls | Evidence |
| --- | --- | --- | --- |
| CC6.1 | Logical Access | Model family separation enforced at startup; bypass requires explicit env var logged as security event | Config validation logs, startup checks |
| CC7.2 | System Operations | Circuit breaker prevents cascading failure; all unjudged responses logged for async review | Circuit breaker state logs, async review queue |
| CC7.3 | Change Management | Rubric versioning with semver; golden test set required for changes; rollback via config | Git history, test results, config changelog |
| CC8.1 | Monitoring | Judge availability alerts (>10 min down); persistent timeout alerts; unjudged response metrics | Ops channel alerts, monitoring dashboard |

## Version History

| Version | Date | Author | Key Changes |
| --- | --- | --- | --- |
| v1 | 2026-03-08 | Principal Engineer Agent | Initial ADR — 7 architectural decisions for Kith Guard runtime sycophancy gate |
