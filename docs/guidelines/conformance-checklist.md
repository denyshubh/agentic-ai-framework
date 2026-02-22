# EAAGF Conformance Checklist

**Document ID:** EAAGF-GUIDE-CHECKLIST  
**Version:** 1.0.0  
**Status:** Draft  
**Last Updated:** 2025-07-14  
**Owner:** AI Governance Team

---

## Purpose

This checklist helps product teams verify that their agent meets all EAAGF requirements before requesting promotion to STAGING or PRODUCTION. Complete all applicable items for your agent's Risk Tier before submitting a lifecycle transition request.

Use this checklist at two key transition points:

- **DEVELOPMENT → STAGING**: All items in the "STAGING Gate" column must be satisfied.
- **STAGING → PRODUCTION**: All items in both the "STAGING Gate" and "PRODUCTION Gate" columns must be satisfied.

For the full specification, see the [EAAGF Specification](../eaagf-specification/01-overview.md). For lifecycle transition rules, see [11 — Lifecycle Management Standard](../eaagf-specification/11-lifecycle-management-standard.md).

---

## How to Use This Checklist

1. Copy this checklist into your agent's onboarding ticket or governance review document.
2. Check off each item as you verify conformance.
3. Items marked **All Tiers** apply to every agent. Items marked with a specific tier (e.g., **T3/T4**) apply only to agents at that tier or above.
4. Attach evidence (links to configurations, test results, approval records) for each checked item.
5. Submit the completed checklist to the AI Governance Team as part of your transition request.

---

## Domain 1: Agent Identity and Registration

_Reference: [02 — Agent Identity Standard](../eaagf-specification/02-agent-identity-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 1.1 | Agent is registered in the Agent_Registry with a valid UUID v4 identifier | All Tiers | ✅ Required | ✅ Required |
| 1.2 | Agent record contains all required fields: name, version (semver), owning_team, platform, risk_tier, lifecycle_state, conformance_profile, identity, created_at, created_by | All Tiers | ✅ Required | ✅ Required |
| 1.3 | Agent has a valid Agent_Identity credential (X.509 certificate or OAuth 2.0 token) bound to its UUID v4 | All Tiers | ✅ Required | ✅ Required |
| 1.4 | Credential rotation is configured to trigger at 80% TTL | All Tiers | ✅ Required | ✅ Required |
| 1.5 | Decommission credential revocation is verified to complete within 60 seconds | All Tiers | — | ✅ Required |
| 1.6 | Agent manifest (YAML) conforms to the EAAGF manifest schema (`apiVersion: eaagf/v1`, `kind: AgentManifest`) | All Tiers | ✅ Required | ✅ Required |

---

## Domain 2: Risk Classification and Tiering

_Reference: [03 — Risk Classification Standard](../eaagf-specification/03-risk-classification-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 2.1 | Agent has a valid Risk_Tier assignment (T1, T2, T3, or T4) | All Tiers | ✅ Required | ✅ Required |
| 2.2 | Risk classification evaluated all three dimensions: autonomy level, data sensitivity, action scope | All Tiers | ✅ Required | ✅ Required |
| 2.3 | Assigned tier matches the classification matrix (highest applicable tier across all dimensions) | All Tiers | ✅ Required | ✅ Required |
| 2.4 | For multi-platform agents: maximum tier rule applied across all platform contexts | Multi-platform | ✅ Required | ✅ Required |
| 2.5 | Self-service risk classification questionnaire completed and recommendation reviewed | All Tiers | ✅ Required | ✅ Required |
| 2.6 | Re-classification triggers are understood and documented (capability changes trigger re-evaluation within 24 hours) | All Tiers | — | ✅ Required |

---

## Domain 3: Authorization and Least Privilege

_Reference: [04 — Authorization Standard](../eaagf-specification/04-authorization-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 3.1 | Conformance_Profile declares only the minimum permissions required for the agent's task scope | All Tiers | ✅ Required | ✅ Required |
| 3.2 | All resource permissions are explicitly declared (no wildcard permissions) | All Tiers | ✅ Required | ✅ Required |
| 3.3 | Credential TTL conforms to tier maximum: ≤3600s for T1/T2, ≤900s for T3/T4 | All Tiers | ✅ Required | ✅ Required |
| 3.4 | `max_session_duration_seconds` in Conformance_Profile does not exceed tier maximum | All Tiers | ✅ Required | ✅ Required |
| 3.5 | Network egress endpoints are declared in `approved_egress_endpoints` (agents with no declaration are denied all outbound connections) | All Tiers | ✅ Required | ✅ Required |
| 3.6 | Context_Compartments are declared in `context_compartments` for all resources the agent accesses | All Tiers | ✅ Required | ✅ Required |
| 3.7 | Rate limit (`max_actions_per_minute`) conforms to tier maximum: ≤100 for T1/T2, ≤20 for T3/T4 | All Tiers | ✅ Required | ✅ Required |
| 3.8 | If agent acts on behalf of users: enterprise IdP integration verified (permissions = intersection of agent + user permissions) | If delegated | ✅ Required | ✅ Required |
| 3.9 | T3/T4 write operations on Restricted data are routed through Human_Oversight_Gate | T3/T4 | ✅ Required | ✅ Required |

---

## Domain 4: Observability and Audit Trail

_Reference: [05 — Observability Standard](../eaagf-specification/05-observability-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 4.1 | Agent actions produce audit events in OTLP format with all required `eaagf.*` span attributes | All Tiers | ✅ Required | ✅ Required |
| 4.2 | Correlation ID (`eaagf.task.correlation_id`) is propagated across all events within a task execution | All Tiers | ✅ Required | ✅ Required |
| 4.3 | BLOCKED events are emitted within 500ms of the block decision | All Tiers | — | ✅ Required |
| 4.4 | Audit events are written to immutable storage (write-once, tamper-evident) | All Tiers | — | ✅ Required |
| 4.5 | 7-year retention policy is configured for all audit records | All Tiers | — | ✅ Required |
| 4.6 | SIEM integration is configured (Kafka, Event Hub, Kinesis, or webhook) | All Tiers | — | ✅ Required |
| 4.7 | Telemetry buffering (WAL) is configured with minimum 10,000 event capacity | All Tiers | — | ✅ Required |
| 4.8 | If agent has reasoning chains: structured reasoning summary is logged alongside action events | If applicable | ✅ Required | ✅ Required |

---

## Domain 5: Human Oversight Controls

_Reference: [06 — Human Oversight Standard](../eaagf-specification/06-human-oversight-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 5.1 | Oversight mode is declared in Conformance_Profile and matches tier default (HUMAN_IN_LOOP for T1, SUPERVISED for T2, APPROVAL_REQUIRED for T3/T4) | All Tiers | ✅ Required | ✅ Required |
| 5.2 | T4 agents: oversight mode is APPROVAL_REQUIRED or stricter (FULL_AUTO requires explicit AI Governance Team authorization) | T4 | ✅ Required | ✅ Required |
| 5.3 | Gate notification channels are configured (email, Slack, Teams, or PagerDuty) with 30-second SLA | T2/T3/T4 | ✅ Required | ✅ Required |
| 5.4 | Gate timeout and escalation rules are configured (default: 4-hour timeout, escalation to secondary approver) | T2/T3/T4 | ✅ Required | ✅ Required |
| 5.5 | Designated approvers and secondary approvers are assigned and documented | T2/T3/T4 | ✅ Required | ✅ Required |
| 5.6 | Pause/resume capability is verified — agent can be suspended while preserving execution state | All Tiers | — | ✅ Required |
| 5.7 | Rollback capability is configured for T2/T3/T4 agents (default: last 10 actions) | T2/T3/T4 | — | ✅ Required |
| 5.8 | Plan deviation detection threshold is configured | All Tiers | — | ✅ Required |
| 5.9 | Emergency stop procedure is documented and tested | All Tiers | — | ✅ Required |

---

## Domain 6: Interoperability Standards

_Reference: [07 — Interoperability Standard](../eaagf-specification/07-interoperability-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 6.1 | Agent supports MCP as baseline protocol for tool connections | All Tiers | ✅ Required | ✅ Required |
| 6.2 | All MCP servers used by the agent are listed in the enterprise MCP directory | All Tiers | ✅ Required | ✅ Required |
| 6.3 | `approved_mcp_servers` in Conformance_Profile lists only enterprise-approved MCP server URIs | All Tiers | ✅ Required | ✅ Required |
| 6.4 | If agent delegates to other agents: A2A protocol is used with signed Agent Card | If delegating | ✅ Required | ✅ Required |
| 6.5 | If agent delegates: combined Risk_Tier of agent pair does not exceed task authorization | If delegating | ✅ Required | ✅ Required |
| 6.6 | `protocols_supported` in Conformance_Profile declares all protocols used (MCP_1_0, A2A_1_0) | All Tiers | ✅ Required | ✅ Required |
| 6.7 | Conformance_Profile validates against the EAAGF Conformance Profile JSON Schema | All Tiers | ✅ Required | ✅ Required |
| 6.8 | Platform Adapter is registered and its Conformance_Profile validated by the Governance_Controller | All Tiers | ✅ Required | ✅ Required |
| 6.9 | For non-native MCP/A2A platforms: compatibility shim is deployed and verified | If applicable | ✅ Required | ✅ Required |

---

## Domain 7: Data Governance and Privacy

_Reference: [08 — Data Governance Standard](../eaagf-specification/08-data-governance-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 7.1 | `data_classifications_accessed` in Conformance_Profile accurately lists all data classification levels the agent accesses | All Tiers | ✅ Required | ✅ Required |
| 7.2 | Context_Compartments are configured for Confidential and Restricted data access | If Confidential/Restricted | ✅ Required | ✅ Required |
| 7.3 | Cross-scope transfer of Restricted data is blocked | If Restricted | ✅ Required | ✅ Required |
| 7.4 | PII detection pipeline is configured for agent inputs and outputs | All Tiers | ✅ Required | ✅ Required |
| 7.5 | PII masking/redaction is applied in audit logs per enterprise data retention policy | All Tiers | ✅ Required | ✅ Required |
| 7.6 | Compartment purge is verified to complete within 60 seconds of task completion | If Confidential/Restricted | — | ✅ Required |
| 7.7 | Geographic constraints (`geographic_constraints`) are declared for GDPR/CCPA-scoped data | If GDPR/CCPA | ✅ Required | ✅ Required |
| 7.8 | Data catalog integration is configured (Apache Atlas, Collibra, or Alation) for runtime classification resolution | All Tiers | — | ✅ Required |
| 7.9 | Mixed-classification context handling applies highest classification controls to entire context | All Tiers | ✅ Required | ✅ Required |

---

## Domain 8: Security Controls

_Reference: [09 — Security Standard](../eaagf-specification/09-security-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 8.1 | Prompt injection detection classifier is active with configured threshold (default: 0.85) | All Tiers | ✅ Required | ✅ Required |
| 8.2 | Output schema validation is configured for all agent outputs | All Tiers | ✅ Required | ✅ Required |
| 8.3 | Sandbox isolation is enforced (no access to host resources, env vars, or non-allowlisted network interfaces) | T3/T4 | ✅ Required | ✅ Required |
| 8.4 | Rate limiting is configured per tier (100/min T1/T2, 20/min T3/T4) | All Tiers | ✅ Required | ✅ Required |
| 8.5 | Anomaly detection monitoring is active (unusual data access, unexpected connections, repeated denials) | All Tiers | — | ✅ Required |
| 8.6 | Self-modification prevention is verified (agent cannot modify own Conformance_Profile, Risk_Tier, or governance controls) | All Tiers | ✅ Required | ✅ Required |
| 8.7 | Input/output content filtering is active for harmful, biased, or policy-violating content | T3/T4 | ✅ Required | ✅ Required |
| 8.8 | SIEM/SOAR integration is configured for automated incident response workflows | All Tiers | — | ✅ Required |
| 8.9 | Security scan attestation (SAST + dependency vulnerability scan) is completed and passing | All Tiers | — | ✅ Required |

---

## Domain 9: Compliance and Regulatory Alignment

_Reference: [10 — Compliance Standard](../eaagf-specification/10-compliance-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 9.1 | Automated risk assessment report generated at registration, mapping Risk_Tier to regulatory obligations | All Tiers | ✅ Required | ✅ Required |
| 9.2 | Compliance evidence collection is active (audit logs, policy configs, approval records, test results) | All Tiers | — | ✅ Required |
| 9.3 | If EU AI Act high-risk criteria are met: agent is flagged as `EU_AI_ACT_HIGH_RISK` with additional transparency and oversight obligations enforced | If applicable | ✅ Required | ✅ Required |
| 9.4 | NIST AI RMF-aligned risk management plan is generated for T3/T4 agents | T3/T4 | — | ✅ Required |
| 9.5 | Conformance_Test_Suite has been executed and all tests pass | All Tiers | ✅ Required | ✅ Required |
| 9.6 | Compliance drift detection is active (24-hour SLA for alerts on degraded compliance posture) | All Tiers | — | ✅ Required |

---

## Domain 10: Lifecycle Management

_Reference: [11 — Lifecycle Management Standard](../eaagf-specification/11-lifecycle-management-standard.md)_

| # | Check | Tier | STAGING Gate | PRODUCTION Gate |
|---|---|---|---|---|
| 10.1 | Agent version follows semantic versioning (e.g., `1.2.0`) | All Tiers | ✅ Required | ✅ Required |
| 10.2 | Conformance_Profile is complete and validated against EAAGF schema | All Tiers | ✅ Required | ✅ Required |
| 10.3 | Conformance_Test_Suite run is passing | All Tiers | ✅ Required | ✅ Required |
| 10.4 | Risk_Tier is assigned | All Tiers | ✅ Required | ✅ Required |
| 10.5 | Security scan attestation is attached (SAST + dependency vulnerability scan) | All Tiers | — | ✅ Required |
| 10.6 | Data governance review is completed | All Tiers | — | ✅ Required |
| 10.7 | AI Governance Team approval is obtained | T3/T4 | — | ✅ Required |
| 10.8 | Re-validation period is configured (90 days for T3/T4, 180 days for T1/T2) | All Tiers | — | ✅ Required |
| 10.9 | CI/CD integration is configured with governance quality gates (webhook triggers) | All Tiers | — | ✅ Required |
| 10.10 | Deprecation notification process is documented (30-day notice to dependent agents/systems) | All Tiers | — | ✅ Required |
| 10.11 | Decommission dependency verification is documented | All Tiers | — | ✅ Required |

---

## Summary: Transition Gate Requirements

### DEVELOPMENT → STAGING

All items marked **✅ Required** in the "STAGING Gate" column must be satisfied:

- Agent registered with valid identity and credentials
- Risk_Tier assigned via three-dimension classification
- Conformance_Profile complete and schema-validated
- Conformance_Test_Suite passing
- Least-privilege permissions declared
- Oversight mode configured per tier
- MCP/A2A protocols configured
- Security controls active (prompt injection, output validation, rate limiting)

### STAGING → PRODUCTION

All items in both "STAGING Gate" and "PRODUCTION Gate" columns must be satisfied. Additional PRODUCTION requirements include:

- Security scan attestation (SAST + dependency vulnerability scan)
- Data governance review completed
- AI Governance Team approval (T3/T4 agents)
- Audit trail immutability and 7-year retention configured
- SIEM integration active
- Anomaly detection monitoring active
- Emergency stop procedure documented and tested
- CI/CD governance quality gates configured
- Compliance evidence collection active

---

## Related Documents

- [Agent Registration Guide](./agent-registration-guide.md)
- [Risk Assessment Guide](./risk-assessment-guide.md)
- [Incident Response Playbook](./incident-response-playbook.md)
- [EAAGF Specification Overview](../eaagf-specification/01-overview.md)
- [Glossary](../reference/glossary.md)
