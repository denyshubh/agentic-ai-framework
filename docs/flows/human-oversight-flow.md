# Human Oversight Flow

## Overview

This document describes the human oversight governance flow within the EAAGF. It covers the gate trigger and approval workflow, the oversight mode selection logic, the gate state machine, timeout and escalation handling, and the emergency stop procedure.

Human oversight ensures that humans retain meaningful control over high-stakes agent decisions. The framework provides four oversight modes with increasing levels of human involvement, a gate-based approval workflow with notification and escalation, and an emergency stop capability for immediate intervention.

### Applicable Requirements

| Requirement | Description |
|---|---|
| 5.1 | Support four oversight modes: FULL_AUTO, SUPERVISED, APPROVAL_REQUIRED, HUMAN_IN_LOOP |
| 5.2 | Default T4 agents to APPROVAL_REQUIRED; block FULL_AUTO without AI Governance Team authorization |
| 5.3 | Notify designated approver within 30 seconds when a gate is triggered |
| 5.4 | Escalate to secondary approver on gate timeout (default: 4 hours) |
| 5.8 | Trigger a gate on plan deviation exceeding a configurable threshold |
| 5.9 | Support emergency stop that terminates all agent actions and revokes credentials |
| 5.10 | Emit EMERGENCY_STOP audit event and notify AI Governance Team within 60 seconds |

---

## Oversight Mode Selection Logic

The oversight mode determines which agent actions trigger a Human_Oversight_Gate. The mode is declared in the agent's Conformance_Profile and enforced by the Policy_Engine as part of the authorization decision flow.

```mermaid
flowchart TD
    START([Agent deployed]) --> TIER{Agent Risk_Tier?}

    TIER -->|T1| T1_DEFAULT[Default: HUMAN_IN_LOOP<br/>Team MAY change to any mode]
    TIER -->|T2| T2_DEFAULT[Default: SUPERVISED<br/>Team MAY change to any mode]
    TIER -->|T3| T3_DEFAULT[Default: APPROVAL_REQUIRED<br/>Team MAY change to any mode<br/>including FULL_AUTO]
    TIER -->|T4| T4_DEFAULT[Default: APPROVAL_REQUIRED<br/>Req 5.2]

    T4_DEFAULT --> T4_CHECK{Requested mode<br/>is FULL_AUTO?}
    T4_CHECK -->|No| T4_OK[Set requested mode<br/>SUPERVISED / APPROVAL_REQUIRED /<br/>HUMAN_IN_LOOP]
    T4_CHECK -->|Yes| T4_AUTH{AI Governance Team<br/>authorization present?}
    T4_AUTH -->|No| T4_REJECT[Reject: FULL_AUTO not permitted<br/>for T4 without authorization]
    T4_AUTH -->|Yes| T4_VALIDATE{Authorization valid?<br/>Named member, time-bound<br/>max 90 days}
    T4_VALIDATE -->|No| T4_REJECT
    T4_VALIDATE -->|Yes| T4_FULL_AUTO[Set FULL_AUTO<br/>Record authorization in Audit_Log]

    T1_DEFAULT --> MODE_SET([Oversight mode active])
    T2_DEFAULT --> MODE_SET
    T3_DEFAULT --> MODE_SET
    T4_OK --> MODE_SET
    T4_FULL_AUTO --> MODE_SET

    T4_FULL_AUTO --> EXPIRY_CHECK{Authorization<br/>expired?}
    EXPIRY_CHECK -->|Yes| REVERT[Revert to APPROVAL_REQUIRED<br/>Emit T4_OVERSIGHT_REVERTED event]
    REVERT --> MODE_SET
```

### Oversight Mode Gate Trigger Matrix

The following table defines which action types trigger a Human_Oversight_Gate under each oversight mode (Requirement 5.1):

| Action Type | FULL_AUTO | SUPERVISED | APPROVAL_REQUIRED | HUMAN_IN_LOOP |
|---|---|---|---|---|
| Read-only data access | No gate | No gate | No gate | Gate |
| Write operation (internal) | No gate | Gate | Gate | Gate |
| Write operation (external) | No gate | Gate | Gate | Gate |
| External connection | No gate | No gate | Gate | Gate |
| Agent delegation (A2A) | No gate | No gate | Gate | Gate |
| Multi-step operation | No gate | No gate | Gate | Gate |
| Trivial read (e.g., health check) | No gate | No gate | No gate | Gate |

---

## Gate State Machine

The Human_Oversight_Gate follows a state machine that governs the lifecycle of every gate instance from trigger to resolution (Requirements 5.3, 5.4).

```mermaid
stateDiagram-v2
    [*] --> PENDING : Gate triggered<br/>Notify approver within 30s

    PENDING --> APPROVED : Primary approver approves
    PENDING --> REJECTED : Primary approver rejects
    PENDING --> ESCALATED : Primary timeout expires<br/>(default: 4h T1/T2, 2h T3, 1h T4)

    ESCALATED --> APPROVED : Secondary approver approves
    ESCALATED --> REJECTED : Secondary approver rejects
    ESCALATED --> AUTO_REJECTED : Secondary timeout expires

    APPROVED --> [*] : Resume agent execution
    REJECTED --> [*] : Action denied
    AUTO_REJECTED --> [*] : Action denied

    note right of PENDING
        Agent execution paused.
        Approver notified via
        email / Slack / Teams / PagerDuty.
    end note

    note right of ESCALATED
        GATE_TIMEOUT_ESCALATION
        audit event emitted.
        Secondary approver notified.
    end note

    note right of AUTO_REJECTED
        GATE_AUTO_REJECTED
        audit event emitted.
        Both approvers notified.
    end note
```

### State Definitions

| State | Description | Entry Condition | Exit Condition |
|---|---|---|---|
| PENDING | Gate triggered, awaiting primary approver response. | Gate triggered by oversight mode, plan deviation, or restricted data write. | Approver responds (→ APPROVED or REJECTED) or timeout expires (→ ESCALATED). |
| APPROVED | Gated action approved by a human approver. | Primary or secondary approver approves. | Terminal state. Agent execution resumes. |
| REJECTED | Gated action rejected by a human approver. | Primary or secondary approver rejects. | Terminal state. Action denied. |
| ESCALATED | Primary approver did not respond within timeout. Escalated to secondary approver. | Primary timeout expires without response. | Secondary approver responds or secondary timeout expires (→ AUTO_REJECTED). |
| AUTO_REJECTED | Neither approver responded within their timeouts. Action automatically rejected. | Secondary timeout expires without response. | Terminal state. Action denied. |

### State Transition Rules

1. Terminal states (APPROVED, REJECTED, AUTO_REJECTED) are immutable — once reached, no further transitions occur.
2. An emergency stop forces any active gate to REJECTED immediately, regardless of current state.
3. Each state transition produces an audit event.
4. The total maximum gate duration (primary + secondary timeout) SHALL NOT exceed a configurable maximum (default: 24 hours).

---

## End-to-End Human Oversight Flow

The following diagram shows the complete gate workflow from trigger through notification, approval/rejection, escalation, and resolution.

```mermaid
flowchart TD
    START([Action triggers<br/>Human Oversight Gate]) --> REASON{Gate trigger<br/>reason?}

    REASON -->|Oversight mode<br/>requires gate<br/>Req 5.1| PAUSE[Pause agent execution<br/>Preserve execution state]
    REASON -->|T3/T4 write on<br/>restricted data| PAUSE
    REASON -->|Plan deviation<br/>exceeds threshold<br/>Req 5.8| PAUSE

    PAUSE --> EMIT_TRIGGER[Emit GATE_TRIGGERED<br/>audit event]
    EMIT_TRIGGER --> NOTIFY[Notify designated approver<br/>within 30 seconds<br/>via configured channels<br/>Req 5.3]

    NOTIFY --> NOTIFY_CHANNELS{Notification<br/>channels}
    NOTIFY_CHANNELS --> CH_EMAIL[Email]
    NOTIFY_CHANNELS --> CH_SLACK[Slack]
    NOTIFY_CHANNELS --> CH_TEAMS[Microsoft Teams]
    NOTIFY_CHANNELS --> CH_PD[PagerDuty]

    CH_EMAIL --> NOTIFY_CONTENT
    CH_SLACK --> NOTIFY_CONTENT
    CH_TEAMS --> NOTIFY_CONTENT
    CH_PD --> NOTIFY_CONTENT

    NOTIFY_CONTENT[Notification includes:<br/>• Agent name, UUID, Risk_Tier<br/>• Action type and target resource<br/>• Gate trigger reason<br/>• Link to approval interface<br/>• Timeout duration and escalation target]

    NOTIFY_CONTENT --> WAIT{Primary approver<br/>response?}

    WAIT -->|Approved| EMIT_APPROVED[Emit GATE_RESOLVED<br/>decision: APPROVED]
    EMIT_APPROVED --> RESUME[Resume agent execution]
    RESUME --> DONE([Gate resolved])

    WAIT -->|Rejected| EMIT_REJECTED[Emit GATE_RESOLVED<br/>decision: REJECTED]
    EMIT_REJECTED --> DENY[Block action<br/>Return denial to agent]
    DENY --> DONE

    WAIT -->|No response<br/>within timeout<br/>Req 5.4| ESCALATE[Transition to ESCALATED state]
    ESCALATE --> EMIT_ESCALATION[Emit GATE_TIMEOUT_ESCALATION<br/>audit event]
    EMIT_ESCALATION --> NOTIFY_SECONDARY[Notify secondary approver<br/>within 30 seconds]

    NOTIFY_SECONDARY --> WAIT2{Secondary approver<br/>response?}

    WAIT2 -->|Approved| EMIT_APPROVED
    WAIT2 -->|Rejected| EMIT_REJECTED

    WAIT2 -->|No response<br/>within secondary timeout| AUTO_REJECT[Transition to AUTO_REJECTED]
    AUTO_REJECT --> EMIT_AUTO[Emit GATE_AUTO_REJECTED<br/>audit event]
    EMIT_AUTO --> NOTIFY_BOTH[Notify both primary and<br/>secondary approvers of<br/>auto-rejection]
    NOTIFY_BOTH --> DENY
```

### Timeout Configuration by Risk Tier

| Risk Tier | Primary Timeout | Secondary Timeout | Maximum Gate Duration |
|---|---|---|---|
| T1 | 4 hours | 4 hours | 24 hours |
| T2 | 4 hours | 4 hours | 24 hours |
| T3 | 2 hours | 2 hours | 12 hours |
| T4 | 1 hour | 1 hour | 8 hours |

---

## Plan Deviation Gate (Requirement 5.8)

When an agent's action sequence deviates from its declared task plan beyond a configurable threshold, the Governance_Controller automatically triggers a Human_Oversight_Gate.

```mermaid
flowchart TD
    A[Agent executes action] --> B[Compare action against<br/>declared task plan]
    B --> C{Action matches<br/>declared plan?}
    C -->|Yes| D[Continue execution<br/>Update deviation counter]
    C -->|No| E[Increment deviation<br/>counter]
    E --> F{Cumulative deviation<br/>exceeds threshold?}
    F -->|No| D
    F -->|Yes| G[Trigger PLAN_DEVIATION gate]
    G --> H[Pause agent execution]
    H --> I[Emit PLAN_DEVIATION<br/>audit event]
    I --> J[Notify approver with:<br/>• Declared plan<br/>• Actual action sequence<br/>• Deviation percentage<br/>• Deviating action details]
    J --> K{Approver decision?}
    K -->|Approve and continue| L[Resume with modified plan]
    K -->|Approve and update plan| M[Update declared plan<br/>Resume execution]
    K -->|Reject| N[Deny deviating action<br/>Pause agent for inspection]
```

### Default Deviation Thresholds by Risk Tier

| Risk Tier | Default Deviation Threshold |
|---|---|
| T1 | 30% |
| T2 | 25% |
| T3 | 20% |
| T4 | 10% |

Agents in FULL_AUTO mode are exempt from plan deviation detection but deviation metrics are still logged for post-hoc analysis.

---

## Emergency Stop Procedure (Requirements 5.9, 5.10)

The emergency stop is a single authorized command that immediately terminates all agent actions and revokes all credentials. It is the most severe intervention available.

```mermaid
flowchart TD
    START([Emergency stop<br/>command issued]) --> AUTH{Authorized role?}

    AUTH -->|AI Governance Team<br/>member| PROCEED[Execute emergency stop]
    AUTH -->|Agent owning<br/>team lead| PROCEED
    AUTH -->|Operator with<br/>EMERGENCY_STOP permission| PROCEED
    AUTH -->|Unauthorized| DENY[Deny command<br/>Emit unauthorized<br/>access event]

    PROCEED --> STEP1[1. Terminate all agent actions<br/>Cancel pending, in-flight,<br/>and queued actions<br/>within 5 seconds]

    STEP1 --> STEP2[2. Revoke ALL credentials<br/>Session credentials,<br/>delegated credentials,<br/>platform tokens]

    STEP2 --> STEP3[3. Block new actions<br/>Agent cannot initiate<br/>any actions until<br/>re-authorized by AI Gov Team]

    STEP3 --> STEP4[4. Preserve execution state<br/>Task queue, completed actions,<br/>pending actions, reasoning context,<br/>active data — for investigation]

    STEP4 --> CANCEL_GATES{Active Human<br/>Oversight Gates?}
    CANCEL_GATES -->|Yes| FORCE_REJECT[Force all active gates<br/>to REJECTED state]
    CANCEL_GATES -->|No| EMIT_EVENT

    FORCE_REJECT --> EMIT_EVENT[Emit EMERGENCY_STOP<br/>audit event within 5 seconds<br/>Req 5.10]

    EMIT_EVENT --> NOTIFY_GOV[Notify AI Governance Team<br/>within 60 seconds<br/>Req 5.10]

    NOTIFY_GOV --> NOTIFY_CONTENT[Notification includes:<br/>• Agent name, UUID, Risk_Tier, platform<br/>• Operator who initiated stop<br/>• Reason for emergency stop<br/>• Link to preserved execution state<br/>• Link to incident response runbook]

    NOTIFY_CONTENT --> NOTIFY_CHANNELS{Notification channels}
    NOTIFY_CHANNELS --> SLACK_TEAMS[Slack / Teams<br/>AI Gov Team primary channel]
    NOTIFY_CHANNELS --> PD{T3 or T4 agent?}
    PD -->|Yes| PAGERDUTY[PagerDuty incident]
    PD -->|No| COMPLETE

    SLACK_TEAMS --> COMPLETE
    PAGERDUTY --> COMPLETE

    COMPLETE --> RETRY_CHECK{Notification<br/>delivered?}
    RETRY_CHECK -->|Yes| DONE([Emergency stop complete<br/>Agent requires AI Gov Team<br/>re-authorization to resume])
    RETRY_CHECK -->|No| RETRY[Retry every 30 seconds<br/>Emit EMERGENCY_STOP_NOTIFICATION_FAILURE]
    RETRY --> RETRY_CHECK
```

### Emergency Stop Characteristics

| Property | Value |
|---|---|
| Action termination SLA | Within 5 seconds |
| Credential revocation | All credentials (session, delegated, platform) |
| Audit event emission | Within 5 seconds |
| AI Governance Team notification | Within 60 seconds |
| Reversibility | Non-reversible. Requires AI Gov Team re-authorization. |
| Gate override | Forces all active gates to REJECTED |
| State preservation | Full execution state preserved for investigation |

---

## Audit Event Coverage

Every path through the human oversight flow produces audit events for complete observability:

| Flow Path | Audit Events Emitted |
|---|---|
| Gate triggered → Approved | `GATE_TRIGGERED`, `GATE_NOTIFICATION_SENT`, `GATE_RESOLVED` (APPROVED) |
| Gate triggered → Rejected | `GATE_TRIGGERED`, `GATE_NOTIFICATION_SENT`, `GATE_RESOLVED` (REJECTED) |
| Gate triggered → Timeout → Escalated → Approved | `GATE_TRIGGERED`, `GATE_NOTIFICATION_SENT`, `GATE_TIMEOUT_ESCALATION`, `GATE_RESOLVED` (APPROVED) |
| Gate triggered → Timeout → Escalated → Rejected | `GATE_TRIGGERED`, `GATE_NOTIFICATION_SENT`, `GATE_TIMEOUT_ESCALATION`, `GATE_RESOLVED` (REJECTED) |
| Gate triggered → Timeout → Escalated → Auto-rejected | `GATE_TRIGGERED`, `GATE_NOTIFICATION_SENT`, `GATE_TIMEOUT_ESCALATION`, `GATE_AUTO_REJECTED` |
| Plan deviation gate | `PLAN_DEVIATION`, `GATE_TRIGGERED`, `GATE_NOTIFICATION_SENT`, `GATE_RESOLVED` |
| Emergency stop | `EMERGENCY_STOP`, `EMERGENCY_STOP_NOTIFICATION_SENT` (or `EMERGENCY_STOP_NOTIFICATION_FAILURE`) |

---

## Cross-References

- [Human Oversight Standard](../eaagf-specification/06-human-oversight-standard.md) — Normative oversight rules, gate state machine, and emergency stop procedure
- [Authorization Standard](../eaagf-specification/04-authorization-standard.md) — Policy evaluation that triggers gates for T3/T4 restricted data writes
- [Observability Standard](../eaagf-specification/05-observability-standard.md) — Audit event schema and emission SLAs
- [Agent Action Flow](./agent-action-flow.md) — How gates integrate into the action governance flow
- [Incident Response Flow](./incident-response-flow.md) — Post-emergency-stop incident handling
