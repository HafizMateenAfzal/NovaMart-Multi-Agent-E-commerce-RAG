# NovaMart Multi-Agent E-commerce RAG

A production-style **multi-agent customer support system** built with the [Strands Agents SDK](https://github.com/strands-agents/sdk-python) and **Amazon Bedrock AgentCore**. An Orchestrator agent understands each customer request, routes it to specialist worker agents, and hands the collected findings to a Communication agent that writes the final reply.

> Project for the **Future AWS Agent Engineer** program (a Udacity Nanodegree sponsored by AWS). Result: **120/120 on the automated test suite, passed project review.**

---

## What it does

NovaMart (a fictional e-commerce company) gets thousands of support requests: order status, returns and refunds, policy questions. Agents used to look up orders and cross-check policies by hand. This system automates that workflow end to end:

- Routes each request to the right specialist
- Reads order and customer data from DynamoDB
- Answers policy questions with **parallel multi-agent RAG** over three Bedrock Knowledge Bases
- Decides return eligibility by customer tier (30-day Standard, 60-day Premium)
- Enforces safety rules with **Bedrock Guardrails**
- Keeps session context in **AgentCore Memory**
- Traces every request through **CloudWatch and AWS X-Ray**

## Architecture

```
Customer Request
       |
OrchestratorAgent  (Claude Haiku 4.5, temp 0.0 - routing only, owns WorkflowState)
       |
  +----+---------------+-----------------+--------------------+
  |                    |                 |                    |
InventoryAgent     PolicyAgent      RefundAgent       CommunicationAgent
(DynamoDB)        (multi-agent RAG) (DynamoDB)        (final customer reply)
                       |
      +----------------+----------------+
 ReturnsRetriever  ShippingRetriever  WarrantyRetriever    <- run in PARALLEL
  (Bedrock KB)       (Bedrock KB)       (Bedrock KB)

Shared state: DynamoDB WorkflowState table with optimistic locking
```

### Agents

| Agent | Model | Temp | Responsibility | Tools |
|---|---|---|---|---|
| **Orchestrator** | Claude Haiku 4.5 | 0.0 | Routes requests, manages WorkflowState, never answers directly | `initialize_session`, `route_to_inventory_agent`, `route_to_policy_agent`, `route_to_refund_agent`, `route_to_communication_agent` |
| **Inventory** | Claude Sonnet 4.5 | 0.1 | Gathers order and customer facts, never decides | `check_order_status`, `get_customer_tier`, `list_customer_orders` |
| **Policy** | Claude Sonnet 4.5 | 0.2 | Coordinates parallel retrieval and synthesizes a grounded answer | `search_all_policies` |
| **Refund** | Claude Sonnet 4.5 | 0.1 | Decides return eligibility and starts the return | `get_inventory_context`, `initiate_refund` |
| **Communication** | Claude Sonnet 4.5 | 0.3 | Writes the final warm, professional reply | `get_full_workflow_context` |

The three retriever sub-agents inside the Policy agent use temperature 0.0 for deterministic retrieval.

### Routing rules (Orchestrator)

| # | Trigger | Action |
|---|---|---|
| 1 | Every request | Call `initialize_session` first |
| 2 | Order status / return / refund | Inventory agent first, then Refund agent for returns |
| 3 | Policy meaning questions | Policy agent |
| 4 | Account questions ("what is my tier?") | Inventory agent, never the Policy agent |
| 5 | Math / calculation questions | Straight to Communication agent |
| 6 | Every request, last step | Communication agent composes the final reply |

### Shared WorkflowState

Each worker writes its findings to a DynamoDB record keyed by `session_id`. Updates use **optimistic locking** (`expected_version`), so concurrent agents cannot overwrite each other's results.

## AWS services used

- **Amazon Bedrock**: Claude Sonnet 4.5 and Haiku 4.5, Titan Text Embeddings V2
- **Bedrock Knowledge Bases** with **S3 Vectors** (three KBs: returns, shipping, warranty)
- **Bedrock AgentCore**: Runtime (deployed with the AgentCore CLI) and Memory (session summaries, 7-day retention)
- **Bedrock Guardrails**: content filters, PII handling, denied topics, profanity filter
- **DynamoDB**: orders, customers, workflow state
- **CloudWatch Logs and AWS X-Ray**: logging and distributed tracing
- **CloudFormation**: foundation infrastructure

## Guardrail policy

- **Content filters:** sexual, violence and hate at HIGH; insults and misconduct at MEDIUM
- **PII:** block credit card numbers and SSNs; anonymize emails and phone numbers
- **Denied topics:** competitor products, pricing negotiations, legal threats. Plain arithmetic on a stated price and discount is explicitly allowed, so "5 items at $29.99 with 10% off" is answered rather than blocked.
- **Words:** managed profanity list
- Published as a **numbered version** (not DRAFT) and attached to every agent's model at runtime

## Results

| Check | Result |
|---|---|
| Automated test suite (`python tests/test_agent.py all`) | **120 / 120** |
| Parallel retrieval from all three Knowledge Bases | Passing |
| X-Ray service map | Orchestrator connected to all workers and Knowledge Bases |

Screenshots are in [`screenshots/`](screenshots/):

- `tests_120_1.png` to `tests_120_5.png`: the full test run ending at 120/120
- `xray_map_full.png`, `xray_map_top.png`, `xray_map_bottom.png`: X-Ray trace map
- `xray_list.png`: trace map list view

## Project structure

```
.
├── src/
│   ├── agent_orchestrator.py    # agent builders, routing tools, guardrail, deployment (my implementation)
│   ├── agent_observability.py   # X-Ray tracing and CloudWatch logging (provided)
│   ├── agent_utils.py           # terminal trace UI (provided)
│   ├── agentcore_cli.py         # AgentCore CLI wrapper (provided)
│   ├── bedrock_kb_retrieval.py  # Knowledge Base retrieval helper (provided)
│   └── demo.py                  # one end-to-end request (provided)
├── infrastructure/
│   ├── starter_stack.yaml       # CloudFormation stack
│   ├── seed_data.py             # mock customers, orders, policy documents
│   └── cleanup.py               # deletes everything the project created
├── tests/test_agent.py          # automated test suite
├── agentcore/                   # AgentCore CLI project
├── diagrams/                    # architecture diagrams
├── config.py                    # central configuration
├── .env.example                 # environment variable template
└── requirements.txt
```

## Getting started

**Prerequisites:** Python 3.12+, Node.js 20+, [uv](https://docs.astral.sh/uv/), AWS CLI, an AWS account with Bedrock model access in `us-east-1` (Claude Sonnet 4.5, Claude Haiku 4.5, Titan Embed Text v2).

```powershell
# 1. Install tools
npm install -g @aws/agentcore@0.30.0
python -m venv venv
venv\Scripts\activate            # macOS/Linux: source venv/bin/activate
pip install -r requirements.txt

# 2. Credentials (temporary lab keys shown; use your own profile otherwise)
$env:AWS_ACCESS_KEY_ID="..."
$env:AWS_SECRET_ACCESS_KEY="..."
$env:AWS_SESSION_TOKEN="..."
$env:AWS_DEFAULT_REGION="us-east-1"

# 3. Foundation infrastructure and sample data
aws cloudformation deploy --template-file infrastructure/starter_stack.yaml --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1
python infrastructure/seed_data.py
cp .env.example .env
python config.py                 # prints resource names
```

**Create the three Knowledge Bases** (returns, shipping, warranty) in the Bedrock console or CLI:

- S3 prefixes `policies/returns/`, `policies/shipping/`, `policies/warranty/`
- Embedding model Titan Text Embeddings V2
- Existing S3 Vectors bucket and the matching index (`returns-policy-index`, etc.)
- Sync each data source, then add `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID` to `.env`

```powershell
# 4. Test, then deploy guardrail + runtime + memory + observability
python tests/test_agent.py task2
python tests/test_agent.py task5
python src/agent_orchestrator.py deploy
# copy AGENTCORE_RUNTIME_ARN, GUARDRAIL_ID, GUARDRAIL_VERSION into .env

# 5. Run end to end
python src/agent_orchestrator.py test        # 3 scenarios + X-Ray trace IDs
python src/agent_orchestrator.py chat        # interactive
python tests/test_agent.py all               # full suite
```

### Example scenarios

| Customer says | Route |
|---|---|
| "I want to return my order ORD-27176" | Orchestrator -> Inventory -> Refund -> Communication |
| "What is the return policy for premium customers?" | Orchestrator -> Policy (3 parallel KB retrievers) -> Communication |
| "How much are 5 items at $29.99 with 10% off?" | Orchestrator -> Communication |

### Clean up

```powershell
python infrastructure/cleanup.py          # dry run
python infrastructure/cleanup.py --yes    # delete everything
```

Delete the Knowledge Bases manually if they were created outside the stack.

## What I learned

- Designing an orchestrator-worker hierarchy with clear tool boundaries and deterministic routing
- Running retrieval sub-agents in parallel with `ThreadPoolExecutor`
- Coordinating agents through shared state with optimistic locking
- Writing guardrail topic definitions that block abuse without false positives on normal requests
- Reading distributed traces to confirm the real call chain across agents and Knowledge Bases

## Ideas for next steps

- Run adversarial guardrail tests (prompt injection, competitor and legal-threat prompts) and document the blocks
- Add persistent chat sessions with a DynamoDB session store
- Build a CloudWatch dashboard for invocation counts, latency per agent, and guardrail triggers
- Add Cognito authentication and a React front end for the AgentCore endpoint

## Credits

Project brief, starter code, CloudFormation template, seed data, test suite and diagrams are provided by **Udacity** (Future AWS Agent Engineer program). My implementation is in `src/agent_orchestrator.py`.

## Author

**Hafiz Mateen Afzal**
[LinkedIn](https://www.linkedin.com/in/hafizmateena/) | [GitHub](https://github.com/HafizMateenAfzal)
