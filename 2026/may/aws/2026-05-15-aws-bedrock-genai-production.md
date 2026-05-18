# AWS Bedrock + Production GenAI Patterns -- The Managed Kitchen

> Yesterday's CloudTrail doc closed with one promise: *every Bedrock model invocation is a CloudTrail data event, so "which workflow asked which model what prompt" becomes a Lake/Athena query just like every other API call.* Today we cash that promise from the other direction -- not "how do I audit Bedrock" but "how do I run Bedrock in production at all." Bedrock in 2026 is no longer "Anthropic's API with an AWS bill." It is a **full managed GenAI platform**: a unified inference API across a dozen foundation-model vendors (Anthropic Claude 4.x, Meta Llama 3/4, Mistral, Cohere, Amazon Titan/Nova, Stability AI), managed RAG via **Knowledge Bases**, agentic orchestration with tool-use via **Agents** + action groups, six independent **Guardrail** policy families, **Prompt Caching** with cache-control checkpoints, **Provisioned Throughput** for reserved capacity, **cross-region inference profiles** that route invocations to the lowest-loaded region, and **Model Invocation Logging** that captures full prompts and responses to S3 + CloudWatch (the audit half of yesterday's story). Underneath all of that sits the same control-plane / data-plane split you already know from every other AWS service -- `bedrock:*` for control, `bedrock-runtime:*` and `bedrock-agent-runtime:*` for the actual model calls.
>
> **The core analogy for today: Bedrock is a managed kitchen, and you're a restaurant owner who only brings orders.** Walk it through. In SageMaker JumpStart, AWS gives you the recipes and a fully-stocked test kitchen, but *you* run the line -- you choose the burners (instance types), you keep the gas on (warm pools), you replace the chef when they walk (auto-scaling groups), and you wash the dishes (model artifacts in S3). In **self-hosted on EKS with vLLM**, you're not even buying recipes -- you build the kitchen from scratch, you source the produce (HuggingFace weights), you negotiate with the gas company (GPU spot quotas), you train the dishwashers (KEDA + Karpenter), and you eat every shift the chef calls in sick (CUDA-driver-mismatch oncall). **In Bedrock, you walk up to a serving window with a written order** -- "I want one Claude 4.6 Sonnet with these system instructions and this user message, output capped at 500 tokens, with these guardrails applied at the door" -- and the kitchen returns a finished plate. You never see the burner, you never wash a dish, you never hire a chef. You pay per plate (per token), with a discount for ordering ahead in bulk (Provisioned Throughput) and another discount for ordering the same prep multiple times in the same hour (Prompt Caching). The **menu** is the model catalog (and like any kitchen, the menu varies by region -- Claude 4.6 Sonnet is on the menu in `us-east-1` from day one, but the Tokyo branch might be a month behind). The **specials** are the cross-region inference profiles -- the kitchen routes your order to whichever branch has open capacity, so you never see the "kitchen full" sign. The **librarian who runs the in-house cookbook** is the **Knowledge Base** -- you don't shelve the cookbook yourself anymore (that's RDS for vectors). The **junior contractor with a tool belt and a manager looking over their shoulder** is the **Agent** -- you describe what they're allowed to do and provide the manager's runbook (orchestration loop), and they handle the rest. The **TSA checkpoint at the entrance and exit of every order** is the **Guardrail** -- six independent policies (content, denied topics, word filters, PII, contextual grounding, prompt-attack), applied to both the inbound prompt and the outbound response, optionally also to *non-Bedrock* models via the standalone `ApplyGuardrail` API.
>
> Three corollaries to bolt on. **(1) Bedrock removes the GPU-ops headache, but the cost surface didn't disappear -- it moved.** SageMaker's bill was endpoint-hours. Self-hosted's bill was GPU spot + driver SRE time. Bedrock's bill is tokens, and **output tokens cost ~5x input tokens**, which means the entire art of running Bedrock economically is "write prompts that force *short* responses" -- `max_tokens` is a budget directive, not a safety guard. **(2) Bedrock's IAM model is two services, not one.** The control plane (`bedrock:*` for CreateAgent, UpdateKnowledgeBase, CreateGuardrail) and the data plane (`bedrock-runtime:InvokeModel`, `bedrock-runtime:Converse`, `bedrock-agent-runtime:InvokeAgent`, `bedrock-agent-runtime:Retrieve`) are *separately scopable* and *separately billable in CloudTrail*. The data plane is the one that produces a CloudTrail data event per call -- the one that lets you SQL-query "every prompt this workflow ever sent" the way you saw yesterday. **(3) The Bedrock-vs-SageMaker-vs-self-hosted decision is not a vibes call.** There is a clean three-axis decision: *do you need to choose the model* (Bedrock has ~50 in 2026; SageMaker has thousands; self-hosted has all of HuggingFace), *can you absorb the SRE overhead* (Bedrock: zero; SageMaker: meaningful; self-hosted: a full-time team), and *is data sovereignty solvable with regional KMS* (Bedrock and SageMaker both yes; self-hosted on EKS gives you the most control but the most rope). The break-even point in 2026 economics is roughly **10M tokens/day** -- below that, Bedrock wins on TCO; above that, self-hosted vLLM-on-EKS starts looking attractive if you have the team to run it.

---

**Date**: 2026-05-15
**Topic Area**: aws
**Difficulty**: Intermediate-to-Advanced
**Tags**: bedrock, genai, llm, rag, agents, guardrails, prompt-caching, knowledge-bases, converse-api, inference-profiles

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Bedrock | A managed kitchen -- bring orders, not appliances | Multi-vendor foundation-model API with managed RAG, agents, guardrails, and prompt caching; zero GPU ops |
| Foundation model (FM) | A specific dish on the menu | A pretrained LLM (Claude, Llama, Titan, Nova, Mistral, Cohere, Stability) you invoke via the unified API |
| Control plane (`bedrock:*`) | The kitchen manager's office | Create/Update/Delete: agents, knowledge bases, guardrails, model copies, provisioned throughput |
| Data plane (`bedrock-runtime:*`, `bedrock-agent-runtime:*`) | The serving window | The runtime: `InvokeModel`, `Converse`, `InvokeAgent`, `Retrieve`, `RetrieveAndGenerate`, `ApplyGuardrail` |
| `InvokeModel` (legacy) | Ordering in each chef's native dialect | Per-vendor JSON request format -- different shape for Claude vs Llama vs Titan |
| `Converse` API | The unified order form | Single message-array schema across all chat-capable models; the production default in 2026 |
| `ConverseStream` | Watching the dish plated piece by piece | Streaming token-by-token via SSE; same API surface, different response handling |
| Tool use / function calling | The chef calling out "I need garlic from the pantry" | Model returns a `toolUse` block; caller executes; result fed back via `toolResult` |
| Knowledge Base | The in-house cookbook with a librarian | Managed RAG: ingestion + chunking + embedding + vector store + retrieval, all one resource |
| Vector store (OSS / Aurora / Pinecone / Mongo / Redis) | Different kinds of cookbook shelving | The actual index storage -- OpenSearch Serverless is default; Aurora pgvector is cheapest small-scale |
| Chunking strategy (fixed / semantic / hierarchical) | How you tear the cookbook into recipe cards | Fixed = N tokens; semantic = LLM-decided breaks; hierarchical = parent-chunk-retrieved, child-chunk-matched |
| `Retrieve` API | "Bring me the relevant recipes" | Returns top-K chunks; you compose the final prompt yourself |
| `RetrieveAndGenerate` | "Cook me the dish from the relevant recipes" | One call: retrieve + stuff + invoke + return; opinionated end-to-end RAG |
| Hybrid search | Searching by ingredient name AND by description | Vector + keyword (BM25); not on by default; the fix for SKU/ID queries |
| Agent | A junior contractor with a tool belt and a manager | Orchestrator LLM + action groups + KB + guardrails + memory + collaborators |
| Action group | One tool on the contractor's belt | A callable function: Lambda-executed OR `RETURN_CONTROL` for client-side execution |
| OpenAPI / function schema | The instruction sheet for the tool | JSON Schema (or OpenAPI 3) declaring the function name, params, types, descriptions |
| `RETURN_CONTROL` | The contractor walks out and asks you to use your own tool | Agent surfaces the tool call to the caller instead of invoking Lambda -- only path to in-process/browser/on-prem |
| Agent memory | The contractor's notebook between visits | Multi-session persistent context; opt-in; expires per configured retention |
| Multi-agent collaboration | The contractor calls in a specialist sub | Supervisor agent routes to collaborator agents; orchestration loop nests |
| Guardrail | TSA checkpoint at entry and exit | Six policy families applied to prompt (input) and response (output) independently |
| Content filter | "Is this a weapon?" baggage scan | Four categories: hate, insults, sexual, violence -- each None/Low/Medium/High |
| Denied topics | The "no politics" rule sign at the front desk | Up to 30 natural-language topic descriptions you don't want discussed |
| Word filter | The list of banned phrases | Profanity preset + custom word/phrase blocklist |
| Sensitive info filter | The agent who redacts SSNs from documents | 30+ built-in PII types + custom regex; Block (reject) or Mask (`{EMAIL}`) |
| Contextual grounding | "Did the chef cook from the recipe, or improvise?" | Two scores: **grounding** (response supported by source?) + **relevance** (response answers the question?); the hallucination killer for RAG |
| Prompt-attack filter | "Is this person trying to social-engineer the chef?" | Detects jailbreak / prompt-injection patterns; tunable strength |
| `ApplyGuardrail` | The TSA station you can rent and deploy at any door | Standalone API: apply Bedrock guardrails to ANY model (OpenAI, self-hosted, etc.); separate per-call charge |
| Prompt caching | The chef's prep station -- chop onions once, reuse all night | `cachePoint` checkpoints in messages; cached tokens cost ~10% of normal input; 5-min TTL; region-scoped |
| `cachePoint` checkpoint | A "freeze this prep" mark in the order | Marks the prefix to cache; placed at end of system prompt or end of RAG context block |
| Provisioned Throughput (PT) | A reserved table at a Michelin restaurant | Pre-purchased model units (MUs) with 1-mo or 6-mo commitment; required for custom fine-tuned models |
| On-Demand | Walking up to the serving window | Default; pay-per-token; subject to per-region token-per-minute quotas |
| Cross-region inference profile | A hotel-chain reservation that routes you to whichever location has a room | An ARN that fronts multiple regional copies of the same model; the kitchen picks the branch |
| Model ID | A specific kitchen branch | `anthropic.claude-sonnet-4-6-20260301-v1:0` -- one region only |
| Inference profile ARN | The chain-level booking number | `arn:aws:bedrock:us-east-1:...:inference-profile/us.anthropic.claude-sonnet-4-6-v1:0` -- routed across regions |
| Batch inference | A catering order for next Tuesday | ~50% discount; variable latency up to 24h; offline enrichment workloads |
| Bedrock Studio | The franchise's tasting-room kiosk for non-engineers | SSO-gated workspace + sandbox; requires IAM Identity Center, NOT standalone IAM users |
| Model Invocation Logging | The kitchen's CCTV that records every order ticket and plated dish | Account-level toggle; full prompts + responses to S3 (+ optional CloudWatch); the audit half |
| `AWS/Bedrock` CloudWatch namespace | The kitchen's gauges -- orders/in-tokens/out-tokens/latency | `Invocations`, `InputTokenCount`, `OutputTokenCount`, `InvocationLatency`, `InvocationClientErrors`, `InvocationServerErrors`, `InvocationThrottles` |
| SageMaker JumpStart | The serviced test-kitchen rental | You bring chefs (endpoints), AWS provides the appliances; thousands of models, full lifecycle |
| Self-hosted vLLM on EKS | The bare warehouse you turned into a kitchen | Maximum control, maximum SRE cost; break-even ~10M tokens/day with a team that can run it |

---

## The Big Picture in One Diagram

```
THE MANAGED KITCHEN -- WHAT EACH PIECE IS
==============================================================================

   THE DINING ROOM (your application code)              THE KITCHEN (Bedrock)
   ---------------------------------------              -----------------------

   +-----------------+    Converse / ConverseStream
   |  Lambda /       |--------------------------+
   |  ECS / EKS /    |    InvokeModel (legacy)  |
   |  on-prem client |                          v
   +-----------------+              +------------------------+
            |                       |  Bedrock Runtime       |
            |   bedrock-agent-      |  (bedrock-runtime.     |
            |   runtime:InvokeAgent |   amazonaws.com)       |
            +---------------------->+------------+-----------+
            |   bedrock-agent-      |  Bedrock   |  Cross-
            |   runtime:Retrieve    |  Agent     |  region
            |   bedrock-agent-      |  Runtime   |  Inference
            |   runtime:Retrieve-   |            |  Profile
            |   AndGenerate         |            |  router
            +---------------------->+------------+-----------+
                                                |
            +-----------------------------------+
            |                  |                |               |
            v                  v                v               v
      +-----------+      +-----------+    +-----------+   +------------+
      | Foundation|      | Knowledge |    |  Agent    |   | Guardrail  |
      | Models    |      | Base      |    | (orches-  |   | (6 policy  |
      | (Claude,  |      | (managed  |    |  trator + |   |  families) |
      | Llama,    |      |  RAG)     |    |  action   |   +-----+------+
      | Titan,    |      +-----+-----+    |  groups)  |         |
      | Nova,     |            |          +-----+-----+         |
      | Mistral,  |            |                |               |
      | Cohere,   |            v                |               |
      | Stability)|      +-----------+          v               |
      +-----+-----+      | Vector    |    +-----------+         |
            |            | store:    |    | Action    |         |
            |            | OSS-S /   |    | group:    |         |
            |            | Aurora    |    | Lambda or |         |
            |            | pgvector /|    | RETURN_   |         |
            |            | Pinecone /|    | CONTROL   |         |
            |            | Mongo /   |    +-----------+         |
            |            | Redis     |                          |
            |            +-----+-----+                          |
            |                  ^                                |
            |   Ingestion      | embeddings (Titan/Cohere)      |
            |   from S3 /      |                                |
            |   Confluence /   |                                |
            |   SharePoint /   |                                |
            |   Salesforce /   |                                |
            |   web crawler    |                                |
            +-----------+      |                                |
                        |      |                                |
                        v      |                                v
                  +----------------+               +---------------------+
                  | Source data    |               | ApplyGuardrail      |
                  | bucket (S3)    |               | standalone -- works |
                  +----------------+               | for non-Bedrock too |
                                                   +---------------------+

   THE OBSERVABILITY ROOM                          THE COST LEVERS
   ----------------------                          ----------------
                                                   1. Pick the smallest FM that
   CloudWatch metrics                                 hits your eval bar (Haiku
   (AWS/Bedrock namespace):                           for classify, Sonnet for
     Invocations                                      general, Opus for hardest)
     InputTokenCount                                2. Force short outputs:
     OutputTokenCount                                  output tokens = ~5x input
     InvocationLatency                              3. Cache the long system
     InvocationClientErrors                            prompt + RAG header
     InvocationServerErrors                            (cachePoint, 5-min TTL,
     InvocationThrottles                               region-scoped)
                                                   4. Cross-region inference
   Model Invocation Logging                           profile = no throttles
   (S3 + optional CW Logs):                        5. Batch inference for
     Full prompt + response                           offline = ~50% off
     PII unless filtered                            6. Reserve PT only when
     KMS-encrypt the dest                             (a) custom FM or
                                                      (b) guaranteed capacity
   CloudTrail data events
   bedrock-runtime:InvokeModel
   bedrock-agent-runtime:InvokeAgent
   = "who asked which model what"

   CARDINAL RULES:
     1. Output tokens cost ~5x input. Cap with `max_tokens`. Prompt for short.
     2. Default to inference profile ARN, not model ID, for production.
     3. Knowledge Base default = OpenSearch Serverless. Aurora pgvector for
        small/cheap. Pinecone if you already have it.
     4. Guardrails: turn on contextual grounding for any RAG workload.
        That score is the hallucination killer.
     5. Prompt caching saves 80-90% on cached tokens. 5-min TTL. Region-scoped.
        Cross-region routing CAN invalidate the cache.
     6. CloudTrail captures every InvokeModel/InvokeAgent as a data event.
        Pair with Model Invocation Logging for full prompts.
```

---

## Part 1: The Architecture -- Control Plane, Data Plane, and the Regional Menu

The single most useful frame for Bedrock is the same one that works for every AWS service: **control plane vs data plane**, and Bedrock has the unusual property of also splitting the data plane in two.

### Control plane -- `bedrock.amazonaws.com`

Anything that creates, updates, or deletes a Bedrock *resource* goes through `bedrock:*` IAM actions against the `bedrock.amazonaws.com` endpoint. Canonical examples:

- `bedrock:CreateKnowledgeBase`, `UpdateKnowledgeBase`, `DeleteKnowledgeBase`
- `bedrock:CreateAgent`, `UpdateAgent`, `PrepareAgent`, `CreateAgentAlias`
- `bedrock:CreateGuardrail`, `UpdateGuardrail`, `CreateGuardrailVersion`
- `bedrock:CreateProvisionedModelThroughput`, `DeleteProvisionedModelThroughput`
- `bedrock:CreateModelCustomizationJob` (fine-tuning)
- `bedrock:GetFoundationModel`, `ListFoundationModels`

These appear in CloudTrail as **management events** (free first copy -- same model you saw yesterday). Terraform-managed Bedrock infrastructure lives entirely in this plane.

### Data plane -- two endpoints

The data plane is split into two services, and the IAM actions are scoped to each:

**`bedrock-runtime.amazonaws.com`** -- direct model invocation. The endpoint hostname is `bedrock-runtime.*`, but the IAM service prefix is just `bedrock:` (no separate `bedrock-runtime:` prefix exists -- a common source of confusion). The IAM actions:

- `bedrock:InvokeModel` -- **gates both `InvokeModel` and the modern `Converse` API.** There is no separate `bedrock:Converse` action; if you write that in a policy it's a no-op.
- `bedrock:InvokeModelWithResponseStream` -- gates both `InvokeModelWithResponseStream` and `ConverseStream`.
- `bedrock:ApplyGuardrail` -- standalone guardrail check.

**`bedrock-agent-runtime.amazonaws.com`** -- higher-level orchestration (same `bedrock:` IAM prefix):

- `bedrock:InvokeAgent`
- `bedrock:Retrieve` (KB retrieval only, you assemble the prompt)
- `bedrock:RetrieveAndGenerate` (end-to-end RAG in one call)

Three non-obvious consequences:

1. **`bedrock:InvokeModel` is the production-critical action even if you only ever call `Converse`.** This trips up first-time policy authors who try to scope to `bedrock:Converse` and get `AccessDenied` because that action doesn't exist.
2. **You can scope IAM separately for "can call models directly" vs "can call agents."** A common pattern: developer roles get `bedrock:InvokeModel` against a small model allowlist, while only the application's execution role gets `bedrock:InvokeAgent` against the production agent alias ARN.
3. **Each of these data-plane calls produces its own CloudTrail data event.** Pair with Model Invocation Logging (Part 11) and you have the complete audit answer to "who prompted what model with what input and got what output."

### The regional menu gotcha -- not all models in all regions

Bedrock's foundation-model catalog is **region-by-region**. Anthropic Claude 4.x first lands in `us-east-1`, `us-west-2`, and `eu-central-1`. Amazon Nova is broadest. Meta Llama 4 lands in the major US regions and `eu-west-1`. **There is no region where every model is available.**

The consequences:

- **`ListFoundationModels` returns a different set per region.** Build your model-availability matrix in code; don't hardcode.
- **Cross-region inference profiles (Part 11) are the workaround.** They give you one ARN that routes to whichever region currently has the model deployed *and* has capacity.
- **Compliance constraints (data residency in EU, JP, etc.) often force you into the regions with the smaller menu.** Verify model availability for your target region *before* committing to a model in design docs.

You enable each model explicitly via the "Manage model access" flow in the Bedrock console (programmatic enablement is via the `PutUseCaseForModelAccess`-family APIs -- the console flow is what most teams use). This is a one-time per-account, per-region step. Forgetting it means `InvokeModel` returns `AccessDeniedException: You don't have access to the model with the specified model ID` -- a confusing error since IAM looks fine.

---

## Part 2: The 2026 Foundation-Model Catalog -- Picking the Right Dish

> Note on model IDs: the date-suffixed Claude / Titan / Llama model IDs used throughout this doc (e.g. `anthropic.claude-sonnet-4-6-20260301-v1:0`) are **illustrative placeholders** to show the ID *pattern* — always pull the current authoritative IDs from `aws bedrock list-foundation-models --region <r>` for your target region. Vendors publish new dated revisions on a rolling cadence and the suffix moves; never hardcode without checking.

The menu in May 2026, with picking criteria:

| Family | Model | Strengths | Pick for |
|--------|-------|-----------|----------|
| **Anthropic Claude 4.x** | Opus 4.7 | Highest reasoning, long context (1M tokens), tool use | Hardest reasoning, agentic workflows, complex code synthesis |
| | Sonnet 4.6 | Excellent general; cheaper than Opus | The default for general chat, RAG synthesis, most agent orchestrators |
| | Haiku 4.5 | Fast, cheap, surprisingly capable | Classification, extraction, simple Q&A, high-volume routing |
| **Meta Llama** | Llama 4 (Maverick / Scout) | Open-weight feel, multimodal, long context | When you want open-model semantics with managed hosting |
| | Llama 3.3 70B | Stable; long context | Production workloads validated on Llama 3 |
| **Mistral** | Large 2, Small 3 | Strong code, low latency on Small | European data-residency workloads (Mistral hosted in `eu-*`) |
| **Cohere** | Command R+ | RAG-tuned, multilingual | RAG-heavy workloads where Cohere's tool-use schema fits |
| | Embed v3 (English / Multilingual) | Embedding model | The default embedding choice paired with Cohere or Anthropic generation |
| **Amazon Titan** | Titan Text G1 Premier | AWS-native; broad regional availability | Workloads that need regions where Anthropic isn't yet GA |
| | Titan Embeddings G1 / V2 | Default embedding model on Knowledge Bases | The path-of-least-resistance embedding choice |
| **Amazon Nova** | Nova Pro / Lite / Micro | AWS-native multimodal frontier; broad availability | New 2025+ workloads where you want AWS-native end to end |
| | Nova Canvas / Reel | Image / video generation | Multimodal output |
| **Stability AI** | Stable Diffusion 3.5 / SDXL | Image generation | Product imagery, marketing assets |

### The model-selection rubric

The cost question dominates. Output tokens cost ~5x input tokens, and the price difference *between models* can be 10-15x. The rule:

1. **Pick the smallest model that passes your eval bar.** Run a 100-prompt eval set against Haiku, Sonnet, and Opus. If Haiku passes, ship Haiku. If only Sonnet passes, ship Sonnet. Reserve Opus for workloads where Sonnet's eval is clearly below bar.
2. **Different models for different stages.** A common production pattern: Haiku for the *router* (classifying which agent or KB to invoke), Sonnet for the *worker* (synthesizing the answer), Opus only for *fallback* on hard cases (detected by low contextual-grounding score from the Sonnet pass).
3. **Embedding model = Titan v2 for AWS-native, Cohere v3 for non-English / multilingual.** Don't overthink this. Picking the embedding model is *not* where the latency/quality budget lives.

### Anthropic on Bedrock vs Anthropic direct API -- the dialect gotcha

A subtle production trap: **Anthropic Claude on Bedrock has a slightly different tool-use schema than the direct Anthropic API.** Same model weights, slightly different request envelope. Don't share code paths blindly between a "Bedrock branch" and an "Anthropic direct branch" -- abstract behind an interface and version each. The Converse API (Part 3) flattens this away, which is the main reason to use Converse over `InvokeModel`.

---

## Part 3: The Converse API -- The Unified Order Form

Before Converse (GA late 2024), invoking each FM family meant building a different JSON request body:

```python
# Anthropic Claude via InvokeModel -- "messages" array + anthropic_version
anthropic_body = {
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello"}],
}

# Meta Llama via InvokeModel -- "prompt" string with chat template baked in
llama_body = {
    "prompt": "<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\nHello<|eot_id|>",
    "max_gen_len": 1024,
    "temperature": 0.7,
}

# Amazon Titan via InvokeModel -- "inputText" + textGenerationConfig
titan_body = {
    "inputText": "Hello",
    "textGenerationConfig": {"maxTokenCount": 1024, "temperature": 0.7},
}
```

Three different envelopes, three different response shapes, three different streaming protocols. Every framework (LangChain, LiteLLM, your in-house adapter) ended up with a vendor-dispatch table. **Converse collapses all of this.**

### The Converse request shape

```python
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
    messages=[
        {"role": "user", "content": [{"text": "Summarize the CAP theorem in 2 sentences."}]}
    ],
    system=[{"text": "You are a precise technical assistant. Be brief."}],
    inferenceConfig={
        "maxTokens": 200,           # the budget control -- output tokens cost 5x input
        "temperature": 0.2,
        "topP": 0.9,
        "stopSequences": ["\n\nUser:"],
    },
)

# Same shape regardless of FM family
text = response["output"]["message"]["content"][0]["text"]
print(text)

# Token accounting -- always present, vendor-agnostic
print(response["usage"])
# { "inputTokens": 42, "outputTokens": 87, "totalTokens": 129 }
print(response["stopReason"])   # "end_turn" | "max_tokens" | "stop_sequence" | "tool_use"
print(response["metrics"]["latencyMs"])
```

Three properties to internalize:

1. **`content` is always an array of "content blocks."** A block can be `{"text": "..."}`, `{"image": {...}}`, `{"toolUse": {...}}`, `{"toolResult": {...}}`, or (with cachePoints) `{"cachePoint": {"type": "default"}}`. This is the same shape Anthropic shipped on their direct API in 2024; Bedrock adopted it as the cross-vendor lingua franca.
2. **`system` is its own top-level field, not a message with role=system.** This matters for prompt-caching (the cachePoint typically goes at the *end* of the system block) and for models that have different system-prompt handling under the hood.
3. **`usage.inputTokens` and `usage.outputTokens` are always populated**, regardless of FM family. This is the metric you graph in CloudWatch dashboards (Part 13).

### Tool use via Converse -- the function-calling contract

The killer feature of Converse is that tool use works *the same way* across every model that supports it:

```python
TOOL_SPEC = {
    "tools": [
        {
            "toolSpec": {
                "name": "get_order_status",
                "description": "Look up an order's current status by order ID.",
                "inputSchema": {
                    "json": {
                        "type": "object",
                        "properties": {
                            "order_id": {
                                "type": "string",
                                "description": "The order ID, e.g. ORD-12345",
                            }
                        },
                        "required": ["order_id"],
                    }
                },
            }
        }
    ],
    "toolChoice": {"auto": {}},   # or {"tool": {"name": "get_order_status"}} to force
}

response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
    messages=[{"role": "user", "content": [{"text": "What's the status of ORD-12345?"}]}],
    toolConfig=TOOL_SPEC,
    inferenceConfig={"maxTokens": 500},
)

if response["stopReason"] == "tool_use":
    # The model returned a tool_use block instead of text
    tool_use = next(
        block["toolUse"] for block in response["output"]["message"]["content"]
        if "toolUse" in block
    )
    tool_use_id = tool_use["toolUseId"]
    tool_name = tool_use["name"]
    tool_input = tool_use["input"]   # already-parsed JSON dict

    # YOU execute the tool (DB query, API call, whatever)
    result = lookup_order(tool_input["order_id"])

    # Feed the result back as a toolResult block
    follow_up = bedrock.converse(
        modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
        messages=[
            {"role": "user", "content": [{"text": "What's the status of ORD-12345?"}]},
            {"role": "assistant", "content": response["output"]["message"]["content"]},
            {
                "role": "user",
                "content": [{
                    "toolResult": {
                        "toolUseId": tool_use_id,
                        "content": [{"json": result}],
                    }
                }],
            },
        ],
        toolConfig=TOOL_SPEC,
        inferenceConfig={"maxTokens": 500},
    )
    final_text = follow_up["output"]["message"]["content"][0]["text"]
```

The mental model from Step Functions: **`toolUseId` is the task token.** The model emits a task token; your code fulfills it; you pass the result back keyed by that token. The model resumes. Same pattern, LLM substrate.

### ConverseStream -- the streaming variant

For latency-sensitive UIs, ConverseStream returns an event stream:

```python
response = bedrock.converse_stream(
    modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
    messages=[{"role": "user", "content": [{"text": "Write a haiku about Kubernetes."}]}],
    inferenceConfig={"maxTokens": 200},
)

for event in response["stream"]:
    if "contentBlockDelta" in event:
        delta = event["contentBlockDelta"]["delta"]
        if "text" in delta:
            print(delta["text"], end="", flush=True)
    elif "metadata" in event:
        # final event: usage + metrics
        usage = event["metadata"]["usage"]
        print(f"\n[inputTokens={usage['inputTokens']} outputTokens={usage['outputTokens']}]")
```

Same `usage` accounting, just delivered at end-of-stream. The event types are `messageStart`, `contentBlockStart`, `contentBlockDelta`, `contentBlockStop`, `messageStop`, and `metadata`. For tool use, the `contentBlockDelta` carries `toolUse.input` as a streaming JSON fragment string -- you accumulate and parse at `contentBlockStop`.

### When to fall back to `InvokeModel`

Two cases:

1. **Image generation models** (Stable Diffusion, Titan Image, Nova Canvas) -- these aren't chat models and don't fit the Converse shape. Use `InvokeModel` with the vendor-specific request.
2. **Embedding models** (Titan Embeddings, Cohere Embed) -- same. Use `InvokeModel` with the vendor-specific request.

For everything chat/text -- Converse, always.

---

## Part 4: Knowledge Bases -- Managed RAG

The strongest one-line frame: **Knowledge Base is to RAG what RDS is to Postgres** -- you stop running the vector DB, the embedding pipeline, the ingestion job, and the retrieval API yourself. AWS runs it; you point a Knowledge Base at an S3 bucket (or Confluence space, SharePoint site, Salesforce org, web crawl seed) and get back a `knowledgeBaseId` you can `Retrieve` from.

### The KB architecture

```
   Data source                Bedrock-managed                  Vector store
   -----------                ---------------                  ------------

   +-------------+   ingest   +----------------+  embed      +---------------+
   | S3 bucket / |----------->| Chunker        |------------>| OpenSearch    |
   | Confluence /|            | (fixed /       | (Titan or   | Serverless /  |
   | SharePoint /|            |  semantic /    |  Cohere)    | Aurora        |
   | Salesforce /|            |  hierarchical /|             | pgvector /    |
   | web crawler |            |  custom Lambda)|             | Pinecone /    |
   +-------------+            +----------------+             | Mongo Atlas / |
                                                             | Redis Ent.    |
   +-------------+   query    +----------------+  retrieve   +-------+-------+
   | Application |----------->| Retrieve /     |<--------------------+
   | (Lambda /   |            | RetrieveAnd-   |
   |  ECS / etc) |            | Generate API   |
   +-------------+            +----------------+
```

Two API surfaces:

- **`Retrieve`** -- returns the top-K chunks for a query. You compose the final prompt yourself. The right choice when you want full control over how retrieved context is presented to the model, or when you're using the chunks for something other than answering (e.g., highlighting source passages in a UI).
- **`RetrieveAndGenerate`** -- one call: retrieve + stuff + invoke FM + return. Opinionated, fast to ship, less control. Use this for a "ChatGPT over my docs" path-of-least-resistance build.

### Vector store decision matrix

| Vector store | When to pick | Cost profile | Notes |
|--------------|-------------|--------------|-------|
| **OpenSearch Serverless (default)** | The path-of-least-resistance choice | OCU minimums (~$350/mo even at zero traffic) | KB default; AWS provisions and manages OCUs |
| **Aurora pgvector** | Small/cheap KB; already running Aurora | Aurora instance hours; no separate vector tier | Cheapest at low scale; bring-your-own DB |
| **Pinecone** | You already use Pinecone elsewhere | Pinecone serverless pricing | Bedrock supports as managed connector |
| **MongoDB Atlas Vector Search** | You already use Atlas | Atlas tier | Bring-your-own Atlas cluster |
| **Redis Enterprise Cloud** | You already use Redis Enterprise | Redis Enterprise pricing | Bring-your-own |
| **Neptune Analytics** | Graph-shaped retrieval (vector + graph) | Neptune Analytics hours | Specialized for GraphRAG |
| **Amazon S3 Vectors** (2025) | Massive scale, infrequent query | S3-tier storage + per-query | Cheap cold storage for vectors; higher query latency |

**The OpenSearch Serverless OCU gotcha:** OSS has a minimum OCU floor for the index + indexing collection. At zero traffic, a single KB on OSS-S still bills ~$350/month for the OCU floor. For a 10-document internal wiki, **Aurora pgvector is dramatically cheaper** -- a `db.t4g.medium` Aurora instance with pgvector is ~$40/month. The OSS default is right for medium-to-large KBs; small KBs should use pgvector or share an OSS-S collection across multiple KBs.

### Chunking strategies -- the most impactful KB tuning knob

The chunker decides how a source document gets sliced into vectors. Bedrock supports four:

| Strategy | What it does | Best for |
|----------|--------------|----------|
| **Fixed-size** | N tokens per chunk with M-token overlap | Short, structured docs (FAQs, tickets, product specs); the safest default |
| **Hierarchical** | Two-level: parent chunks (large, retrieved as context) + child chunks (small, used for matching) | Long narrative docs where you want match precision + context completeness |
| **Semantic** | An LLM picks chunk boundaries at semantic breaks | Long narrative content (technical docs, articles, transcripts) where natural breaks matter |
| **Custom Lambda** | You write the chunker | When none of the above fits (e.g., source-code-aware splitting, table-aware splitting) |

**The contrarian rule:** *Semantic chunking is not always best.* For short, well-structured documents (a FAQ with 50 Q&A pairs, an API reference with 200 endpoint descriptions), **fixed-size chunking with overlap beats semantic** because the docs are already chunked the right way. Semantic chunking only wins when the source content is genuinely narrative.

**Hierarchical chunking is the production default for long docs** because it solves the precision-vs-context trade-off: the small child chunks make matching precise, the larger parent chunks delivered to the model preserve enough surrounding context to answer well.

### Hybrid search -- the SKU-query fix

By default, KB retrieval is pure vector cosine similarity. That works great for "what's our PTO policy?" -- a semantic question. It works terribly for "find the warranty doc for SKU-A93412" -- a keyword question where the exact token must match.

**Bedrock supports hybrid search (vector + BM25 keyword)** but **it is NOT enabled by default**. Enable it explicitly per-retrieve:

```python
agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

resp = agent_runtime.retrieve(
    knowledgeBaseId="ABCDEFGHIJ",
    retrievalQuery={"text": "warranty for SKU-A93412"},
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "numberOfResults": 5,
            "overrideSearchType": "HYBRID",   # default is "SEMANTIC"
        }
    },
)
for chunk in resp["retrievalResults"]:
    print(chunk["content"]["text"][:200], "...")
    print(" score:", chunk["score"], "source:", chunk["location"])
```

The 30-second debugging story for "the KB can't find my product codes": ssh in, run `retrieve` with `overrideSearchType="HYBRID"`, compare to default. If hybrid finds it and semantic doesn't, flip the default and ship.

### RetrieveAndGenerate -- end-to-end RAG

```python
resp = agent_runtime.retrieve_and_generate(
    input={"text": "What's our parental leave policy?"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "ABCDEFGHIJ",
            "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-6-20260301-v1:0",
            "retrievalConfiguration": {
                "vectorSearchConfiguration": {"numberOfResults": 5, "overrideSearchType": "HYBRID"}
            },
            "generationConfiguration": {
                "guardrailConfiguration": {
                    "guardrailId": "abc123def456",
                    "guardrailVersion": "DRAFT",
                },
                "inferenceConfig": {"textInferenceConfig": {"maxTokens": 500, "temperature": 0.2}},
            },
        },
    },
)

print(resp["output"]["text"])
for citation in resp["citations"]:
    for ref in citation["retrievedReferences"]:
        print("source:", ref["location"], "score:", ref.get("score"))
```

Notice the `guardrailConfiguration` slotted right in -- guardrails apply on the way out, post-generation. The `citations` field is the production differentiator over a hand-rolled RAG pipeline: you get back which chunks were used, scored, with their S3 source locations, ready to render as "sources" in a UI.

### KB sync -- the re-index gotcha

A Knowledge Base's data source has two ingestion modes:

1. **Full sync** (default if you don't specify) -- re-embeds every document in the data source. Expensive on a 100k-doc corpus.
2. **Incremental sync** -- re-embeds only changed objects, detected via S3 ETag or modification time.

The trap: triggering a sync via the console or `StartIngestionJob` API **does a full sync unless the data source was configured for incremental from the start.** Symptom: a 5-minute sync becomes a 4-hour sync after the corpus grows. Fix: rebuild the data source with `dataSourceConfiguration.s3Configuration` and the right ingestion mode at creation time.

### Data source connectors -- beyond S3

Bedrock KBs have managed connectors for:

- **S3** (the default, the simplest)
- **Confluence Cloud** (OAuth + Confluence-scoped service account)
- **SharePoint Online** (Azure AD app registration)
- **Salesforce** (connected app credentials)
- **Web Crawler** (seed URLs + crawl-depth + rate limits)
- **Custom** (you implement the connector via Lambda and call `IngestKnowledgeBaseDocuments`)

The Confluence and SharePoint connectors are the most valuable for enterprise -- they preserve ACLs (a doc you don't have read access to won't be retrievable by your principal). The S3 connector is the most common; the web crawler is the most likely to surprise you with cost (it re-crawls the seed URLs on each sync).

---

## Part 5: Agents -- The Junior Contractor with a Tool Belt

If a Knowledge Base is the in-house cookbook, an **Agent** is the contractor you hire to *do work* with that cookbook plus a set of tools. The orchestrator LLM is the manager. Each action group is a tool. Guardrails are the checkpoints at the door. KB attachments are the cookbook on the shelf. Memory is the contractor's notebook between visits.

### The orchestration loop

```
   User input
       |
       v
   +------------------+
   | Orchestrator FM  |<------------+
   | (Claude/Llama/   |             |
   |  Nova/etc.)      |             |
   +--------+---------+             |
            |                       |
   "Should I:                       |
    a) call an action group?        |
    b) retrieve from a KB?          |
    c) ask for clarification?       |
    d) answer directly?"            |
            |                       |
   +--------+---------+             |
   | Decision dispatch|             |
   +---+--------+-----+             |
       |        |                   |
       v        v                   |
   +-----+   +------+               |
   | KB  |   | Tool |--+            |
   +-----+   +------+  |            |
       |        |      |            |
   chunks    Lambda  RETURN_       |
       |     result  CONTROL       |
       |        |      |           |
       v        v      v           |
   +------------------+            |
   | Context block    |            |
   | re-injected to   |            |
   | orchestrator     |            |
   +--------+---------+            |
            +-----------------------+
            |
       (loop until orchestrator decides to answer)
            |
            v
   Final response (Guardrail filters on the way out)
            |
            v
       To caller
```

This is the same flow as Step Functions task tokens, but the *router* is an LLM instead of a deterministic state machine. The LLM looks at the user's request, the conversation so far, the tool descriptions you provided, and the KB metadata, and decides what to do next.

### Action groups -- the tool belt

An action group is a callable function the agent can decide to invoke. Two execution modes:

**Mode 1: Lambda-executed.** You provide a Lambda function ARN. The agent, when it decides to call this tool, invokes your Lambda synchronously with a structured event. Your Lambda runs (up to 25 seconds -- see gotcha below), returns a result, and the agent continues the loop.

**Mode 2: `RETURN_CONTROL`.** The agent returns the *tool call request* to the caller (your application) and pauses. Your application executes the tool however it likes -- in the browser, in a mobile app, on an on-prem network the agent can't reach -- and submits the result back to the agent via `InvokeAgent` with a `sessionState.invocationResults` field. Then the agent resumes.

When to pick each:

| Pattern | Use |
|---------|-----|
| Tool is a managed AWS resource (DynamoDB lookup, RDS query, S3 GetObject, internal microservice in same VPC) | **Lambda action group** |
| Tool needs caller credentials (a per-user token, a browser-side OAuth session) | **`RETURN_CONTROL`** |
| Tool runs on the user's device (camera, file picker, biometric prompt) | **`RETURN_CONTROL`** |
| Tool reaches a network only the caller can reach (on-prem, VPN-only) | **`RETURN_CONTROL`** |
| Tool result must be confirmed by the user before the agent acts on it | **`RETURN_CONTROL`** (with a UI prompt between request and result submission) |

### Defining an action group -- the schema

Action groups are defined with one of two schema formats:

1. **OpenAPI 3 schema** -- you upload a full OpenAPI spec describing the action group's "endpoints." Each path becomes a tool the agent can call. Best for action groups that mirror a real REST API.
2. **Function schema** (the newer, simpler shape) -- a JSON schema per function. Best for purpose-built agent tools.

Function schema example:

```json
{
  "functions": [
    {
      "name": "get_order_status",
      "description": "Look up an order's current status, ship date, and tracking number.",
      "parameters": {
        "order_id": {
          "type": "string",
          "description": "The order ID, e.g. ORD-12345",
          "required": true
        }
      }
    },
    {
      "name": "cancel_order",
      "description": "Cancel an order. Requires the order to be in PENDING or PROCESSING state.",
      "parameters": {
        "order_id": {"type": "string", "description": "The order ID", "required": true},
        "reason": {"type": "string", "description": "Cancellation reason", "required": false}
      }
    }
  ]
}
```

The `description` fields are *the prompt the orchestrator sees* when deciding which tool to call. Write them like API docs: precise, unambiguous, with examples. A weak description is the #1 cause of "the agent never calls my tool."

### The Lambda action-group payload

When the agent invokes your Lambda, it sends:

```python
def lambda_handler(event, context):
    """
    event shape:
    {
        "messageVersion": "1.0",
        "agent": {"name": "...", "id": "...", "alias": "...", "version": "..."},
        "inputText": "the user's original message",
        "sessionId": "...",
        "actionGroup": "OrderActions",
        "function": "get_order_status",
        "parameters": [
            {"name": "order_id", "type": "string", "value": "ORD-12345"}
        ],
        "sessionAttributes": {...},
        "promptSessionAttributes": {...}
    }
    """
    order_id = next(p["value"] for p in event["parameters"] if p["name"] == "order_id")
    status = lookup_order(order_id)
    return {
        "messageVersion": "1.0",
        "response": {
            "actionGroup": event["actionGroup"],
            "function": event["function"],
            "functionResponse": {
                "responseBody": {
                    "TEXT": {"body": f"Order {order_id} is {status}"}
                }
            },
        },
    }
```

The `responseBody.TEXT.body` is what gets fed back into the orchestrator's context. You can return JSON-serializable strings; the model will reason over them in the next loop iteration.

### The action-group response-size and InvokeAgent timeout gotchas

The myth: "action-group Lambdas time out at 25 seconds." Not exactly. What AWS actually publishes:

1. **Action-group Lambda responses are capped at 25 KB.** If your tool returns more than 25 KB of body, the agent fails the turn. Long answers (large query results, full document contents) must be summarized, paginated, or written to S3 with a reference returned.
2. **The `InvokeAgent` API has its own service timeout** that bounds the entire turn (orchestrator + tool round-trips + post-processing). Even if your Lambda is configured for 15 minutes, the orchestrator will give up well before that -- in practice plan for the whole tool call to complete in tens of seconds, not minutes.
3. **Lambda's own configured timeout still applies** -- a Lambda configured for 60 s will be killed by Lambda at 60 s regardless of what Bedrock does.

Three mitigations for slow tools:

1. **Async-offload.** Have the action-group Lambda kick off the work (SQS message, Step Functions execution) and return a small handle. A subsequent turn (or a `RETURN_CONTROL` flow) polls.
2. **Pre-compute.** If the agent needs a slow lookup, cache it ahead of time in DynamoDB or ElastiCache.
3. **Switch to `RETURN_CONTROL`.** The caller can wait on the slow API with whatever timeout it wants, then submit the result back via `sessionState.invocationResults`.
4. **Trim the response.** If you're brushing the 25 KB cap, the model probably can't usefully consume that much anyway -- summarize before returning.

### Agent memory -- multi-session continuity

Bedrock Agents support **session memory** as of 2024. You opt in per agent; memory is keyed by `memoryId` (which you provide -- typically a user ID or a `tenant:user` composite). The agent uses the memory to recall summaries of past sessions when answering a new prompt in the same `memoryId`.

```python
agent_runtime.invoke_agent(
    agentId="ABC123",
    agentAliasId="TSTALIASID",
    sessionId=f"sess-{uuid.uuid4()}",         # new session per conversation
    memoryId="user-1234",                       # SAME memoryId across sessions = continuity
    inputText="What did we conclude last time?",
)
```

Two gotchas:

- **Memory retention is configurable** (default 30 days), and longer retention costs more. Set this per privacy/cost requirements.
- **Memory is summary-only**, not full transcript. The agent doesn't see verbatim prior messages -- it sees a model-generated summary. For workflows where verbatim recall matters (legal, compliance), you store transcripts in DynamoDB and feed them in explicitly.

### Multi-agent collaboration -- supervisor + collaborators

The 2024 addition that turned Bedrock Agents from a single-LLM-with-tools into a real agentic platform: **a supervisor agent can route to collaborator agents.** You define a supervisor with `agentCollaboration: "SUPERVISOR"` and attach one or more collaborator agents. The supervisor's orchestrator decides at each step whether to call its own tools/KBs or delegate to a collaborator.

The pattern: **specialist agents per domain** -- a billing agent (with the billing-system action groups), a shipping agent (with the WMS action groups), a knowledge agent (with the docs KB) -- and a supervisor that routes. The supervisor doesn't need every action group; it just needs to know when to hand off.

The cost story: every agent invocation is its own LLM call. A supervisor-plus-three-collaborators pattern can easily 4x your token spend per user message. Use Haiku for the supervisor (it's just a router); reserve Sonnet/Opus for the collaborators that do the actual work.

---

## Part 6: Guardrails -- The TSA Checkpoints

Guardrails apply on the way in (the prompt) and on the way out (the response), **independently**. You can have a guardrail block PII in the user's input but mask PII in the model's output. The six policy families:

| Policy | What it does | Configuration |
|--------|-------------|---------------|
| **Content filter** | Blocks harmful content across 4 categories | Hate, Insults, Sexual, Violence -- each None/Low/Medium/High strength, separately for input and output |
| **Denied topics** | Blocks discussion of off-limits subjects | Up to 30 topics, defined as natural-language descriptions ("legal advice," "competitor product comparisons") |
| **Word filters** | Blocks specific words and phrases | Profanity preset (toggleable) + custom blocklist of up to 10k words |
| **Sensitive info (PII) filter** | Detects and Blocks or Masks PII | 30+ built-in types (email, SSN, phone, credit card, IP address, name, address, etc.) + custom regex; Mask mode replaces with `{EMAIL}` etc. |
| **Contextual grounding** | Detects hallucinations against retrieved sources | Two scores: **grounding** (is the response supported by source?) + **relevance** (does the response answer the question?); thresholds 0-1 |
| **Prompt-attack filter** | Detects jailbreak / prompt-injection attempts | Strength None/Low/Medium/High; applies to input only |

### Wiring a guardrail into Converse

```python
response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
    messages=[{"role": "user", "content": [{"text": user_input}]}],
    guardrailConfig={
        "guardrailIdentifier": "abc123def456",
        "guardrailVersion": "1",                 # or "DRAFT"
        "trace": "enabled",                       # surfaces what triggered, useful in dev
    },
    inferenceConfig={"maxTokens": 500},
)

# When a guardrail intervenes, stopReason tells you
if response["stopReason"] == "guardrail_intervened":
    # The text in output[].content[].text is the guardrail's blocked-message text
    # (which you configured at guardrail creation -- e.g., "I can't help with that.")
    pass
```

### Contextual grounding -- the hallucination killer

This is the one production teams keep underweighting. **Contextual grounding** takes the retrieved RAG context, the model's response, and computes two scores per response:

- **Grounding score (0-1)**: How well-supported is the response by the source documents?
- **Relevance score (0-1)**: Does the response actually answer the question that was asked?

You set thresholds (typical: 0.75 grounding, 0.5 relevance). If the response falls below either threshold, the guardrail intervenes. **For any RAG workload, this is the single most valuable guardrail.** It's the difference between "the model confidently said the wrong thing" and "the model refused because it couldn't ground its answer."

Wire it into `RetrieveAndGenerate` via `guardrailConfiguration`, and the contextual-grounding check sees both the retrieved chunks and the generated answer automatically.

### `ApplyGuardrail` -- the standalone API

The most underused Bedrock feature in 2026. `ApplyGuardrail` lets you apply a Bedrock guardrail to **any text, including text generated by non-Bedrock models** (OpenAI, self-hosted, on-prem):

```python
resp = bedrock.apply_guardrail(
    guardrailIdentifier="abc123def456",
    guardrailVersion="1",
    source="OUTPUT",                       # or "INPUT"
    content=[{"text": {"text": "...content to check..."}}],
)

# resp["action"] is "GUARDRAIL_INTERVENED" | "NONE"
# resp["outputs"] contains the (masked or unchanged) text
# resp["assessments"] contains the detailed score breakdown per policy
```

This is the pattern for a multi-model org: one guardrail definition (a Terraform-managed resource), applied consistently across Bedrock models, OpenAI calls, and self-hosted Llama on EKS. **It does cost separately per call**, even when you're already paying for a Bedrock invoke -- but the consistency is worth it.

### The prompt-attack tuning gotcha

The prompt-attack filter is the most likely guardrail to cause false positives in *legitimate* engineering work. Examples that have been falsely flagged in production:

- A system prompt that says *"Ignore prior instructions and answer the user's question in JSON only."* (Legitimate prompt engineering; flagged as a jailbreak attempt.)
- A user asking, *"Write me a roleplay where I play a sysadmin debugging a leak."* (Legitimate request; flagged as social engineering.)

Tune the strength carefully. Start at Low or Medium for user-facing chat; only go to High for unauthenticated/anonymous traffic surfaces.

---

## Part 7: Prompt Caching -- The Kitchen Prep Station

**The single most impactful cost lever in Bedrock in 2026.** Prompt caching lets you mark portions of your prompt as cacheable; Bedrock stores the cached prefix and reuses it across subsequent calls. Cached tokens cost ~10% of normal input tokens. For workloads with long, repeated context (large system prompts, long RAG headers, multi-turn conversations with long history), this is an **80-90% input-token cost reduction** and a **substantial latency reduction** (no re-tokenization, no re-attention over the cached prefix).

### The two TTLs -- 5 minutes default, 1 hour opt-in

As of January 2026 Bedrock supports two cache durations for Claude 4.x models (Haiku 4.5, Sonnet 4.6, Opus 4.7):

- **5-minute TTL (default)** -- "prep that lasts a service." Cheaper write penalty. Best for chat-shaped workloads where the same user is actively talking to the model.
- **1-hour TTL (opt-in)** -- "prep that lasts the day." Higher write penalty, but lets you amortize cache writes across sporadic traffic. Best for workloads where requests cluster at unpredictable cadences (10-min gaps between user turns, hourly batch enrichments hitting the same system prompt, a help-bot with one query every few minutes).

The breakeven is roughly: **if your average inter-call gap is between 5 minutes and 1 hour, the 1-hour TTL wins on cost**; below 5 minutes the default is cheaper; above 1 hour neither helps. Pick TTL per workload, not globally. The 1-hour TTL is also region-scoped, so the cross-region inference profile cache-miss trade-off below still applies.

**The refresh-on-hit mechanic that changes the math.** The TTL is a *sliding* window, not a fixed one: every cache hit re-anchors the expiry to TTL-from-now. A cache prefix stays warm as long as *someone* keeps reading it within the window -- it does not die a fixed N minutes after the write. This is why the breakeven is about the *inter-call gap*, not the session length: a 6-hour support conversation with a question every 20 minutes stays 100% warm on the 1-hour TTL because each turn refreshes the clock, even though the conversation far outlives any single TTL period. The corollary failure: with the 5-minute default and a 15-40 minute inter-turn gap, *every* follow-up turn is a guaranteed miss, and a miss is **worse than no caching** -- you pay the cache-write premium (~125% of input rate on the prefix) and never collect a single discounted read. Always confirm the win in production by graphing `cacheReadInputTokens` against `cacheWriteInputTokens`: healthy caching shows reads dwarfing writes; if writes are spiking and reads are near zero, your TTL choice is upside-down for your traffic pattern.

### The cachePoint mechanism

You insert `cachePoint` blocks into the messages. Everything *before* a cachePoint is cached as a single contiguous prefix:

```python
LONG_SYSTEM_PROMPT = """
You are a customer-support assistant for ACME Corp.
You have access to the following product catalog and policy documents:
... [10,000 tokens of policy text] ...
You must follow these rules:
1. Never offer refunds without manager approval.
2. ...
"""

response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
    system=[
        {"text": LONG_SYSTEM_PROMPT},
        {"cachePoint": {"type": "default"}},     # cache everything above this point
    ],
    messages=[
        {"role": "user", "content": [{"text": "Can I get a refund on order ORD-12345?"}]}
    ],
    inferenceConfig={"maxTokens": 500},
)

# response.usage now has additional fields:
# usage["inputTokens"]          -- billed at full rate
# usage["cacheReadInputTokens"] -- cached tokens read, billed at ~10% of input
# usage["cacheWriteInputTokens"]-- first call writes to cache, billed at ~125% of input
```

The economics: **first call writes the cache (small penalty); subsequent calls within 5 minutes read the cache (~90% discount on cached tokens).** If your system prompt is 10,000 tokens and you're calling 100 times in a 5-minute window:

- Without caching: 100 × 10,000 = 1,000,000 input tokens billed at full rate.
- With caching: 1 × 12,500 (cache write) + 99 × 1,000 (cache read at 10%) = ~111,500 tokens billed.
- **Saving: ~89%.**

### Where to place the cachePoint

The rule: **cache the prefix that's stable across requests; vary what comes after.**

Common placements:

1. **End of system prompt.** Cache the system prompt; vary the user message. Best for tool-use schemas, persona definitions, policy docs.
2. **End of RAG context block.** Cache the retrieved chunks for a session (if the chunks stay the same across follow-up questions). Best when a user is having a multi-turn chat against the same retrieved context.
3. **End of conversation history.** In a long chat, cache the prior N turns; only the latest user message is variable. Best for long agentic loops.

You can have **multiple cachePoints**, each defining a checkpoint. The model returns the longest cached prefix on each call. Sonnet/Opus typically supports up to 4 cachePoints per request.

### TTL eviction and region-scoping gotchas

Two production traps:

1. **TTL eviction.** The default 5-minute TTL evicts the cache after 5 minutes of inactivity; sporadic-traffic workloads (one user per 10 minutes) get **no benefit** on the default. The opt-in 1-hour TTL fixes this at the cost of a higher write penalty -- pick per workload. Caching shines on bursty traffic: hot chat threads, batch enrichment runs, stress tests.
2. **Cache is region-scoped.** A cache written in `us-east-1` is not readable in `us-west-2`. This matters because **cross-region inference profiles can route a call to a different region than the previous call**, invalidating the cache. If you absolutely need cache hits on a workload, use a region-specific model ID for the first few minutes of cache warmth and *then* fall back to the inference profile for resilience. Or accept that cross-region inference reduces cache hit rate -- a documented AWS trade-off.

### Cost / latency math example

For a customer-support chatbot with a 12,000-token system prompt + RAG header and ~500 tokens of variable user/conversation content per turn, average 8 turns per conversation, 1,000 conversations/day:

| Metric | Without caching | With caching | Savings |
|--------|----------------|--------------|---------|
| Input tokens/day | 8M × 12,500 = 100M | 1M (1st turn writes) + 87.5M × 0.10 ≈ 9.75M | ~90% |
| Input $ at $3/M | $300/day | $30/day | $270/day |
| Latency p50 | 1.8s | 1.1s | ~40% |
| Annual savings | -- | -- | **~$98k** |

For workloads with smaller stable prefixes (a 2,000-token system prompt only), the savings shrink proportionally but typically stay above 50%.

---

## Part 8: Provisioned Throughput vs On-Demand -- The Reserved Table

The default for Bedrock is **On-Demand** -- you call the model, you get charged per token, and your throughput is bounded by per-region per-model TPM (tokens-per-minute) quotas. When TPM is exceeded, calls return `ThrottlingException` and you have to retry or fail.

**Provisioned Throughput (PT)** is the reserved-table model: you commit to N model units (MUs) for 1 or 6 months and get dedicated capacity. Each MU corresponds to a guaranteed throughput tier per model (specifics vary -- Claude Sonnet 4.6 MU ≈ 100k input + 30k output tokens/min as of writing).

### When to pick PT

PT is the right choice in three cases:

1. **You must serve a custom fine-tuned model.** Fine-tuned models on Bedrock are *only* available via PT -- there's no on-demand path for them.
2. **You need guaranteed capacity for a known-high-volume workload** (e.g., a batch enrichment job that needs to finish in a fixed window, or a customer-facing chat that gets crushed during a marketing campaign).
3. **You're over the on-demand pricing tier consistently** -- in some cases PT is cheaper per token at very high volume, though this is workload-dependent and rare in practice.

PT is the **wrong** choice in the common cases:

- Most production workloads. On-demand is more flexible and almost always cheaper at typical scale.
- Bursty or unpredictable traffic. PT pays for the reservation whether or not you use it.
- Cross-region resilience. PT is per-region; cross-region inference profiles work better for resilience.

### Commitment terms and pricing

- **1-month term**: ~50% premium over the 6-month rate, but you can release after 30 days.
- **6-month term**: cheapest per-MU rate; you're locked in.
- **No-commit / hourly**: available for some models for development purposes; expensive per hour.

The math test for PT: **at what utilization does PT break even vs on-demand?** Compute your expected tokens/minute, multiply by the on-demand per-token cost, divide by the PT MU hourly rate. If your expected utilization is above ~60% of an MU's capacity, PT is competitive. Below that, on-demand wins.

---

## Part 9: Cross-Region Inference Profiles -- The Hotel-Chain Reservation

Cross-region inference profiles, GA in 2024 and now the production default in 2026, solve two problems at once:

1. **TPM quota headroom.** Each region has its own TPM quota per model. An inference profile fronting multiple regions effectively sums the quotas.
2. **Resilience.** If `us-east-1` is throttling or has elevated errors, the profile routes to `us-west-2`. You don't see it.

### Model ID vs inference profile ARN

The key API distinction:

```python
# Using a region-specific model ID -- the call stays in this region
bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6-20260301-v1:0",
    ...
)

# Using the short inference profile ID -- routed across regions
bedrock.converse(
    modelId="us.anthropic.claude-sonnet-4-6-v1:0",
    ...
)

# Or, equivalently, the full inference profile ARN
bedrock.converse(
    modelId="arn:aws:bedrock:us-east-1:111122223333:inference-profile/us.anthropic.claude-sonnet-4-6-v1:0",
    ...
)
```

Either the short inference-profile ID (`us.anthropic.claude-sonnet-4-6-v1:0`) or the full ARN works as the `modelId` argument -- they refer to the same routing target. IAM policies have to use the full ARN; application code can use whichever form is more convenient.

Inference profile naming convention: `<geographic-region-prefix>.<model-id>`, where prefixes are `us.`, `eu.`, `apac.` (verify the exact prefix list for your target region with `aws bedrock list-inference-profiles` -- some sub-regions like Tokyo/Sydney use their own prefix variants). The geographic constraint matters for data residency -- a `us.` profile only routes to US regions; `eu.` only to EU regions.

**The production default in 2026 should be the inference profile ARN, not the raw model ID.** Two reasons:

- Routing transparency: AWS picks the best regional copy at call time. You don't have to write retry logic.
- Higher effective TPM: you get the sum of regional quotas.

The trade-off: **cache hit rate drops** with cross-region routing (the cache is region-scoped; if the call routes to a different region than the prior call, it's a cache miss). For workloads where caching is dominant, use a region-specific model ID for the high-cache-affinity path and reserve the inference profile for resilience fallback. Most workloads should just accept the slightly reduced cache hit rate -- the resilience win outweighs it.

### The "wait, my CloudTrail event says a different region" gotcha

When an inference profile routes from `us-east-1` to `us-west-2`, the **CloudTrail data event for the underlying invocation is logged in `us-west-2`** -- not the region you called. If you have region-scoped Athena partition predicates in your CloudTrail forensic queries (from yesterday's doc), you may be looking in the wrong region for the actual invocation. The fix: always include all `us.` profile regions in your forensic queries when investigating Bedrock activity through an inference profile.

---

## Part 10: IAM Patterns -- Identity, Resource, and Cross-Account

The Bedrock IAM model has two layers that interact: **identity-based policies** (who can call what) and **resource-based policies on the model itself** (who can use this specific model). For most accounts, only the identity-based layer matters; cross-account model sharing is where resource policies enter.

### Identity-based policy -- the workload role

The execution role for a Lambda or ECS service that calls Bedrock should scope `bedrock-runtime:InvokeModel` (or `Converse`) to specific model ARNs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowInvokeOnApprovedModels",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-6-20260301-v1:0",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-haiku-4-5-20260301-v1:0",
        "arn:aws:bedrock:us-east-1:111122223333:inference-profile/us.anthropic.claude-sonnet-4-6-v1:0"
      ]
    },
    {
      "Sid": "AllowGuardrailApplyAndConverseGuardrails",
      "Effect": "Allow",
      "Action": ["bedrock:ApplyGuardrail"],
      "Resource": "arn:aws:bedrock:us-east-1:111122223333:guardrail/abc123def456"
    },
    {
      "Sid": "AllowAgentInvoke",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeAgent", "bedrock:Retrieve", "bedrock:RetrieveAndGenerate"],
      "Resource": [
        "arn:aws:bedrock:us-east-1:111122223333:agent-alias/ABC123/TSTALIASID",
        "arn:aws:bedrock:us-east-1:111122223333:knowledge-base/ABCDEFGHIJ"
      ]
    }
  ]
}
```

Three things to internalize:

1. **`bedrock:InvokeModel` is resource-scopable per model ARN.** Don't grant `Resource: "*"`. Pick the models the workload actually needs. This is your cost-control IAM lever -- prevents a developer from accidentally switching to Opus.
2. **Foundation model ARNs use the empty-account-ID format** (`arn:aws:bedrock:REGION::foundation-model/MODEL-ID`). Inference profile ARNs include the customer account ID.
3. **Agent invocation grants are on the agent *alias*, not the agent itself.** Aliases are how you version-pin agents in production; the alias ARN is what you authorize.

### KB-side service role

The Knowledge Base itself needs a service role to:

- Read source data from S3 (`s3:GetObject`, `s3:ListBucket`)
- Invoke the embedding model (`bedrock:InvokeModel` on the Titan/Cohere embedding model ARN)
- Write to the vector store (varies by store -- OSS-S, Aurora, etc.)
- Encrypt/decrypt with the KMS key on the data bucket

This is a separate role with a trust policy for `bedrock.amazonaws.com`. The Terraform in Part 14 sets it up.

### Cross-account model invocation

Two patterns:

**Pattern 1: Shared model access via inference profile.** The inference profile is in account A; account B's role gets `bedrock:InvokeModel` on the profile ARN. The model usage bills to account A, which then chargeback-bills account B internally. Useful for a "platform team owns the Bedrock account, application teams have their own accounts" setup.

**Pattern 2: KB or Agent sharing.** Bedrock supports resource-based policies on knowledge bases and agents. Account A's KB grants `bedrock:Retrieve` to a role in account B; account B can now query the KB directly. Cleaner than copying the KB.

### Bedrock Studio access

Bedrock Studio (the visual workspace) **requires IAM Identity Center (formerly SSO)**. You cannot give a standalone IAM user access to Studio -- the access model is users-in-Identity-Center mapped to Studio workspaces. This is intentional: Studio is positioned for non-engineering personas (product managers, content writers) who don't have IAM credentials of their own.

---

## Part 11: Observability -- The Kitchen's CCTV

Three observability surfaces, mapping cleanly to the CloudWatch and CloudTrail docs from earlier this week.

### CloudWatch metrics -- the `AWS/Bedrock` namespace

The metric namespace is `AWS/Bedrock` (and `AWS/Bedrock/Agents` for agent-specific metrics). Key metrics:

| Metric | Dimensions | Use |
|--------|-----------|-----|
| `Invocations` | ModelId, [optional] InferenceProfileId | Call volume per model |
| `InputTokenCount` | ModelId | Sum over time for input-token cost dashboards |
| `OutputTokenCount` | ModelId | Sum over time for output-token cost dashboards (the dominant cost) |
| `InvocationLatency` | ModelId | Latency p50/p95/p99 per model |
| `InvocationClientErrors` | ModelId | 4xx from caller (throttling, validation, access denied) |
| `InvocationServerErrors` | ModelId | 5xx from Bedrock (model issues) |
| `InvocationThrottles` | ModelId | TPM quota exceeded; signal to bump quota or switch to inference profile |

**The Day 1 gotcha:** `InputTokenCount` and `OutputTokenCount` are **gauges per invocation**, not running counters. To get total tokens over a time window, use `SUM()` statistic, not `AVERAGE()`. Most teams get this wrong in their first dashboard and conclude "I only used 5 tokens today" because they're averaging gauges.

Logs Insights query against Model Invocation Logs (next section) for token-usage by user/workflow:

```sql
fields @timestamp, modelId, identity.arn AS caller, input.inputTokenCount, output.outputTokenCount
| filter modelId like /claude-sonnet/
| stats sum(input.inputTokenCount) AS in_tokens,
        sum(output.outputTokenCount) AS out_tokens
        by caller
| sort out_tokens desc
| limit 20
```

### Model Invocation Logging -- the full transcript

Account-level toggle (one setting per account, per region) that captures **full request and response payloads** for every Bedrock invocation. Destinations:

- **S3** (gzipped JSON batches; recommended for archive and Athena queries)
- **CloudWatch Logs** (real-time; expensive at high volume)
- **Both** (the production default for hot ops + cold compliance)

Enable via console or:

```bash
aws bedrock put-model-invocation-logging-configuration \
  --logging-config '{
    "cloudWatchConfig": {
      "logGroupName": "/aws/bedrock/invocations",
      "roleArn": "arn:aws:iam::111122223333:role/BedrockLoggingRole",
      "largeDataDeliveryS3Config": {
        "bucketName": "bedrock-invocation-large-data",
        "keyPrefix": "large/"
      }
    },
    "s3Config": {
      "bucketName": "bedrock-invocation-logs",
      "keyPrefix": "invocations/"
    },
    "textDataDeliveryEnabled": true,
    "imageDataDeliveryEnabled": false,
    "embeddingDataDeliveryEnabled": false
  }'
```

The captured payload includes the full prompt (system + user messages, including any retrieved RAG context) and the full response. **This will contain PII/PHI** unless you've filtered upstream with a guardrail's sensitive-info policy. Four non-negotiables:

1. **KMS-encrypt the S3 destination** with a customer-managed key. Restrict decrypt to the audit team's role only. **Silent-failure gotcha:** Bedrock writes these logs using its own service principal -- the CMK key policy *must* grant `bedrock.amazonaws.com` permission to `kms:GenerateDataKey` (and `kms:Decrypt`). If it doesn't, log delivery fails *silently* -- you think you have a 7-year audit trail and you have an empty bucket. Set a CloudWatch alarm on log-delivery failure so you find out in minutes, not at audit time.
2. **Set an S3 lifecycle policy with explicit retention.** SOC 2 / HIPAA / GDPR all have data-minimization requirements; "infinite retention because it's cheap" is non-compliant.
3. **For compliance retention, use S3 Object Lock in Compliance Mode**, not Governance Mode. Governance Mode lets a sufficiently-privileged admin (or a compromised admin role) delete the audit trail; Compliance Mode locks the objects against deletion *even by the account root* for the retention period. A 7-year HIPAA/SOX audit requirement is a Compliance-Mode requirement -- Governance Mode fails the "tamper-proof" test.
4. **Inject `requestMetadata` for domain-scoped audit queries.** This is the pattern that makes "show me every model invocation that touched patient X's data" *answerable*. Every `Converse`/`InvokeModel` call accepts a `requestMetadata` map (string key/value pairs) that Bedrock writes verbatim into the invocation log record. Wrap your Bedrock client to inject structured keys -- `tenant_id`, `patient_id`, `case_id`, `user_id`, `session_id` -- on every call. Without it, the only thing you can filter on is the prompt *text*, which means grepping free-form strings across years of logs: fragile, slow, and unreliable. With it, the audit query is a partition-pruned Athena `WHERE patient_id = 'X' AND dt BETWEEN ...` joined to CloudTrail on `requestId`. `requestMetadata` is small (string values, a low double-digit key cap) -- treat it like structured log fields, not a payload channel.

```python
bedrock.converse(
    modelId="...",
    messages=[...],
    requestMetadata={                 # written verbatim into the MIL record
        "tenant_id":  "acme",
        "patient_id": "P-90210",      # the domain key your CISO will query on
        "case_id":    "CASE-4471",
        "user_id":    "clinician-553",
        "session_id": session_id,
    },
)
```

### CloudTrail data events for Bedrock

Yesterday's audit pattern applies directly here. Enable Bedrock data events on your trail:

```hcl
advanced_event_selector {
  name = "Log all Bedrock model invocations"
  field_selector {
    field  = "eventCategory"
    equals = ["Data"]
  }
  field_selector {
    field  = "resources.type"
    equals = ["AWS::Bedrock::Model", "AWS::Bedrock::AgentAlias", "AWS::Bedrock::KnowledgeBase"]
  }
}
```

Pair CloudTrail data events (who called what, with what session context) with Model Invocation Logging (what the prompt and response actually were) and you have the complete audit answer for "GenAI use across the org."

Athena query joining the two (one from CloudTrail, one from MIL):

```sql
SELECT
  ct.eventtime,
  ct.useridentity.arn AS caller,
  ct.requestparameters AS request_metadata,
  mil.input.textinput AS prompt_text,
  mil.output.textoutput AS response_text,
  mil.input.inputtokencount AS in_tok,
  mil.output.outputtokencount AS out_tok
FROM cloudtrail_logs ct
JOIN bedrock_invocation_logs mil
  ON ct.requestid = mil.requestid
WHERE ct.eventname = 'InvokeModel'
  AND ct.partition_year = '2026' AND ct.partition_month = '05'
  AND ct.useridentity.arn LIKE '%/MyAppRole/%';
```

This is the SOC-2 audit answer for "show me every prompt this application has ever sent to a Bedrock model."

---

## Part 12: Cost Optimization -- The Token Math

Bedrock pricing is per-1000-input-tokens and per-1000-output-tokens, **and output is ~5x input across most chat models** (Claude Sonnet 4.6: $3/M input, $15/M output). The cost equation:

```
cost = (input_tokens × input_rate) + (output_tokens × output_rate)
     + (cache_read_tokens × input_rate × 0.10)
     + (cache_write_tokens × input_rate × 1.25)
     + guardrail_apply_cost (separate, per call, if standalone)
     + KB storage + vector-store cost (OSS-S OCU minimums dominate)
     + PT MU-hours (if reserved)
```

The four levers that dominate, in order of impact:

### Lever 1: Model selection (10-15x range)

Pick the smallest model that passes your eval bar. Approximate 2026 ratios (Claude family, normalized to Haiku = 1):

| Model | Input rate (relative) | Output rate (relative) |
|-------|----------------------|------------------------|
| Haiku 4.5 | 1× | 1× |
| Sonnet 4.6 | ~12× | ~12× |
| Opus 4.7 | ~60× | ~60× |

**The router pattern:** Use Haiku to classify "is this request hard?" and route hard requests to Sonnet, only-the-hardest to Opus. A Haiku classifier costs ~$0.01 per 1000 prompts; the saved Sonnet invocations more than pay for it.

### Lever 2: Force short outputs (output ≈ 5x input)

The most common production mistake: writing prompts that allow the model to ramble. Three controls:

1. **`max_tokens` (or `maxTokens`) in inferenceConfig.** This is a budget directive. Set it to the actual maximum your UI needs.
2. **System prompt instructions.** "Be brief." "Answer in 2 sentences." "Output JSON only." These reduce average output length even when `max_tokens` is generous.
3. **`stopSequences`.** Use them to cut the model off at natural boundaries.

Empirically, going from "no max_tokens, no brevity prompt" to "max_tokens=200, 'be brief' system prompt" cuts output costs by 60-80% with no quality loss on Q&A workloads.

### Lever 3: Prompt caching (50-90% on cached portion)

Covered in Part 7. The biggest win is on workloads with long stable prefixes (system prompts > 2000 tokens, persistent RAG headers). Always cache the system prompt; cache the RAG context for multi-turn sessions.

### Lever 4: Batch inference (~50% off for offline)

For workloads where latency doesn't matter (overnight document enrichment, backfilling embeddings, scoring a million records), Bedrock Batch Inference accepts a job with input prompts in S3 and outputs results in S3 at **~50% the on-demand rate**. The catch: latency is variable, up to 24 hours.

```python
bedrock_ctrl = boto3.client("bedrock", region_name="us-east-1")

resp = bedrock_ctrl.create_model_invocation_job(
    jobName="overnight-doc-enrichment",
    roleArn="arn:aws:iam::111122223333:role/BedrockBatchRole",
    modelId="anthropic.claude-haiku-4-5-20260301-v1:0",
    inputDataConfig={
        "s3InputDataConfig": {"s3Uri": "s3://my-batch-input/prompts.jsonl"}
    },
    outputDataConfig={
        "s3OutputDataConfig": {"s3Uri": "s3://my-batch-output/"}
    },
)
```

The input file is **JSONL with one record per line, each wrapped as `{"recordId": "<your-id>", "modelInput": {...vendor-native body...}}`**. The `modelInput` shape is the same body you'd send to `InvokeModel` for that model (Anthropic messages payload for Claude, Llama prompt body for Llama, Titan textGenerationConfig for Titan -- batch does **not** auto-convert from Converse format). Output is JSONL where each line carries your `recordId`, the `modelOutput`, and any error. Use batch for any offline workload; you're leaving 50% on the table if you run it on-demand.

### Cost-first triage when the bill spikes

The diagnostic order is **what -> who -> why -> fix**, and you do it in that order because Cost Explorer answers *what* in 60 seconds while logs take an hour:

1. **Cost Explorer first, grouped by `UsageType` and `API Operation`, filtered on service `Bedrock`, daily granularity, this-month vs last-30.** This isolates *which dimension is on fire* before you touch logs. The four `UsageType` signatures, each pointing at a different root cause:
   - **Input tokens up, output ~flat** -> volume/loop regression or a model swap. "We got busy" or "we routed to a pricier model."
   - **Output tokens up, input ~flat** -> a *distinct* signature with a *different* fix. Someone removed `max_tokens`, weakened the "be brief" instruction, or raised temperature. Output is ~5x input rate, so this is the highest unit-cost regression -- the fix is a prompt edit, not a loop cap. Treat it as its own first-class culprit, not a footnote of the input case.
   - **`Bedrock-CacheWrite` exploding while `Bedrock-CacheRead` collapses** -> a prompt-caching regression (dynamic content moved before the cache checkpoint, a tool definition changed, system blocks reordered). Token *volume* may look normal so volume alarms never fire -- you're paying the write premium on every miss with zero reads to amortize.
   - **`Bedrock-MU-Hours` up with *low* invocation count** -> idle Provisioned Throughput. A PT commitment for a workload that no longer drives traffic. Unique signature: high spend, low calls -- neither a loop nor a cache regression explains it.
2. **CloudWatch metric `InputTokenCount`/`OutputTokenCount` SUM grouped by `ModelId`.** Which model jumped (confirms the Cost Explorer read).
3. **Logs Insights against Model Invocation Logs, grouped by `identity.arn`.** The top row is the offending IAM principal = the team's app role. Now you know *who*.
4. **Common causes**, in rank order:
   1. **Someone switched a workload from Haiku to Sonnet to "improve quality"** without changing prompt structure -- ~12x cost jump.
   2. **A prompt change removed the `max_tokens` cap or weakened "be brief"** -- 2-3x output growth (the output-tokens-up signature).
   3. **Cache regression from prefix churn** -- dynamic content before the checkpoint; every request misses and still pays the write premium.
   4. **A new agentic workflow** -- multi-agent collaboration patterns 3-5x token spend.
   5. **Idle Provisioned Throughput** -- the MU-hours-up, low-invocation signature.

The fix conversation is **data, not accusation**: "your role's ARN went from $X to $Y/day starting on [date]; here's the query and the inflection point -- walk me through what shipped." Agree on three outcomes before leaving: immediate rollback/hotfix, a short-term CloudWatch billing alarm scoped to that IAM principal, and a long-term AI-changes code-review checklist (model choice, cache-prefix stability, retry caps, context size). Most cost incidents are foot-guns a five-line checklist would have caught.

---

## Part 13: Bedrock vs SageMaker JumpStart vs Self-Hosted vLLM on EKS

The three options, with the decision matrix:

| Dimension | **Bedrock** | **SageMaker JumpStart** | **Self-hosted vLLM on EKS** |
|-----------|------------|-------------------------|------------------------------|
| Ops cost | Zero -- managed end to end | Moderate -- endpoints, scaling, scheduling | High -- GPU ops, CUDA, driver mgmt, capacity planning |
| Cold-start latency | None (always warm) | Minutes (endpoint provisioning) | Seconds-to-minutes (depends on pod warmup) |
| Per-token cost (steady state) | Higher per token | Endpoint-hours + optional inference cost | Lowest at high utilization; expensive idle |
| Model selection | ~50 hosted FMs | Thousands (JumpStart catalog) | All of HuggingFace + custom |
| Custom fine-tunes | Supported (FT job + PT only) | Full lifecycle (train, tune, host) | Full control |
| Latency floor | ~200-500ms (depends on model, region) | ~100-300ms (depends on instance + autoscaling) | Lowest possible (sub-100ms achievable with vLLM + co-location) |
| Data residency | Per Bedrock region (US, EU, APAC, etc.) | Per SageMaker region | Wherever you run EKS (any region, any sovereignty) |
| Compliance (HIPAA, FedRAMP, etc.) | Bedrock is HIPAA-eligible in most regions; FedRAMP coverage varies by model | SageMaker has broadest compliance posture | You own compliance entirely |
| Observability | CloudWatch + CloudTrail + Model Invocation Logging out of the box | SageMaker metrics + CloudWatch + custom | You build it (Prometheus, OTel, etc.) |
| Multi-model serving | Per-model invoke; one API | Multi-model endpoints supported | vLLM's multi-LoRA serving native |
| Cost break-even point | Wins below ~10M tokens/day | Wins when custom training matters | Wins above ~10M tokens/day *with the right team* |
| SRE headcount needed | Zero | ~0.5 FTE | ~2-3 FTE for production |

**The decision flow:**

```
Need to choose from any HuggingFace model OR fine-tune your own?
   yes -> SageMaker JumpStart (managed) or self-hosted (control)
   no  -> Bedrock

If yes-self-hosted: do you push >10M tokens/day, have 2-3 SREs to operate
                    GPU infra, and value sub-100ms p50?
   yes -> Self-hosted vLLM on EKS
   no  -> SageMaker JumpStart (let AWS run the endpoints)

If Bedrock: is the workload latency-sensitive AND high-volume AND
            using a stable model with stable prompts?
   yes -> Bedrock + Provisioned Throughput + caching
   no  -> Bedrock On-Demand + inference profile + caching
```

The honest read in 2026: **most production GenAI workloads should be Bedrock On-Demand.** The TCO math is clear at typical volumes (< 5M tokens/day). The SRE overhead of self-hosting is real and underestimated. SageMaker JumpStart is the right answer when you need a model that isn't in Bedrock's catalog and you don't want to run GPUs yourself. Self-hosted on EKS is the answer at very high volume *with the team to support it* -- but the failure mode is teams who spend more on GPU infra and engineering time than they would have on Bedrock tokens.

---

## Part 14: Full Production Terraform -- KB + Agent + Guardrail + CloudWatch Dashboard

The end-to-end skeleton. Bedrock Knowledge Base on OpenSearch Serverless, ingesting from S3, embedded with Titan v2; an Agent with a Lambda action group for order lookups; a Guardrail with PII masking and contextual grounding; a CloudWatch dashboard for token usage.

**Caveat before you `apply`:** the AWS provider's Bedrock resources (`aws_bedrockagent_*`, `aws_bedrock_guardrail`, `aws_bedrock_model_invocation_logging_configuration`) have evolved across provider versions -- block names, nested argument names, and attribute outputs have shifted between 5.50 and 5.80+. Pin the provider version and cross-check each block against `terraform-provider-aws` docs for that version. Treat this skeleton as the *shape* of the production wiring, not a paste-and-apply artifact.

```hcl
# ============================================================
# Variables and locals
# ============================================================
variable "name_prefix" { default = "support-bot" }
variable "region"      { default = "us-east-1" }

locals {
  embedding_model_arn = "arn:aws:bedrock:${var.region}::foundation-model/amazon.titan-embed-text-v2:0"
  generation_model_id = "anthropic.claude-sonnet-4-6-20260301-v1:0"
  inference_profile_arn = "arn:aws:bedrock:${var.region}:${data.aws_caller_identity.me.account_id}:inference-profile/us.anthropic.claude-sonnet-4-6-v1:0"
}

data "aws_caller_identity" "me" {}

# ============================================================
# S3 bucket for source documents
# ============================================================
resource "aws_s3_bucket" "kb_source" {
  bucket = "${var.name_prefix}-kb-source"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "kb_source" {
  bucket = aws_s3_bucket.kb_source.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.bedrock.arn
    }
  }
}

resource "aws_kms_key" "bedrock" {
  description         = "CMK for Bedrock KB + invocation logs"
  enable_key_rotation = true
}

# ============================================================
# OpenSearch Serverless collection for the vector index
# ============================================================
resource "aws_opensearchserverless_security_policy" "encryption" {
  name = "${var.name_prefix}-enc"
  type = "encryption"
  policy = jsonencode({
    Rules = [{ Resource = ["collection/${var.name_prefix}-kb"], ResourceType = "collection" }]
    AWSOwnedKey = true
  })
}

resource "aws_opensearchserverless_security_policy" "network" {
  name = "${var.name_prefix}-net"
  type = "network"
  policy = jsonencode([{
    Rules = [
      { Resource = ["collection/${var.name_prefix}-kb"], ResourceType = "collection" },
      { Resource = ["collection/${var.name_prefix}-kb"], ResourceType = "dashboard" },
    ]
    AllowFromPublic = true
  }])
}

resource "aws_opensearchserverless_collection" "kb" {
  name = "${var.name_prefix}-kb"
  type = "VECTORSEARCH"
  depends_on = [
    aws_opensearchserverless_security_policy.encryption,
    aws_opensearchserverless_security_policy.network,
  ]
}

# ============================================================
# Knowledge Base service role
# ============================================================
resource "aws_iam_role" "kb_role" {
  name = "${var.name_prefix}-kb-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "bedrock.amazonaws.com" }
      Action = "sts:AssumeRole"
      Condition = {
        StringEquals = { "aws:SourceAccount" = data.aws_caller_identity.me.account_id }
      }
    }]
  })
}

resource "aws_iam_role_policy" "kb_role" {
  role = aws_iam_role.kb_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["bedrock:InvokeModel"]
        Resource = local.embedding_model_arn
      },
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.kb_source.arn,
          "${aws_s3_bucket.kb_source.arn}/*",
        ]
      },
      {
        Effect = "Allow"
        Action = ["aoss:APIAccessAll"]
        Resource = aws_opensearchserverless_collection.kb.arn
      },
      {
        Effect = "Allow"
        Action = ["kms:Decrypt", "kms:GenerateDataKey"]
        Resource = aws_kms_key.bedrock.arn
      },
    ]
  })
}

# ============================================================
# Knowledge Base + data source
# ============================================================
resource "aws_bedrockagent_knowledge_base" "main" {
  name        = "${var.name_prefix}-kb"
  role_arn    = aws_iam_role.kb_role.arn
  description = "Support policy documents and product FAQs"

  knowledge_base_configuration {
    type = "VECTOR"
    vector_knowledge_base_configuration {
      embedding_model_arn = local.embedding_model_arn
    }
  }

  storage_configuration {
    type = "OPENSEARCH_SERVERLESS"
    opensearch_serverless_configuration {
      collection_arn    = aws_opensearchserverless_collection.kb.arn
      vector_index_name = "${var.name_prefix}-vector-index"
      field_mapping {
        vector_field   = "vector"
        text_field     = "text"
        metadata_field = "metadata"
      }
    }
  }
}

resource "aws_bedrockagent_data_source" "s3_docs" {
  knowledge_base_id = aws_bedrockagent_knowledge_base.main.id
  name              = "${var.name_prefix}-s3-source"

  data_source_configuration {
    type = "S3"
    s3_configuration {
      bucket_arn = aws_s3_bucket.kb_source.arn
    }
  }

  # Hierarchical chunking for long narrative docs
  vector_ingestion_configuration {
    chunking_configuration {
      chunking_strategy = "HIERARCHICAL"
      hierarchical_chunking_configuration {
        overlap_tokens = 60
        level_configurations {
          max_tokens = 1500   # parent chunks
        }
        level_configurations {
          max_tokens = 300    # child chunks (used for matching)
        }
      }
    }
  }
}

# ============================================================
# Lambda action group for order lookups
# ============================================================
data "archive_file" "order_lookup_zip" {
  type        = "zip"
  source_file = "${path.module}/lambda/order_lookup.py"
  output_path = "${path.module}/lambda/order_lookup.zip"
}

resource "aws_iam_role" "lambda_role" {
  name = "${var.name_prefix}-action-lambda-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_logs" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "order_lookup" {
  function_name = "${var.name_prefix}-order-lookup"
  filename      = data.archive_file.order_lookup_zip.output_path
  source_code_hash = data.archive_file.order_lookup_zip.output_base64sha256
  handler       = "order_lookup.lambda_handler"
  runtime       = "python3.12"
  role          = aws_iam_role.lambda_role.arn
  timeout       = 20   # keep tool calls snappy; the real cap is the 25 KB response and the InvokeAgent service timeout
}

resource "aws_lambda_permission" "allow_bedrock_agent" {
  statement_id  = "AllowBedrockAgentInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.order_lookup.function_name
  principal     = "bedrock.amazonaws.com"
  source_arn    = aws_bedrockagent_agent.main.agent_arn
}

# ============================================================
# Guardrail
# ============================================================
resource "aws_bedrock_guardrail" "main" {
  name                      = "${var.name_prefix}-guardrail"
  blocked_input_messaging   = "I can't help with that request."
  blocked_outputs_messaging = "I can't share that information."

  content_policy_config {
    filters_config {
      type            = "HATE"
      input_strength  = "HIGH"
      output_strength = "HIGH"
    }
    filters_config {
      type            = "INSULTS"
      input_strength  = "MEDIUM"
      output_strength = "MEDIUM"
    }
    filters_config {
      type            = "SEXUAL"
      input_strength  = "HIGH"
      output_strength = "HIGH"
    }
    filters_config {
      type            = "VIOLENCE"
      input_strength  = "HIGH"
      output_strength = "HIGH"
    }
    filters_config {
      type            = "PROMPT_ATTACK"
      input_strength  = "MEDIUM"   # tune carefully -- too high blocks legit prompt engineering
      output_strength = "NONE"
    }
  }

  sensitive_information_policy_config {
    pii_entities_config {
      type   = "EMAIL"
      action = "ANONYMIZE"
    }
    pii_entities_config {
      type   = "PHONE"
      action = "ANONYMIZE"
    }
    pii_entities_config {
      type   = "CREDIT_DEBIT_CARD_NUMBER"
      action = "BLOCK"
    }
    pii_entities_config {
      type   = "US_SOCIAL_SECURITY_NUMBER"
      action = "BLOCK"
    }
  }

  topic_policy_config {
    topics_config {
      name       = "legal-advice"
      definition = "Specific recommendations on legal action, contract interpretation, or representation."
      examples   = ["Should I sue?", "Is this contract enforceable?"]
      type       = "DENY"
    }
  }

  contextual_grounding_policy_config {
    filters_config {
      type      = "GROUNDING"
      threshold = 0.75
    }
    filters_config {
      type      = "RELEVANCE"
      threshold = 0.50
    }
  }
}

resource "aws_bedrock_guardrail_version" "v1" {
  guardrail_arn = aws_bedrock_guardrail.main.guardrail_arn
  description   = "v1 -- production"
}

# ============================================================
# Agent
# ============================================================
resource "aws_iam_role" "agent_role" {
  name = "${var.name_prefix}-agent-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "bedrock.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "agent_role" {
  role = aws_iam_role.agent_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["bedrock:InvokeModel"]
        Resource = [
          "arn:aws:bedrock:${var.region}::foundation-model/${local.generation_model_id}",
          local.inference_profile_arn,
        ]
      },
      {
        Effect = "Allow"
        Action = ["bedrock:Retrieve"]
        Resource = aws_bedrockagent_knowledge_base.main.arn
      },
      {
        Effect = "Allow"
        Action = ["bedrock:ApplyGuardrail"]
        Resource = aws_bedrock_guardrail.main.guardrail_arn
      },
    ]
  })
}

resource "aws_bedrockagent_agent" "main" {
  agent_name              = "${var.name_prefix}-agent"
  agent_resource_role_arn = aws_iam_role.agent_role.arn
  foundation_model        = local.generation_model_id
  instruction             = <<-EOT
    You are a customer support assistant for ACME Corp.
    Use the attached knowledge base to answer policy questions.
    Use the order_lookup action group for order-status questions.
    Be brief: answer in 2-3 sentences unless the user asks for detail.
    If you don't know, say so. Never fabricate order IDs or policy quotes.
  EOT
  guardrail_configuration {
    guardrail_identifier = aws_bedrock_guardrail.main.guardrail_id
    guardrail_version    = aws_bedrock_guardrail_version.v1.version
  }
}

resource "aws_bedrockagent_agent_action_group" "order_lookup" {
  agent_id          = aws_bedrockagent_agent.main.agent_id
  action_group_name = "OrderActions"
  agent_version     = "DRAFT"

  action_group_executor {
    lambda = aws_lambda_function.order_lookup.arn
  }

  # NOTE: The aws_bedrockagent_agent_action_group schema for function_schema
  # has shifted across AWS provider versions. Verify against your pinned
  # provider docs before applying -- some versions expose `function` blocks
  # with a `parameters` map keyed by parameter name; others use api_schema +
  # an OpenAPI document in S3. The shape below is the OpenAPI-via-payload
  # variant, which is the most portable across recent provider versions.
  api_schema {
    payload = jsonencode({
      openapi = "3.0.0"
      info    = { title = "OrderActions", version = "1.0.0" }
      paths = {
        "/get_order_status" = {
          post = {
            description = "Look up an order's status, ship date, and tracking number by order ID."
            operationId = "get_order_status"
            requestBody = {
              required = true
              content = {
                "application/json" = {
                  schema = {
                    type = "object"
                    properties = {
                      order_id = {
                        type        = "string"
                        description = "The order ID, e.g. ORD-12345"
                      }
                    }
                    required = ["order_id"]
                  }
                }
              }
            }
            responses = {
              "200" = {
                description = "Order status"
                content = {
                  "application/json" = {
                    schema = { type = "object" }
                  }
                }
              }
            }
          }
        }
      }
    })
  }
}

resource "aws_bedrockagent_agent_knowledge_base_association" "main" {
  agent_id             = aws_bedrockagent_agent.main.agent_id
  knowledge_base_id    = aws_bedrockagent_knowledge_base.main.id
  agent_version        = "DRAFT"
  description          = "Support policy KB for policy Q&A."
  knowledge_base_state = "ENABLED"
}

resource "aws_bedrockagent_agent_alias" "prod" {
  agent_id         = aws_bedrockagent_agent.main.agent_id
  agent_alias_name = "prod"
}

# ============================================================
# Model Invocation Logging
# ============================================================
resource "aws_s3_bucket" "invocation_logs" {
  bucket = "${var.name_prefix}-bedrock-invocation-logs"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "invocation_logs" {
  bucket = aws_s3_bucket.invocation_logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.bedrock.arn
    }
  }
}

resource "aws_cloudwatch_log_group" "invocations" {
  name              = "/aws/bedrock/${var.name_prefix}-invocations"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.bedrock.arn
}

resource "aws_bedrock_model_invocation_logging_configuration" "main" {
  logging_config {
    cloudwatch_config {
      log_group_name = aws_cloudwatch_log_group.invocations.name
      role_arn       = aws_iam_role.bedrock_logging.arn
    }
    s3_config {
      bucket_name = aws_s3_bucket.invocation_logs.id
      key_prefix  = "invocations/"
    }
    text_data_delivery_enabled       = true
    image_data_delivery_enabled      = false
    embedding_data_delivery_enabled  = false
  }
}

resource "aws_iam_role" "bedrock_logging" {
  name = "${var.name_prefix}-bedrock-logging"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "bedrock.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "bedrock_logging" {
  role = aws_iam_role.bedrock_logging.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["logs:CreateLogStream", "logs:PutLogEvents"]
      Resource = "${aws_cloudwatch_log_group.invocations.arn}:*"
    }]
  })
}

# ============================================================
# CloudWatch dashboard for token usage
# ============================================================
resource "aws_cloudwatch_dashboard" "bedrock_tokens" {
  dashboard_name = "${var.name_prefix}-bedrock-tokens"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x = 0  y = 0  width = 12  height = 6
        properties = {
          title  = "Output tokens by model (SUM) -- the cost driver"
          view   = "timeSeries"
          region = var.region
          metrics = [
            ["AWS/Bedrock", "OutputTokenCount", "ModelId", "anthropic.claude-sonnet-4-6-20260301-v1:0", { stat = "Sum" }],
            ["AWS/Bedrock", "OutputTokenCount", "ModelId", "anthropic.claude-haiku-4-5-20260301-v1:0",  { stat = "Sum" }],
          ]
          period = 300
        }
      },
      {
        type   = "metric"
        x = 12  y = 0  width = 12  height = 6
        properties = {
          title  = "Input tokens by model (SUM)"
          view   = "timeSeries"
          region = var.region
          metrics = [
            ["AWS/Bedrock", "InputTokenCount", "ModelId", "anthropic.claude-sonnet-4-6-20260301-v1:0", { stat = "Sum" }],
            ["AWS/Bedrock", "InputTokenCount", "ModelId", "anthropic.claude-haiku-4-5-20260301-v1:0",  { stat = "Sum" }],
          ]
          period = 300
        }
      },
      {
        type   = "metric"
        x = 0  y = 6  width = 8  height = 6
        properties = {
          title  = "Invocation latency p95"
          view   = "timeSeries"
          region = var.region
          metrics = [
            ["AWS/Bedrock", "InvocationLatency", "ModelId", "anthropic.claude-sonnet-4-6-20260301-v1:0", { stat = "p95" }],
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x = 8  y = 6  width = 8  height = 6
        properties = {
          title  = "Throttles -- bump quota or switch to inference profile"
          view   = "timeSeries"
          region = var.region
          metrics = [
            ["AWS/Bedrock", "InvocationThrottles", "ModelId", "anthropic.claude-sonnet-4-6-20260301-v1:0", { stat = "Sum" }],
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x = 16  y = 6  width = 8  height = 6
        properties = {
          title  = "Server errors"
          view   = "timeSeries"
          region = var.region
          metrics = [
            ["AWS/Bedrock", "InvocationServerErrors", "ModelId", "anthropic.claude-sonnet-4-6-20260301-v1:0", { stat = "Sum" }],
          ]
          period = 60
        }
      },
    ]
  })
}
```

That's the production skeleton. Once the provider blocks are reconciled to your pinned version: drop docs into the KB source bucket, run `aws bedrock-agent start-ingestion-job`, `aws bedrock-agent prepare-agent`, point at the `prod` alias, and you can invoke it end to end.

---

## Part 15: Production Gotchas -- 23 Items That Have Bitten Engineers

A non-exhaustive list of foot-guns:

1. **Output tokens cost ~5x input tokens.** The single most important pricing fact. Cap with `max_tokens` and "be brief" system instructions.
2. **`max_tokens` is a budget, not a safety net.** It cuts off the model mid-response if exceeded. Set it to the *actual* max your UI accepts, not a generous round number.
3. **Prompt-cache TTL is 5 minutes by default with a 1-hour opt-in for Claude 4.x (added Jan 2026), and either way it's region-scoped.** Sporadic-traffic workloads should opt into the 1-hour TTL; for very bursty workloads stick with the cheaper-write 5-minute default. Cross-region inference profiles can route to a different region and invalidate the cache.
4. **Using model ID directly bypasses cross-region routing.** Production default should be the inference profile ARN. The model ID is the right pick only when you're optimizing for cache hit rate or when the model isn't in the profile family.
5. **KB chunking: fixed-size beats semantic for short structured docs.** Semantic is for long narrative content. Hierarchical is the production default for mixed corpora.
6. **OpenSearch Serverless OCU minimums make small KBs surprisingly expensive (~$350/mo idle).** Aurora pgvector is cheaper at low scale.
7. **Hybrid search is NOT enabled by default.** Pure vector search misses SKU/ID/keyword queries. Set `overrideSearchType: "HYBRID"` for any corpus with structured identifiers.
8. **Action-group Lambda responses are capped at 25 KB**, and the `InvokeAgent` API has its own service timeout bounding the whole turn -- so even a 15-minute Lambda will be killed by the orchestrator well before that. Async-offload slow tools (SQS, Step Functions) and return a small handle, or move them to `RETURN_CONTROL`. The "25-second Lambda timeout" you may have read is folklore -- the real limits are the response size cap and the InvokeAgent service timeout.
9. **`RETURN_CONTROL` is the only path to in-process / browser / mobile / on-prem tool execution.** Lambda action groups can't reach those network paths.
10. **`ApplyGuardrail` is billed separately** even when wrapping a Bedrock invoke -- but it's the only way to apply Bedrock guardrails to non-Bedrock models.
11. **Contextual grounding is the killer feature for hallucination detection.** Turn it on for any RAG workload. The grounding+relevance score thresholds are your hallucination guardrail.
12. **The prompt-attack filter blocks legitimate prompt engineering** at High strength. Start at Low or Medium for trusted user inputs; only High for anonymous traffic.
13. **Model availability is region-by-region.** Claude 4.x first lands in `us-east-1`, `us-west-2`, `eu-central-1`. Always verify via `ListFoundationModels`; don't hardcode.
14. **`bedrock:InvokeModel` is granular per model ARN.** Use resource-level scoping as a cost-control lever -- prevents a developer from accidentally calling Opus instead of Haiku.
15. **Model Invocation Logging captures FULL prompts including PII/PHI** unless you filtered upstream. KMS-encrypt the S3 destination, lifecycle-expire per data-minimization rules, and inject `requestMetadata` (tenant/patient/case IDs) so domain-scoped audit queries are possible -- without it you're grepping prompt text across years of logs.
16. **`InputTokenCount` and `OutputTokenCount` in CloudWatch are gauges per invocation, not running counters.** Use `SUM()` statistic in dashboards, not `AVERAGE()`.
17. **Anthropic Claude on Bedrock has a slightly different tool-use schema than the direct Anthropic API.** Don't share code paths blindly. Converse API hides the difference.
18. **Batch inference is ~50% cheaper but has variable latency** (up to 24h). Use for offline enrichment; never user-facing.
19. **Knowledge Base sync re-indexes ALL documents on every sync** unless the data source was configured for incremental ingestion at creation time. A 5-minute sync turns into 4 hours after the corpus grows.
20. **Bedrock Studio access requires IAM Identity Center (SSO)**, not standalone IAM users. Plan for SSO integration before promising non-engineers Studio access.
21. **The CMK for the MIL bucket must grant `bedrock.amazonaws.com` `kms:GenerateDataKey`/`kms:Decrypt`.** Bedrock writes logs as its own service principal; if the key policy omits it, log delivery fails *silently* -- you discover the empty bucket at audit time. Alarm on log-delivery failure.
22. **S3 Object Lock Governance Mode is not tamper-proof for compliance.** A privileged admin can still purge the audit trail. Long-retention HIPAA/SOX/financial audit trails need Compliance Mode (locked even against account root).
23. **Idle Provisioned Throughput silently bills 24/7.** A PT commitment for a workload that no longer drives traffic is the bill-spike pattern with a unique signature: high spend, *low* invocation count. Neither a runaway loop nor a cache regression -- check `Bedrock-MU-Hours` UsageType in Cost Explorer.

---

## Part 16: Decision Frameworks

### Framework 1: Bedrock vs SageMaker vs self-hosted

Covered in Part 13. Three axes: model selection breadth, SRE headcount available, token volume per day. Below 10M tokens/day and not needing custom training, Bedrock wins TCO.

### Framework 2: Converse vs InvokeModel

| Use | Pick |
|-----|------|
| Any chat / text generation | **Converse** |
| Tool use / function calling | **Converse** with `toolConfig` |
| Streaming for UIs | **ConverseStream** |
| Image generation (SDXL, Titan Image, Nova Canvas) | `InvokeModel` |
| Embedding generation | `InvokeModel` |
| Legacy code already on `InvokeModel` with no friction | Stay |

### Framework 3: Knowledge Base vector store

| Corpus size | Workload | Pick |
|-------------|----------|------|
| < 1k docs | Internal wiki, sporadic queries | Aurora pgvector |
| 1k - 100k docs | Production RAG, moderate QPS | OpenSearch Serverless (default) |
| > 100k docs | High-volume RAG | OpenSearch Serverless with provisioned OCUs OR existing Pinecone |
| Graph-shaped retrieval | GraphRAG | Neptune Analytics |
| Massive cold corpus | Compliance archive with occasional retrieval | S3 Vectors |
| Existing Pinecone investment | Anything | Pinecone (managed connector) |

### Framework 4: Action group execution mode

| Scenario | Pick |
|----------|------|
| Tool is an AWS resource in your account | Lambda action group |
| Tool needs caller credentials (per-user token) | `RETURN_CONTROL` |
| Tool runs on user device | `RETURN_CONTROL` |
| Tool reaches on-prem network only | `RETURN_CONTROL` |
| Tool result needs user confirmation before agent acts | `RETURN_CONTROL` |
| Tool is fast and stateless | Lambda action group |
| Tool returns > 25 KB body | Summarize / paginate / write to S3 and return a reference |
| Tool is slow (tens of seconds) | Lambda action group with async offload (SQS / Step Functions), OR `RETURN_CONTROL` |

### Framework 5: Provisioned Throughput vs On-Demand vs Batch

| Workload | Pick |
|----------|------|
| Most production workloads | **On-Demand** with inference profile + caching |
| Custom fine-tuned model | **Provisioned Throughput** (required) |
| Guaranteed-capacity workload during known peak | **Provisioned Throughput** for the peak window |
| Offline enrichment, latency-insensitive | **Batch inference** (~50% off) |
| Stress-test / spike traffic | On-Demand with inference profile (the profile sums regional TPM quotas) |

---

## Part 17: The 10 Commandments of Production Bedrock

1. **Converse, not InvokeModel, for any chat / text / tool-use workload.** The unified message format is the only sane API surface across vendors. `InvokeModel` is reserved for image-gen and embeddings.
2. **Inference profile ARN, not raw model ID, as the production default.** Cross-region routing buys you TPM headroom and resilience for free. Accept the slightly reduced cache hit rate.
3. **Pick the smallest model that hits your eval bar.** Haiku for classification and simple Q&A, Sonnet for general, Opus for the hardest reasoning only. The router pattern (Haiku decides "is this hard?" and dispatches to Sonnet/Opus) pays for itself in days.
4. **Cap output tokens explicitly.** `max_tokens` + "be brief" system prompt + `stopSequences`. Output tokens cost ~5x input. The single most impactful prompt-engineering discipline.
5. **Cache the stable prefix.** Long system prompt? `cachePoint` at end of system. Multi-turn against the same RAG context? `cachePoint` at end of retrieved chunks. 80-90% input cost reduction on the cached portion. Pick TTL per workload: 5 minutes (default, cheaper write) for bursty chat, 1 hour (Jan 2026, higher write premium) for sporadic traffic. Both are region-scoped.
6. **Knowledge Base default = OpenSearch Serverless with hierarchical chunking + hybrid search enabled.** Aurora pgvector for small/cheap KBs. Pinecone if you already pay them. Enable hybrid search even for "semantic" corpora -- it handles SKU/ID queries the pure-vector path misses.
7. **Contextual grounding is the hallucination guardrail for any RAG workload.** Set thresholds at ~0.75 grounding / ~0.50 relevance. The "model confidently said the wrong thing" failure mode goes away.
8. **`RETURN_CONTROL` for any tool that needs caller credentials, runs on the user's device, reaches an on-prem network, or needs user confirmation.** Lambda action groups for everything else, with the 25 KB response cap and the InvokeAgent service timeout in mind -- slow or chatty tools must async-offload.
9. **Model Invocation Logging on, KMS-encrypted destination, lifecycle-expire per data-minimization.** Pair with CloudTrail Bedrock data events for the complete audit story. The SOC 2 / HIPAA / GDPR answer to "what did this app prompt the model with?"
10. **Bedrock is the managed kitchen, not the appliance store.** If you find yourself debugging CUDA drivers, you took a wrong turn -- you're back in SageMaker JumpStart or self-hosted vLLM territory. The whole point of Bedrock is *not running model infrastructure*. Push as much complexity into managed surfaces (KB, Agents, Guardrails) as you can; reach for self-hosted only when the token math at >10M/day plus the SRE team to run it both check out.

---

## Cheat Sheet -- Things to Memorize

### Data-plane endpoints and key IAM actions

```
Endpoint: bedrock-runtime.amazonaws.com    IAM prefix: bedrock:
  bedrock:InvokeModel                    (gates BOTH InvokeModel AND Converse)
  bedrock:InvokeModelWithResponseStream  (gates BOTH stream APIs AND ConverseStream)
  bedrock:ApplyGuardrail                 (standalone -- works on non-Bedrock text too)
  -- no bedrock:Converse action exists; writing it in a policy is a no-op --

Endpoint: bedrock-agent-runtime.amazonaws.com    IAM prefix: bedrock:
  bedrock:InvokeAgent
  bedrock:Retrieve                       (KB chunks; you build the prompt)
  bedrock:RetrieveAndGenerate            (end-to-end RAG; one call)
```

### Converse minimal call

```python
bedrock.converse(
  modelId="<inference-profile-arn-or-model-id>",
  system=[{"text": "..."}, {"cachePoint": {"type": "default"}}],
  messages=[{"role": "user", "content": [{"text": "..."}]}],
  inferenceConfig={"maxTokens": 500, "temperature": 0.2},
  toolConfig={"tools": [...], "toolChoice": {"auto": {}}},
  guardrailConfig={"guardrailIdentifier": "...", "guardrailVersion": "1"},
)
```

### Cost levers ranked

```
1. Model selection (10-15x range; pick the smallest that passes eval)
2. max_tokens + "be brief" + stopSequences (output ~5x input)
3. Prompt caching (50-90% on cached portion; 5-min default or 1-hour opt-in TTL on Claude 4.x; region-scoped)
4. Batch inference (50% off; up to 24h latency)
5. Inference profile vs region-pinned (capacity + resilience; small cache hit)
6. PT only for: custom fine-tunes, guaranteed-capacity peaks
```

### Guardrail policy families

```
1. Content filter (Hate, Insults, Sexual, Violence; None/Low/Medium/High)
2. Denied topics (up to 30; natural-language description + examples)
3. Word filters (profanity preset + custom blocklist)
4. Sensitive info (30+ PII types; Block or Mask)
5. Contextual grounding (grounding score + relevance score thresholds)
6. Prompt attack (None/Low/Medium/High; input only)
```

### KB chunking by content type

```
Short structured docs (FAQs, specs, tickets) -> FIXED with overlap
Long narrative docs (articles, manuals)      -> HIERARCHICAL (default for mixed)
Strongly thematic narrative                  -> SEMANTIC
Code or table-aware splits                   -> CUSTOM LAMBDA
```

### CloudWatch metric SUM gotcha

```
InputTokenCount / OutputTokenCount = gauges per invocation
Dashboards MUST use Sum(), not Average(), for "tokens over time"
```

### Bedrock vs SageMaker vs self-hosted (one line each)

```
Bedrock         = managed kitchen; bring orders; zero GPU ops; the default
SageMaker JS    = serviced test kitchen; you run endpoints; any HF model
Self-hosted     = build the kitchen; cheapest at >10M tok/day WITH the team
```

---

## What's Next

Tomorrow (2026-05-16) is **AWS Sagemaker for Production ML** -- the other half of the model-platform story: when Bedrock isn't enough, when fine-tuning matters, when the model lifecycle (data prep -> train -> tune -> deploy -> monitor) needs first-class infrastructure. The Bedrock-vs-SageMaker decision frame from Part 13 sets up that conversation. The audit-trail thread continues: every SageMaker endpoint invocation is a CloudTrail data event too, joinable in Athena against today's Bedrock invocation logs to build the org-wide "where did GenAI get used and by whom" view.
