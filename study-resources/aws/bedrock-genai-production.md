# AWS Bedrock + Production GenAI Patterns — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-05-15
**Difficulty:** Intermediate / Advanced

## Overview

This plan covers AWS Bedrock as a production GenAI platform from a DevOps/SRE lens: how the Converse API, Knowledge Bases (managed RAG), Agents (action groups), Guardrails, prompt caching, and cross-region inference fit together — plus the IAM, observability, and cost-optimization patterns you'd defend in an interview. By the end you'll be able to design and justify a RAG chatbot architecture (KB + Agent + Guardrail + CloudWatch token dashboard) and articulate when to pick Bedrock over SageMaker JumpStart or a self-hosted vLLM-on-EKS stack.

## Resources

### 1. Amazon Bedrock or Amazon SageMaker AI? (Decision Guide) ⏱️ 12 min

- **URL:** <https://docs.aws.amazon.com/decision-guides/latest/bedrock-or-sagemaker/bedrock-or-sagemaker.html>
- **Type:** Official Docs (Decision Guide)
- **Summary:** AWS's own framing of when to pick Bedrock (managed FM API, no GPU ops) vs SageMaker (custom training, full model lifecycle). Read this first to anchor your mental model of where Bedrock sits in the AWS AI stack, and to compare against your existing EKS/ECS workload patterns.

### 2. Inference using the Converse API (with tool use) ⏱️ 20 min

- **URL:** <https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html>
- **Type:** Official Docs
- **Summary:** The canonical reference for the unified Converse / ConverseStream API — message format, system prompts, `toolConfig` for function calling, and streaming. This is the API you'll actually call from Lambda, so understand the difference vs the legacy `InvokeModel` and how tool use mirrors the Step Functions task-token pattern you already know. Skim the linked "tool use examples" page for the JSON schema shape.

### 3. Knowledge Bases for Amazon Bedrock (Prescriptive Guidance) ⏱️ 18 min

- **URL:** <https://docs.aws.amazon.com/prescriptive-guidance/latest/retrieval-augmented-generation-options/rag-fully-managed-bedrock.html>
- **Type:** Official Docs (Prescriptive Guidance)
- **Summary:** Managed RAG end-to-end: supported vector stores (OpenSearch Serverless, Aurora pgvector, Pinecone, Neptune Analytics, MongoDB, Redis Enterprise, S3 Vectors), embedding model choices (Titan, Cohere), chunking strategies (fixed / semantic / hierarchical / custom Lambda), and `RetrieveAndGenerate` vs `Retrieve` APIs. The single best doc to plan the KB half of your RAG chatbot build.

### 4. How Amazon Bedrock Agents work + Return Control ⏱️ 15 min

- **URL:** <https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html>
- **Type:** Official Docs
- **Summary:** Agent orchestration loop, action groups (Lambda executor vs OpenAPI schema vs function schema), and the critical `RETURN_CONTROL` pattern that lets your own caller execute the action instead of Bedrock invoking Lambda directly — useful when actions need credentials or network paths Bedrock can't reach. Also read the linked OpenAPI schema page for action group contracts.

### 5. Detect and filter harmful content with Bedrock Guardrails ⏱️ 15 min

- **URL:** <https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html>
- **Type:** Official Docs
- **Summary:** The six guardrail policy types — content filters, denied topics, word filters, sensitive information (PII) filters with Block/Mask modes, contextual grounding (hallucination detection with grounding + relevance score thresholds), and prompt-attack detection. Note that guardrails apply independently on input and output and can be attached to KBs and Agents, not just raw model calls. Follow the link to the contextual grounding page — it's the production differentiator for RAG.

### 6. Prompt caching + Global cross-Region inference ⏱️ 15 min

- **URL:** <https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>
- **Type:** Official Docs (read both linked pages)
- **Summary:** Two huge cost/latency levers. Prompt caching gives up to ~90% cost reduction on cached tokens and ~85% latency reduction for repeated context (long system prompts, RAG headers). Then read <https://docs.aws.amazon.com/bedrock/latest/userguide/global-cross-region-inference.html> for cross-region inference profiles — and importantly, the gotcha that cross-region routing reduces cache hit rates because caches are regional.

### 7. Monitor model invocation using CloudWatch Logs and S3 ⏱️ 12 min

- **URL:** <https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html>
- **Type:** Official Docs
- **Summary:** How to wire up full request/response logging (account-level toggle, S3 gzipped JSON batches or CloudWatch Logs, 100 KB body cap, large-data S3 offload). Pair with the parent monitoring page (<https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring.html>) for the CloudWatch metrics namespace — `InvocationCount`, `InputTokenCount`, `OutputTokenCount`, `InvocationLatency`, `InvocationThrottles`, `InvocationClientErrors` — which is exactly the dataset for your token-usage dashboard. Builds directly on the CloudWatch deep dive from 2026-05-12.

### 8. Monitoring Generative AI applications with Bedrock + CloudWatch (AWS Cloud Ops blog) ⏱️ 15 min

- **URL:** <https://aws.amazon.com/blogs/mt/monitoring-generative-ai-applications-using-amazon-bedrock-and-amazon-cloudwatch-integration/>
- **Type:** AWS Blog
- **Summary:** End-to-end practitioner walkthrough showing how to combine model invocation logs, CloudWatch Logs Insights queries, and dashboards to track per-model token spend, latency p95, throttles, and guardrail interventions. Read this as the implementation template for the "CloudWatch dashboard for token usage" half of today's build, and notice how it composes with the OpenTelemetry patterns you already covered.

### 9. Build responsible AI applications with Bedrock Guardrails (AWS ML blog) ⏱️ 10 min

- **URL:** <https://aws.amazon.com/blogs/machine-learning/build-responsible-ai-applications-with-amazon-bedrock-guardrails/>
- **Type:** AWS Blog
- **Summary:** Real-world design patterns for layering guardrails — independent input vs output policies, applying the same guardrail across multiple FMs for consistent policy, and using `ApplyGuardrail` as a standalone API to protect non-Bedrock or self-hosted models. Closes the loop on the "guardrails as a cross-cutting policy layer" mental model that's often missed in shallow tutorials.

## Study Tips

- **Map every Bedrock concept back to something you already know.** Converse tool use ~ Step Functions task tokens with a JSON schema contract. Agents ~ EventBridge + Lambda orchestration but with an LLM as the router. Knowledge Bases ~ managed ETL + vector index with `RetrieveAndGenerate` as a fronting API. Guardrails ~ WAF for LLM I/O. This compression is what interviews reward.
- **Pay attention to IAM and the data plane.** The control plane is `bedrock:*` (Create/Update Agent, KB, Guardrail). The data plane is `bedrock-runtime:*` (InvokeModel, Converse, InvokeAgent, Retrieve). Cross-account model access goes through inference profiles + resource policies, and KB ingestion needs S3 read + the embedding model invoke permission. Be ready to whiteboard the trust relationships.
- **Cost = tokens x model x (1 - cache_hit_rate) + KB storage + OSS/Aurora vector store + Provisioned Throughput hours.** Default to on-demand; only reach for Provisioned Throughput when you need guaranteed capacity or custom fine-tuned models. Pick the smallest model that hits your eval bar, then optimize with prompt caching before considering PT.

## Next Steps

- Build the hands-on RAG chatbot: drop a few PDFs in S3, create a Knowledge Base backed by OpenSearch Serverless with semantic chunking, attach a Guardrail with PII-Mask and a denied-topics policy, then wire an Agent with a Lambda action group (e.g. "lookup order status") and a CloudWatch dashboard pulling `InputTokenCount` / `OutputTokenCount` / `InvocationThrottles` per model ID.
- Explore Bedrock AgentCore (the newer runtime + observability layer for production agents) and how its tracing model compares to the OpenTelemetry stack you covered earlier.
- Compare your Bedrock RAG against a self-hosted vLLM-on-EKS deployment at 10M+ tokens/day to internalize the break-even point — the 80-95% cost-reduction claim is real but assumes you can absorb the SRE overhead, which is exactly the trade-off interviewers probe.
