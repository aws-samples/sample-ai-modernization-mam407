# AgentCore Resources — Workshop Modules

> [!NOTE]
> **🚀 Coming Soon:** A future update will add a local test environment with a deployable CloudFormation template, allowing you to run the full workshop independently.

> [!IMPORTANT]
> **Interested in running this workshop?** If you are an AWS customer and would like to conduct this workshop, please reach out to your AWS account team or AWS Partner team.

This folder contains two Jupyter notebook modules that extend the [AI Modernization Workshop (MAM407)](https://catalog.workshops.aws/ai-modernization/en-US). These notebooks serve as open-source reference material for hosting MCP servers and AI agents on **Amazon Bedrock AgentCore Runtime**.

The workshop walks through the full end-to-end setup (infrastructure, knowledge bases, etc.), while the resources here are meant to be customized with your own DynamoDB table names, Knowledge Base IDs, or any other data sources you're connecting to.

## Shared Resources

| File | Description |
|------|-------------|
| `utils.py` | Helper functions for setting up Amazon Cognito user pools, authenticating users, and creating AgentCore IAM execution roles |
| `product_description.md` | Product descriptions used to populate the Bedrock Knowledge Base (headphones, webcam, t-shirt, running shoes, coffee maker) |

---

## Module 3: Hosting an MCP Server on AgentCore Runtime

**Notebook:** `module_3/hosting_mcp_server.ipynb`

This module covers creating, testing, and deploying an MCP (Model Context Protocol) server to AgentCore Runtime. The server exposes five tools that interact with DynamoDB and Bedrock Knowledge Bases.

### MCP Server Tools

| Tool | Description |
|------|-------------|
| `search_product_knowledge(query)` | Semantic product search via Bedrock Knowledge Base |
| `list_products()` | Lists all products from DynamoDB |
| `get_product(product_id)` | Gets detailed product info |
| `check_inventory(product_id)` | Checks stock levels and availability |
| `place_order(product_id, quantity)` | Places an order, updates inventory, records transaction |

### What You'll Do

1. Install dependencies (`requirements.txt`)
2. Create the MCP server (`mcp_server.py`) using FastMCP with `stateless_http=True`
3. Test locally with a client (`my_mcp_client.py`) using two terminal sessions
4. Set up Cognito authentication for AgentCore
5. Deploy the MCP server to AgentCore Runtime
6. Invoke the deployed server remotely

### Key Configuration

- The MCP server listens on `0.0.0.0:8000/mcp` (AgentCore requirement)
- You must paste your **Knowledge Base ID** (from [Module 2 of the workshop](https://catalog.workshops.aws/ai-modernization/en-US)) into `mcp_server.py` before deploying
- DynamoDB table names default to `secure-ecommerce-stack-product-catalog` and `secure-ecommerce-stack-transaction-logs` — update these to match your own tables

### Dependencies

```
mcp>=1.10.0
boto3<1.40.45
bedrock-agentcore<=0.1.7
bedrock-agentcore-starter-toolkit<=0.1.5
```

---

## Module 4: Hosting a Strands Agent on AgentCore Runtime

**Notebook:** `module_4/hosting_agent.ipynb`

This module covers creating, testing, and deploying a **Strands agent** that connects to the MCP server from Module 3. The agent provides natural-language e-commerce assistance by calling the MCP tools.

### What You'll Do

1. Install dependencies (`requirements_agent.txt`)
2. Test the Strands agent locally by connecting to the deployed MCP server
3. Define the agent as a FastAPI app (`agent_agentcore.py`) with an `/invocations` endpoint
4. Set up Cognito authentication
5. Configure and launch the agent to AgentCore Runtime
6. Add IAM permissions for the agent to access SSM parameters and Secrets Manager
7. Store agent configuration (ARN, bearer token) for remote invocation

### Agent Details

- Uses **Claude Haiku** (`global.anthropic.claude-haiku-4-5-20251001-v1:0`) as the underlying model
- Includes a system prompt with tool usage strategy, customer interaction guidelines, and error handling instructions
- Retrieves MCP server ARN and bearer token from SSM Parameter Store and Secrets Manager at runtime

### Key Configuration

- Module 3 must be completed first — the agent connects to the deployed MCP server
- The agent's system prompt and model ID can be customized in `agent_agentcore.py`
- Lambda functions (`websocket_lambda.py`, `processor_lambda.py`) are deployed to the CloudFormation-provisioned Lambdas to connect the agent to the front-end WebSocket API

### Dependencies

```
strands-agents<=1.10.0
boto3<1.40.45
mcp>=1.0.0
fastapi>=0.116.1
pydantic<=2.11.9
```

---

## Running Locally

```bash
cd MAM407-mcp-workshop-opensource/agentcore_resources
pip install jupyter
jupyter notebook
```

Then open either `module_3/hosting_mcp_server.ipynb` or `module_4/hosting_agent.ipynb` and run cells sequentially.

## Customization

These notebooks are designed as starting points. Before running them, update the following to match your environment:

- `mcp_server.py` → `table_name`, `transaction_name`, and `knowledge_base_id` variables
- `agent_agentcore.py` → system prompt, model ID, and tool configuration
- Swap out the Bedrock Knowledge Base for any other data source (OpenSearch, RDS, etc.)

For the full workshop context and earlier modules (infrastructure setup, knowledge base creation), see the [AI Modernization Workshop](https://catalog.workshops.aws/ai-modernization/en-US).

## Security Best Practices for Production

These notebooks are built for workshop and learning purposes. When moving to production, follow these AWS security best practices:

### Secrets and Configuration Management

- Store sensitive values like Knowledge Base IDs, DynamoDB table names, and credentials in **AWS Systems Manager Parameter Store** or **AWS Secrets Manager** rather than hardcoding them in source files. This improves both security and maintainability — you can rotate values without redeploying code.
- See: [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html), [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

### Least-Privilege IAM Policies

- The notebooks currently grant broad DynamoDB and Bedrock permissions for convenience. In production, scope IAM policies to the specific actions and resources each component actually needs. For example, the MCP server only needs `dynamodb:GetItem` and `dynamodb:Scan` on the product table — it doesn't need `BatchWriteItem` or access to unrelated tables.
- See: [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

### Input Validation and Guardrails

- The current MCP tools do not implement input validation or sanitization. In production, validate and sanitize all inputs to prevent injection attacks and abuse. Use [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) to add content filtering, topic restrictions, and prompt attack detection to your agent interactions. This is on the workshop roadmap.

### Error Handling and Observability

- The notebooks return raw error messages for debugging convenience. In production, implement secure error handling that logs detailed errors internally (e.g., via CloudWatch) while returning generic messages to end users. Use [Amazon Bedrock AgentCore Observability](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore-observability.html) for monitoring agent performance, tracing tool invocations, and debugging issues. This is also on the workshop roadmap.

### WebSocket and Lambda Security

- The WebSocket API and Lambda architecture is functional but will evolve. Planned improvements include:
  - Streaming responses for better user experience
  - Front-end user authentication tied to Cognito so permissions are scoped per user
- In the meantime, lock down IAM permissions so the processor Lambda is only invokable from the WebSocket Lambda, which is only invokable from the API Gateway. This way, even if a connection ID were compromised, the downstream resources remain protected.
- See: [API Gateway WebSocket Security](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-protect.html), [Lambda Resource-Based Policies](https://docs.aws.amazon.com/lambda/latest/dg/access-control-resource-based.html)

---

## Project Structure

```
agentcore_resources/
├── README.md
├── utils.py                          # Cognito & IAM helper functions
├── product_description.md            # Product catalog descriptions
├── module_3/
│   ├── hosting_mcp_server.ipynb      # MCP server notebook
│   ├── requirements.txt
│   └── images/
└── module_4/
    ├── hosting_agent.ipynb           # Strands agent notebook
    ├── requirements_agent.txt
    └── images/
```
