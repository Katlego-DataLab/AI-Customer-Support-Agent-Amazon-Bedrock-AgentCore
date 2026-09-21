# AI Customer Support Agent, Amazon Bedrock AgentCore

[![AWS](https://img.shields.io/badge/AWS-Bedrock%20AgentCore-orange?logo=amazon-aws)](https://aws.amazon.com/bedrock/agentcore/)
[![Python](https://img.shields.io/badge/Python-3.12-blue?logo=python)](https://www.python.org/)
[![Framework](https://img.shields.io/badge/Framework-Strands%20Agents-blueviolet)](https://strandsagents.com/)
[![Protocol](https://img.shields.io/badge/Protocol-MCP-informational)](https://modelcontextprotocol.io/)
[![Status](https://img.shields.io/badge/Status-Complete-brightgreen)]()
[![License](https://img.shields.io/badge/License-MIT-lightgrey)]()

A production-style AI customer support agent built with the Strands Agents SDK and deployed to Amazon Bedrock AgentCore Runtime. It handles order tracking, refunds, product and policy Q&A, cross-session memory, loyalty discount math, and live web lookups through a single conversational interface, integrating five distinct AgentCore capabilities into one coherent system rather than isolated demos.

## What it does

- **Order tracking and refunds** — calls backend Lambda functions through the AgentCore Gateway using the Model Context Protocol (MCP), with one target fronted by an API Gateway REST API and another invoked directly as a Lambda ARN.
- **Grounded product and policy answers** — a Retrieval-Augmented Generation (RAG) tool queries a Bedrock Knowledge Base built from a product catalog, rather than letting the model guess.
- **Cross-session memory** — a custom hook retrieves relevant customer facts and preferences from AgentCore Memory before each response and persists new interactions afterward, so the agent recalls a customer across separate sessions.
- **Exact loyalty discount math** — a sandboxed AgentCore Code Interpreter tool runs deterministic arithmetic for points redemption and tier discounts, instead of relying on the LLM to do math.
- **Live web lookups** — an AgentCore Browser Tool lets the agent retrieve real content from a live page when asked.

## Architecture

```
Customer prompt
      |
      v
Bedrock AgentCore Runtime (main.py, direct code deploy)
      |
      +--> AgentCore Gateway (MCP) ---> API Gateway REST API ---> Lambda (order-tracker)
      |                            \--> Lambda ARN target ------> Lambda (refund-processor)
      |
      +--> Bedrock Knowledge Base (Titan Embeddings v2 + OpenSearch Serverless)
      |
      +--> AgentCore Memory (semantic facts + user preference strategies)
      |
      +--> AgentCore Code Interpreter (loyalty discount calculation)
      |
      +--> AgentCore Browser Tool (live page retrieval)
```

## Repository structure

```
.
├── main.py                  # Agent entrypoint, tools, memory hook, Gateway client
├── lambda/
│   ├── order_tracker.py     # Order and customer lookups (API Gateway proxy target)
│   ├── refund_processor.py  # Refunds and return labels (Lambda ARN target)
│   └── lambda_schema         # MCP tool schema for the refund-processor target
├── product_catalog.txt      # Source data for the Knowledge Base
├── pyproject.toml           # Dependencies (uv-managed)
├── reflection.txt           # Design decisions, challenges, and production considerations
└── project 2-Screenshot/    # Test evidence for all six verification scenarios
```

## Design decisions worth noting

**Loyalty discount capping.** `calculate_loyalty_discount` caps points redemption at 50% of the order's value and rounds down to the nearest 500-point increment before converting to a dollar amount. This mirrors how real loyalty programs prevent an order from being reduced to near-zero through points alone, and keeps the arithmetic deterministic for the Code Interpreter rather than leaving edge cases to the model's judgment.

**Scoped IAM over broad managed policies.** The Runtime's execution role is granted narrowly-scoped inline permissions (`bedrock:Retrieve` on the specific Knowledge Base ARN, specific Browser session actions on the AWS-managed browser resource) rather than a wide managed policy. This came directly out of debugging a real failure mode described below.

**Explicit Gateway failure handling.** The Gateway MCP connection is wrapped in a try/except that distinguishes `TimeoutError` and `ConnectionError` from other exceptions, logging each case clearly via `logger.exception` rather than letting a Gateway outage crash the agent or fail silently.

## A real bug worth mentioning

Early on, the Knowledge Base and Browser tools failed without raising a visible error — the agent instead returned a generic "access issue" message and, in one case, fabricated plausible-but-incorrect loyalty tier benefits instead of admitting it had no data. Tracing this through CloudWatch logs showed the Runtime's auto-created execution role had permissions for memory and the code interpreter, but nothing scoped to Knowledge Base retrieval or Browser session actions. The fix was two targeted inline policy statements, not a broader policy grant.

The lesson: a missing permission on a customer-facing agent doesn't always look like a crash — it can look like a wrong answer delivered with full confidence, which is a worse failure mode for a support agent than an outright error.

## Verification

All six required test scenarios were run against the deployed agent and are documented in `project 2-Screenshot/`:

| Test | Scenario | Result |
|---|---|---|
| 1 | Order tracking | Shipping status, tracking number, carrier, delivery date returned |
| 2 | Refund processing | Refund ID, approved status, credit timeline returned |
| 3 | Knowledge Base (RAG) | Platinum tier benefits correctly grounded in catalog data |
| 4 | Cross-session memory | Customer name and preference recalled in a separate session |
| 5 | Loyalty discount calculation | Points redeemed, tier discount, final total, remaining points computed |
| 6 | Browser tool | Live page content retrieved from a real URL |

## Tech stack

- **Amazon Bedrock AgentCore** — Runtime, Gateway, Memory, Knowledge Bases, Code Interpreter, Browser Tool
- **Strands Agents SDK** — agent orchestration, tool definitions, hooks
- **Model Context Protocol (MCP)** — Gateway-to-agent tool exposure
- **AWS Lambda, API Gateway, OpenSearch Serverless, S3, IAM**
- **Python 3.12**, managed with `uv`

## Notes

This was built as a Udacity Cloud AWS AI Agent project. All AWS resources used a temporary sandboxed lab account and have since been decommissioned; this repository preserves the source code, configuration, and test evidence.
Built with 🤍 by Katlego Mathebula
