# EAAGF Glossary

**Document ID:** EAAGF-REF-GLOSSARY  
**Version:** 1.0.0  
**Status:** Draft  
**Last Updated:** 2025-07-14  
**Owner:** AI Governance Team

---

This glossary defines the canonical terminology used throughout the Enterprise AI Agent Governance Framework (EAAGF). All specification documents, guidelines, templates, and reference materials use these terms as defined below.

Terms are listed alphabetically. Cross-references to related terms are indicated with → arrows and link to the relevant entry within this document. Links to the primary specification document where each term is normatively defined are provided under each entry.

---

## Terms

### A2A

**Agent-to-Agent Protocol** — The protocol for peer-to-peer task delegation between agents. When one agent delegates a sub-task to another, the delegation MUST use A2A with a signed Agent Card that includes the delegating agent's identity and [→ Risk_Tier](#risk_tier). The [→ Governance_Controller](#governance_controller) enforces that the combined Risk_Tier of the agent pair does not exceed the maximum tier authorized for the task context.

**Related terms:** [→ Agent](#agent), [→ MCP](#mcp), [→ Agent_Identity](#agent_identity), [→ Platform_Adapter](#platform_adapter)  
**Primary specification:** [07 — Interoperability Standard](../eaagf-specification/07-interoperability-standard.md)

---

### Agent

An AI-powered autonomous or semi-autonomous software entity that perceives inputs, reasons, and takes actions on behalf of a user or system. Every agent deployed in the enterprise MUST be registered in the [→ Agent_Registry](#agent_registry), assigned an [→ Agent_Identity](#agent_identity), classified into a [→ Risk_Tier](#risk_tier), and governed by the [→ Governance_Controller](#governance_controller) throughout its lifecycle.

**Related terms:** [→ Agent_Registry](#agent_registry), [→ Agent_Identity](#agent_identity), [→ Risk_Tier](#risk_tier), [→ Conformance_Profile](#conformance_profile), [→ Governance_Controller](#governance_controller)  
**Primary specification:** [02 — Agent Identity Standard](../eaagf-specification/02-agent-identity-standard.md)

---

### Agent_Identity

A unique, cryptographically verifiable [→ Non-Human Identity (NHI)](#nhi) assigned to each agent instance. Agent_Identity credentials take the form of X.509 certificates or short-lived OAuth 2.0 tokens issued by the [→ Agent_Registry](#agent_registry). Credentials are automatically rotated when they reach 80% of their configured TTL and are revoked within 60 seconds when an agent is decommissioned.

**Related terms:** [→ Agent](#agent), [→ Agent_Registry](#agent_registry), [→ NHI](#nhi), [→ Governance_Controller](#governance_controller), [→ Least_Privilege](#least_privilege)  
**Primary specification:** [02 — Agent Identity Standard](../eaagf-specification/02-agent-identity-standard.md)

---

### Agent_Registry

The central catalog that records all registered agents, their identities, classifications, and lifecycle state. The Agent_Registry is the authoritative source of truth for agent identities. It assigns globally unique identifiers (UUID v4), issues [→ Agent_Identity](#agent_identity) credentials, manages credential rotation, supports bulk registration of up to 1,000 agents via declarative manifests, and retains registration records for a minimum of 7 years.

**Related terms:** [→ Agent](#agent), [→ Agent_Identity](#agent_identity), [→ Risk_Tier](#risk_tier), [→ Conformance_Profile](#conformance_profile), [→ Governance_Controller](#governance_controller)  
**Primary specification:** [02 — Agent Identity Standard](../eaagf-specification/02-agent-identity-standard.md)

---

### Audit_Log

An immutable, tamper-evident record of all agent actions, decisions, and identity events. Once written, audit records SHALL NOT be modifiable or deletable by any agent, [→ Platform_Adapter](#platform_adapter), or application team. All records are retained for a minimum of 7 years to satisfy EU AI Act and enterprise compliance requirements. The Audit_Log is populated by the [→ Telemetry_Emitter](#telemetry_emitter) and queryable via the [→ Governance_Controller](#governance_controller) query API.

**Related terms:** [→ Telemetry_Emitter](#telemetry_emitter), [→ Governance_Controller](#governance_controller), [→ Conformance_Test_Suite](#conformance_test_suite)  
**Primary specification:** [05 — Observability Standard](../eaagf-specification/05-observability-standard.md)

---

### Conformance_Profile

A machine-readable declaration of which EAAGF capabilities an agent or [→ Platform_Adapter](#platform_adapter) implements. The Conformance_Profile is defined as a JSON Schema and includes: declared capabilities, permissions, approved [→ MCP](#mcp) servers, approved egress endpoints, data classifications accessed, oversight mode, session duration limits, rate limits, [→ Context_Compartment](#context_compartment) assignments, geographic constraints, and supported protocols. The [→ Policy_Engine](#policy_engine) uses the Conformance_Profile to enforce [→ Least_Privilege](#least_privilege) authorization.

**Related terms:** [→ Agent](#agent), [→ Platform_Adapter](#platform_adapter), [→ Policy_Engine](#policy_engine), [→ Least_Privilege](#least_privilege), [→ Context_Compartment](#context_compartment), [→ Conformance_Test_Suite](#conformance_test_suite)  
**Primary specification:** [07 — Interoperability Standard](../eaagf-specification/07-interoperability-standard.md)

---

### Conformance_Test_Suite

The automated test suite used to verify that an agent or [→ Platform_Adapter](#platform_adapter) meets EAAGF requirements. The test suite produces machine-readable results that serve as ISO 42001 audit evidence. A passing Conformance_Test_Suite run is a prerequisite for transitioning an agent from DEVELOPMENT to STAGING in the agent lifecycle.

**Related terms:** [→ Conformance_Profile](#conformance_profile), [→ Platform_Adapter](#platform_adapter), [→ Audit_Log](#audit_log), [→ Governance_Controller](#governance_controller)  
**Primary specification:** [10 — Compliance Standard](../eaagf-specification/10-compliance-standard.md)

---

### Context_Compartment

An isolated data scope that prevents cross-contamination of sensitive information between agent tasks. When an agent accesses data classified as Confidential or Restricted, the [→ Governance_Controller](#governance_controller) creates a Context_Compartment to isolate that data. If an agent attempts to access a resource outside its declared Context_Compartment, the [→ Policy_Engine](#policy_engine) denies the access. All Confidential and Restricted data is purged from the compartment within 60 seconds of task completion.

**Related terms:** [→ Governance_Controller](#governance_controller), [→ Policy_Engine](#policy_engine), [→ Conformance_Profile](#conformance_profile), [→ Least_Privilege](#least_privilege)  
**Primary specification:** [08 — Data Governance Standard](../eaagf-specification/08-data-governance-standard.md)

---

### Governance_Controller

The EAAGF runtime component that enforces policy, mediates agent actions, and emits audit events. The Governance_Controller is the central enforcement point in the governance architecture. It validates [→ Agent_Identity](#agent_identity) credentials, delegates authorization decisions to the [→ Policy_Engine](#policy_engine), triggers [→ Human_Oversight_Gate](#human_oversight_gate) workflows, directs the [→ Telemetry_Emitter](#telemetry_emitter) to record events, manages agent lifecycle transitions, and enforces security controls including prompt injection detection and rate limiting.

**Related terms:** [→ Agent_Identity](#agent_identity), [→ Policy_Engine](#policy_engine), [→ Human_Oversight_Gate](#human_oversight_gate), [→ Telemetry_Emitter](#telemetry_emitter), [→ Risk_Tier](#risk_tier), [→ Audit_Log](#audit_log), [→ Platform_Adapter](#platform_adapter)  
**Primary specification:** [01 — Overview](../eaagf-specification/01-overview.md) (architecture-wide; referenced across all domain specifications)

---

### Human_Oversight_Gate

A mandatory approval checkpoint that pauses agent execution and requires human authorization before proceeding. The framework supports four oversight modes: FULL_AUTO, SUPERVISED, APPROVAL_REQUIRED, and HUMAN_IN_LOOP. When a gate is triggered, the designated approver is notified within 30 seconds. If no response is received within the configured timeout (default: 4 hours), the gate escalates to a secondary approver. The [→ Governance_Controller](#governance_controller) also supports pause, resume, rollback, and emergency stop capabilities.

**Related terms:** [→ Governance_Controller](#governance_controller), [→ Risk_Tier](#risk_tier), [→ Telemetry_Emitter](#telemetry_emitter), [→ Audit_Log](#audit_log)  
**Primary specification:** [06 — Human Oversight Standard](../eaagf-specification/06-human-oversight-standard.md)

---

### Least_Privilege

The security principle that agents receive only the minimum permissions required to complete their assigned task. The [→ Policy_Engine](#policy_engine) enforces Least_Privilege by granting each agent only the permissions declared in its [→ Conformance_Profile](#conformance_profile), issuing scope-bound credentials with tier-appropriate TTLs (1 hour for T1/T2, 15 minutes for T3/T4), and immediately revoking session credentials when a task completes or times out.

**Related terms:** [→ Policy_Engine](#policy_engine), [→ Conformance_Profile](#conformance_profile), [→ Context_Compartment](#context_compartment), [→ Risk_Tier](#risk_tier), [→ Agent_Identity](#agent_identity)  
**Primary specification:** [04 — Authorization Standard](../eaagf-specification/04-authorization-standard.md)

---

### MCP

**Model Context Protocol** — The baseline interoperability protocol for agent-to-tool connections. Every [→ Platform_Adapter](#platform_adapter) MUST implement MCP server/client interfaces. Agents may only connect to MCP servers listed in the enterprise MCP directory — a curated catalog of approved MCP servers with security attestations. If an agent attempts to connect to an unapproved MCP server, the [→ Governance_Controller](#governance_controller) blocks the connection.

**Related terms:** [→ A2A](#a2a), [→ Platform_Adapter](#platform_adapter), [→ Governance_Controller](#governance_controller), [→ Conformance_Profile](#conformance_profile)  
**Primary specification:** [07 — Interoperability Standard](../eaagf-specification/07-interoperability-standard.md)

---

### NHI

**Non-Human Identity** — Machine identities such as API keys, service accounts, OAuth tokens, and X.509 certificates used by agents. Each [→ Agent_Identity](#agent_identity) is a specific type of NHI issued and managed by the [→ Agent_Registry](#agent_registry). NHIs are subject to [→ Least_Privilege](#least_privilege) controls and credential lifecycle management including automatic rotation and revocation.

**Related terms:** [→ Agent_Identity](#agent_identity), [→ Agent_Registry](#agent_registry), [→ Least_Privilege](#least_privilege)  
**Primary specification:** [02 — Agent Identity Standard](../eaagf-specification/02-agent-identity-standard.md)

---

### Platform_Adapter

A thin integration layer that translates platform-specific agent APIs into EAAGF-compliant interfaces. Platform_Adapters exist for each supported platform: Databricks, Salesforce AgentForce, Snowflake Cortex, Microsoft Copilot Studio, AWS Bedrock, Azure AI Foundry, and GCP Vertex AI. Each adapter MUST implement [→ MCP](#mcp) server/client interfaces and normalize agent action requests for the [→ Governance_Controller](#governance_controller). Where a platform does not natively support MCP or [→ A2A](#a2a), the adapter implements a compatibility shim.

**Related terms:** [→ MCP](#mcp), [→ A2A](#a2a), [→ Governance_Controller](#governance_controller), [→ Conformance_Profile](#conformance_profile), [→ Conformance_Test_Suite](#conformance_test_suite)  
**Primary specification:** [07 — Interoperability Standard](../eaagf-specification/07-interoperability-standard.md)

---

### Policy_Engine

The component that evaluates agent action requests against governance policies and returns permit/deny decisions. The Policy_Engine uses an attribute-based access control (ABAC) model that considers agent attributes, action attributes, resource attributes, and environment attributes. It enforces [→ Least_Privilege](#least_privilege), [→ Context_Compartment](#context_compartment) isolation, egress allowlists, rate limits, and integrates with enterprise identity providers (LDAP, Azure AD, Okta) to resolve human-delegated permissions.

**Related terms:** [→ Governance_Controller](#governance_controller), [→ Least_Privilege](#least_privilege), [→ Context_Compartment](#context_compartment), [→ Conformance_Profile](#conformance_profile), [→ Human_Oversight_Gate](#human_oversight_gate)  
**Primary specification:** [04 — Authorization Standard](../eaagf-specification/04-authorization-standard.md)

---

### Risk_Tier

A classification level (T1–T4) assigned to an agent based on autonomy, data sensitivity, and action scope. The four tiers are:

| Tier | Name | Description |
|---|---|---|
| T1 | Informational | Read-only agents on non-sensitive data with human approval for every action |
| T2 | Transactional | Agents performing transactional writes on internal data with human-in-the-loop for writes |
| T3 | Autonomous | Agents operating autonomously on confidential data or performing multi-step writes |
| T4 | Critical | Agents on restricted data, external systems, financial transactions, or with self-modification capability |

The Risk_Tier determines credential TTLs, oversight modes, re-validation periods, rate limits, and whether AI Governance Team approval is required for production deployment. When an agent spans multiple platforms, the highest applicable tier is assigned.

**Related terms:** [→ Agent](#agent), [→ Governance_Controller](#governance_controller), [→ Human_Oversight_Gate](#human_oversight_gate), [→ Least_Privilege](#least_privilege), [→ Policy_Engine](#policy_engine)  
**Primary specification:** [03 — Risk Classification Standard](../eaagf-specification/03-risk-classification-standard.md)

---

### Telemetry_Emitter

The EAAGF component responsible for emitting standardized OpenTelemetry-compatible (OTLP) observability data. The Telemetry_Emitter records an audit event for every agent action, including agent ID, action type, target resource, timestamps, [→ Risk_Tier](#risk_tier), platform, outcome, and a correlation ID for end-to-end trace reconstruction. It supports SIEM integration via configurable webhooks or message queues (Kafka, Azure Event Hub, AWS Kinesis) and buffers events locally when the telemetry backend is unavailable.

**Related terms:** [→ Audit_Log](#audit_log), [→ Governance_Controller](#governance_controller), [→ Human_Oversight_Gate](#human_oversight_gate)  
**Primary specification:** [05 — Observability Standard](../eaagf-specification/05-observability-standard.md)

---

## See Also

- [EAAGF Specification Overview](../eaagf-specification/01-overview.md)
- [Architecture Overview](./architecture-overview.md)
- [Error Codes Reference](./error-codes-reference.md)
- [Compliance Mapping Registry](./compliance-mapping-registry.md)
