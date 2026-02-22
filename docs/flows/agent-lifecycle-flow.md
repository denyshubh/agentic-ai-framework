# Agent Lifecycle Flow

## Overview

This document describes the agent lifecycle flow within the EAAGF, covering the four-stage lifecycle state machine (DEVELOPMENT → STAGING → PRODUCTION → DECOMMISSIONED), transition gate prerequisites, re-validation enforcement and automatic downgrade, deprecation notification, and decommission dependency verification.

The lifecycle is a binding governance control — agents can only transition between stages by satisfying the prerequisites enforced by the Governance_Controller. Direct stage-skipping (e.g., DEVELOPMENT → PRODUCTION) is not permitted.

### Applicable Requirements

| Requirement | Description |
|---|---|
| 10.1 | Define and enforce four lifecycle stages: DEVELOPMENT, STAGING, PRODUCTION, DECOMMISSIONED |
| 10.2 | DEVELOPMENT → STAGING requires: Conformance_Profile, passing tests, Risk_Tier assignment |
| 10.3 | STAGING → PRODUCTION requires: security scan, data governance review, AI Gov approval (T3/T4) |
| 10.5 | Support canary deployment pattern for production promotion |
| 10.6 | Notify dependent agents and systems 30 days before deprecation |
| 10.7 | Automatically downgrade to STAGING if re-validation is overdue |
| 10.11 | Verify dependency migration before completing decommission |

---

## Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> DEVELOPMENT : Agent registered<br/>(Req 10.1)

    DEVELOPMENT --> STAGING : Conformance_Profile complete<br/>+ tests pass<br/>+ Risk_Tier assigned<br/>(Req 10.2)

    STAGING --> PRODUCTION : Security scan attestation<br/>+ data governance review<br/>+ (T3/T4: AI Gov approval)<br/>(Req 10.3)

    PRODUCTION --> STAGING : Re-validation overdue<br/>OR compliance drift<br/>OR failed conformance tests<br/>(Req 10.7)

    PRODUCTION --> DECOMMISSIONED : Decommission authorized<br/>+ dependencies migrated<br/>(Req 10.11)

    STAGING --> DECOMMISSIONED : Decommission authorized<br/>+ dependencies migrated

    DEVELOPMENT --> DECOMMISSIONED : Decommission authorized

    DECOMMISSIONED --> [*] : Records retained 7 years

    note right of DEVELOPMENT
        No production data access.
        Sandbox/development resources only.
    end note

    note right of STAGING
        Pre-production validation.
        Synthetic/anonymized data only.
    end note

    note right of PRODUCTION
        Full production access.
        Continuous monitoring.
        Subject to re-validation.
    end note

    note right of DECOMMISSIONED
        All credentials revoked.
        No actions permitted.
        Read-only record access.
    end note
```

### Lifecycle Stage Summary

| Stage | Permitted Actions | Data Access | Monitoring |
|---|---|---|---|
| DEVELOPMENT | Tool calls against sandbox resources, conformance testing | Development/sandbox only | Basic |
| STAGING | Tool calls against staging resources, integration testing, security scanning | Synthetic/anonymized only | Standard |
| PRODUCTION | All declared actions within Conformance_Profile scope | Production data | Continuous |
| DECOMMISSIONED | None (read-only record access for audit) | None | Archived |

---

## DEVELOPMENT → STAGING Transition (Requirement 10.2)

```mermaid
flowchart TD
    A([Transition request:<br/>DEVELOPMENT → STAGING]) --> B{Conformance_Profile<br/>complete and valid?}

    B -->|No| C[DENY transition<br/>Return: Conformance_Profile<br/>incomplete or invalid]
    B -->|Yes| D{Conformance_Test_Suite<br/>passing?<br/>Run within last 24 hours}

    D -->|No| E[DENY transition<br/>Return: Tests not passing<br/>or stale results]
    D -->|Yes| F{Risk_Tier<br/>assigned?}

    F -->|No| G[DENY transition<br/>Return: CLASSIFICATION_REQUIRED]
    F -->|Yes| H[All prerequisites met]

    H --> I[Update lifecycle_state<br/>to STAGING]
    I --> J[Grant access to<br/>staging resources]
    J --> K[Emit LIFECYCLE_TRANSITION<br/>audit event]
    K --> L([Agent in STAGING])
```

### Prerequisites Checklist

| # | Prerequisite | Validation |
|---|---|---|
| 1 | Conformance_Profile complete | Validated against EAAGF Conformance Profile JSON Schema |
| 2 | Conformance_Test_Suite passing | Full suite run within last 24 hours, all applicable tests pass |
| 3 | Risk_Tier assigned | T1, T2, T3, or T4 recorded in Agent_Registry |

---

## STAGING → PRODUCTION Transition (Requirement 10.3)

```mermaid
flowchart TD
    A([Transition request:<br/>STAGING → PRODUCTION]) --> B{Security scan<br/>attestation on file?<br/>Generated within 7 days}

    B -->|No| C[DENY transition<br/>Return: Security scan<br/>missing or expired]
    B -->|Yes| D{SAST: no critical/<br/>high findings?}

    D -->|No| E[DENY transition<br/>Return: Critical/high<br/>security findings]
    D -->|Yes| F{Dependency vuln scan:<br/>no exploitable vulns?}

    F -->|No| G[DENY transition<br/>Return: Exploitable<br/>vulnerabilities found]
    F -->|Yes| H{Data governance<br/>review complete?}

    H -->|No| I[DENY transition<br/>Return: Data governance<br/>review pending]
    H -->|Yes| J{Conformance_Test_Suite<br/>passing within 24 hours?}

    J -->|No| K[DENY transition<br/>Return: Tests not passing]
    J -->|Yes| L{Agent Risk_Tier?}

    L -->|T1 / T2| M[No AI Gov approval needed]
    L -->|T3 / T4| N{AI Governance Team<br/>approval granted?}

    N -->|No| O[DENY transition<br/>Return: AI Gov approval<br/>required for T3/T4]
    N -->|Yes| M

    M --> P{Canary deployment<br/>requested?<br/>Req 10.5}

    P -->|Yes| Q[Deploy canary<br/>Route configured % of traffic<br/>to new version]
    P -->|No| R[Full deployment]

    Q --> S[Monitor canary<br/>Default: 24h T1/T2, 72h T3/T4]
    S --> T{Canary success<br/>criteria met?}
    T -->|Yes| U[Promote canary<br/>to full production]
    T -->|No| V[Rollback canary<br/>Downgrade to STAGING<br/>Emit CANARY_ROLLBACK event]

    R --> W[Update lifecycle_state<br/>to PRODUCTION]
    U --> W

    W --> X[Grant production<br/>resource access]
    X --> Y[Initialize continuous<br/>compliance monitoring]
    Y --> Z[Emit LIFECYCLE_TRANSITION<br/>audit event]
    Z --> DONE([Agent in PRODUCTION])
```

### Prerequisites Checklist

| # | Prerequisite | Validation |
|---|---|---|
| 1 | Security scan attestation | SAST + dependency scan, no critical/high findings, generated within 7 days |
| 2 | Data governance review | Data access patterns, classification handling, geographic constraints compliant |
| 3 | Conformance_Test_Suite passing | Full suite run within last 24 hours |
| 4 | AI Governance Team approval (T3/T4 only) | Approver identity, timestamp, and conditions recorded |

---

## Re-Validation and Automatic Downgrade (Requirement 10.7)

Agents in PRODUCTION must be periodically re-validated. Failure to re-validate triggers automatic downgrade to STAGING.

```mermaid
flowchart TD
    A[Agent in PRODUCTION] --> B{Re-validation<br/>deadline approaching?}

    B -->|30 days before| C[Notify owning team:<br/>Re-validation due in 30 days]
    B -->|14 days before| D[Notify owning team:<br/>Re-validation due in 14 days]
    B -->|7 days before| E[Notify owning team:<br/>Re-validation due in 7 days]

    C --> F{Re-validation<br/>completed?}
    D --> F
    E --> F

    F -->|Yes| G[Re-validation successful<br/>Reset re-validation period]
    G --> A

    F -->|No, deadline passed| H[Enter grace period<br/>Flag: REVALIDATION_OVERDUE]
    H --> I[Emit COMPLIANCE_DRIFT alert]

    I --> J{Re-validation completed<br/>during grace period?}
    J -->|Yes| G
    J -->|No, grace period expired| K[AUTOMATIC DOWNGRADE<br/>to STAGING<br/>Req 10.7]

    K --> L[Revoke production<br/>resource access]
    L --> M[Emit LIFECYCLE_DOWNGRADE<br/>audit event]
    M --> N[Notify owning team<br/>within 30 minutes]
    N --> O([Agent in STAGING<br/>Must re-satisfy all<br/>STAGING → PRODUCTION<br/>prerequisites])
```

### Re-Validation Periods and Grace Periods

| Risk Tier | Re-Validation Period | Grace Period |
|---|---|---|
| T1 | 180 days | 14 days |
| T2 | 180 days | 14 days |
| T3 | 90 days | 7 days |
| T4 | 90 days | 7 days |

### Re-Validation Requirements

| # | Requirement | Applies To |
|---|---|---|
| 1 | Passing Conformance_Test_Suite run (full suite) | All tiers |
| 2 | Current security scan attestation (within 7 days) | All tiers |
| 3 | Conformance_Profile review (still accurate) | All tiers |
| 4 | AI Governance Team confirmation of current risk assessment | T3/T4 only |

---

## Deprecation Notification Flow (Requirement 10.6)

When an agent version is deprecated, all dependent agents and systems receive 30 days notice before the effective decommission date.

```mermaid
flowchart TD
    A([Agent version<br/>marked as deprecated]) --> B[Record deprecation:<br/>• Deprecation timestamp<br/>• Effective decommission date<br/>• Replacement version if available<br/>• Authorizing human identity]

    B --> C[Query Agent_Registry<br/>for dependent agents]
    C --> D[Identify dependents via:<br/>• approved_mcp_servers references<br/>• A2A delegation targets<br/>• Recent communication history]

    D --> E[Notify all dependents<br/>within 24 hours]
    E --> F[Emit DEPRECATION_DECLARED<br/>audit event]

    F --> G[Deprecation notice includes:<br/>• Deprecated agent ID, name, version<br/>• Effective decommission date<br/>• Replacement version if available<br/>• Migration guidance]

    G --> H{14 days before<br/>decommission date}
    H --> I[Send reminder notification<br/>Emit DEPRECATION_REMINDER event]

    I --> J{7 days before<br/>decommission date}
    J --> K[Send final reminder<br/>Emit DEPRECATION_REMINDER event]

    K --> L{Decommission date<br/>reached}
    L --> M{All dependents<br/>migrated?}
    M -->|Yes| N[Proceed to decommission]
    M -->|No| O[Block decommission<br/>Escalate to AI Governance Team]
    O --> P{AI Gov Team<br/>override?}
    P -->|Yes| N
    P -->|No| Q[Extend deprecation period<br/>Re-notify dependents]
```

---

## Decommission Dependency Verification (Requirement 10.11)

Before completing a decommission, the Governance_Controller verifies that all dependent agents and systems have migrated away.

```mermaid
flowchart TD
    A([Decommission<br/>requested]) --> B[Perform dependency<br/>verification check]

    B --> C[Query Agent_Registry<br/>for agents referencing<br/>target agent in<br/>Conformance_Profile]
    B --> D[Query observability backend<br/>for A2A communication<br/>within last 30 days]

    C --> E[Compile dependency report]
    D --> E

    E --> F{Active dependencies<br/>found?}

    F -->|No dependencies| G[Dependencies clear<br/>Proceed with decommission]

    F -->|Active dependencies<br/>in DEV/STAGING/PRODUCTION| H[BLOCK decommission<br/>Emit DECOMMISSION_BLOCKED<br/>audit event]
    H --> I[Return dependency report<br/>to requesting team]

    I --> J{AI Governance Team<br/>override authorization?}
    J -->|No| K([Decommission blocked<br/>Team must migrate<br/>dependents first])

    J -->|Yes| L[Record override:<br/>• Authorizer identity<br/>• Justification<br/>• Impact acknowledgment]
    L --> G

    G --> M[Transition lifecycle_state<br/>to DECOMMISSIONED]
    M --> N[Revoke ALL credentials<br/>within 60 seconds]
    N --> O[Revoke production<br/>resource access]
    O --> P[Emit LIFECYCLE_TRANSITION<br/>audit event]
    P --> Q[Notify previously identified<br/>dependents of decommission]
    Q --> R[Retain registration record<br/>and audit history<br/>for 7 years]
    R --> DONE([Agent decommissioned])
```

### Dependency Verification Sources

| Source | What It Checks |
|---|---|
| Agent_Registry | Agents referencing target in Conformance_Profile (A2A targets, MCP server references) |
| Observability backend | Agent-to-agent communication involving target within last 30 days |

### Decommission Override Requirements

An override requires:
- Authorizing AI Governance Team member's identity
- Documented justification
- Acknowledgment of impact on dependent agents

---

## Canary Deployment Pattern (Requirement 10.5)

```mermaid
flowchart TD
    A([New version promoted<br/>to PRODUCTION]) --> B{Canary deployment<br/>configured?}

    B -->|No| C[Full deployment<br/>100% traffic to new version]
    B -->|Yes| D[Route canary_percentage<br/>of traffic to new version<br/>Default: 10%]

    D --> E[Both versions in<br/>PRODUCTION state<br/>Exception to single-version rule]

    E --> F[Monitor canary version<br/>Same observability and<br/>security controls]

    F --> G{Canary duration<br/>elapsed?<br/>Default: 24h T1/T2, 72h T3/T4}
    G -->|No| H{Rollback trigger<br/>condition met?}
    H -->|No| F
    H -->|Yes, error rate /<br/>latency / policy violation| I[ROLLBACK canary]

    I --> J[Route 100% traffic<br/>to previous version]
    J --> K[Downgrade canary version<br/>to STAGING]
    K --> L[Emit CANARY_ROLLBACK<br/>audit event]
    L --> M[Notify owning team]
    M --> DONE_ROLLBACK([Canary rolled back])

    G -->|Yes| N{Success criteria met?<br/>Zero critical errors,<br/>no policy violations,<br/>no gate rejections}
    N -->|No| I
    N -->|Yes| O[Promote canary<br/>Route 100% to new version]
    O --> P[Deprecate previous version<br/>Subject to deprecation notice]
    P --> Q[Emit CANARY_PROMOTED<br/>audit event]
    Q --> DONE_PROMOTED([Canary promoted<br/>to full production])
```

---

## Audit Event Coverage

| Event | Trigger | Key Fields |
|---|---|---|
| `LIFECYCLE_TRANSITION` | Agent transitions between lifecycle stages | agent_id, previous_state, new_state, authorizing_identity, prerequisites_verified |
| `LIFECYCLE_DOWNGRADE` | Agent automatically downgraded from PRODUCTION | agent_id, downgrade_reason, timestamp |
| `DEPRECATION_DECLARED` | Agent version marked as deprecated | agent_id, effective_date, replacement_version, authorizer |
| `DEPRECATION_REMINDER` | Reminder sent before deprecation effective date | agent_id, days_remaining, dependent_count |
| `DECOMMISSION_BLOCKED` | Decommission blocked due to active dependencies | agent_id, dependency_count, dependency_report |
| `CANARY_ROLLBACK` | Canary deployment rolled back | agent_id, version, rollback_trigger, canary_duration |
| `CANARY_PROMOTED` | Canary deployment promoted to full production | agent_id, version, canary_duration, success_metrics |
| `AGENT_VERSION_REGISTERED` | New agent version registered | agent_id, version, predecessor_version |

---

## Cross-References

- [Lifecycle Management Standard](../eaagf-specification/11-lifecycle-management-standard.md) — Normative lifecycle rules, transition gates, and re-validation enforcement
- [Agent Identity and Registration Standard](../eaagf-specification/02-agent-identity-standard.md) — Credential revocation on decommission, record retention
- [Risk Classification Standard](../eaagf-specification/03-risk-classification-standard.md) — Risk_Tier assignment prerequisite
- [Compliance Standard](../eaagf-specification/10-compliance-standard.md) — Compliance drift triggers for downgrade
- [Agent Registration Flow](./agent-registration-flow.md) — Initial lifecycle state assignment
- [Credential Lifecycle Flow](./credential-lifecycle-flow.md) — Credential revocation on decommission
- [Incident Response Flow](./incident-response-flow.md) — Incident-triggered lifecycle actions
