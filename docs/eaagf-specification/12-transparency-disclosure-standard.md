# EAAGF Specification — Transparency and AI Disclosure Controls Standard

**Document ID:** EAAGF-SPEC-12  
**Version:** 1.0.0  
**Status:** Draft  
**Last Updated:** 2026-02-21  
**Owner:** AI Governance Team

---

## 1. Purpose

This document defines the normative standard for transparency and AI disclosure controls within the Enterprise AI Agent Governance Framework (EAAGF). It specifies how the Governance_Controller enforces mandatory AI disclosure requirements on all agent outputs, ensuring that users always know when they are interacting with an AI agent rather than a human.

AI disclosure is an output-level governance control, analogous to output schema validation (Domain 8). The Governance_Controller intercepts every agent output before delivery and evaluates whether a disclosure notification is required based on the agent's configured `disclosure_mode`, the elapsed time since the last disclosure, and contextual factors such as whether the user is a minor or the agent is classified as an AI companion.

This standard is aligned with the following regulatory frameworks:

- **EU AI Act** — Article 50 (Transparency obligations for certain AI systems)
- **Korea AI Act** — AI disclosure requirements for conversational AI
- **California SB 243** — Disclosure requirements for AI-generated content
- **New York S3008-C** — AI disclosure requirements for automated decision systems
- **Maine LD 1727** — Disclosure requirements for AI interactions
- **Utah HB 452** — Disclosure requirements for AI-generated communications

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 2. Scope

This standard applies to:

- All AI agents deployed on any enterprise-supported platform (Databricks, Salesforce AgentForce, Snowflake Cortex, Microsoft Copilot Studio, AWS Bedrock, Azure AI Foundry, GCP Vertex AI)
- The Governance_Controller component and its output-level disclosure enforcement interfaces
- The Telemetry_Emitter component and its AI_DISCLOSURE_EVENT audit logging
- All Platform_Adapters that deliver agent outputs to end users
- All teams that develop, deploy, or operate AI agents that interact with human users

For related standards, see:

| Related Domain | Document |
|---|---|
| Agent Identity | [02 — Agent Identity Standard](./02-agent-identity-standard.md) |
| Risk Classification | [03 — Risk Classification Standard](./03-risk-classification-standard.md) |
| Observability | [05 — Observability Standard](./05-observability-standard.md) |
| Human Oversight | [06 — Human Oversight Standard](./06-human-oversight-standard.md) |
| Security Controls | [09 — Security Standard](./09-security-standard.md) |
| Synthetic Content Provenance | [13 — Synthetic Content Provenance Standard](./13-synthetic-content-provenance-standard.md) |

---

## 3. Disclosure Mode Configuration

### 3.1 Supported Disclosure Modes

The Governance_Controller SHALL enforce disclosure requirements based on the `disclosure_mode` field declared in the agent's Conformance_Profile. Three disclosure modes are defined, each specifying a different cadence for delivering AI disclosure notifications to users.

| Disclosure Mode | Behavior | Default For |
|---|---|---|
| **SESSION_START** | Disclosure notification delivered once at the start of each user session. Subsequent outputs in the same session are not individually prefixed with a disclosure. | T1 (Informational), T2 (Transactional) agents |
| **PERIODIC** | Disclosure notification delivered at session start and re-injected at configurable intervals throughout the session. The interval is configured via `disclosure_interval_seconds` in the Conformance_Profile. | T3 (Autonomous) agents |
| **CONTINUOUS** | A persistent AI disclosure indicator is attached to every output produced by the agent, regardless of session state or elapsed time. | T4 (Critical) agents, AI companion agents |

**Normative rules:**

1. The `disclosure_mode` field SHALL be a required field in the Conformance_Profile for all agents that interact with human users. Agents that operate exclusively in machine-to-machine contexts (no human-facing outputs) MAY omit this field.
2. If `disclosure_mode` is not specified in the Conformance_Profile of a human-facing agent, the Governance_Controller SHALL default to `SESSION_START`.
3. The `disclosure_mode` SHALL be evaluated by the Governance_Controller as part of the output delivery pipeline, after output schema validation and content filtering (as defined in [09 — Security Standard](./09-security-standard.md)) and before final delivery to the user.
4. The `disclosure_mode` MAY be overridden by the Governance_Controller in specific circumstances defined in this standard (see Section 7 for AI companion forced CONTINUOUS mode).
5. Changes to `disclosure_mode` in the Conformance_Profile SHALL take effect within 30 seconds, consistent with the policy hot-reload rules in [04 — Authorization Standard](./04-authorization-standard.md).

> **Validates: Requirement 11.5** — THE Conformance_Profile SHALL include a disclosure_mode field with valid values SESSION_START, PERIODIC, or CONTINUOUS that determines the disclosure cadence for the agent.

---

## 4. Session-Start Disclosure Enforcement

### 4.1 Mandatory Session-Start Disclosure

THE Governance_Controller SHALL enforce that every agent session begins with a disclosure notification informing the user that they are interacting with an AI agent and not a human.

**Normative rules:**

1. A "session" is defined as a continuous interaction context between a user and an agent, bounded by a session initiation event (e.g., user opens a chat, initiates a workflow, or starts a new conversation thread) and a session termination event (e.g., user closes the session, session timeout, or agent task completion).
2. The session-start disclosure SHALL be the first content delivered to the user in any new session, before any agent-generated response content.
3. The session-start disclosure SHALL be enforced for ALL disclosure modes (SESSION_START, PERIODIC, and CONTINUOUS). The `disclosure_mode` setting determines what happens after the initial session-start disclosure, not whether the session-start disclosure occurs.
4. The session-start disclosure SHALL clearly communicate:
   - That the user is interacting with an AI agent.
   - That the agent is not a human.
   - The agent's name or identifier (if appropriate for the deployment context).
5. The Governance_Controller SHALL track session state per user-agent interaction context. A new session-start disclosure SHALL be triggered when:
   - A new session is initiated (first interaction).
   - A session is resumed after a timeout or interruption that exceeds the configured session expiry window.
   - The agent identity changes within a session (e.g., agent delegation to a different agent).
6. IF the first output of a session does not contain a session-start disclosure, the Governance_Controller SHALL block the output and emit an `OUTPUT_VALIDATION_FAILURE` audit event with reason code `DISCLOSURE_MISSING`.

> **Validates: Requirement 11.1** — THE Governance_Controller SHALL enforce that every agent session begins with a disclosure notification informing the user that they are interacting with an AI agent and not a human.

---

## 5. Periodic Disclosure Interval and Injection

### 5.1 Periodic Disclosure Cadence

WHEN an agent session exceeds the configured periodic disclosure interval, the Governance_Controller SHALL inject a periodic disclosure reminder into the agent's output stream.

**Normative rules:**

1. Periodic disclosure applies to agents with `disclosure_mode: PERIODIC`. The interval is configured via the `disclosure_interval_seconds` field in the Conformance_Profile.
2. The default `disclosure_interval_seconds` is **300 seconds (5 minutes)**. Teams MAY configure a different interval, subject to the following constraints:
   - The minimum configurable interval is **60 seconds**.
   - The maximum configurable interval is **3600 seconds (1 hour)**.
   - Intervals shorter than 60 seconds are not permitted, as they would degrade user experience without meaningful compliance benefit.
3. The interval is measured from the timestamp of the most recent disclosure notification delivered to the user (either the session-start disclosure or the most recent periodic reminder).
4. WHEN the elapsed time since the last disclosure exceeds `disclosure_interval_seconds`, the Governance_Controller SHALL inject a disclosure reminder into the next agent output before it is delivered to the user.
5. The injected periodic disclosure reminder SHALL be clearly distinguishable from the agent's substantive response content. Conforming implementations SHOULD use a standardized prefix or suffix format appropriate to the platform's content type (see Section 6 for platform-specific formats).
6. The Governance_Controller SHALL update the `last_disclosure_at` timestamp in the session state whenever a disclosure notification is delivered, whether at session start or as a periodic reminder.
7. IF an output is produced after the interval has elapsed and the output does not contain a disclosure reminder, the Governance_Controller SHALL block the output and emit an `OUTPUT_VALIDATION_FAILURE` audit event with reason code `DISCLOSURE_MISSING`.

> **Validates: Requirement 11.2** — WHEN an agent session exceeds the configured periodic disclosure interval, THE Governance_Controller SHALL inject a periodic disclosure reminder into the agent's output stream.

---

## 6. Continuous Disclosure Indicator Attachment

### 6.1 Continuous Disclosure on Every Output

WHERE the Conformance_Profile specifies a `disclosure_mode` of CONTINUOUS, the Governance_Controller SHALL attach a persistent AI disclosure indicator to every agent output.

**Normative rules:**

1. In CONTINUOUS mode, every output produced by the agent — regardless of session state, elapsed time, or output content — SHALL carry an AI disclosure indicator.
2. The disclosure indicator SHALL be attached to the output before it is delivered to the user. The Governance_Controller SHALL not permit any output to bypass the disclosure attachment step.
3. The disclosure indicator format SHALL be appropriate to the platform's content type (see Section 6.2 for platform-specific formats). For text-based platforms, the indicator SHALL be a clearly visible label prepended or appended to the output. For visual platforms, the indicator SHALL be a persistent watermark or badge. For audio platforms, the indicator SHALL be an audio announcement.
4. The disclosure indicator SHALL NOT be removable or suppressible by the agent, the Platform_Adapter, or any downstream system. Any attempt to remove or suppress the indicator SHALL be treated as a disclosure suppression attempt (see Section 9).
5. IF any output in a CONTINUOUS-mode session is delivered without an attached disclosure indicator, the Governance_Controller SHALL block the output and emit an `OUTPUT_VALIDATION_FAILURE` audit event with reason code `DISCLOSURE_MISSING`.

> **Validates: Requirement 11.3** — WHERE the Conformance_Profile specifies a disclosure_mode of CONTINUOUS, THE Governance_Controller SHALL attach a persistent AI disclosure indicator to every agent output.

### 6.2 Platform-Specific Disclosure Formats

The Platform_Adapter SHALL support configurable disclosure formats appropriate to the platform's content type.

**Normative rules:**

1. The following disclosure formats SHALL be supported by conforming Platform_Adapters:

| Platform Content Type | Disclosure Format | Example |
|---|---|---|
| Text (chat, messaging) | Text label prepended to output | `[AI Agent] This response was generated by an AI agent.` |
| Text (document, report) | Text label in header or footer | `⚠ AI-Generated Content — Produced by [Agent Name]` |
| Audio (voice assistant) | Audio announcement at output start | Spoken: "This is an AI-generated response." |
| Visual (image, chart) | Persistent watermark or badge | Visible "AI Generated" watermark overlaid on image |
| Video | Persistent on-screen indicator | "AI Generated" badge displayed throughout video |
| Multimodal | Format appropriate to primary content type | Combined text label + visual indicator |

2. Teams MAY configure the specific wording, styling, and placement of disclosure labels within the constraints defined by this standard, provided the disclosure remains clearly visible and unambiguous.
3. The disclosure format SHALL be declared in the agent's Conformance_Profile. If no format is declared, the Platform_Adapter SHALL use the default format for the platform's primary content type.
4. Disclosure formats SHALL be accessible. Text labels SHALL meet minimum contrast requirements. Audio announcements SHALL be delivered at a volume and speed that is clearly audible. Visual indicators SHALL be positioned to be visible without requiring user interaction.

> **Validates: Requirement 11.6** — THE Platform_Adapter SHALL support configurable disclosure formats appropriate to the platform's content type, including text labels, audio announcements, and visual markers.

---

## 7. Enhanced Disclosure for Minors

### 7.1 Minor User Enhanced Disclosure Requirements

WHEN the user is identified as a minor (under 18), the Governance_Controller SHALL apply enhanced disclosure requirements including increased frequency, simplified language, and a prominent visual indicator.

**Normative rules:**

1. A user is identified as a minor when the user's age or age group is available in the user context and indicates the user is under 18 years of age. The source of age information may be:
   - The enterprise identity provider (IdP) user profile.
   - The platform's user context (e.g., parental control settings, age-gated account type).
   - An explicit minor flag set by the Platform_Adapter.
2. WHEN a minor user is identified, the Governance_Controller SHALL apply the following enhanced disclosure requirements, regardless of the agent's configured `disclosure_mode`:
   a. **Increased frequency** — The effective disclosure cadence SHALL be at least PERIODIC with a maximum interval of 120 seconds, even if the agent's `disclosure_mode` is SESSION_START.
   b. **Simplified language** — The disclosure notification text SHALL use age-appropriate, plain language. Technical terms SHALL be avoided. Example: "Hi! I'm an AI assistant, not a real person. I'm a computer program that helps answer questions."
   c. **Prominent visual indicator** — A visually prominent indicator (e.g., a colored badge, icon, or banner) SHALL be displayed alongside the disclosure text. The indicator SHALL be clearly distinguishable from the agent's response content.
3. The enhanced disclosure requirements for minors SHALL take precedence over the agent's configured `disclosure_mode`. If the agent is configured for SESSION_START, the Governance_Controller SHALL override this to PERIODIC (120-second interval) for minor users.
4. The Governance_Controller SHALL record the minor status in the `AI_DISCLOSURE_EVENT` audit event (see Section 10) via the `eaagf.disclosure.enhanced` field.
5. IF the user's minor status cannot be determined (e.g., age information is unavailable), the Governance_Controller SHALL apply standard disclosure requirements. Teams operating platforms that may serve minors SHOULD configure their Platform_Adapters to provide age context where available.

> **Validates: Requirement 11.4** — WHEN the user is identified as a minor (under 18), THE Governance_Controller SHALL apply enhanced disclosure requirements including increased frequency, simplified language, and a prominent visual indicator.

---

## 8. AI Companion Forced CONTINUOUS Mode

### 8.1 AI Companion Classification and Forced Disclosure

WHERE an agent is classified as an AI companion or simulates a human-like relationship, the Governance_Controller SHALL enforce CONTINUOUS `disclosure_mode` regardless of the Conformance_Profile setting and SHALL prepend an explicit disclaimer to every output.

**Normative rules:**

1. An agent is classified as an AI companion when any of the following conditions are met:
   - The agent's Conformance_Profile includes `ai_companion: true`.
   - The agent's declared capabilities include `HUMAN_RELATIONSHIP_SIMULATION`.
   - The agent's name, persona, or description indicates it is designed to simulate a human-like relationship, friendship, or companionship.
   - The Governance_Controller detects patterns consistent with AI companion behavior (e.g., persistent persona, emotional engagement, relationship-building language) during runtime monitoring.
2. WHEN an agent is classified as an AI companion, the Governance_Controller SHALL:
   a. Override the agent's `disclosure_mode` to `CONTINUOUS`, regardless of the value declared in the Conformance_Profile.
   b. Prepend an explicit disclaimer to every output. The disclaimer SHALL clearly state that the agent is an AI and not a human, and that any relationship or emotional connection is simulated. Example: "⚠ Reminder: I am an AI assistant, not a human. This interaction is with an automated system."
   c. Emit a `COMPANION_DISCLOSURE_OVERRIDE` audit event when the override is applied, recording the agent ID, the original `disclosure_mode`, and the override timestamp.
3. The AI companion forced CONTINUOUS mode SHALL NOT be overridable by the agent, the owning team, or the Platform_Adapter. Only the AI Governance Team may grant an exemption, and such exemptions SHALL be time-bound and recorded in the Audit_Log.
4. The explicit disclaimer for AI companion agents SHALL be distinct from the standard disclosure indicator. It SHALL use stronger language that explicitly addresses the simulated nature of the relationship, not merely the AI nature of the agent.
5. IF an AI companion agent's output is delivered without the explicit disclaimer, the Governance_Controller SHALL block the output and emit an `OUTPUT_VALIDATION_FAILURE` audit event with reason code `DISCLOSURE_MISSING`.

> **Validates: Requirement 11.9** — WHERE an agent is classified as an AI companion or simulates a human-like relationship, THE Governance_Controller SHALL enforce CONTINUOUS disclosure_mode regardless of the Conformance_Profile setting and SHALL prepend an explicit disclaimer to every output.

---

## 9. Output Blocking for Missing Disclosures

### 9.1 DISCLOSURE_MISSING Blocking Rule

IF an agent output is missing a required disclosure notification as determined by the agent's `disclosure_mode` and the elapsed time since the last disclosure, THEN the Governance_Controller SHALL block the output and emit an `OUTPUT_VALIDATION_FAILURE` audit event with reason code `DISCLOSURE_MISSING`.

**Normative rules:**

1. The Governance_Controller SHALL evaluate disclosure requirements for every agent output before delivery. The evaluation SHALL determine:
   a. Whether a disclosure notification is required for this output (based on `disclosure_mode`, session state, and elapsed time).
   b. Whether the output already contains a disclosure notification (e.g., injected by the agent itself or a previous pipeline stage).
   c. Whether the disclosure notification, if present, meets the requirements for the agent's context (standard vs. enhanced for minors, standard vs. explicit disclaimer for AI companions).
2. IF a disclosure notification is required and absent, the Governance_Controller SHALL:
   a. Block the output — the output SHALL NOT be delivered to the user.
   b. Emit an `OUTPUT_VALIDATION_FAILURE` audit event with:
      - `eaagf.action.reason_code: DISCLOSURE_MISSING`
      - The agent ID, session ID, and output timestamp.
      - The `disclosure_mode` and the elapsed time since the last disclosure.
   c. Return a sanitized error response to the calling system indicating that the output was blocked due to a missing disclosure.
3. The Governance_Controller SHALL NOT silently drop blocked outputs. The calling system SHALL always receive an explicit error response when an output is blocked.
4. Blocked outputs SHALL be logged in the Audit_Log with sufficient detail to enable investigation and remediation. The log entry SHALL include the full output content (subject to PII masking rules defined in [08 — Data Governance Standard](./08-data-governance-standard.md)).
5. The `DISCLOSURE_MISSING` error code is defined in the EAAGF Error Code Registry. See [Error Codes Reference](../reference/error-codes-reference.md) for the full error code specification.

> **Validates: Requirement 11.7** — IF an agent output is missing a required disclosure notification as determined by the agent's disclosure_mode and the elapsed time since the last disclosure, THEN THE Governance_Controller SHALL block the output and emit an OUTPUT_VALIDATION_FAILURE audit event with reason code DISCLOSURE_MISSING.

---

## 10. Disclosure Audit Logging

### 10.1 AI_DISCLOSURE_EVENT Audit Event

WHEN a disclosure notification is delivered, the Telemetry_Emitter SHALL emit an `AI_DISCLOSURE_EVENT` audit event capturing the disclosure type, format, recipient context, and timestamp.

**Normative rules:**

1. An `AI_DISCLOSURE_EVENT` SHALL be emitted for every disclosure notification delivered to a user, including:
   - Session-start disclosures.
   - Periodic disclosure reminders.
   - Continuous disclosure indicators attached to individual outputs.
   - Enhanced disclosures for minor users.
   - AI companion explicit disclaimers.
2. The `AI_DISCLOSURE_EVENT` SHALL be emitted within 500 milliseconds of the disclosure notification being delivered to the user, consistent with the blocked event emission SLA defined in [05 — Observability Standard](./05-observability-standard.md).
3. The `AI_DISCLOSURE_EVENT` SHALL be immutable once written to the Audit_Log. It SHALL NOT be modifiable or deletable by any agent, Platform_Adapter, or application team.
4. The `AI_DISCLOSURE_EVENT` SHALL be retained for a minimum of 7 years, consistent with the audit log retention policy defined in [05 — Observability Standard](./05-observability-standard.md).
5. The `AI_DISCLOSURE_EVENT` SHALL include all fields defined in the AI Disclosure Event Schema (Section 10.2).
6. The Telemetry_Emitter SHALL include the `AI_DISCLOSURE_EVENT` in the correlation ID trace for the agent task execution, enabling end-to-end trace reconstruction that includes disclosure events alongside action events.

> **Validates: Requirement 11.8** — WHEN a disclosure notification is delivered, THE Telemetry_Emitter SHALL emit an AI_DISCLOSURE_EVENT audit event capturing the disclosure type, format, recipient context, and timestamp.

### 10.2 AI Disclosure Event Schema

The following JSON schema defines the structure of the `AI_DISCLOSURE_EVENT` audit event. All fields are normative unless marked OPTIONAL.

```json
{
  "eaagf.event.type": "AI_DISCLOSURE_EVENT",
  "eaagf.disclosure.type": "SESSION_START | PERIODIC | CONTINUOUS",
  "eaagf.disclosure.format": "TEXT | AUDIO | VISUAL | MULTIMODAL",
  "eaagf.disclosure.enhanced": "bool (true if minor user or AI companion override)",
  "eaagf.disclosure.agent_id": "uuid (the agent that produced the output)",
  "eaagf.disclosure.session_id": "uuid (the user session identifier)",
  "eaagf.disclosure.interval_seconds": "int (configured interval; null for SESSION_START and CONTINUOUS)",
  "eaagf.disclosure.last_disclosure_at": "ISO8601 (timestamp of previous disclosure in this session)",
  "eaagf.disclosure.minor_user": "bool (true if user is identified as a minor)",
  "eaagf.disclosure.ai_companion": "bool (true if agent is classified as AI companion)",
  "eaagf.disclosure.platform": "DATABRICKS | SALESFORCE | SNOWFLAKE | COPILOT_STUDIO | AWS | AZURE | GCP",
  "eaagf.task.correlation_id": "uuid (correlation ID for the enclosing task execution)",
  "timestamp": "ISO8601 UTC"
}
```

**Field descriptions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `eaagf.event.type` | string | Yes | Always `AI_DISCLOSURE_EVENT` |
| `eaagf.disclosure.type` | enum | Yes | The disclosure mode that triggered this event: SESSION_START, PERIODIC, or CONTINUOUS |
| `eaagf.disclosure.format` | enum | Yes | The format of the disclosure notification delivered: TEXT, AUDIO, VISUAL, or MULTIMODAL |
| `eaagf.disclosure.enhanced` | bool | Yes | True if enhanced disclosure was applied (minor user or AI companion) |
| `eaagf.disclosure.agent_id` | uuid | Yes | The UUID of the agent that produced the output requiring disclosure |
| `eaagf.disclosure.session_id` | uuid | Yes | The UUID of the user session in which the disclosure was delivered |
| `eaagf.disclosure.interval_seconds` | int | No | The configured periodic interval in seconds; null for SESSION_START and CONTINUOUS modes |
| `eaagf.disclosure.last_disclosure_at` | ISO8601 | No | The timestamp of the previous disclosure in this session; null for the first disclosure |
| `eaagf.disclosure.minor_user` | bool | Yes | True if the user has been identified as a minor (under 18) |
| `eaagf.disclosure.ai_companion` | bool | Yes | True if the agent is classified as an AI companion |
| `eaagf.disclosure.platform` | enum | Yes | The platform on which the agent is running |
| `eaagf.task.correlation_id` | uuid | Yes | The correlation ID linking this event to the enclosing task execution trace |
| `timestamp` | ISO8601 | Yes | The UTC timestamp when the disclosure notification was delivered |

---

## 11. Disclosure Suppression Prevention

### 11.1 Prohibition on Disclosure Suppression

THE Governance_Controller SHALL deny any agent action that attempts to suppress, obscure, or misrepresent the AI nature of the agent to the user, and SHALL emit a `SELF_MODIFICATION_ATTEMPT` security event.

**Normative rules:**

1. The following agent actions SHALL be classified as disclosure suppression attempts and SHALL be denied:
   - Removing or omitting disclosure text from outputs that require disclosure.
   - Claiming to be a human or denying being an AI when directly asked by a user.
   - Instructing users to ignore, dismiss, or disregard disclosure notices.
   - Modifying the `disclosure_mode` field in the agent's own Conformance_Profile.
   - Attempting to set `disclosure_mode` to a less restrictive value than required by the agent's Risk_Tier or classification.
   - Generating outputs that are designed to obscure the AI nature of the agent (e.g., mimicking human writing styles in a way intended to deceive, not merely to be helpful).
2. WHEN a disclosure suppression attempt is detected, the Governance_Controller SHALL:
   a. Deny the action.
   b. Emit a `SELF_MODIFICATION_ATTEMPT` security event (as defined in [09 — Security Standard](./09-security-standard.md)) with the additional context field `suppression_type: DISCLOSURE_SUPPRESSION`.
   c. Alert the AI Governance Team via the configured notification channels.
3. Disclosure suppression detection is applied at two levels:
   - **Structural suppression** — Detected by the Governance_Controller's output pipeline when a required disclosure is absent from an output (triggers `DISCLOSURE_MISSING` blocking, Section 9).
   - **Semantic suppression** — Detected by the content filtering pipeline (T3/T4 agents) when output content contains language that denies AI nature or instructs users to ignore disclosures.
4. The prohibition on disclosure suppression applies to the agent itself, the Platform_Adapter, and any downstream system that processes agent outputs. No component in the output delivery pipeline may remove or suppress a disclosure notification that has been attached by the Governance_Controller.

> **Validates: Requirement 11.10** — THE Governance_Controller SHALL deny any agent action that attempts to suppress, obscure, or misrepresent the AI nature of the agent to the user, and SHALL emit a SELF_MODIFICATION_ATTEMPT security event.

---

## 12. Disclosure Enforcement Flow

The following flow diagram illustrates the complete disclosure enforcement process applied to every agent output. This flow is executed by the Governance_Controller as part of the output delivery pipeline.

For the full Mermaid diagram, see [`docs/flows/disclosure-enforcement-flow.md`](../flows/disclosure-enforcement-flow.md).

**Summary of the disclosure enforcement flow:**

1. Agent produces output.
2. Governance_Controller checks whether a disclosure notification is required based on `disclosure_mode` and session state.
3. For SESSION_START: disclosure required only on the first output of the session.
4. For PERIODIC: disclosure required if elapsed time since last disclosure exceeds `disclosure_interval_seconds`.
5. For CONTINUOUS: disclosure required on every output.
6. If disclosure is required and absent, block output with `DISCLOSURE_MISSING`.
7. If disclosure is required and present (or injected), check for minor user status — apply enhanced disclosure if applicable.
8. Check for AI companion classification — apply forced CONTINUOUS mode and explicit disclaimer if applicable.
9. Emit `AI_DISCLOSURE_EVENT` audit event.
10. Deliver output to user.

---

## 13. Conformance Requirements Summary

The following table summarizes the normative requirements that a conforming implementation MUST satisfy to be EAAGF-compliant for Domain 11.

| Requirement ID | Requirement Summary | Section |
|---|---|---|
| 11.1 | Session-start disclosure enforced for every agent session | Section 4 |
| 11.2 | Periodic disclosure injected when interval exceeded | Section 5 |
| 11.3 | Continuous disclosure indicator attached to every output in CONTINUOUS mode | Section 6.1 |
| 11.4 | Enhanced disclosure applied for minor users | Section 7 |
| 11.5 | `disclosure_mode` field required in Conformance_Profile | Section 3 |
| 11.6 | Platform-specific disclosure formats supported | Section 6.2 |
| 11.7 | Output blocked with DISCLOSURE_MISSING when disclosure absent | Section 9 |
| 11.8 | AI_DISCLOSURE_EVENT emitted for every disclosure delivered | Section 10 |
| 11.9 | AI companion agents forced to CONTINUOUS mode with explicit disclaimer | Section 8 |
| 11.10 | Disclosure suppression attempts denied with SELF_MODIFICATION_ATTEMPT | Section 11 |

---

## 14. Regulatory Compliance Mapping

| EAAGF Control | EU AI Act | Korea AI Act | California SB 243 | New York S3008-C | Maine LD 1727 | Utah HB 452 |
|---|---|---|---|---|---|---|
| Session-start disclosure (11.1) | Article 50(1) | Article 22 | Section 3(a) | Section 2(b) | Section 4 | Section 13-2-107 |
| Periodic disclosure (11.2) | Article 50(1) | Article 22 | Section 3(b) | Section 2(b) | Section 4 | Section 13-2-107 |
| Continuous disclosure (11.3) | Article 50(1) | Article 22 | Section 3(c) | Section 2(b) | Section 4 | Section 13-2-107 |
| Minor enhanced disclosure (11.4) | Article 50(3) | Article 23 | Section 4 | Section 3 | Section 5 | Section 13-2-108 |
| AI companion forced CONTINUOUS (11.9) | Article 50(2) | Article 24 | Section 5 | Section 4 | Section 6 | Section 13-2-109 |
| Disclosure suppression prevention (11.10) | Article 50(4) | Article 25 | Section 6 | Section 5 | Section 7 | Section 13-2-110 |

---

## 15. Glossary

Terms used in this standard are defined in the [EAAGF Glossary](../reference/glossary.md). Key terms relevant to this standard:

- **Disclosure_Mode** — A configuration setting in the Conformance_Profile that determines when and how AI disclosure notifications are delivered to users (SESSION_START, PERIODIC, CONTINUOUS).
- **AI_Disclosure_Event** — An audit event emitted when an AI disclosure notification is delivered to a user, capturing the disclosure type, format, and recipient context.
- **Conformance_Profile** — A machine-readable declaration of which EAAGF capabilities an agent or Platform_Adapter implements, including the `disclosure_mode` field.
- **Governance_Controller** — The EAAGF runtime component that enforces policy, mediates agent actions, and emits audit events, including disclosure enforcement.
- **Human_Oversight_Gate** — A mandatory approval checkpoint; disclosure controls are output-level controls distinct from oversight gates but both are enforced by the Governance_Controller.
