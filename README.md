# Multi-Agent Customer Support System — LangGraph + AWS Bedrock

[![CI](https://github.com/parikshit06reddy-cloud/langgraph-bedrock-multi-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/parikshit06reddy-cloud/langgraph-bedrock-multi-agent/actions/workflows/ci.yml)
[![Security](https://github.com/parikshit06reddy-cloud/langgraph-bedrock-multi-agent/actions/workflows/security.yml/badge.svg)](https://github.com/parikshit06reddy-cloud/langgraph-bedrock-multi-agent/actions/workflows/security.yml)
[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE.txt)

> Production-grade multi-agent customer support system orchestrating five specialized agents on AWS Bedrock with LangGraph, with a Bedrock Knowledge Base RAG layer, MCP-compatible tool gateway, and enterprise security controls (Cognito, IAM isolation, VPC PrivateLink).

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Agent roles](#agent-roles)
- [Key features](#key-features)
- [Repository layout](#repository-layout)
- [Quick start](#quick-start)
- [Environments and configuration](#environments-and-configuration)
- [Production hardening](#production-hardening)
- [Observability](#observability)
- [Security](#security)
- [Contributing](#contributing)
- [License](#license)

---

## Why this exists

This project is an opinionated reference for running a **supervisor + specialized subagents** pattern on AWS using LangGraph and Amazon Bedrock. It mirrors a production architecture used to orchestrate parallel, stateless agents with strong context isolation and measurable improvements in end-to-end task completion time.

It is intended as a starting point for teams building:

- Multi-domain assistants (support, ops, finance, healthcare, etc.)
- Hybrid retrieval systems mixing structured DB tools with unstructured KB RAG
- Agentic platforms that need to coexist with existing REST/Lambda tooling via MCP

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│       React Frontend (S3 + CloudFront + Cognito)            │
└──────────────────────┬──────────────────────────────────────┘
                       │ WebSocket / GraphQL (AppSync)
┌──────────────────────▼──────────────────────────────────────┐
│              Supervisor Agent (LangGraph)                    │
│   Plans, routes, and consolidates across all sub-agents      │
└──┬──────────┬────────────┬─────────────┬─────────────────────┘
   │          │            │             │
┌──▼──┐  ┌───▼───┐  ┌─────▼────┐  ┌────▼──────┐  ┌──────────────┐
│Order│  │Product│  │Personal- │  │Trouble-   │  │  RAG Agent   │
│ Mgmt│  │Recom. │  │ization   │  │shooting   │  │(Knowledge    │
│Agent│  │Agent  │  │Agent     │  │Agent      │  │ Base Bedrock)│
└──┬──┘  └───────┘  └──────────┘  └───────────┘  └──────────────┘
   │
┌──▼──────────────────────────────────────────────────────────┐
│         MCP Tool Gateway (Lambda + API Gateway)              │
│   Aurora PostgreSQL → Agent-ready tools via semantic routing │
└─────────────────────────────────────────────────────────────┘
   │
┌──▼──────────────────────────────────────────────────────────┐
│         Security & Observability Layer                       │
│  Cognito Auth │ IAM Role Isolation │ VPC PrivateLink │       │
│  CloudWatch Dashboards │ OpenTelemetry Traces                │
└─────────────────────────────────────────────────────────────┘
```

---

## Tech stack

| Layer | Technologies |
|---|---|
| Agent orchestration | LangGraph (stateful graph workflows) |
| Foundation model | Anthropic Claude 3.5 Haiku / Sonnet on Amazon Bedrock |
| RAG | Bedrock Knowledge Base (unstructured), Aurora PostgreSQL (structured) |
| Tool gateway | Model Context Protocol (MCP), AWS Lambda, API Gateway |
| Frontend | React 19 + TypeScript, Vite 7, AWS Amplify, AppSync (GraphQL + WebSocket) |
| Auth | Amazon Cognito (user + identity pools) |
| Data | Aurora PostgreSQL (Serverless v2), Amazon S3, Bedrock Knowledge Base |
| Compute | Amazon ECS Fargate (one container per specialized agent) |
| Observability | Amazon CloudWatch (dashboards, alarms), OpenTelemetry |
| Security | IAM least-privilege, Secrets Manager, VPC PrivateLink, Cognito |
| IaC | AWS CDK (TypeScript) |

---

## Agent roles

| Agent | Responsibility | Tools |
|---|---|---|
| **Supervisor** | Plans tasks, routes to sub-agents, consolidates responses | All agent tools |
| **Order Management** | Order tracking, returns, inventory queries | Aurora PostgreSQL via RDS Data API |
| **Product Recommendation** | Personalized product suggestions | Bedrock Knowledge Base, embeddings |
| **Personalization** | Customer profile + preference tracking | DynamoDB, Aurora |
| **Troubleshooting** | Technical issue resolution | Knowledge Base RAG, Lambda |
| **RAG Agent** *(extension)* | Unstructured retrieval from PDFs / docs | Bedrock KB, Pinecone |

---

## Key features

- **Subagents pattern** — Centralized supervisor + stateless specialists; supports parallel tool calling
- **Bedrock Knowledge Base RAG** — Unstructured retrieval alongside structured Aurora queries
- **MCP tool gateway** — Existing Lambda + Aurora queries exposed as MCP-compatible agent tools
- **Real-time streaming** — Token-level streaming via AWS AppSync WebSocket to React UI
- **Enterprise security** — Cognito auth, per-agent IAM roles, PrivateLink to Bedrock, Secrets Manager
- **CloudWatch observability** — Per-agent dashboards: latency, tool selection accuracy, error rate, session duration
- **Environment-aware infra** — `prod` defaults flip to deletion-protected DB, container health checks on, ECS Exec off, ALB deletion-protected

---

## Repository layout

```
.
├── agents/                 # One Python service per specialized agent
│   ├── supervisor-agent/
│   ├── order-management-agent/
│   ├── product-recommendation-agent/
│   ├── personalization-agent/
│   ├── troubleshooting-agent/
│   └── knowledge-base/     # Seed data + KB scaffolding
├── frontend/               # React + Vite + Amplify UI
├── infra/                  # AWS CDK (TypeScript) stacks
│   └── lib/                # ecs / database / load-balancer / monitoring / etc.
├── scripts/                # Local helpers
├── tests/                  # Cross-service test scaffolding
├── deploy.sh               # End-to-end deploy helper
└── .github/workflows/      # CI + Security pipelines
```

---

## Quick start

Prerequisites:

- Python `>=3.10`, Node.js `>=20`
- AWS CLI v2 configured for the target account
- Docker (for `cdk deploy` to build agent images)
- AWS Bedrock model access enabled for Claude 3.5 Haiku/Sonnet

```bash
git clone https://github.com/parikshit06reddy-cloud/langgraph-bedrock-multi-agent.git
cd langgraph-bedrock-multi-agent

cp .env.example .env

cd frontend && npm install && cd ..
cd infra    && npm install && cd ..

cd infra
npx cdk bootstrap
npx cdk deploy --all --context environment=dev
```

For each agent service:

```bash
cd agents/supervisor-agent
pip install -e ".[dev]"
pytest
```

---

## Environments and configuration

The CDK stacks read `environment` from CDK context. The default is `dev`. Set `prod` to flip safer defaults:

```bash
npx cdk deploy --all --context environment=prod
```

| Setting | dev | prod |
|---|---|---|
| Aurora `deletionProtection` | `false` | `true` |
| Aurora `log_statement` | `all` | `ddl` |
| ECS container `healthCheck` | off | `/health` curl every 30s |
| ECS `enableExecuteCommand` | `true` | `false` |
| ECS deploy `circuitBreaker.rollback` | `false` | `true` |
| ALB `deletionProtection` | `false` | `true` |

---

## Production hardening

- ALB listeners do **not** auto-open `0.0.0.0/0` (`open: false`); rely on explicit security group rules
- Aurora PostgreSQL: KMS-encrypted at rest, private subnets only, RDS Data API enabled
- Per-agent IAM roles scope Bedrock, Secrets Manager, RDS Data API, DynamoDB, AppSync to the minimum needed
- Bedrock and Secrets Manager accessed via VPC endpoints where supported
- Container images built via CDK `DockerImageAsset` and pinned by digest at deploy time

---

## Observability

Each agent emits to CloudWatch:

| Metric | Description |
|---|---|
| `AgentLatencyP95` | 95th percentile per-agent response time |
| `ToolSelectionAccuracy` | % of tool calls hitting the correct tool |
| `ErrorRate` | Failed agent runs per minute |
| `SessionDuration` | Average multi-turn session length |
| `RAGFaithfulness` | RAGAS faithfulness on sampled responses |

OpenTelemetry traces flow through to CloudWatch / your APM of choice via the OTel collector sidecar.

---

## Security

- Continuous: CodeQL (Python + TypeScript), Gitleaks, Trivy filesystem, `npm audit`, `pip-audit`
- Weekly: Dependabot for pip, npm, GitHub Actions, Docker
- Push protection: GitHub secret scanning blocks secrets at push time

Report vulnerabilities via [SECURITY.md](SECURITY.md).

---

## Contributing

PRs welcome. CI runs lint + per-agent tests, frontend build, and `cdk synth` on every PR. Security workflows must be green before merge.

---

## License

Apache-2.0. See [LICENSE.txt](LICENSE.txt).

---

**Author:** Parikshit Reddy — [LinkedIn](https://www.linkedin.com/in/parikshitr/) · [GitHub](https://github.com/parikshit06reddy-cloud)

> Extended fork of [aws-solutions-library-samples/guidance-for-multi-agent-orchestration-langgraph-on-aws](https://github.com/aws-solutions-library-samples/guidance-for-multi-agent-orchestration-langgraph-on-aws) with RAG agent, MCP gateway, and enterprise security patterns.
