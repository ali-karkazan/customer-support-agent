# Customer Support AI Agent — Udacity Project Submission

This project implements a customer support agent using Amazon Bedrock AgentCore and the Strands SDK. It supports order tracking, mock refunds, knowledge-base retrieval, cross-session customer memory, loyalty calculations through Code Interpreter, and live web browsing.

## Repository contents

| Path | Purpose |

|---|---|
| `starter/main.py` | Agent implementation |
| `starter/pyproject.toml` | Python requirements and dependencies |
| `starter/uv.lock` | Resolved dependency versions |
| `starter/.python-version` | Python 3.13 pin |
| `starter/product_catalog.txt` | Knowledge Base source data |
| `starter/lambda/` | Supplied Lambda functions and refund tool schema |
| `evidence/screenshots/` | Test results, tool evidence, and alarm configuration |
| `reflection.md` | Design reflection and production considerations |
| `LICENSE.txt` | License supplied with the starter repository |

## Architecture

The deployed `BedrockAgentCoreApp` exposes an asynchronous entrypoint. Each invocation creates a Strands agent with the configured model, tools, system instructions, and memory hook.

- **Model:** `global.amazon.nova-2-lite-v1:0`.
- **Order tools:** AgentCore Gateway → API Gateway REST API → order-tracking Lambda.
- **Refund tools:** AgentCore Gateway → direct refund Lambda invocation.
- **RAG:** `search_knowledge_base` calls the Bedrock Retrieve API and joins retrieved text chunks.
- **Memory:** A `MemoryHook` retrieves customer facts and preferences before responding and saves the original user query and assistant response afterward.
- **Calculations:** `calculate_loyalty_discount` executes self-contained Python through AgentCore Code Interpreter, with a tier-only fallback.
- **Browser:** `AgentCoreBrowser` retrieves live webpage content.

The MCP connection remains open while the agent executes its tools.

## Setup decisions and deviations

### Python 3.13

The starter manifest specified Python 3.14+, while the Environment Setup document permitted Python 3.13+.

Under Python 3.14, the Browser tool’s `nest_asyncio` integration caused shutdown errors and disrupted subsequent MCP connections. The project therefore pins Python 3.13. Browser followed by order tracking succeeded in the same local server process under this version.

### Managed Knowledge Base

The sandbox denied `aoss:CreateSecurityPolicy`, preventing the prescribed OpenSearch Serverless collection from being created. A Bedrock Managed Knowledge Base was used instead.

This is a deviation from the infrastructure instructions. The required application behavior remains implemented: the agent calls the Bedrock Retrieve API, and retrieval was tested locally and in the deployed runtime.

### Gateway target types

The implementation follows the Environment Setup document and rubric: one API Gateway target for order tracking and one direct Lambda target for refunds. Both targets returned successful results through MCP.

### Browser test URL

The project Instructions page requests `https://www.udacity.com`, while the original README uses Amazon. The submitted Browser test uses Udacity.

### Observability

Runtime logs and a CloudWatch error-count alarm were configured. Transaction Search and trace delivery setup were blocked by sandbox permissions. Enabling observability in the generated configuration did not resolve that restriction.

## Prerequisites

- Python 3.13
- `uv`
- AWS CLI v2
- Udacity sandbox credentials
- AWS resources in `us-east-1`
- Access to the configured Bedrock model

This submission uses the Python-based `bedrock-agentcore-starter-toolkit` workflow specified by the project. The CLI displays a migration advisory, but this project retains the course’s configure, deploy, and invoke workflow.

## Install dependencies

From the repository root:

```bash
cd starter
uv sync --locked
source .venv/bin/activate
python --version
aws sts get-caller-identity
```

Configure valid sandbox credentials before making AWS calls. Temporary credentials require their session token as well as the access key and secret key. Do not commit credentials.

## AWS resource setup

### Lambda and API Gateway

Deploy the supplied Python Lambda functions without modifying their mock business logic.

Configure the order-tracking Lambda behind a REST API using Lambda proxy integration:

| Method and path | Operation name |

|---|---|
| `GET /orders/{order_id}` | `get_order` |
| `GET /customers/{customer_id}/orders` | `get_customer_orders` |
| `GET /customers/{customer_id}` | `get_customer` |

Deploy the API to a stage such as `prod`. This project uses IAM authorization for the REST API.

### AgentCore Gateway

Create two MCP targets:

- `order-tracker`: the REST API’s deployed stage, exposing all three operations.
- `refund-processor`: the refund Lambda, using `starter/lambda/lambda_schema`.

The Gateway exposes these six tools:

```text
order-tracker___get_order
order-tracker___get_customer
order-tracker___get_customer_orders
refund-processor___initiate_refund
refund-processor___check_refund_status
refund-processor___get_return_label
```

The Gateway uses no inbound authorization to match the supplied course connection code. The deployed Runtime separately uses IAM authorization. An authenticated Gateway should be used for a production system.

### Knowledge Base

Upload `starter/product_catalog.txt` to a private S3 bucket, connect it to the Knowledge Base, and synchronize the data source.

### Memory

Create a memory resource with these built-in strategies:

| Strategy | Name | Namespace template |

|---|---|---|
| Semantic | `customer_facts` | `cs_agent/{actorId}/facts` |
| User preference | `customer_preferences` | `cs_agent/{actorId}/preferences` |

The helper supports both `namespaceTemplates` and the legacy `namespaces` response field.

### Agent configuration

Update these values in `starter/main.py` for your environment:

```python
GATEWAY_URL = "<your Gateway URL ending in /mcp>"
KB_ID = "<your Knowledge Base ID>"
REGION = "us-east-1"
MEMORY_ID = "<your Memory ID>"
```

The committed identifiers refer to the original sandbox and are not a guarantee that those resources remain available.

## Permissions verified during the project

Local execution uses developer credentials. Cloud execution uses a separate Runtime execution role.

The following permissions required explicit attention:

| Role | Permission | Scope |

|---|---|---|
| Gateway execution role | `execute-api:Invoke` | Project REST API stage and GET routes |
| Gateway execution role | `lambda:InvokeFunction` | Refund Lambda |
| Runtime execution role | `bedrock:Retrieve` | Project Knowledge Base |
| Runtime execution role | `bedrock-agentcore:StartBrowserSession` | Built-in Browser |
| Runtime execution role | `bedrock-agentcore:GetBrowserSession` | Built-in Browser |
| Runtime execution role | `bedrock-agentcore:StopBrowserSession` | Built-in Browser |
| Runtime execution role | `bedrock-agentcore:ConnectBrowserAutomationStream` | Built-in Browser |

The built-in Browser resource used was:

```text
arn:aws:bedrock-agentcore:us-east-1:aws:browser/aws.browser.v1
```

Retain the required model, Memory, Code Interpreter, and logging permissions on the Runtime role. The table records specific integration fixes, not a complete standalone IAM policy.

## Run locally

From `starter`, with the environment activated:

```bash
python main.py
```

In a second terminal:

```bash
curl -sS http://localhost:8080/invocations \
  -H 'Content-Type: application/json' \
  -d '{"prompt":"Can you track order ORD-001?","customer_id":"CUST-123","session_id":"local-test-1"}'
```

Both `prompt` and `customer_id` are required. A session ID is generated when omitted. Local execution still calls real AWS services.

## Deploy

From `starter`:

```bash
agentcore configure --entrypoint main.py --name customer_support_agent
agentcore deploy
agentcore status
```

During configuration, select:

- Direct Code Deploy
- Python 3.13
- `us-east-1`
- IAM Runtime authorization
- The existing project memory resource

The machine-specific `.bedrock_agentcore.yaml` and generated `.bedrock_agentcore/` directory are excluded from Git. Regenerate configuration in your own checkout.

## Cloud test commands

Run these from the configured `starter` directory. Each command supplies a fresh Runtime session ID.

### Test 1 — Order tracking

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"Can you track order ORD-001?","customer_id":"CUST-123","session_id":"t1"}'
```

Expected: `SHIPPED`, UPS, tracking number `TRK987654321`, and an estimated delivery date. The mock Lambda calculates delivery as two days after execution.

### Test 2 — Refund

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"I want to return my Kindle Paperwhite (ORD-002). Please initiate a refund.","customer_id":"CUST-123","session_id":"t2"}'
```

Expected: refund approval, refund ID, amount `$139.99`, and a 3–5-business-day message. This is a mock operation; it does not transfer money.

### Test 3 — RAG

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"What are the benefits of the Platinum loyalty tier?","customer_id":"CUST-123","session_id":"t3"}'
```

Expected retrieved benefits: free same-day shipping, 15% discount, and priority customer support.

### Test 4 — Cross-session memory

Session A:

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"Hi, I am Jane. I prefer concise responses.","customer_id":"CUST-123","session_id":"s-A"}'
```

Wait for long-term memory extraction before running session B. Extraction was not always complete after 30 seconds during testing.

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"Do you remember my name and communication preference?","customer_id":"CUST-123","session_id":"s-B"}'
```

Expected: Jane and a preference for concise responses.

The payload session ID identifies the memory conversation; the CLI `--session-id` identifies the Runtime session. Both differ between A and B, while the customer ID stays the same.

Additional evidence uses a fresh customer, `CUST-MEMCHECK-0915`, to demonstrate new cloud persistence and recall of the name Mira and a preference for numbered troubleshooting lists. The saved source event was independently inspected.

### Test 5 — Loyalty calculation

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"I am a Gold member with 4250 points. Calculate my discount on a $150 standard order.","customer_id":"CUST-123","session_id":"t5"}'
```

Expected:

- 4,000 points redeemed for `$40`
- 10% tier discount on the remaining `$110`, saving `$11`
- `$99` final total and `$51` total savings
- 99 earned points and 349 remaining points

The implementation redeems points in increments of 500, caps redemption at 50% of the order value, and applies the tier discount afterward. Whole points are earned on the final amount paid, rounded down.

The calculator follows the starter and graded test’s Gold discount behavior. The catalog separately limits Gold’s 10% benefit to accessories; this inconsistency is retained and documented.

### Test 6 — Browser

```bash
agentcore invoke \
  --session-id "$(python -c 'import uuid; print(uuid.uuid4())')" \
  '{"prompt":"Go to https://www.udacity.com and tell me the page title.","customer_id":"CUST-123","session_id":"t6"}'
```

Expected: Browser retrieves the live page title. The title observed during testing was “Learn the Latest Tech Skills; Advance Your Career | Udacity”.

Browser session names must use lowercase letters, digits, and hyphens.

## Evidence

Screenshots are stored in `evidence/screenshots/`:

| Evidence | Filename |

|---|---|
| Order response | `test-01-order-tracking.png` |
| Order tool input and result | `test-01-order-tool-log.png` |
| Refund response | `test-02-refund-processing.png` |
| Refund tool input and result | `test-02-refund-tool-log.png` |
| Knowledge Base response | `test-03-knowledge-base.png` |
| Official memory sessions | `test-04-memory-session-a.png`, `test-04-memory-session-b.png` |
| Fresh-customer memory sessions | `test-04-fresh-memory-a.png`, `test-04-fresh-memory-b.png` |
| Loyalty calculation | `test-05-loyalty-calculation.png` |
| Browser response | `test-06-browser.png` |
| Alarm configuration | `cloudwatch-alarm-configuration.png` |

Both Gateway targets were verified with successful, non-empty tool results. Browser logs also confirmed session creation, navigation, and title retrieval.

`ToolEvidenceHook` is opt-in through `ENABLE_TOOL_EVIDENCE=true` in the process running the agent. It is disabled by default because detailed tool inputs and results are unnecessary for routine operation. Setting this variable only in a local terminal does not enable it inside an already deployed Runtime.

## Monitoring

The configured alarm is `CustomerSupportAgent-HighErrorCount`:

- Namespace: `CustomerSupportAgent`
- Metric: `ErrorCount`
- Log filter: `ERROR`
- Metric value: `1`; default value: `0`
- Statistic: Sum
- Period: five minutes
- Threshold: greater than five
- Evaluation: one of one datapoints
- Missing data: not breaching
- Notification actions: none

This counts matching error log events, not unique failed requests. A handler returning a friendly error message can still produce a Runtime “invocation completed successfully” entry, so that message alone does not establish functional success.

## Limitations and production considerations

- Order and refund data are mocked. Refund requests are not persisted or deduplicated, and return-label data includes a fixed expiry date.
- Customer IDs are supplied in the request. Prompt-based ownership checks do not replace authenticated identity and backend authorization.
- Retrieved memories can preserve earlier inaccurate assistant statements. Production use needs controls for correction, provenance, and retention.
- Prompt grounding reduced unsupported claims but does not guarantee their elimination.
- Loyalty calculations do not update a real customer balance.
- The tier-only fallback redeems no points and awards no new points.
- Managed-KB substitution and Python-version changes are documented deviations, not claims of grading approval.

## Cleanup

After preserving submission evidence and following the course’s resource-retention guidance:

1. Use `agentcore destroy` for the configured Runtime and inspect which resources it removes.
2. Remove project Gateway targets and Gateway, Memory, and Knowledge Base resources.
3. Remove the project data source and storage resources that remain, including any partially created OpenSearch resources.
4. Remove the project REST API, Lambda functions, alarm, and custom metric filter when no longer needed.
5. Remove project-specific roles and deployment artifacts only after confirming they are not shared.

Do not delete unrelated sandbox resources or shared deployment buckets.

## Reflection

See [reflection.md](reflection.md) for the design decision, troubleshooting experience, and production improvement.

## Attribution

This submission builds on the [Udacity cd14763-project-starter repository](https://github.com/udacity/cd14763-project-starter). The supplied order-tracking and refund Lambda implementations are retained as provided. The original license is preserved in `LICENSE.txt`.
