# EAAGF Incident Response Playbook

**Document ID:** EAAGF-GUIDE-IRP  
**Version:** 1.0.0  
**Status:** Draft  
**Last Updated:** 2025-07-14  
**Owner:** AI Governance Team

---

## Purpose

This playbook provides step-by-step incident response procedures for AI agent incidents governed by the EAAGF. It includes tier-specific runbooks (T1–T4), emergency stop procedures, evidence collection guidance, escalation paths, and post-incident review processes.

Use this playbook when:

- An agent exhibits anomalous or unauthorized behavior
- A security event is detected (prompt injection, output validation failure, rate limit breach)
- An emergency stop is executed
- Compliance drift is escalated
- An external security report involves an EAAGF-governed agent

For the normative specification, see [06 — Human Oversight Standard](../eaagf-specification/06-human-oversight-standard.md) and [11 — Lifecycle Management Standard](../eaagf-specification/11-lifecycle-management-standard.md). For the incident response flow diagrams, see [Incident Response Flow](../flows/incident-response-flow.md).

---

## Incident Severity Classification

Before selecting a runbook, determine the incident severity based on the affected agent's Risk_Tier and the nature of the incident.

| Severity | Risk Tier | Response Lead | Notification |
|---|---|---|---|
| SEV-4 (Low) | T1 | Owning team | Team notification only |
| SEV-3 (Medium) | T2 | Owning team | Team notification + AI Governance Team informed |
| SEV-2 (High) | T3 | Owning team + AI Governance Team | AI Governance Team notified, provides oversight |
| SEV-1 (Critical) | T4 | AI Governance Team leads | AI Governance Team + executive stakeholders + Security team |

If the incident involves data exfiltration, credential compromise, or external system impact, escalate severity by one level regardless of the agent's Risk_Tier.

---

## Emergency Stop Procedure

The emergency stop is the highest-priority intervention. Use it when an agent poses an immediate risk and must be terminated without delay.

### Who Can Execute

- AI Governance Team members (all)
- Agent's owning team lead
- Any operator with the `EMERGENCY_STOP` permission in the enterprise IdP

### Steps

1. **Issue the emergency stop command** targeting the agent's UUID.
2. **Verify termination** — Confirm within 5 seconds that:
   - All pending, in-flight, and queued actions are cancelled.
   - All credentials (session, delegated, platform tokens) are revoked.
   - The agent is blocked from initiating new actions.
3. **Verify notification** — Confirm within 60 seconds that:
   - An `EMERGENCY_STOP` audit event has been emitted.
   - The AI Governance Team has been notified via Slack/Teams AND PagerDuty (for T3/T4).
4. **Preserve state** — The Governance_Controller automatically preserves the agent's execution state (task queue, completed actions, reasoning context, active data) for investigation.
5. **Declare incident** — Proceed to the appropriate tier-specific runbook below.

### After Emergency Stop

- The agent cannot be resumed without explicit AI Governance Team re-authorization.
- A new approval workflow is required before the agent can operate again.
- The preserved execution state is available for forensic investigation.

> Reference: [06 — Human Oversight Standard, Section 10](../eaagf-specification/06-human-oversight-standard.md)

---

## Evidence Bundle Collection

The Governance_Controller automatically collects evidence when an incident is declared. This section describes what is collected and the expected SLAs.

### Automatic Collection

When an incident is declared, the Governance_Controller:

1. Determines the incident time window (default: 24 hours before incident to current time).
2. Collects the following artifacts in parallel:
   - Audit logs (all events for the agent within the window)
   - Telemetry snapshots (performance metrics, error rates, latency)
   - Configuration state (Conformance_Profile, Risk_Tier, lifecycle state, credential status)
   - Credential records (issuance, rotation, revocation events)
   - Policy decisions (all PERMIT, DENY, GATE decisions)
   - Human oversight records (gate triggers, approvals, rejections, escalations)
   - Data access records (compartment creation, purge, lineage)
   - Security events (prompt injection detections, output validation failures, rate limit events, anomalies)
3. Packages artifacts into an integrity-protected bundle (ZIP/TAR.GZ) with SHA-256 hash.
4. Emits an `EVIDENCE_BUNDLE_COLLECTED` audit event.

### Collection SLAs

| Risk Tier | Collection SLA |
|---|---|
| T1 / T2 | 1 hour |
| T3 | 30 minutes |
| T4 | 15 minutes |

### Manual Evidence Supplement

If the automatic collection is insufficient, the investigation team may request additional evidence:

- Extended time window (beyond the default 24 hours)
- Cross-agent correlation (events from agents that interacted via A2A)
- Platform-specific logs from the underlying platform (Databricks, Salesforce, etc.)
- Network traffic logs for egress-related incidents

> Reference: [Incident Response Flow — Evidence Bundle Collection](../flows/incident-response-flow.md)

---

## T1 Incident Response Runbook (Informational Agents)

**Severity:** SEV-4 (Low)  
**Response Lead:** Owning team  
**AI Governance Team Role:** Informed only

### Phase 1: Containment (SLA: 4 hours)

| Step | Action | Owner |
|---|---|---|
| 1 | Pause the agent if still active | Owning team |
| 2 | Restrict resource access (revoke read permissions if needed) | Owning team |
| 3 | Preserve execution state for review | Automatic |
| 4 | Notify owning team lead | Owning team |

### Phase 2: Evidence Collection (SLA: 1 hour)

| Step | Action | Owner |
|---|---|---|
| 1 | Verify Governance_Controller has auto-collected evidence bundle | Owning team |
| 2 | Review evidence bundle manifest for completeness | Owning team |
| 3 | Request additional evidence if needed (extended time window, platform logs) | Owning team |

### Phase 3: Investigation (SLA: 5 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Review audit logs for the incident window | Owning team |
| 2 | Review telemetry for anomalous patterns | Owning team |
| 3 | Review configuration state (Conformance_Profile, credentials) | Owning team |
| 4 | Identify root cause | Owning team |
| 5 | Document findings | Owning team |

### Phase 4: Remediation (SLA: 10 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Fix root cause (code change, configuration update, or Conformance_Profile update) | Owning team |
| 2 | Re-run Conformance_Test_Suite — all tests must pass | Owning team |
| 3 | If agent was downgraded to STAGING: request re-promotion to PRODUCTION | Owning team |
| 4 | Verify agent is operating correctly in production | Owning team |

### Phase 5: Post-Incident Review (SLA: 15 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Document root cause, timeline, and resolution | Owning team |
| 2 | Update runbook if the incident revealed gaps | Owning team |
| 3 | Update compliance records | Owning team |
| 4 | Close incident | Owning team |

---

## T2 Incident Response Runbook (Transactional Agents)

**Severity:** SEV-3 (Medium)  
**Response Lead:** Owning team  
**AI Governance Team Role:** Informed, available for consultation

### Phase 1: Containment (SLA: 4 hours)

| Step | Action | Owner |
|---|---|---|
| 1 | Pause the agent immediately | Owning team |
| 2 | Revoke session credentials for the current task | Owning team |
| 3 | Restrict resource access | Owning team |
| 4 | Assess whether rollback is needed (default: last 10 actions) | Owning team |
| 5 | Execute rollback if data integrity is at risk | Owning team |
| 6 | Notify owning team lead and inform AI Governance Team | Owning team |

### Phase 2: Evidence Collection (SLA: 1 hour)

| Step | Action | Owner |
|---|---|---|
| 1 | Verify Governance_Controller has auto-collected evidence bundle | Owning team |
| 2 | Review evidence bundle manifest for completeness | Owning team |
| 3 | Review data access records for potential data integrity issues | Owning team |

### Phase 3: Investigation (SLA: 5 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Review audit logs and telemetry | Owning team |
| 2 | Review write operations and their outcomes | Owning team |
| 3 | Verify data integrity of affected resources | Owning team |
| 4 | Review human oversight gate history (approvals, rejections) | Owning team |
| 5 | Identify root cause | Owning team |
| 6 | Document findings | Owning team |

### Phase 4: Remediation (SLA: 10 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Fix root cause | Owning team |
| 2 | Restore data integrity if rollback was incomplete | Owning team |
| 3 | Re-run Conformance_Test_Suite — all tests must pass | Owning team |
| 4 | If agent was downgraded: request re-promotion | Owning team |
| 5 | Verify agent is operating correctly | Owning team |

### Phase 5: Post-Incident Review (SLA: 15 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Document root cause, timeline, and resolution | Owning team |
| 2 | Update runbook if needed | Owning team |
| 3 | Update compliance records | Owning team |
| 4 | Share findings with AI Governance Team if broadly applicable | Owning team |
| 5 | Close incident | Owning team |

---

## T3 Incident Response Runbook (Autonomous Agents)

**Severity:** SEV-2 (High)  
**Response Lead:** Owning team with AI Governance Team oversight  
**AI Governance Team Role:** Notified immediately, provides oversight throughout

### Phase 1: Containment (SLA: 2 hours)

| Step | Action | Owner |
|---|---|---|
| 1 | Pause agent immediately | Owning team |
| 2 | Revoke ALL session credentials | Owning team |
| 3 | Restrict ALL resource access | Owning team |
| 4 | Assess whether emergency stop is warranted | AI Governance Team |
| 5 | Execute rollback if data integrity or confidentiality is at risk | Owning team + AI Gov |
| 6 | Notify AI Governance Team via Slack/Teams + PagerDuty | Automatic |
| 7 | AI Governance Team acknowledges and provides oversight | AI Governance Team |

### Phase 2: Evidence Collection (SLA: 30 minutes)

| Step | Action | Owner |
|---|---|---|
| 1 | Verify Governance_Controller has auto-collected evidence bundle | Owning team |
| 2 | Review evidence bundle for completeness | AI Governance Team |
| 3 | Request cross-agent correlation if A2A delegation was involved | AI Governance Team |
| 4 | Collect platform-specific logs if needed | Owning team |

### Phase 3: Investigation (SLA: 3 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Review audit logs, telemetry, and security events | Owning team + AI Gov |
| 2 | Review data access patterns and compartment integrity | Owning team + AI Gov |
| 3 | Review A2A delegation history if applicable | AI Governance Team |
| 4 | Assess data exposure scope (Confidential data handling) | AI Governance Team |
| 5 | Identify root cause with AI Governance Team input | Owning team + AI Gov |
| 6 | Document findings | Owning team |

### Phase 4: Remediation (SLA: 7 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Fix root cause | Owning team |
| 2 | Re-run Conformance_Test_Suite and security tests | Owning team |
| 3 | Complete data governance review if data handling was affected | Owning team + AI Gov |
| 4 | Obtain AI Governance Team re-approval for PRODUCTION promotion | AI Governance Team |
| 5 | Verify agent is operating correctly under enhanced monitoring | Owning team |

### Phase 5: Post-Incident Review (SLA: 10 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Document root cause, timeline, impact, and resolution | Owning team |
| 2 | Brief AI Governance Team on findings and lessons learned | Owning team |
| 3 | Update runbook with new procedures if gaps were identified | AI Governance Team |
| 4 | Update compliance records | Owning team |
| 5 | Distribute lessons learned to relevant teams | AI Governance Team |
| 6 | Close incident | AI Governance Team |

---

## T4 Incident Response Runbook (Critical Agents)

**Severity:** SEV-1 (Critical)  
**Response Lead:** AI Governance Team  
**AI Governance Team Role:** Leads the entire response

### Phase 1: Containment (SLA: 1 hour)

| Step | Action | Owner |
|---|---|---|
| 1 | Execute EMERGENCY STOP — terminate all agent actions | AI Governance Team |
| 2 | Revoke ALL credentials (session, delegated, platform tokens) | Automatic |
| 3 | Block ALL resource access | Automatic |
| 4 | Verify emergency stop completed within 5 seconds | AI Governance Team |
| 5 | Verify AI Governance Team notification within 60 seconds | Automatic |
| 6 | Notify executive stakeholders | AI Governance Team |
| 7 | Notify Security team | AI Governance Team |
| 8 | If external systems were affected: notify external partners | AI Governance Team |
| 9 | Assess whether dependent agents need to be paused | AI Governance Team |

### Phase 2: Evidence Collection (SLA: 15 minutes)

| Step | Action | Owner |
|---|---|---|
| 1 | Verify Governance_Controller has auto-collected evidence bundle | AI Governance Team |
| 2 | Preserve ALL agent state for forensic analysis | Automatic |
| 3 | Collect cross-agent correlation data (A2A delegation chains) | AI Governance Team |
| 4 | Collect platform-specific forensic logs | AI Governance Team |
| 5 | Collect network traffic logs for egress-related incidents | Security team |
| 6 | Verify evidence bundle integrity (SHA-256 hash) | AI Governance Team |

### Phase 3: Investigation (SLA: 2 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Conduct full forensic review of audit logs and telemetry | AI Governance Team |
| 2 | Review all security events (prompt injection, output validation, anomalies) | Security team |
| 3 | Trace data lineage and credential usage | AI Governance Team |
| 4 | Review A2A delegation history and combined tier enforcement | AI Governance Team |
| 5 | Assess data exposure scope (Restricted data handling, PII exposure) | AI Governance Team |
| 6 | Assess regulatory impact (EU AI Act, GDPR, CCPA implications) | Compliance team |
| 7 | Identify root cause | AI Governance Team + Security |
| 8 | Document findings in formal incident report | AI Governance Team |

### Phase 4: Remediation (SLA: 5 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Fix root cause | Owning team (supervised by AI Gov) |
| 2 | Full conformance and security re-certification | Owning team + AI Gov |
| 3 | Complete data governance review | AI Governance Team |
| 4 | If data breach occurred: initiate data breach response per enterprise policy | Compliance team |
| 5 | If GDPR/CCPA data was affected: assess data subject notification requirements | Compliance team |
| 6 | AI Governance Team re-approval REQUIRED before any resumption | AI Governance Team |
| 7 | Deploy agent under enhanced monitoring (reduced rate limits, stricter oversight mode) | Owning team + AI Gov |

### Phase 5: Post-Incident Review (SLA: 7 business days)

| Step | Action | Owner |
|---|---|---|
| 1 | Produce formal incident report | AI Governance Team |
| 2 | Executive briefing | AI Governance Team |
| 3 | Update compliance records and regulatory filings if required | Compliance team |
| 4 | Update runbook with new procedures | AI Governance Team |
| 5 | Distribute lessons learned to all agent-operating teams | AI Governance Team |
| 6 | Review and update EAAGF controls if systemic issues were identified | AI Governance Team |
| 7 | Close incident | AI Governance Team |

---

## Escalation Paths

### Standard Escalation

| Level | Escalation Target | Trigger |
|---|---|---|
| L1 | Owning team on-call | Initial incident detection |
| L2 | Owning team lead | L1 cannot contain within SLA |
| L3 | AI Governance Team | T3/T4 incident, or L2 cannot resolve |
| L4 | Executive stakeholders + Security team | T4 incident, data breach, or regulatory impact |

### Emergency Escalation (bypass standard path)

Use emergency escalation when:

- An agent is actively exfiltrating data
- An agent is executing unauthorized financial transactions
- An agent has been compromised by prompt injection and is performing unauthorized actions
- Multiple agents are affected simultaneously

Emergency escalation goes directly to L3 (AI Governance Team) + L4 (Executive + Security), regardless of the agent's Risk_Tier.

---

## Incident Response SLA Summary

| Phase | T1 | T2 | T3 | T4 |
|---|---|---|---|---|
| Containment | 4 hours | 4 hours | 2 hours | 1 hour |
| Evidence Collection | 1 hour | 1 hour | 30 minutes | 15 minutes |
| Investigation | 5 business days | 5 business days | 3 business days | 2 business days |
| Remediation | 10 business days | 10 business days | 7 business days | 5 business days |
| Post-Incident Review | 15 business days | 15 business days | 10 business days | 7 business days |

---

## Incident Triggers Reference

| Trigger | Description | Expected Severity |
|---|---|---|
| Observability alert | Anomalous behavior detected by monitoring | Varies by tier |
| `PROMPT_INJECTION_DETECTED` | Prompt injection classifier flagged input | SEV-2 or SEV-1 |
| `OUTPUT_VALIDATION_FAILURE` | Agent output failed schema validation | SEV-3 or SEV-2 |
| `RATE_LIMIT_EXCEEDED` | Agent exceeded action rate limit | SEV-4 or SEV-3 |
| `DATA_EXFILTRATION_ATTEMPT` | Restricted data transfer outside authorized scope | SEV-1 |
| `SELF_MODIFICATION_ATTEMPT` | Agent attempted to modify own governance controls | SEV-1 |
| `COMPARTMENT_VIOLATION` | Cross-compartment access denied | SEV-3 or SEV-2 |
| `EMERGENCY_STOPPED` | Agent emergency-stopped by operator | Matches agent tier |
| `COMPLIANCE_DRIFT` | Compliance posture degraded beyond threshold | SEV-3 or SEV-2 |
| `PLAN_DEVIATION` | Agent deviated from declared task plan | SEV-3 or SEV-2 |
| External report | Vulnerability or incident reported externally | SEV-1 |

---

## AI Governance Team Notification Requirements

The AI Governance Team must be notified for the following events. Notification channels and SLAs are defined in [06 — Human Oversight Standard](../eaagf-specification/06-human-oversight-standard.md).

| Event | Notification SLA | Channels | Tier |
|---|---|---|---|
| Emergency stop executed | 60 seconds | Slack/Teams + PagerDuty | All (PagerDuty for T3/T4) |
| T3 incident declared | 30 minutes | Slack/Teams | T3 |
| T4 incident declared | 15 minutes | Slack/Teams + PagerDuty + Email | T4 |
| Data exfiltration attempt | 15 minutes | Slack/Teams + PagerDuty | All |
| Self-modification attempt | 15 minutes | Slack/Teams + PagerDuty | All |
| Containment SLA breach | At SLA expiry | Slack/Teams | All |

---

## Post-Incident Checklist

Use this checklist to verify all post-incident activities are completed before closing an incident.

| # | Item | T1/T2 | T3 | T4 |
|---|---|---|---|---|
| 1 | Root cause identified and documented | Required | Required | Required |
| 2 | Evidence bundle archived (7-year retention) | Required | Required | Required |
| 3 | Remediation implemented and verified | Required | Required | Required |
| 4 | Conformance_Test_Suite re-run and passing | Required | Required | Required |
| 5 | Security tests re-run and passing | — | Required | Required |
| 6 | AI Governance Team re-approval obtained | — | Required | Required |
| 7 | Compliance records updated | Required | Required | Required |
| 8 | Runbook updated if gaps identified | If needed | Required | Required |
| 9 | Lessons learned documented and distributed | If needed | Required | Required |
| 10 | Executive briefing completed | — | — | Required |
| 11 | Regulatory notifications filed (if applicable) | — | If needed | Required |
| 12 | Incident formally closed | Required | Required | Required |

---

## Related Documents

- [Human Oversight Standard](../eaagf-specification/06-human-oversight-standard.md) — Emergency stop, pause/resume, rollback
- [Lifecycle Management Standard](../eaagf-specification/11-lifecycle-management-standard.md) — Incident response runbook templates, evidence bundles
- [Observability Standard](../eaagf-specification/05-observability-standard.md) — Audit event schema, retention
- [Security Standard](../eaagf-specification/09-security-standard.md) — Security event detection
- [Compliance Standard](../eaagf-specification/10-compliance-standard.md) — Compliance drift, evidence collection
- [Incident Response Flow](../flows/incident-response-flow.md) — Flow diagrams
- [Conformance Checklist](./conformance-checklist.md) — Pre-deployment verification
