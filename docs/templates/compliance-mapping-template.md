# EAAGF Compliance Mapping Template

> **Purpose:** Map EAAGF governance controls to regulatory obligations under EU AI Act, NIST AI RMF, and ISO 42001. Use this template to document how each control satisfies applicable regulatory requirements and to track compliance evidence.
>
> **Requirement:** 9.1 (compliance mapping registry)
>
> **References:**
> - [Compliance Standard](../eaagf-specification/10-compliance-standard.md)
> - [Conformance Checklist](../guidelines/conformance-checklist.md)

---

## Instructions

1. Copy this template for each governance domain or agent-level compliance assessment.
2. Fill in the EAAGF control ID and name from the specification.
3. Map each control to the applicable regulatory articles, functions, and clauses.
4. Identify evidence sources that demonstrate compliance.
5. Update the status column as controls are implemented and verified.

---

## Status Legend

| Status | Meaning |
|---|---|
| Not Started | Control not yet implemented |
| In Progress | Implementation underway |
| Implemented | Control implemented, pending verification |
| Verified | Control verified by conformance test suite |
| Audited | Control reviewed and confirmed by external auditor |
| Non-Applicable | Control does not apply to this agent or context |

---

## Domain: Agent Identity and Registration

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-ID-001 | Unique Agent Identifier (UUID v4) | Article 16 — Obligations of providers | GOVERN 1.1 — Legal and regulatory requirements | Clause 6.1 — Actions to address risks | Agent_Registry records | |
| EAAGF-ID-002 | Agent Record Metadata | Article 12 — Record-keeping | GOVERN 1.2 — Trustworthy AI characteristics | Clause 7.5 — Documented information | Registration audit logs | |
| EAAGF-ID-003 | Credential Issuance (X.509/OAuth 2.0) | Article 15 — Accuracy, robustness, cybersecurity | MANAGE 2.2 — Risk tolerance determination | Clause 8.4 — AI system operation | Credential issuance records | |
| EAAGF-ID-004 | Credential Auto-Rotation (80% TTL) | Article 15 — Accuracy, robustness, cybersecurity | MANAGE 2.4 — Risk prioritization | Clause 8.4 — AI system operation | Rotation event logs | |
| EAAGF-ID-005 | Decommission Credential Revocation | Article 22 — Corrective actions | MANAGE 4.1 — Risk treatment | Clause 8.4 — AI system operation | Revocation audit events | |

## Domain: Risk Classification

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-RISK-001 | Four-Tier Risk Classification | Article 6 — Classification rules for high-risk AI | GOVERN 1.1 — Legal and regulatory requirements | Clause 6.1 — Actions to address risks | Classification records | |
| EAAGF-RISK-002 | Three-Dimension Evaluation | Article 9 — Risk management system | MAP 1.1 — Intended purpose and context | Clause 6.1.2 — AI risk assessment | Assessment questionnaire results | |
| EAAGF-RISK-003 | Re-Classification on Change | Article 9 — Risk management system | MANAGE 2.2 — Risk tolerance determination | Clause 8.4 — AI system operation | Re-classification audit logs | |

## Domain: Authorization and Least Privilege

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-AUTH-001 | Least Privilege Authorization | Article 9 — Risk management | GOVERN 1.1 — Legal and regulatory requirements | Clause 6.1 — Risk assessment | Policy engine logs, permission denial events | |
| EAAGF-AUTH-002 | Scope-Bound Credential TTL | Article 15 — Cybersecurity | MANAGE 2.2 — Risk tolerance | Clause 8.4 — AI system operation | Credential issuance records | |
| EAAGF-AUTH-003 | Egress Allowlist Enforcement | Article 15 — Cybersecurity | MANAGE 2.4 — Risk prioritization | Clause 8.4 — AI system operation | Egress denial logs | |

## Domain: Observability and Audit Trail

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-OBS-001 | Standardized Audit Events (OTLP) | Article 12 — Record-keeping | MEASURE 2.5 — AI system monitoring | Clause 9.1 — Monitoring, measurement, analysis | Telemetry backend records | |
| EAAGF-OBS-002 | Immutable Audit Log | Article 12 — Record-keeping | GOVERN 1.2 — Trustworthy AI | Clause 7.5 — Documented information | Audit log integrity checks | |
| EAAGF-OBS-003 | 7-Year Retention | Article 12 — Record-keeping | GOVERN 1.5 — Ongoing monitoring | Clause 7.5 — Documented information | Retention policy configuration | |

## Domain: Human Oversight

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-HOC-001 | Four Oversight Modes | Article 14 — Human oversight | GOVERN 1.4 — Oversight mechanisms | Clause 8.4 — AI system operation | Oversight mode configuration | |
| EAAGF-HOC-002 | T4 Default APPROVAL_REQUIRED | Article 14 — Human oversight | MANAGE 2.2 — Risk tolerance | Clause 6.1 — Risk assessment | Agent configuration records | |
| EAAGF-HOC-003 | Emergency Stop Capability | Article 14 — Human oversight | MANAGE 4.1 — Risk treatment | Clause 8.4 — AI system operation | Emergency stop audit events | |

## Domain: Interoperability

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-IOP-001 | MCP Baseline Protocol | Article 13 — Transparency | MAP 3.2 — Benefits and costs | Clause 8.2 — AI system realization | Platform adapter conformance tests | |
| EAAGF-IOP-002 | A2A Delegation Protocol | Article 13 — Transparency | MAP 3.2 — Benefits and costs | Clause 8.2 — AI system realization | A2A delegation audit logs | |
| EAAGF-IOP-003 | Conformance Profile Validation | Article 16 — Provider obligations | GOVERN 1.1 — Legal requirements | Clause 8.2 — AI system realization | Conformance test results | |

## Domain: Data Governance and Privacy

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-DG-001 | Data Classification Enforcement | Article 10 — Data governance | MAP 1.5 — Data requirements | Clause 8.4 — AI system operation | Data access audit logs | |
| EAAGF-DG-002 | Context Compartment Isolation | Article 10 — Data governance | MANAGE 2.4 — Risk prioritization | Clause 8.4 — AI system operation | Compartment creation logs | |
| EAAGF-DG-003 | PII Detection and Masking | Article 10 — Data governance | MAP 1.5 — Data requirements | Clause 8.4 — AI system operation | PII detection pipeline logs | |
| EAAGF-DG-004 | Geographic Residency Enforcement | GDPR Article 44 — Transfers | GOVERN 1.1 — Legal requirements | Clause 6.1 — Risk assessment | Residency enforcement logs | |

## Domain: Security Controls

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-SEC-001 | Prompt Injection Detection | Article 15 — Cybersecurity | MANAGE 2.4 — Risk prioritization | Clause 8.4 — AI system operation | Security event logs | |
| EAAGF-SEC-002 | Output Schema Validation | Article 15 — Cybersecurity | MEASURE 2.5 — Monitoring | Clause 8.4 — AI system operation | Validation failure logs | |
| EAAGF-SEC-003 | Rate Limiting | Article 15 — Cybersecurity | MANAGE 2.4 — Risk prioritization | Clause 8.4 — AI system operation | Rate limit event logs | |
| EAAGF-SEC-004 | Self-Modification Prevention | Article 15 — Cybersecurity | MANAGE 4.1 — Risk treatment | Clause 8.4 — AI system operation | Self-modification attempt logs | |

## Domain: Compliance and Regulatory Alignment

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-CMP-001 | Compliance Mapping Registry | Article 9 — Risk management | GOVERN 1.1 — Legal requirements | Clause 6.1 — Risk assessment | Mapping registry records | |
| EAAGF-CMP-002 | Automated Risk Assessment | Article 9 — Risk management | MAP 1.1 — Intended purpose | Clause 6.1.2 — AI risk assessment | Risk assessment reports | |
| EAAGF-CMP-003 | Continuous Evidence Collection | Article 12 — Record-keeping | MEASURE 2.5 — Monitoring | Clause 9.1 — Monitoring | Compliance dashboard data | |
| EAAGF-CMP-004 | Quarterly Compliance Reporting | Article 62 — Reporting | GOVERN 1.5 — Ongoing monitoring | Clause 9.1 — Monitoring | Quarterly report archives | |

## Domain: Lifecycle Management

| EAAGF Control ID | Control Name | EU AI Act Article | NIST AI RMF Function | ISO 42001 Clause | Evidence Sources | Status |
|---|---|---|---|---|---|---|
| EAAGF-LCM-001 | Four-Stage Lifecycle | Article 9 — Risk management | GOVERN 1.4 — Oversight mechanisms | Clause 8.2 — AI system realization | Lifecycle state records | |
| EAAGF-LCM-002 | Transition Gate Prerequisites | Article 9 — Risk management | MANAGE 2.2 — Risk tolerance | Clause 8.2 — AI system realization | Gate evaluation logs | |
| EAAGF-LCM-003 | Semantic Versioning | Article 16 — Provider obligations | GOVERN 1.2 — Trustworthy AI | Clause 7.5 — Documented information | Version registry records | |
| EAAGF-LCM-004 | Incident Response Runbooks | Article 22 — Corrective actions | MANAGE 4.1 — Risk treatment | Clause 10.2 — Nonconformity and corrective action | Incident evidence bundles | |

---

## Blank Row Template

Copy and paste the row below to add additional controls:

```
| EAAGF-___-___ | | | | | | |
```
