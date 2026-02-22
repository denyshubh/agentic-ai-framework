# EAAGF Risk Assessment Questionnaire

> **Purpose:** Guide product teams through the three risk classification dimensions to determine the appropriate Risk Tier (T1–T4) for their AI agent.
>
> **Requirements:** 2.1 (four-tier classification), 2.2 (three-dimension evaluation), 2.9 (self-service questionnaire)
>
> **References:**
> - [Risk Classification Standard](../eaagf-specification/03-risk-classification-standard.md)
> - [Risk Assessment Guide](../guidelines/risk-assessment-guide.md)
> - [Risk Classification Flow](../flows/risk-classification-flow.md)

---

## Instructions

1. Answer each question in the three sections below by selecting the option that best describes your agent.
2. Record the score for each answer.
3. Use the scoring matrix at the end to determine the recommended Risk Tier.
4. If your agent spans multiple platforms, complete this questionnaire for each platform context and apply the **maximum tier** across all contexts.

---

## Section 1: Autonomy Level

**How much human involvement is required for the agent to complete its tasks?**

| Option | Description | Score |
|---|---|---|
| **A** | Human approves every action before the agent executes it | 1 |
| **B** | Human reviews and approves write operations; read operations are automatic | 2 |
| **C** | Agent executes multi-step workflows autonomously; human is notified but does not approve each step | 3 |
| **D** | Agent operates fully autonomously with no per-action or per-workflow human approval | 4 |

**Your answer:** ____  **Score:** ____

---

## Section 2: Data Sensitivity

**What is the highest classification level of data the agent will access?**

| Option | Description | Score |
|---|---|---|
| **A** | Public data only (e.g., published knowledge base articles, public APIs) | 1 |
| **B** | Internal data (e.g., internal wikis, non-sensitive business metrics) | 2 |
| **C** | Confidential data (e.g., customer records, employee data, business strategies) | 3 |
| **D** | Restricted data (e.g., PII subject to GDPR/CCPA, financial records, trade secrets) | 4 |

**Your answer:** ____  **Score:** ____

---

## Section 3: Action Scope

**What types of actions will the agent perform?**

| Option | Description | Score |
|---|---|---|
| **A** | Read-only — the agent retrieves and presents information but never modifies data | 1 |
| **B** | Read + Write to internal systems (e.g., updates CRM records, writes to internal databases) | 2 |
| **C** | Read + Write + Multi-step operations (e.g., orchestrates workflows across multiple systems) | 3 |
| **D** | Write + External-facing + Financial (e.g., sends data to external APIs, executes financial transactions, modifies other agents or governance controls) | 4 |

**Your answer:** ____  **Score:** ____

---

## Scoring and Tier Assignment

### Step 1: Record Your Scores

| Dimension | Your Score (1–4) |
|---|---|
| Autonomy Level | ____ |
| Data Sensitivity | ____ |
| Action Scope | ____ |
| **Maximum Score** | ____ |

### Step 2: Determine Risk Tier

The Risk Tier is determined by the **maximum score** across all three dimensions. A single high-scoring dimension elevates the entire agent to the corresponding tier.

| Maximum Score | Risk Tier | Label |
|---|---|---|
| 1 | **T1** | Informational |
| 2 | **T2** | Transactional |
| 3 | **T3** | Autonomous |
| 4 | **T4** | Critical |

### Step 3: Verify Against Tier Definitions

Confirm your result matches the tier definition:

| Tier | Autonomy | Data Sensitivity | Action Scope |
|---|---|---|---|
| **T1** | Human approves every action | Public / Internal | Read-only |
| **T2** | Human-in-loop for writes | Internal / Confidential | Read + Write (internal) |
| **T3** | Autonomous multi-step | Confidential | Read + Write + Multi-step |
| **T4** | Fully autonomous | Restricted | Write + External + Financial |

---

## Governance Implications by Tier

Once you have your Risk Tier, the following governance controls apply automatically:

| Control | T1 | T2 | T3 | T4 |
|---|---|---|---|---|
| Default Oversight Mode | HUMAN_IN_LOOP | SUPERVISED | APPROVAL_REQUIRED | APPROVAL_REQUIRED |
| Credential TTL | 1 hour | 1 hour | 15 minutes | 15 minutes |
| Rate Limit (actions/min) | 100 | 100 | 20 | 20 |
| Re-validation Period | 180 days | 180 days | 90 days | 90 days |
| AI Gov Team Approval for Production | No | No | Yes | Yes |
| EU AI Act High-Risk Candidate | No | Possible | Likely | Yes |
| Security Scan Required for Production | Yes | Yes | Yes | Yes |
| Sandbox Isolation | No | No | Yes | Yes |

---

## Multi-Platform Resolution

If your agent operates across multiple platforms, complete this questionnaire for each platform context. The final Risk Tier is the **maximum tier** across all platform assessments.

**Example:** An agent classified as T2 on Salesforce and T3 on Databricks receives a final classification of **T3**.

| Platform | Autonomy Score | Data Score | Scope Score | Max Score | Tier |
|---|---|---|---|---|---|
| __________ | ____ | ____ | ____ | ____ | ____ |
| __________ | ____ | ____ | ____ | ____ | ____ |
| __________ | ____ | ____ | ____ | ____ | ____ |
| **Final Tier (maximum):** | | | | | ____ |

---

## Next Steps

1. Record the assigned Risk Tier in your [Agent Manifest](./agent-manifest-template.yaml).
2. Complete the [Conformance Profile](./conformance-profile-template.json) with tier-appropriate settings.
3. Submit for registration following the [Agent Registration Guide](../guidelines/agent-registration-guide.md).
4. If you believe the recommended tier is incorrect, contact the AI Governance Team to request a manual review. Re-classification must be completed within 24 hours of the request.
