# AWS Bedrock Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on AWS Bedrock to the EAAGF governance framework. It covers Bedrock Agents API integration, Lambda hooks, the gateway + SDK deployment pattern, MCP/A2A compatibility shim setup, and AWS-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. An AWS account with Amazon Bedrock enabled in your target region.
2. Bedrock Agents configured for your use case.
3. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
4. Access to the EAAGF Governance Controller endpoint.
5. IAM roles and policies configured for Bedrock agent execution.

---

## Architecture Overview

AWS Bedrock does not natively support MCP or A2A. The EAAGF Platform Adapter for AWS uses a **gateway + SDK deployment pattern** where Lambda action group handlers route agent actions through the EAAGF gateway, and an SDK library provides governance integration within the Lambda execution context.

```
┌──────────────────────────────────────────────────┐
│  AWS Bedrock                                      │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Bedrock Agent    │  │  Lambda Action       │  │
│  │  (Agent Runtime)  │──┤  Group Handlers      │  │
│  │                   │  │  (EAAGF SDK)         │  │
│  └──────────────────┘  └──────────┬───────────┘  │
└───────────────────────────────────┼──────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  EAAGF Gateway        │
                         │  (Platform Adapter)   │
                         │  ┌────────────────┐   │
                         │  │ MCP Shim        │   │
                         │  │ A2A Shim        │   │
                         │  │ Identity Proxy  │   │
                         │  │ Telemetry Fwd   │   │
                         │  └────────────────┘   │
                         └──────────┬────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  Governance Controller│
                         └───────────────────────┘
```

---

## Step 1: Bedrock Agents API Integration

### 1.1 Create the Bedrock Agent

Create your Bedrock agent with action groups that route through the EAAGF governance layer:

```python
import boto3

bedrock_agent = boto3.client('bedrock-agent')

response = bedrock_agent.create_agent(
    agentName='sales-analysis-agent',
    description='EAAGF-managed agent for sales data analysis',
    foundationModel='anthropic.claude-3-sonnet-20240229-v1:0',
    agentResourceRoleArn='arn:aws:iam::123456789012:role/BedrockAgentRole',
    instruction='You are a sales analysis agent. All actions are governed by EAAGF.',
    tags={
        'eaagf:agent-id': 'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
        'eaagf:risk-tier': 'T2',
        'eaagf:owning-team': 'revenue-analytics'
    }
)
```

### 1.2 Configure Action Groups with Lambda Hooks

Each Bedrock agent action group is backed by a Lambda function. The EAAGF SDK is included in the Lambda layer to provide governance integration:

```python
# Lambda action group handler with EAAGF SDK
import json
from eaagf_sdk import GovernanceClient, ActionRequest

governance = GovernanceClient(
    gateway_endpoint="https://eaagf-gateway.internal.company.com",
    agent_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    credential_path="/opt/eaagf/credential.json"
)

def lambda_handler(event, context):
    """Bedrock Agent action group handler with EAAGF governance."""
    
    action = event.get('actionGroup')
    api_path = event.get('apiPath')
    parameters = event.get('parameters', [])
    
    # Step 1: Evaluate governance policy
    decision = governance.evaluate(ActionRequest(
        action_type="TOOL_CALL",
        target_resource=f"bedrock://{action}/{api_path}",
        input_summary=json.dumps(parameters)[:1024]
    ))
    
    if decision.outcome == "DENIED":
        return {
            'statusCode': 403,
            'body': json.dumps({
                'error': decision.reason_code,
                'message': f'Action denied by EAAGF governance: {decision.reason_code}'
            })
        }
    
    if decision.outcome == "GATED":
        return {
            'statusCode': 202,
            'body': json.dumps({
                'message': 'Action pending human approval',
                'gate_id': decision.gate_id
            })
        }
    
    # Step 2: Execute the action (only if PERMITTED)
    result = execute_action(action, api_path, parameters)
    
    # Step 3: Report completion to governance
    governance.report_completion(decision.request_id, result)
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }
```

### 1.3 EAAGF SDK Lambda Layer

Package the EAAGF SDK as a Lambda layer so all action group handlers can use it:

```bash
# Build the EAAGF SDK Lambda layer
pip install eaagf-sdk -t python/
zip -r eaagf-sdk-layer.zip python/

# Publish the layer
aws lambda publish-layer-version \
  --layer-name eaagf-sdk \
  --zip-file fileb://eaagf-sdk-layer.zip \
  --compatible-runtimes python3.11 python3.12
```

Attach the layer to all action group Lambda functions:

```python
aws lambda update-function-configuration \
  --function-name my-agent-action-handler \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:eaagf-sdk:1
```

---

## Step 2: Lambda Hooks for Lifecycle Events

### 2.1 Pre-Processing Lambda Hook

Configure a Bedrock agent pre-processing Lambda that validates the agent's identity and session before any action:

```python
def pre_processing_handler(event, context):
    """Validate EAAGF identity before agent processes input."""
    
    session_id = event.get('sessionId')
    agent_id = event.get('agentId')
    
    # Validate agent identity with Governance Controller
    identity_valid = governance.validate_identity(agent_id)
    
    if not identity_valid:
        return {
            'preProcessingParsedResponse': {
                'isValidInput': False,
                'rationale': 'Agent identity validation failed (IDENTITY_UNREGISTERED)'
            }
        }
    
    # Check rate limits
    rate_ok = governance.check_rate_limit(agent_id)
    
    if not rate_ok:
        return {
            'preProcessingParsedResponse': {
                'isValidInput': False,
                'rationale': 'Rate limit exceeded (RATE_LIMIT_EXCEEDED)'
            }
        }
    
    return {
        'preProcessingParsedResponse': {
            'isValidInput': True,
            'rationale': 'EAAGF governance checks passed'
        }
    }
```

### 2.2 Post-Processing Lambda Hook

Configure a post-processing Lambda that validates agent outputs and emits telemetry:

```python
def post_processing_handler(event, context):
    """Validate agent output and emit telemetry."""
    
    output = event.get('response', {}).get('output', '')
    
    # Validate output against schema
    validation = governance.validate_output(output)
    
    if not validation.is_valid:
        governance.emit_event('OUTPUT_VALIDATION_FAILURE', {
            'agent_id': event.get('agentId'),
            'validation_errors': validation.errors
        })
        return {
            'postProcessingParsedResponse': {
                'responseText': 'Output validation failed. Please try again.'
            }
        }
    
    # Emit completion telemetry
    governance.emit_event('ACTION_COMPLETED', {
        'agent_id': event.get('agentId'),
        'session_id': event.get('sessionId'),
        'output_summary': output[:1024]
    })
    
    return {
        'postProcessingParsedResponse': {
            'responseText': output
        }
    }
```

---

## Step 3: Gateway + SDK Deployment

### 3.1 Gateway Configuration

```yaml
# eaagf-aws-gateway.yaml
gateway:
  platform: "AWS"
  governance_controller_endpoint: "https://governance.internal.company.com"

  aws:
    region: "us-east-1"
    bedrock:
      agent_arn_prefix: "arn:aws:bedrock:us-east-1:123456789012:agent/"
    authentication:
      type: "iam_role"
      role_arn: "arn:aws:iam::123456789012:role/EAAGFGatewayRole"

  mcp_shim:
    enabled: true
    version: "MCP_1_0"
    enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
    refresh_interval_seconds: 300
    aws_mappings:
      - aws_operation: "bedrock:InvokeAgent"
        mcp_tool: "mcp://enterprise-catalog/bedrock-agent"
      - aws_operation: "lambda:InvokeFunction"
        mcp_tool: "mcp://enterprise-catalog/aws-lambda"
      - aws_operation: "s3:GetObject"
        mcp_tool: "mcp://enterprise-catalog/aws-s3"

  a2a_shim:
    enabled: true
    version: "A2A_1_0"
    agent_card_signing_key_path: "/etc/eaagf/signing-key.pem"

  telemetry:
    otlp_endpoint: "https://otel-collector.internal.company.com:4317"
    service_name: "eaagf-aws-gateway"
```

### 3.2 Deploy the Gateway

The gateway can be deployed as an ECS Fargate service or on the enterprise Kubernetes cluster:

```yaml
# ECS Task Definition
{
  "family": "eaagf-aws-gateway",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "gateway",
      "image": "eaagf-registry.internal.company.com/aws-gateway:latest",
      "portMappings": [
        { "containerPort": 8080, "protocol": "tcp" },
        { "containerPort": 8081, "protocol": "tcp" }
      ],
      "secrets": [
        {
          "name": "EAAGF_CONFIG",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:eaagf-aws-gateway-config"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/eaagf-aws-gateway",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "gateway"
        }
      }
    }
  ]
}
```

---

## Step 4: MCP Compatibility Shim

### 4.1 How the MCP Shim Works

1. The Bedrock agent invokes an action group (Lambda function).
2. The EAAGF SDK in the Lambda translates the call into an MCP tool call request.
3. The SDK forwards the MCP request to the gateway for policy evaluation.
4. If permitted, the gateway routes the call to the target MCP server or back to the AWS service.
5. The response is returned to the Lambda handler.

### 4.2 Declaring MCP Servers in Your Manifest

```yaml
spec:
  approved_mcp_servers:
    - "mcp://enterprise-catalog/aws-s3"
    - "mcp://enterprise-catalog/aws-dynamodb"
    - "mcp://enterprise-catalog/bedrock-agent"
  protocols_supported:
    - "MCP_1_0"
```

---

## Step 5: A2A Compatibility Shim

### 5.1 Delegating to Other Agents

When a Bedrock agent needs to delegate a sub-task, the EAAGF SDK provides a delegation method:

```python
# In your Lambda action group handler
delegation_result = governance.delegate_task(
    target_agent_id="target-agent-uuid",
    sub_task_description="Analyze customer sentiment from CRM data",
    permitted_actions=["TOOL_CALL", "DATA_READ"],
    permitted_resources=["salesforce://sobject/Case"],
    max_risk_tier="T2"
)

if delegation_result.status == "COMPLETED":
    return delegation_result.result
elif delegation_result.status == "DENIED":
    return {"error": f"Delegation denied: {delegation_result.reason_code}"}
```

---

## Step 6: AWS-Specific Conformance Profile Examples

### T2 AWS Bedrock Agent (Document Analysis Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "document-analysis-agent"
  version: "1.0.0"
  owning_team: "legal-ops"
  platform: "AWS"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "s3://legal-documents/contracts"
      actions: ["READ"]
    - resource: "dynamodb://legal-analysis/results"
      actions: ["READ", "WRITE"]
    - resource: "bedrock://knowledge-base/legal-kb"
      actions: ["QUERY"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/aws-s3"
    - "mcp://enterprise-catalog/aws-dynamodb"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "legal-analysis-context"
  geographic_constraints:
    - "US"
  protocols_supported:
    - "MCP_1_0"
```

### T4 AWS Bedrock Agent (Payment Processing Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "payment-processing-agent"
  version: "2.0.1"
  owning_team: "treasury-operations"
  platform: "AWS"
spec:
  risk_tier: "T4"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - EXTERNAL_CONNECTION
  declared_permissions:
    - resource: "rds://finance/payments/transactions"
      actions: ["SELECT", "INSERT", "UPDATE"]
    - resource: "s3://finance-restricted/audit-reports"
      actions: ["READ"]
    - resource: "secretsmanager://finance/payment-keys"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/payment-gateway"
    - "mcp://enterprise-catalog/finance-db"
  approved_egress_endpoints:
    - "payments.provider.com"
    - "api.internal.company.com"
  data_classifications_accessed:
    - "CONFIDENTIAL"
    - "RESTRICTED"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "payment-processing-context"
    - "finance-restricted-context"
  geographic_constraints:
    - "US"
  protocols_supported:
    - "MCP_1_0"
    - "A2A_1_0"
```

---

## Validation Checklist

Before requesting promotion to STAGING, verify:

- [ ] Agent manifest passes schema validation against the EAAGF conformance schema.
- [ ] EAAGF gateway is deployed and healthy (`/health` endpoint returns 200).
- [ ] Lambda action group handlers include the EAAGF SDK layer.
- [ ] Pre-processing and post-processing Lambda hooks are configured.
- [ ] MCP shim correctly translates AWS-native calls to MCP format.
- [ ] IAM roles follow least-privilege (only permissions needed for the agent's declared scope).
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.
- [ ] For T3/T4: A2A delegation is configured via the EAAGF SDK.
- [ ] Bedrock agent is tagged with `eaagf:agent-id` and `eaagf:risk-tier`.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
