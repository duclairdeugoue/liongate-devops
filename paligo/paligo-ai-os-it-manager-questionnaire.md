# Paligo AI OS — IT Manager Infrastructure Questionnaire

**Document Purpose:** Pre-setup discovery questions to be addressed by the IT Manager before
development begins or continues on the Paligo AI Operating System.  
**Scope:** Local development environment · Cloud/online infrastructure · Dev-to-deployment pipeline · Testing strategy  
**Reference Document:** `05-ai-operating-system-review.md` — Paligo AI OS v1.0  
**Date:** June 2026  

---

## Table of Contents

1. [Current Infrastructure Inventory](#1-current-infrastructure-inventory)
2. [Local Development Environment](#2-local-development-environment)
3. [Cloud & Online Infrastructure](#3-cloud--online-infrastructure)
4. [Private LLM Infrastructure](#4-private-llm-infrastructure)
5. [Vector Database & Embedding Pipeline](#5-vector-database--embedding-pipeline)
6. [MCP Tool Layer & Service APIs](#6-mcp-tool-layer--service-apis)
7. [Data Residency & Security](#7-data-residency--security)
8. [CI/CD & Deployment Pipeline](#8-cicd--deployment-pipeline)
9. [Testing Strategy](#9-testing-strategy)
10. [Monitoring, Observability & Logging](#10-monitoring-observability--logging)
11. [Team Access, Credentials & Secrets Management](#11-team-access-credentials--secrets-management)
12. [Cost & Budget Alignment](#12-cost--budget-alignment)
13. [What Is Missing — Gap Assessment](#13-what-is-missing--gap-assessment)

---

## 1. Current Infrastructure Inventory

> Goal: Understand what already exists so we do not duplicate or conflict with live systems.

### 1.1 Existing Servers & Compute

| # | Question |
|---|----------|
| 1 | What servers or virtual machines are currently provisioned and running (on-prem or cloud)? Please list hostname, provider, region, CPU/RAM/disk, and current purpose. |
| 2 | Is there an existing Kubernetes cluster or Docker Swarm in use? If yes, which version and who manages it? |
| 3 | Are there any GPU-equipped instances already provisioned? If yes, what GPU model and how many VRAM GB? |
| 4 | What is the current total monthly cloud infrastructure spend, and what provider(s) are used (AWS, GCP, Azure, Hetzner, OVH, other)? |

### 1.2 Existing Databases

| # | Question |
|---|----------|
| 5 | Which PostgreSQL version is currently running, and does it have the `pgvector` extension installed? |
| 6 | Is MongoDB already deployed for the product catalogue? What version, and is it a managed service or self-hosted? |
| 7 | Is ClickHouse already deployed for the market pricing index? If yes, what version and who manages the schema? |
| 8 | Is there a Redis or Valkey instance in use (for caching or queuing)? |
| 9 | Are all databases backed up? What is the backup frequency and retention period? |

### 1.3 Existing Application Services

| # | Question |
|---|----------|
| 10 | Is Directus already running? What version, and is the schema migrated and stable? |
| 11 | Is Odoo already deployed? What version, and which modules are active? Is the Odoo Wiki/Knowledge module in use? |
| 12 | Is Keycloak already running? What version, and have the Paligo realms and client configurations been created? |
| 13 | What other application services are currently running (Nuxt frontend, API gateway, logistics connector, etc.)? |

---

## 2. Local Development Environment

> Goal: Ensure every developer — Germany and Douala — can run the AI OS stack locally with identical tooling.

### 2.1 Developer Machines

| # | Question |
|---|----------|
| 14 | What is the standard developer machine specification (OS, CPU, RAM, disk)? Is there a difference between the Germany team and the Douala team? |
| 15 | Is there a minimum RAM requirement enforced for running the full local stack? (The AI OS stack with a local LLM likely requires ≥ 32 GB RAM.) |
| 16 | Are GPU-equipped developer machines available for testing local LLM inference? If not, what is the plan for local LLM testing? |
| 17 | Are developer machines running on Apple Silicon (M-series), Intel/AMD Linux, or Windows with WSL2? |

### 2.2 Local Tooling & Dependencies

| # | Question |
|---|----------|
| 18 | Is Docker Desktop / Docker Engine already installed on all developer machines? What version? |
| 19 | Is there a `docker-compose.yml` or Dev Container configuration that launches the full local stack (Directus, Odoo, Keycloak, PostgreSQL, MongoDB, Redis)? If yes, where is it stored? |
| 20 | Is there an `.env.example` or secrets template file that developers use to configure their local environment variables? |
| 21 | Is Ollama or any local LLM runner already installed on developer machines for local AI testing? |
| 22 | What package managers and runtime versions are standardised? (Node.js version, Python version, pnpm/yarn/npm, pip/uv) |
| 23 | Is there a `Makefile`, `justfile`, or shell script that automates the local stack bootstrap? |

### 2.3 Local AI OS Development

| # | Question |
|---|----------|
| 24 | Is there a local vector database instance (pgvector or Weaviate) available for development and testing of the embedding pipeline? |
| 25 | Is there a local embedding model (e.g., `nomic-embed-text` via Ollama) available so developers can test vector ingestion without hitting paid APIs? |
| 26 | Is there a mock or stub layer for the MCP tools so developers can test agent logic without live database or API connections? |
| 27 | Are there seed data scripts that populate the local databases with representative Paligo data (products, merchants, orders) for AI OS testing? |

---

## 3. Cloud & Online Infrastructure

> Goal: Understand the online topology from staging to production and how the AI OS fits in.

### 3.1 Environments

| # | Question |
|---|----------|
| 28 | How many deployment environments exist today (e.g., `dev`, `staging`, `production`)? Are they fully separated (different cloud accounts / VPCs)? |
| 29 | Is there a staging environment that mirrors production where the AI OS components can be tested with real data before going live? |
| 30 | What is the DNS and domain structure? Which domains and subdomains are reserved for AI OS services (e.g., `ai.paligo.de`, `api-ai.paligo.de`)? |

### 3.2 Networking & Security Groups

| # | Question |
|---|----------|
| 31 | Is there a VPC or private network isolating internal services from the public internet? What CIDR ranges are in use? |
| 32 | Are AI OS services (vector DB, private LLM endpoint, MCP tool layer) intended to be fully private (internal network only) or partially public-facing? |
| 33 | What is the ingress/load balancer setup? (Nginx, Traefik, AWS ALB, Cloudflare, other) |
| 34 | Are there existing firewall rules or security groups that block specific ports or outbound traffic that could interfere with AI services? |

### 3.3 Container Orchestration

| # | Question |
|---|----------|
| 35 | Will the AI OS microservices (each agent, the embedding pipeline, the MCP tool server) be deployed as Docker containers in Kubernetes or as standalone Docker Compose services? |
| 36 | If Kubernetes: which distribution and version (EKS, GKE, K3s, self-managed)? Is there a Helm chart repository already configured? |
| 37 | Is there a container registry (AWS ECR, GitHub Container Registry, Docker Hub private) where built images are pushed and pulled from? |
| 38 | What is the cluster's current capacity headroom for additional AI OS workloads? |

---

## 4. Private LLM Infrastructure

> Goal: The AI OS requires a self-hosted LLM for all sensitive data (customer PII, merchant pricing, financial data). This is a hard architectural requirement.

### 4.1 GPU Compute

| # | Question |
|---|----------|
| 39 | Has a GPU instance been provisioned or budgeted for the private LLM? (The AI OS review specifies AWS EC2 G5.2xlarge or equivalent as the minimum.) |
| 40 | What GPU model is targeted — NVIDIA A10G (G5), A100 (P4d/P3), or RTX 4090 (gaming-grade for cost reduction)? What is the VRAM capacity? |
| 41 | Is the GPU instance intended to run 24/7 or to be started on demand? What is the expected availability SLA for the private LLM endpoint? |
| 42 | Is the GPU instance in the same cloud region as the primary application servers to minimise latency? |

### 4.2 LLM Runtime & Model

| # | Question |
|---|----------|
| 43 | Which LLM serving framework is planned or already set up — vLLM, Ollama, LM Studio, TGI (Text Generation Inference), or another? |
| 44 | Which base model is targeted for the private LLM — Llama 3.1 (8B, 70B), Mistral Large, Qwen 2.5, or another? What quantisation level (Q4, Q8, BF16)? |
| 45 | Has the model been downloaded and verified to run on the target GPU instance? What is the measured tokens/second throughput? |
| 46 | Is there a private model registry or storage bucket where model weights are stored (so re-provisioning does not require re-downloading from Hugging Face)? |
| 47 | Does the private LLM endpoint expose an OpenAI-compatible API (`/v1/chat/completions`)? If not, what API format does it use? |

---

## 5. Vector Database & Embedding Pipeline

> Goal: The knowledge base is the foundation of the entire AI OS. No vector DB = no grounded AI agents.

### 5.1 Vector Database

| # | Question |
|---|----------|
| 48 | Has a decision been made on the vector database — pgvector (extend existing PostgreSQL) or Weaviate (dedicated managed service)? |
| 49 | If pgvector: is the `pgvector` extension installed and enabled? What version? Have indexes been benchmarked for the expected document volume? |
| 50 | If Weaviate: is a Weaviate Cloud instance provisioned, or will it be self-hosted? What version? |
| 51 | What is the expected total document volume for the initial knowledge base ingestion (Odoo Wiki pages, product specs, logistics docs, etc.)? |
| 52 | Has the embedding dimension been decided? (e.g., 1536 for OpenAI text-embedding-3-small, 768 for nomic-embed-text) Is the vector index created with the correct dimension? |

### 5.2 Embedding Model & Pipeline

| # | Question |
|---|----------|
| 53 | Which embedding model will be used — OpenAI `text-embedding-3-small`, `text-embedding-3-large`, or a self-hosted model (nomic-embed-text, BGE, E5)? |
| 54 | Is there a document ingestion pipeline already built (chunking strategy, metadata tagging, upsert logic)? If yes, in what language and framework (Python + LangChain, LlamaIndex, custom)? |
| 55 | How will the Odoo Wiki content be exported and ingested into the vector database? Is there an Odoo export job or webhook already set up? |
| 56 | Is there a scheduled job or event trigger for the 24-hour re-embedding refresh described in the AI OS architecture? |
| 57 | How is chunk overlap and chunk size configured? Has it been tuned for Paligo's document types (product specs, business rules, pricing tables)? |

---

## 6. MCP Tool Layer & Service APIs

> Goal: Agents need to call live platform services via MCP. These tools must be built, exposed, and reachable.

| # | Question |
|---|----------|
| 58 | Is there a dedicated MCP server already implemented? If yes, in what language/framework (TypeScript MCP SDK, Python MCP SDK, other)? |
| 59 | Which MCP tools are already implemented vs. still to be built? (Reference: `get_order_status`, `list_merchant_products`, `get_tracking_info`, `create_claim`, `get_pricing_index`, `update_product_listing`, `get_merchant_sla_metrics`, `search_web`, `query_knowledge_base`) |
| 60 | Are the underlying service APIs (Order Service, Product Service, Merchant Service, Logistics Connector, Analytics) already stable and documented? Or are they still in development? |
| 61 | What authentication method do MCP tools use to call backend services (API key, JWT, service account, mTLS)? |
| 62 | Is there an OpenAPI / Swagger spec or MCP schema file documenting all available tools and their input/output contracts? |
| 63 | How are MCP tool errors and timeouts handled? Is there a retry strategy or circuit breaker pattern in place? |

---

## 7. Data Residency & Security

> Goal: The AI OS has strict data residency requirements — sensitive data must never leave Paligo infrastructure.

| # | Question |
|---|----------|
| 64 | Is there a formal data classification policy that defines what counts as sensitive (PII, merchant financial data, pricing)? |
| 65 | At the infrastructure level, is there network-level isolation between the private LLM path (sensitive data) and the cloud LLM API path (public data)? |
| 66 | Are API keys for external LLMs (Anthropic Claude, OpenAI) stored in a secrets manager (AWS Secrets Manager, HashiCorp Vault, Doppler, other)? |
| 67 | Is there a data masking or anonymisation step before any data is sent to a cloud LLM API? If yes, how is it implemented? |
| 68 | Is the vector database encrypted at rest and in transit? Is access restricted by network policy and/or authentication? |
| 69 | Is there a GDPR data retention policy defined for AI conversation logs, embedded documents, and agent interaction history? |
| 70 | Who has been designated as the Data Protection Officer (DPO) or data privacy lead for the AI OS? |

---

## 8. CI/CD & Deployment Pipeline

> Goal: Understand how AI OS code changes flow from a developer's machine to staging and production.

### 8.1 Source Control

| # | Question |
|---|----------|
| 71 | Where is the AI OS code hosted — GitHub, GitLab, Gitea, Bitbucket? Is there a dedicated repository or is it a monorepo subdirectory? |
| 72 | What is the branching strategy? (GitFlow, trunk-based, feature branches) Who approves pull requests and merges to `main`? |
| 73 | Are there branch protection rules enforcing code review and passing CI checks before merge? |

### 8.2 CI Pipeline

| # | Question |
|---|----------|
| 74 | What CI platform is in use — GitHub Actions, GitLab CI, Jenkins, CircleCI, other? |
| 75 | Does the CI pipeline build Docker images, run tests, and push to the container registry automatically on merge? |
| 76 | Is there a separate CI pipeline for the AI OS components, or are they part of the main Paligo platform pipeline? |
| 77 | How long does the current CI build take end to end? Are there caching strategies in place for Docker layers and dependency installs? |

### 8.3 CD & Deployment

| # | Question |
|---|----------|
| 78 | Is there an automated deployment to staging on every merge to `main`, or is staging deployment a manual trigger? |
| 79 | What is the deployment mechanism to production — Kubernetes rolling update, blue/green deployment, canary, or manual? |
| 80 | Is there a deployment approval gate before production? Who approves production deployments? |
| 81 | How are database migrations (schema changes for PostgreSQL, pgvector index updates) handled in the deployment pipeline? |
| 82 | What is the rollback procedure if a deployment causes a production issue? How quickly can a rollback be executed? |

---

## 9. Testing Strategy

> Goal: AI systems require specific testing approaches beyond standard unit/integration tests.

### 9.1 Standard Testing

| # | Question |
|---|----------|
| 83 | Is Vitest already configured for the Paligo platform? Are there existing test suites that the AI OS components should integrate with? |
| 84 | What is the current test coverage baseline for the platform? Is there a minimum coverage threshold enforced in CI? |
| 85 | Are integration tests run against real services (local Docker Compose) or mocked dependencies? |

### 9.2 AI-Specific Testing

| # | Question |
|---|----------|
| 86 | Is there a framework or tool in place for **LLM evaluation** — testing that agents produce correct, grounded, non-hallucinated responses? (e.g., RAGAS, DeepEval, custom evals) |
| 87 | Are there golden dataset test cases (question + expected answer pairs) defined for each agent (CS Agent, Merchant Agent, Catalogue Agent, Pricing Agent, Onboarding Agent)? |
| 88 | How will **retrieval quality** be tested — i.e., how do we verify that the vector search is returning the most relevant knowledge chunks for a given query? |
| 89 | Is there a regression test suite that runs after each knowledge base re-ingestion to detect if embedding changes have degraded agent response quality? |
| 90 | How will **MCP tool calls** be tested in isolation — unit testing each tool's logic with mock API responses? |
| 91 | Is there an end-to-end test scenario simulating a full agent interaction (e.g., a customer asking "Where is my order?" through to a logistics API response)? |
| 92 | What is the strategy for testing the **private LLM** — is there a standard benchmark (MMLU, HellaSwag) to verify model quality after deployment? |

### 9.3 Performance & Load Testing

| # | Question |
|---|----------|
| 93 | Has load testing been planned for the AI OS? What is the expected peak concurrent user count (customers, merchants, developers) interacting with agents? |
| 94 | What is the acceptable latency SLA for an agent response (e.g., < 5 seconds for a CS Agent reply)? Has this been benchmarked against the chosen LLMs? |
| 95 | Is there a load testing tool (k6, Locust, Artillery) already in use for the Paligo platform that can be extended for AI OS stress testing? |

---

## 10. Monitoring, Observability & Logging

> Goal: AI systems fail silently. Proper observability is critical for debugging agent behaviour and tracking costs.

### 10.1 General Observability

| # | Question |
|---|----------|
| 96 | What is the current logging stack — ELK (Elasticsearch + Logstash + Kibana), Loki + Grafana, Datadog, CloudWatch, other? |
| 97 | Is distributed tracing in place (OpenTelemetry, Jaeger, Zipkin)? If yes, can AI OS agent interactions be traced end-to-end through MCP tool calls? |
| 98 | Is there a metrics and alerting platform (Prometheus + Grafana, Datadog, New Relic)? Are alerting rules configured for service downtime? |

### 10.2 AI-Specific Observability

| # | Question |
|---|----------|
| 99 | Has **Langfuse** been evaluated or deployed for LLM observability (tracing prompt inputs, outputs, token counts, latency per agent call)? |
| 100 | Are LLM API costs (Claude API, OpenAI API) being tracked at the per-agent level to validate the €500–€2,000/month estimate from the AI OS review? |
| 101 | Is there a plan to log and review agent conversations (with appropriate GDPR controls) to detect quality degradation over time? |
| 102 | Will there be a dashboard for the Product Owner showing AI OS health metrics — agent uptime, average response time, escalation rate, knowledge base freshness? |

---

## 11. Team Access, Credentials & Secrets Management

> Goal: The distributed team (Germany + Douala) needs consistent, secure access to all infrastructure components.

| # | Question |
|---|----------|
| 103 | Is there a centralised secrets manager for all infrastructure credentials (DB passwords, API keys, LLM API keys, Keycloak client secrets)? Which tool — AWS Secrets Manager, Vault, Doppler, `.env` files in a private repo? |
| 104 | Are developer credentials and environment variables provisioned per-person, or is a shared credential used? (Shared credentials are a security risk.) |
| 105 | What is the process for a new developer to receive all necessary access on Day 1? Is there a documented onboarding checklist? |
| 106 | Are Keycloak roles and permissions already defined for AI OS admin, AI OS developer, and read-only observer roles? |
| 107 | How are SSH keys or cloud console access managed for Germany vs. Douala team members? Is there MFA enforced on all cloud accounts? |
| 108 | Is there a VPN or bastion host that developers must use to connect to production infrastructure? Is it configured and accessible from Douala? |

---

## 12. Cost & Budget Alignment

> Goal: Validate that the infrastructure budget aligns with the indicative costs in the AI OS review (€1,200–€4,200/month).

| # | Question |
|---|----------|
| 109 | Has a monthly infrastructure budget been approved for the AI OS? What is the approved figure? |
| 110 | Is the GPU instance (estimated €600–€1,500/month) already approved and within budget? |
| 111 | Are cloud LLM API keys (Anthropic Claude, OpenAI) already provisioned with billing configured and a spend cap set? |
| 112 | Who is responsible for monitoring and approving infrastructure cost overruns — IT Manager, Product Owner, or CTO? |
| 113 | Is there a cost tagging or labelling strategy to attribute cloud spend per service and per environment (dev, staging, prod)? |

---

## 13. What Is Missing — Gap Assessment

> These questions identify blockers that must be resolved before AI OS development can begin or continue.

| # | Question | Criticality |
|---|----------|-------------|
| 114 | Is the PostgreSQL + pgvector instance up and accessible? | 🔴 Blocker — Phase 0 |
| 115 | Is the Odoo Wiki module active with at least some content that can be used for the initial ingestion test? | 🔴 Blocker — Phase 0 |
| 116 | Is there a working embedding pipeline (even a basic script) that can chunk and embed an Odoo document into pgvector? | 🔴 Blocker — Phase 0 |
| 117 | Is a local LLM (Ollama + Llama 3.1) running on at least one machine for agent development and testing? | 🟠 High — Phase 1 |
| 118 | Is the MCP server scaffolding created with at least one working tool (`query_knowledge_base`)? | 🟠 High — Phase 1 |
| 119 | Is a cloud LLM API key (Anthropic Claude) provisioned, tested, and available to the development team? | 🟠 High — Phase 1 |
| 120 | Is there a Docker Compose or Dev Container file that lets any developer boot the full local AI OS stack in under 10 minutes? | 🟡 Medium — pre-Phase 1 |
| 121 | Is Langfuse or an equivalent AI observability tool set up in at least the staging environment? | 🟡 Medium — pre-Phase 2 |
| 122 | Is the GPU cloud instance provisioned for the private LLM, even if the model is not yet deployed? | 🟡 Medium — pre-Phase 3 |
| 123 | Is there a GDPR-compliant data handling policy signed off for AI-processed customer and merchant data? | 🟡 Medium — pre-Phase 3 |

---

## Notes & Instructions for the Interview

- Conduct this interview **before** any Phase 0 development tasks begin.
- Record answers directly in this document, adding an **Answer** column or sub-section under each question.
- For any question answered with "not yet done," create a corresponding ticket in the project backlog immediately.
- Questions marked 🔴 in Section 13 are **hard blockers** — development cannot meaningfully proceed without them.
- Share the completed document with Christian (Tech Lead), Jonathan (Project Manager), and the Douala Tech Lead as the infrastructure baseline record.

---

*This document was generated from the Paligo AI OS Review v1.0 (June 2026).  
Review and update after each infrastructure change or when moving to a new implementation phase.*
