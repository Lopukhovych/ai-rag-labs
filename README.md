# AI-RAG-labs

A connected set of AI engineering projects built around one enterprise knowledge corpus: a permission-aware RAG system, a multi-agent ops desk, a full-stack copilot, and the evaluation harness that measures all of them. Based on AWS (us-east-1, Amazon Bedrock, serverless, AWS CDK in Python, GitHub Actions with OIDC).

## Repositories

| # | Repo | What it does |
|---|------|--------------|
| data | [ai-rag-data](https://github.com/Lopukhovych/ai-rag-data) | EnterpriseRAG-Bench pinned, with permissions, two tenants, and fixed `dev` / `heldout` splits |
| infra | [ai-rag-infra](https://github.com/Lopukhovych/ai-rag-infra) | Shared AWS resources as CDK: OpenSearch, Cognito, Bedrock Guardrails, S3, GitHub OIDC roles |
| 01 | [enterprise-rag-qa](https://github.com/Lopukhovych/enterprise-rag-qa) | Hybrid RAG on OpenSearch with citations, abstention, and role/tenant filters |
| 02 | [ops-desk-agents](https://github.com/Lopukhovych/ops-desk-agents) | Strands multi-agent system on Bedrock AgentCore with approvals and memory |
| 03 | [workplace-copilot](https://github.com/Lopukhovych/workplace-copilot) | React + FastAPI on Lambda behind CloudFront, Cognito sign-in, streaming, fallback |
| 04 | [evalkit](https://github.com/Lopukhovych/evalkit) | Scorers, calibrated LLM judge, CI eval gate, online trace scoring |

Rows are in build order. Each repo has its own README with setup, deploy, and teardown commands.

## How they connect

```
ai-rag-infra (CDK) creates the shared resources and publishes their names in SSM /ai-rag/*:
  S3 buckets · OpenSearch domain · Cognito user pool · Bedrock Guardrail · OIDC roles per repo

ai-rag-data ──publish──> S3 (data bucket)
                             │
                             ▼ ingest.yml (chunk, embed with Bedrock Titan)
                     enterprise-rag-qa ──index──> OpenSearch (`rag` alias: BM25 + vectors + ACLs)
                                                       ▲                 ▲
                                          retrieve()   │                 │  retrieve() / answer()
                                                       │                 │
                     ops-desk-agents (AgentCore) ──────┘                 │
                             ▲                                           │
                             └──invokes── workplace-copilot (CloudFront + Lambda, Cognito sign-in)
                                                       │
                                                       ▼ traces
                                                  S3 (traces bucket)
                                                       │
evalkit: dev dependency + reusable eval-gate workflow in every repo; score-traces reads the traces
```

## Shared model tiers

All AWS-hosted code uses Amazon Bedrock through the Converse API. Model IDs are pinned in each repo's `.env`; check current IDs with `aws bedrock list-inference-profiles`.

| Tier | Amazon Bedrock | Used for |
|------|----------------|----------|
| Cheap | Amazon Nova Micro | Query rewriting, triage, summarization |
| Fast | Amazon Nova Lite | Answers, drafting, extraction, agent tool calls |
| Fallback | Anthropic Claude Haiku | Second vendor: fallback tier and comparison runs |
| Strong | Anthropic Claude Sonnet | LLM judge and Ragas |
| Embeddings | Amazon Titan Text Embeddings V2 (1024 dims) | Shared OpenSearch index |
| Guardrail | Bedrock Guardrails (shared, pinned version) | Prompt attacks, content filters, PII, grounding |

## License

MIT
