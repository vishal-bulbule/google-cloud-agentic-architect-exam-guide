## Section 3 — Developing custom agents (~33%)

**What the exam is really testing**
1. Whether you can pick the *right-sized* building block (model, workflow primitive, session and memory backend, vector store, protocol) for a stated requirement, and name the trade-off you accept.
2. Whether you know the managed Agent Platform services (Agent Runtime, Sessions, Memory Bank, Agent Retrieval, RAG Engine, Agent Search, Agent Registry, Agent Identity) well enough to know their defaults, limits and how they wire into ADK.
3. Whether you can keep a multi-agent system deterministic, governable and least-privilege: graph and workflow agents instead of prompt spaghetti, per-agent identity instead of shared service accounts, registries instead of hard-coded endpoints.

> **2026 naming map.** Vertex AI is now **Gemini Enterprise Agent Platform** ("Agent Platform"). Agent Engine is now **Agent Runtime**; the REST resource is still `reasoningEngines`. Vertex AI Search is now **Agent Search** (API: `discoveryengine`). Vector Search 2.0 is now **Agent Retrieval**, and the original index/endpoint product is **Vector Search 1.0**. ADK class names still carry the old prefix: `VertexAiSessionService`, `VertexAiMemoryBankService`, `VertexAiSearchTool`, `VertexAiRagMemoryService`. The ADK docs call RAG Engine "Knowledge Engine" in places, while the Cloud docs still say **RAG Engine**. If an answer option uses either name, treat them as the same product. (That the product itself has been renamed is unverified.)

---

### 3.1 Designing and building agentic workflows in code

#### 3.1.1 Selecting and configuring the model

> 📊 **Infographic:** Model selection map
>
> [![Model selection map](../infographics/05-model-selection.png)](../infographics/05-model-selection.png)

**Current Gemini line-up on Agent Platform (September 2026).** Generally available: Gemini 3.8 Flash ("workhorse… long-horizon coding and autonomous agents"), 3.7 Flash, 3.6 Flash, 3.5 Flash, **3.5 Flash-Lite** ("lightweight agentic workflows at top speeds and minimal cost"), 3.1 Flash-Lite (most cost-efficient), 2.5 Pro/Flash/Flash-Lite. Preview: Gemini 3.1 Pro (`gemini-3.1-pro-preview`) and Gemini 3 Flash (`gemini-3-flash-preview`). There are also specialised models: 3.8 Flash Cyber, 3.8 Live and 2.5 Flash Live for bidirectional audio, and the image models. ADK samples use the alias `gemini-flash-latest`.

**Consumption options** (these change the cost answer): Standard PayGo, **Flex PayGo** (cheaper, can be slower), **Priority PayGo** (latency-sensitive), **Provisioned Throughput** (reserved capacity for predictable, high-volume traffic), **Batch inference** (offline, cheapest), plus implicit and explicit **context caching** (cached input tokens are billed at about 10% of the normal rate).

**Model selection matrix**

| Requirement signal | Best fit | Why / trade-off |
|---|---|---|
| Complex multi-step reasoning, planning, coding orchestrator | Gemini Pro-class, or the newest Flash with thinking | Accuracy costs latency and tokens. Keep the reasoning model on the root/planner only |
| High-volume classification, routing, extraction, simple sub-agents | Gemini Flash-Lite | Lowest cost and latency. Weaker at long-horizon reasoning |
| "SLM", on-device or edge, air-gapped, or data may not leave the VPC | **Gemma** (open weights), self-deployed from Model Garden, on GKE with vLLM, or on Cloud Run GPU | You own the capacity, scaling, patching and eval. Cheaper per token only at sustained volume |
| Open model, bursty or unpredictable traffic, no MLOps team | **Model as a Service (MaaS)**: serverless managed API in Model Garden (cards named "API Service") | Pay per token, no GPUs to manage. No custom weights |
| Custom or fine-tuned weights, custom pre/post-processing, strict single-tenant or data-residency rules, predictable high volume | **Self-deployed** Model Garden endpoint (one-click, prebuilt vLLM/Hex-LLM/SGLang/TGI/TensorRT-LLM containers, or a custom vLLM container) | Lower TCO at scale and full control of the data path and hardware. Higher engineering and operations cost |
| Must standardise on Kubernetes, share GPU pools, or reuse existing GKE platform | vLLM on **GKE** | Maximum control, maximum operational burden |
| Third-party proprietary model (for example Claude) under Google Cloud billing and controls | Partner model via Model Garden (MaaS) | Proprietary behaviour. Check regional availability |
| Central governance of model traffic: token limits, semantic caching, model routing, Model Armor | **Apigee AI Gateway** in front of the model (`ApigeeLlm` in ADK) | Adds a hop and a component to run. You get quota, cost and safety control |

Self-deployed partner models: licences come through Cloud Marketplace, weights **cannot be exported**, and only the shared public endpoint type is supported.

**How the model is wired into ADK** (verified in the ADK docs):

```python
from google.adk.agents import LlmAgent
from google.adk.models.lite_llm import LiteLlm

# 1. Gemini by name. Set GOOGLE_GENAI_USE_ENTERPRISE=TRUE plus project/location for Agent Platform.
a = LlmAgent(name="planner", model="gemini-flash-latest", instruction="...")

# 2. A Model Garden / fine-tuned endpoint: pass the full endpoint resource string
b = LlmAgent(name="ft", model="projects/P/locations/us-central1/endpoints/1234567890")

# 3. Self-hosted vLLM (GKE / Cloud Run / Model Garden) through LiteLLM's OpenAI-compatible path
c = LlmAgent(
    name="gemma_agent",
    model=LiteLlm(model="openai/google/gemma-4-31B-it",
                  api_base="https://my-vllm.run.app/v1",
                  extra_headers={"Authorization": f"Bearer {id_token}"}),
)
```

- **Gotcha for vLLM:** tools only work if the server has OpenAI-compatible function calling enabled, for example `--enable-auto-tool-choice` plus the right `--tool-call-parser`. Without it, the agent "never calls tools".
- **Gotcha for Gemma 3:** it has no native function calling or system instructions. Use `google.adk.models.Gemma(model="gemma-3-27b-it")` or `Gemma3Ollama()`. Gemma 4 works through the standard `Gemini(...)` class or through vLLM.
- **LiteLLM:** ADK needs `litellm>=1.84`. The docs carry a supply-chain advisory for LiteLLM 1.82.7 and 1.82.8 (March 2026). That is a real-world reminder to pin and scan dependencies.
- **Model routing in ADK:** `RoutedLlm` (model-level) and `RoutedAgent` (agent-level) take a router function, support A/B tests and complexity-based routing, and **fail over** when the chosen model fails *before* producing any output. After output has started, errors propagate. The docs mark both **experimental, ADK TypeScript v1.0**. In Python the usual patterns are:
  - assign different models per agent (Flash-Lite router → Pro specialist);
  - LiteLLM or Apigee AI Gateway routing;
  - a `before_model_callback` that rewrites the request (unverified as a documented pattern).
- `planner=BuiltInPlanner(thinking_config=ThinkingConfig(...))` exposes Gemini thinking budgets. The trade-off is more reasoning for more latency and cost.

**Exam signals**
- "Unpredictable/bursty traffic, no ML ops team, open model" → **MaaS**.
- "Fine-tuned weights / strict residency / predictable high volume / choose GPU type" → **self-deployed Model Garden endpoint**.
- "Already on Kubernetes, need GPU sharing" → **vLLM on GKE**.
- "Cheapest model for a triage/router sub-agent" → **Flash-Lite**. "Hardest reasoning" → Pro/thinking.
- "Rate/token limits and semantic caching across many agents' LLM calls" → **Apigee AI Gateway**.
- "Reserved, guaranteed capacity" → **Provisioned Throughput**. "Overnight bulk" → **Batch**.

**Distractors:** fine-tuning when the problem is missing enterprise knowledge (use RAG). Self-hosting "to save money" at low or irregular volume. Using Pro for every sub-agent.

---

#### 3.1.2 Building custom agents with ADK

**`LlmAgent` (alias `Agent`): the knobs the exam cares about**

| Param | What it does / gotcha |
|---|---|
| `name`, `description` | `description` is what a parent LLM reads when deciding a transfer. Vague descriptions cause mis-routing |
| `instruction` | Supports `{state_key}` templating (and `{artifact.name}`), or an `InstructionProvider` function |
| `tools` | Functions (auto-wrapped as `FunctionTool`), toolsets, `AgentTool`, built-ins |
| `sub_agents` | Enables LLM-driven transfer (AutoFlow `transfer_to_agent`). **Single-parent rule:** one agent instance can only have one parent, otherwise `ValueError` |
| `output_key` | Writes the final text response into `session.state[key]`. This is how steps of a workflow pass data |
| `output_schema` / `input_schema` | Pydantic-enforced JSON. `output_schema` together with `tools` is natively supported only on some models (Gemini 3.x). Otherwise ADK falls back to a `set_model_response` tool that "may not work reliably", so split the formatting into its own agent |
| `include_contents='none'` | Stateless agent that does not see conversation history. Cheaper, and common inside loops |
| `generate_content_config`, `planner` | Temperature, safety settings, thinking |
| `mode` (ADK 2.0) | `chat` (default), `task`, `single_turn`: collaboration mode for **sub-agents only** (see 3.3) |

**Tool types**

- `FunctionTool` (a plain Python function; its docstring becomes the description)
- `LongRunningFunctionTool` (human approval, slow jobs)
- `AgentTool` (another agent used as a tool)
- `McpToolset` (MCP over stdio, SSE or Streamable HTTP; `tool_filter=` restricts the exposed tools)
- `OpenAPIToolset`, `APIHubToolset`, `ApplicationIntegrationToolset` (more than 100 Integration Connectors: Salesforce, SAP, ServiceNow…)
- `BigQueryToolset` (tool config can block writes)
- `VertexAiSearchTool` (Agent Search)
- `google_search`, built-in code execution
- `ApiRegistry.get_toolset()` for Google Cloud MCP servers
- `AgentRegistry.get_mcp_toolset()` and `SkillToolset` (see 3.2)

Built-in tool limitations:
- Before ADK Python 1.16, Google Search, Agent Search and built-in code execution each had to be the **only tool** on an agent. Newer versions work around this (`GoogleSearchTool(bypass_multi_tools_limit=True)`).
- Built-in tools cannot sit in a sub-agent. The exceptions are `GoogleSearchTool` and `VertexAiSearchTool` in Python.
- The classic workaround is still valid: wrap the built-in tool in its own agent and expose it as an `AgentTool`.

**MCP deployment gotcha:** when deploying to Cloud Run, GKE or Agent Runtime, define the agent and its `McpToolset` **synchronously** at module level. The async factory pattern that works in `adk web` fails in deployment. stdio MCP servers (such as `npx`) must be installed inside the same container.

**Callbacks.** There are six hooks: `before_agent_callback`/`after_agent_callback`, `before_model_callback`/`after_model_callback`, and `before_tool_callback`/`after_tool_callback`. Returning `None` continues normally. **Returning an object short-circuits the step:**
- a `types.Content` skips the agent;
- an `LlmResponse` skips the model call (use it for caching or guardrails);
- a `dict` skips the tool and becomes its result.

Callbacks can read and write state through `callback_context.state` or `tool_context.state`.

```python
def block_pii_tool(tool, args, tool_context):
    if tool.name == "export_customer" and not tool_context.state.get("user:is_admin"):
        return {"status": "denied", "reason": "admin only"}   # tool is NOT executed
    return None

agent = LlmAgent(name="crm", model="gemini-flash-latest", tools=[export_customer],
                 before_tool_callback=block_pii_tool)
```

**Plugins.** A plugin subclasses `BasePlugin`, is registered **once on the `Runner`/`App`**, and its callbacks apply **globally** to every agent, model call and tool call under that runner. The docs recommend plugins over per-agent callbacks for security guardrails and policy. Prebuilt plugins:
- `ReflectAndRetryToolPlugin(max_retries=3)`
- BigQuery Agent Analytics
- Model Armor
- Context Filter
- Global Instruction
- Save Files as Artifacts
- Logging

Exam logic: "the same guardrail on every agent and tool" → plugin. "One agent's special rule" → callback.

**Workflow agents (prebuilt, deterministic, no LLM for control flow)**

- `SequentialAgent`: runs its sub-agents in order with a shared `InvocationContext`. Data passes through `output_key` → `{key}`.
- `ParallelAgent`: runs its sub-agents concurrently. Each child gets its own `branch` (isolated history), but **all share the same `session.state`**, so each must write to a distinct key. Follow it with a Sequential "synthesizer" step (fan-out/gather).
- `LoopAgent`: repeats its sub-agents until `max_iterations` is reached or a sub-agent emits `escalate=True` (for example through the `exit_loop` tool or a custom `BaseAgent`). **Always set `max_iterations`**, because otherwise you risk runaway token spend from a reasoning loop.
- Custom `BaseAgent` (override `_run_async_impl`): use it for conditional logic the prebuilt ones cannot express, such as "regenerate only if tone is negative".

```python
from google.adk.agents import LlmAgent, SequentialAgent, ParallelAgent, LoopAgent

fetch = ParallelAgent(name="fetch", sub_agents=[
    LlmAgent(name="policy", model="gemini-flash-latest", output_key="policy_ctx", ...),
    LlmAgent(name="orders", model="gemini-flash-latest", output_key="order_ctx", ...)])
refine = LoopAgent(name="refine", max_iterations=3, sub_agents=[critic, reviser])
pipeline = SequentialAgent(name="pipeline", sub_agents=[fetch, drafter, refine])
```

**Graph-based workflows (ADK 2.0: Python/TypeScript/Go v2.0.0).** `Workflow(name=..., edges=[...])` defines nodes and edges.
- A node can be an agent, a tool, plain Python code (no LLM call) or a nested Workflow.
- Each node's return value is passed as the next node's input, so no state writes are needed.
- **Routing:** a router node returns `Event(route=[...])`, and an edge maps route values to handler nodes.
- **Fan-out/fan-in:** list `'START'` in several rows, then join with `JoinNode`.
- **HITL:** a node yields `RequestInput(message=...)` and the workflow pauses deterministically with no model involved.
- An `LlmAgent` inside a graph must run in `single_turn` or `task` mode. In ADK Python 2.0.0, `task` mode is disabled inside graphs.
- ADK also offers "dynamic workflows": plain code orchestration with loops and recursion, for flows too irregular for a static graph.

```python
from google.adk import Agent, Workflow, Event

classify = Agent(name="classify", model="gemini-flash-latest",
                 instruction="Reply BUG, SUPPORT or LOGISTICS")
def router(node_input: str):
    return Event(route=[r.strip() for r in node_input.split(",")])
root_agent = Workflow(name="triage", edges=[
    ("START", classify, router),
    (router, {"BUG": handle_bug, "SUPPORT": handle_support, "LOGISTICS": handle_logistics}),
])
```

**Agent Config (no-code YAML).** `adk create --type=config my_agent` generates `root_agent.yaml` (name, model, description, instruction, tools, sub_agents). It is **experimental**, supports **Gemini models only**, and needs Python/Java for custom tool code. `LangGraphAgent` and `A2aAgent` are not supported. Supported built-ins include `google_search`, `AgentTool`, `McpToolset`, `LongRunningFunctionTool`, `exit_loop`, `preload_memory`, `VertexAiSearchTool` and others. Signal: "business users edit agent definitions without code changes". Distractor: using Agent Config for a Claude- or vLLM-backed agent.

**Managed agents (ADK Python 2.4, Preview).** `ManagedAgent(agent_id=..., environment={'type':'remote'})` connects to a Google-hosted agent, such as the Antigravity agent, through the Managed Agents / Interactions API.
- Only the `global` location is available.
- Tools are **server-side only**: client function tools and MCP tools raise `NotImplementedError`.
- Only streaming is supported.
- Use it for "powerful out-of-the-box sandboxed agent, little control". Use `LlmAgent` for "fine-grained control".

**Exam signals**
- "Must always execute steps A→B→C" → `SequentialAgent` or a graph Workflow, **not** a long prompt.
- "Independent lookups, cut latency" → `ParallelAgent` with distinct `output_key`s.
- "Iterate until quality check passes" → `LoopAgent` + `max_iterations` + escalate.
- "Conditional branch on classification, some steps pure code, human approval in the middle" → **graph Workflow** with route + `RequestInput`.
- "Apply guardrail/logging to all agents" → **Plugin**.

---

#### 3.1.3 Sessions and memory

> 📊 **Infographic:** State, sessions & memory
>
> [![State, sessions & memory](../infographics/07-state-memory.png)](../infographics/07-state-memory.png)

**Concepts.**
- **Session**: one conversation thread, made up of `events` plus `state`.
- **State**: a key-value scratchpad. Values must be serializable (no clients or objects).
- **Memory**: a searchable, cross-session store.
- The `Runner` commits state through `append_event(session, event)` using the `state_delta` carried on the event. Never mutate `session.state` obtained from `get_session` directly outside a callback or tool context: the change is not persisted.

**State prefixes (high-yield)**

| Prefix | Scope | Persistence |
|---|---|---|
| *(none)* | This session only | Persisted if the SessionService is persistent |
| `user:` | All sessions of this `user_id` within this `app_name` | Persisted (Database / Vertex AI) |
| `app:` | All users and sessions of the app | Persisted |
| `temp:` | Current **invocation** only. Shared by sub-agents through the same `InvocationContext` | **Never persisted** |

Gotcha: `get_user_state()` raises `NotImplementedError` on `VertexAiSessionService`. List the sessions and call `get_session` instead.

**SessionService options**

| Service | Backend | Use when | Trade-off |
|---|---|---|---|
| `InMemorySessionService` | Process memory | Local dev, tests, `adk web` | Lost on restart. Breaks with more than one replica |
| `DatabaseSessionService(db_url=...)` | SQLAlchemy **async** driver: `postgresql+asyncpg` (Cloud SQL / AlloyDB), `aiomysql`, `sqlite+aiosqlite` | You self-host (Cloud Run/GKE) and want sessions in your own DB | You run the DB. Row-level `SELECT … FOR UPDATE` locking for multi-replica. **Schema changed v0 (pickle) → v1 (JSON) in ADK Python 1.22.** Migrate with `adk migrate session` |
| `VertexAiSessionService` → **Agent Platform Sessions** | Fully managed | Production on Agent Runtime, *or* from Cloud Run/GKE/local by pointing at an Agent Runtime instance | Managed and scalable, TTL, feeds Memory Bank directly. Regional availability |
| Firestore session service | Firestore | Java only (`google-adk-firestore-session-service`) | Not Python |
| Redis (`adk-redis`) | Redis Agent Memory / self-hosted Agent Memory Server | Low-latency sessions plus semantic caching | Community integration, not a Google product. Using **Memorystore for Redis** as a backend is (unverified) |

**Agent Platform Sessions specifics**
- It needs an **Agent Runtime instance** but **no code deployment**: `client.runtimes.create(config={"display_name": ...})` creates it in seconds.
- `VertexAiSessionService(project, location, agent_engine_id=...)`. For deployed agents the ID is in `GOOGLE_CLOUD_AGENT_ENGINE_ID`. `AdkApp` on Agent Runtime wires sessions automatically.
- **Every session expires.** You can set `ttl` or `expire_time`. The **default TTL is 365 days**. `ttl` is counted from create time, or from update time for updated sessions.
- `user_id` can be up to 128 characters.
- Custom session ID: if it starts with a letter, up to 63 characters of `[a-z0-9-]`; if it starts with a digit, up to 9 digits with no leading zeros.
- Express mode (free tier, 90 days): 10 session create/update/delete per minute and 30 `append_event` per minute. There is a cap of 10 Agent Runtime instances, and deploying agents requires paid access.

**MemoryService options (ADK Python)**

| Service | Extraction | Search | Use when |
|---|---|---|---|
| `InMemoryMemoryService` | Stores full conversation | Keyword | Prototyping |
| `VertexAiMemoryBankService` → **Memory Bank** | **LLM extracts *and consolidates* facts** per scope | Semantic similarity, scoped | Personalisation that evolves ("remember my preferences"), deduplicated and contradiction-resolved |
| `VertexAiRagMemoryService(rag_corpus=..., similarity_top_k, vector_distance_threshold)` | Stores raw transcripts in a RAG Engine corpus | Vector similarity | You already run RAG Engine or need raw transcript recall |
| `DatabaseMemoryService` (`adk-database-memory`) | Stores sessions | DB-backed | Non-GCP, on-prem or air-gapped (community) |

**Memory Bank: facts to memorise**
- **Scope** is a dict of up to 5 key-value pairs, for example `{"user_id": "123"}`. Consolidation and retrieval are **isolated per exact scope**. When generating from Sessions, the scope defaults to `{"user_id": session.user_id}`.
- **Generate:** `memories.generate` from `vertex_session_source` (optionally a time window), `direct_contents_source` (events you send, for non-Agent-Platform session stores) or `direct_memories_source` (**pre-extracted facts, at most 5 per call**, which are consolidated). `disable_consolidation` makes it extract-only. `wait_for_completion` controls sync or async, and background generation is the latency-friendly default pattern.
- **`memories.create`** skips extraction *and* consolidation, so duplicates are possible.
- **Memory topics:** the managed topics are `USER_PERSONAL_INFO`, `USER_PREFERENCES`, `KEY_CONVERSATION_DETAILS` and `EXPLICIT_INSTRUCTIONS` (all four are on by default). You can add **custom topics** (label + description), and should add few-shot `generate_memories_examples` with them.
- **Defaults:**
  - generation model `gemini-3.5-flash`;
  - similarity embedding `text-embedding-005`;
  - memory revisions kept (default revision TTL 365 days);
  - **no TTL on memories unless you configure `ttl_config`**;
  - memories written in the first person.
- **Retrieve:** `memories.retrieve(scope=..., similarity_search_params={"search_query": q, "top_k": 3})`. The default `top_k` is 3 and results are ranked by Euclidean distance. Without similarity parameters it returns every memory in the scope (paginated). `ListMemories` is not for low-latency paths.
- **Governance:** IAM Conditions restrict which principals can read or write which scopes. Memory **revisions** show how a fact evolved. Input can be multimodal, but only text and files are used for extraction; function calls are ignored.
- **ADK wiring:**
  - pass `VertexAiMemoryBankService(project, location, agent_engine_id)` to `Runner(memory_service=...)`, or run `adk web --memory_service_uri="agentengine://<ID>"`;
  - read memory with the **`preload_memory`** tool (retrieves every turn automatically) or **`load_memory`** (the model decides when);
  - write memory through `add_session_to_memory` (commonly in an `after_agent_callback`) or `add_memory` (set `custom_metadata={"enable_consolidation": True}` to consolidate).

```python
from google.adk.tools import preload_memory

async def save_to_memory(callback_context):
    await callback_context.add_session_to_memory()   # as shown in the ADK memory docs

agent = LlmAgent(name="concierge", model="gemini-flash-latest",
                 tools=[preload_memory], after_agent_callback=save_to_memory)
```

**Exam signals**
- "Conversation lost when Cloud Run scales/restarts" → replace `InMemorySessionService` with `VertexAiSessionService` or `DatabaseSessionService`.
- "Remember user preferences across conversations, avoid duplicate/contradicting facts" → **Memory Bank** (consolidation), *not* state and *not* raw RAG over transcripts.
- "Preference shared by all of a user's sessions but no long-term semantic search needed" → `user:` state.
- "Scratch value between tool calls in one turn" → `temp:`.
- "Global config/discount for every user" → `app:`.
- "GDPR retention on conversations" → session TTL / `expire_time`, plus memory `ttl_config`.
- "Must keep sessions in our own Postgres" → `DatabaseSessionService` on Cloud SQL/AlloyDB with `asyncpg`.

**Distractors:** stuffing all history into the prompt (use memory and compaction). Using `InMemory*` in production. Assuming `temp:` survives to the next turn. Using `memories.create` when you need deduplication.

---

#### 3.1.4 Configuring skills with Agents CLI

**What Agents CLI is.** "CLI and skills for building agents on Google Cloud". It is the Agent Platform toolchain that installs **Agent Skills** (open `SKILL.md` standard, loaded by progressive disclosure) into coding agents (Antigravity, Claude Code, Codex, Cursor…) *and* provides a standalone `agents-cli` binary.

- **Install:** `uvx google-agents-cli setup` (or `pip`/`pipx install google-agents-cli && agents-cli setup`; skills only: `npx skills add google/agents-cli`). Setup flags:
  - `--workspace`: project scope instead of global;
  - `--agent claude-code --agent cursor` or `all`: target coding agents;
  - `--skills-source`: local path, GitHub `owner/repo` or URL, for a private or forked skill set;
  - `--dry-run`, `--dev`, `-i`.
- `agents-cli update [--workspace] [-y]` refreshes skills. `agents-cli extension add|list|remove|update` (experimental, `--global`) adds extensions that contribute commands.
- **The seven bundled skills:**
  - `google-agents-cli-workflow` (lifecycle, model selection)
  - `-adk-code` (ADK API)
  - `-scaffold` (create/enhance/upgrade)
  - `-eval` (datasets, metrics, grade, optimize)
  - `-deploy` (Agent Runtime/Cloud Run/GKE, CI/CD)
  - `-publish` (Gemini Enterprise registration)
  - `-observability` (Cloud Trace, logging)
- **Key commands:**
  - `agents-cli create my-agent --prototype --yes`
  - `agents-cli scaffold enhance --deployment-target agent_runtime`
  - `agents-cli playground` (ADK web on :8080 with hot reload)
  - `agents-cli run "<prompt>"`
  - `agents-cli eval run`
  - `agents-cli deploy [--agent-identity]`
  - `agents-cli infra single-project` (observability/infra)
  - `agents-cli info`
- The scaffold writes `agents-cli-manifest.yaml`, `app/agent.py`, `fast_api_app.py`, `tests/eval`, a `Dockerfile` and `GEMINI.md`. The deployment target lives in `pyproject.toml`.
- **"Agent vs human mode".** The docs describe two usage modes: **agent-assisted mode**, where skills are installed into a coding agent that "uses them to make the right decisions at every step", and **manual mode**, where a human runs "CLI commands directly from your terminal. Every command works standalone." Machine-readable output is available through `--json` on commands such as `cmd-info` and `infra show`. (A literal `--mode agent|human` flag was **not found** in the docs. Treat the exam phrase as this distinction: unverified.)
- Relation to ADK's own CLI: `adk create`/`adk web`/`adk deploy` still exist. The docs point to `adk create` for learning single-file agents, and to Agents CLI when you plan to test, evaluate and deploy.

**Skills at runtime (inside your agent, not your IDE).** `SkillToolset(skills=[...], registry=GCPSkillRegistry(project_id, location))` gives the agent `search_skills` and `load_skill` tools. Skills are fetched on demand from **Skill Registry** instead of bloating the system prompt. Registry validation limits:
- ZIP of at most 10 MB, at most 500 MB unzipped, at most 10,000 files, depth of at most 8, no symlinks;
- `SKILL.md` is required with YAML `name` (at most 64 characters, lowercase/digits/hyphens) and `description` (at most 1024 characters).

Custom skills get a `private-` prefix. **Skill revisions are immutable**, and a skill has a default revision pointer. If the Agent Registry service agent (`service-PROJECT_NUMBER@gcp-sa-agentregistry.iam.gserviceaccount.com`) imports a skill from GCS, it needs `storage.objects.get`.

**Exam signals**
- "Teach Antigravity/Claude Code how to scaffold, eval and deploy ADK agents the Google way" → **Agents CLI setup** (skills).
- "Share skills with the whole repo team" → `--workspace` and commit `.agents/skills`.
- "Hundreds of skills, keep context small, govern versions centrally" → **Skill Registry + SkillToolset**.
- "Register agent in Gemini Enterprise" → the publish skill.

---

### 3.2 Integrating enterprise domain knowledge

#### 3.2.1 RAG pipelines and vector retrieval

> 📊 **Infographic:** RAG pipeline & backend chooser
>
> [![RAG pipeline & backend chooser](../infographics/08-rag-pipeline.png)](../infographics/08-rag-pipeline.png)

**Pipeline stages:** ingest → parse → chunk → embed → index → retrieve (dense / sparse / hybrid + metadata filter) → **rerank** → ground the generation → (optionally) check grounding.

**Parsing and chunking**
- **RAG Engine:** `ChunkingConfig(chunk_size=1024, chunk_overlap=256)` are the **defaults (tokens)**. Smaller chunks give precise embeddings but lose context; larger chunks give general embeddings but miss detail. Parser options are the default parser, the **Document AI layout parser**, or an **LLM parser** (`LlmParserConfig(model_name=..., max_parsing_requests_per_min=...)`). Re-importing a file is skipped if the file *and* the chunking config are unchanged. `max_embedding_requests_per_min` defaults to 1,000 QPM per import job.
- **Agent Search:** turn on *layout-aware chunking* (`documentProcessingConfig.chunkingConfig.layoutBasedChunkingConfig`) plus layout parsing **at data store creation**. It **cannot be toggled later**. You can also bring your own chunks (Preview).

**Embedding models**
- `gemini-embedding-001`: 3072 dimensions by default, reducible with `output_dimensionality`. One input text per request.
- `text-embedding-005`: 768 dimensions. It is the **default for RAG Engine** and for Memory Bank similarity.
- `text-multilingual-embedding-002`.
- Open-source e5 variants through Model Garden.
- **Task types:** `RETRIEVAL_DOCUMENT` for the corpus and `RETRIEVAL_QUERY` (the default) for queries. Asymmetric pairs also exist: `QUESTION_ANSWERING`, `FACT_VERIFICATION` and `CODE_RETRIEVAL_QUERY` for the query side, each paired with `RETRIEVAL_DOCUMENT`. `SEMANTIC_SIMILARITY` is **not** for retrieval.
- **Gotcha:** the embedding model of a RAG corpus is **immutable** for the corpus's lifetime. Query and document vectors must come from the same model and dimension.

**Similarity scoring**
- Vector Search 1.0 distances: `DOT_PRODUCT_DISTANCE` (default), `COSINE_DISTANCE`, `SQUARED_L2_DISTANCE`, `L1_DISTANCE`. Google recommends **`DOT_PRODUCT_DISTANCE` + `UNIT_L2_NORM`** over COSINE: the ranking is identical and it is better optimised. Sparse indexes support dot product only.
- RagManagedDb uses cosine.
- BigQuery `VECTOR_SEARCH(..., distance_type => 'COSINE'|'EUCLIDEAN'|'DOT_PRODUCT')`.
- Memory Bank ranks by Euclidean distance.

**Hybrid search**
- **Vector Search 1.0:** dense and sparse embeddings (TF-IDF/BM25/SPLADE) in one index, merged with **Reciprocal Rank Fusion**. With `rrf_ranking_alpha`, **1 (or unset) = dense only, 0 = sparse only, 0.5 = equal weight**.
- **RAG Engine:** `hybrid_search.alpha` (0 = sparse, 1 = dense, default 0.5). The docs say it is **only available with the Weaviate backend**.
- **AlloyDB:** `ai.hybrid_search()` fuses vector (ScaNN) and full-text (GIN/RUM `tsvector`) results with RRF.
- **BigQuery:** `VECTOR_SEARCH` single-query syntax with `lexical_search_columns` (Preview), or `AI.SEARCH` on tables with autonomous embeddings.
- **Agent Retrieval:** a batch search can fuse semantic and text search with RRF.

**Reranking**
- **Ranking API** (Discovery Engine, `rankingConfigs/default_ranking_config:rank`, location `global`):
  - models `semantic-ranker-default@latest` and `semantic-ranker-fast@latest`; the 004 generation takes 1024 tokens per record, 003 and earlier take 512. The 005 models are in Preview from September 2026, and `@latest` moves to 005 by 1 October 2026;
  - **at most 1000 records per request**; `topN` limits what is returned (all records are still scored); returns a 0–1 relevance score;
  - AlloyDB exposes it in SQL as `ai.rank()`.
- **RAG Engine:** `ranking.rank_service.model_name` (for example `semantic-ranker-512@latest`) **or** `ranking.llm_ranker.model_name` (a Gemini model). The LLM ranker is more flexible but slower and more expensive.
- **Agent Retrieval `VertexRanker`** is *best-effort*:
  - if ranking fails, the search still succeeds with RRF-fused results and a warning (`UNAVAILABLE`, `RESOURCE_EXHAUSTED`, `DEADLINE_EXCEEDED`);
  - but the **whole RPC fails with `FAILED_PRECONDITION` if the Discovery Engine API isn't enabled** on the consumer project.

**Vector store decision table**

| Option | What it is | Pick it when | Watch-outs |
|---|---|---|---|
| **Agent Search** (ex Vertex AI Search) | Google-managed search engine: connectors, parsing, chunking, ranking, grounding, ACL-aware enterprise search; shared with Gemini Enterprise | "Fastest path to high-quality search over enterprise docs/websites with minimal ML work". ADK `VertexAiSearchTool`; MCP `discoveryengine.googleapis.com/mcp` | Less control of embeddings and chunking. Chunking choice fixed at data store creation |
| **RAG Engine** | Managed RAG *framework*: corpus, import from GCS/Drive/Slack/Jira/SharePoint, transformations, embeddings, pluggable vector DB, retrieval config, Gemini `Retrieval(vertex_rag_store=...)` tool | "DIY-but-managed RAG with control over chunking, embedding model, reranker, backend" | Embedding model fixed per corpus. Hybrid only on Weaviate. Allowlist needed for new projects in some US regions. Data residency not supported |
| **Vector Search 1.0** | ANN index-as-a-service (ScaNN tree-AH or brute force), deployed to index endpoints | Billions of vectors, lowest-latency ANN, hybrid dense+sparse, custom recsys | You manage index, endpoint and replicas. **Batch vs streaming choice is permanent** per index. Streaming updates cost $0.45/GB plus compaction. You need a separate store for the payload |
| **Agent Retrieval** (Vector Search 2.0) | Collections of JSON Data Objects with a schema, auto-embedding, KNN/ANN, filtering, RRF + VertexRanker; "single unified data source" (stores payload too) | New builds wanting a self-tuning managed store without VM/replica sizing; RAG Engine serverless default backend | 9 regions. RAG-Engine-backed VS 2.0 corpora are **us-central1 only**. No CMEK via RAG Engine |
| **AlloyDB AI** (pgvector + **ScaNN**, HNSW) | Vectors next to operational rows; `embedding()` generated columns; `ai.hybrid_search`, `ai.rank` | Transactional data plus semantic search with SQL filters and joins, one DB to operate | ScaNN suits very large or low-dimension sets (up to about 10B vectors); HNSW suits sets that fit in memory (about 10–20M) |
| **Cloud SQL for PostgreSQL** (pgvector) | pgvector HNSW/IVFFlat | Small or medium corpora already on Cloud SQL | Scale and performance ceiling below AlloyDB ScaNN (unverified detail) |
| **BigQuery** (`VECTOR_SEARCH`, `CREATE VECTOR INDEX` IVF / TreeAH, `ML.GENERATE_EMBEDDING`, `AI.SEARCH`) | Analytics-native vector search | Data already in BQ, batch similarity joins, analytics plus RAG | Not a low-latency online serving store. TreeAH suits large query batches (tables of 200M rows or fewer); IVF suits small batches. Brute force with `use_brute_force` |

**RAG Engine deployment modes and backends**
- **Serverless mode** is recommended. It provisions a **Vector Search 2.0 collection in your project** (visible, with cost transparency). **No CMEK.**
- **Spanner mode** is the **default** if you don't choose. It uses RagManagedDb on dedicated Spanner with a **Basic** (default, fixed, cheap) or **Scaled** (autoscaling) tier, and is **the CMEK option**. `Unprovisioned` deletes it.
- Backends: `RagManagedDb` (KNN default; switch to ANN when you have more than about 10K files; the ANN index must be rebuilt after large changes), `RagManagedVertexVectorSearch` (VS 2.0), `VertexVectorSearch` (VS 1.0, which you manage), Vertex AI Feature Store (Preview), Weaviate (Preview), Pinecone.
- RAG Engine supports VPC-SC and CMEK. Data residency and AXT are **not** supported.

```python
from vertexai import rag
cfg = rag.RagRetrievalConfig(
    top_k=10,
    filter=rag.Filter(vector_distance_threshold=0.5, metadata_filter='dept == "legal"'),
    ranking=rag.Ranking(rank_service=rag.RankService(model_name="semantic-ranker-default@latest")),
)
```
(The SDK class names follow the RAG API fields `RagRetrievalConfig.filter/ranking/hybrid_search`. The exact Python constructor shape may differ by SDK version: unverified.)

**Exam signals**
- "Keyword-heavy queries (part numbers, SKUs) miss results" → **hybrid search** (sparse + dense, RRF) and/or reranking.
- "Relevant chunks retrieved but ordered badly" → **Ranking API / rank_service**.
- "Tables and headings in PDFs get split mid-structure" → **layout parser** / layout-aware chunking.
- "CMEK required for RAG index" → **RAG Engine Spanner mode (RagManagedDb)**.
- "Transactional product catalog + semantic search in one query with filters" → **AlloyDB ScaNN**.
- "Data already in BigQuery, analysts run batch similarity" → **BigQuery VECTOR_SEARCH**.
- "Minimal effort enterprise doc search with connectors" → **Agent Search**.
- "Real-time index updates within seconds" → **streaming update** index (decided at creation).

**Distractors:** using COSINE instead of DOT_PRODUCT + UNIT_L2_NORM in VS 1.0. Expecting hybrid search on the RagManagedDb backend. Changing the embedding model on an existing corpus. Using `SEMANTIC_SIMILARITY` for retrieval. Fine-tuning instead of RAG for fast-changing facts.

---

#### 3.2.2 Agent permissions: Agent Identity

> 📊 **Infographic:** Protocols, registry & agent identity
>
> [![Protocols, registry & agent identity](../infographics/09-protocols-identity-registry.png)](../infographics/09-protocols-identity-registry.png)

- **What it is.** Each agent gets a **SPIFFE-based, per-agent principal** tied to its hosting resource and lifecycle. It is not shared like a service account.
  - SPIFFE ID: `spiffe://TRUST_DOMAIN/resources/SERVICE/RESOURCE_PATH`
  - IAM principal: `principal://agents.global.org-ORG_ID.system.id.goog/resources/aiplatform/projects/PROJECT_NUMBER/locations/REGION/reasoningEngines/AGENT_ID`
  - Without an organization the trust domain is `agents.global.proj-PROJECT_NUMBER.system.id.goog`. Gemini Enterprise agents use `.../discoveryengine/...`.
- **Credentials.** An auto-provisioned X.509 certificate with **mTLS** to Google APIs, plus **DPoP** across Agent Gateway ("double-bound"). A Google-managed **Context-Aware Access** policy makes tokens un-replayable outside the runtime. Using one elsewhere gives `401 Context-Aware Access requirements are not met`. Opting out through `GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES=False` is strongly discouraged.
- **Default roles on an agent identity:** `roles/aiplatform.agentDefaultAccess` (logging and model calls) and `roles/aiplatform.agentContextEditor` (only **its own** sessions, memories and sandboxes).
- **Opt-in.** Without the identity flag, Agent Runtime keeps using the **AI Platform Reasoning Engine Service Agent** (`service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com`, `roles/aiplatform.reasoningEngineServiceAgent`) or a custom service account. Both are shared across agents. Ways to enable it:
  - `client.runtimes.create(config={"identity_type": types.IdentityType.AGENT_IDENTITY, ...})` (v1beta1);
  - `agents-cli deploy --agent-identity`;
  - `.agent_engine_config.json` `{"identity_type": "AGENT_IDENTITY"}` before `adk deploy`;
  - on Cloud Run: `gcloud run deploy ... --identity-type=agent-identity --functional-type=agent`.
- **Grant before deploy.** Create the Runtime instance with *only* `identity_type` to get the principal, bind IAM, then `runtimes.update(...)` with code.
- **Principal sets.**
  - All Agent Runtime agents in a project: `principalSet://agents.global.org-ORG_ID.system.id.goog/attribute.platformContainer/aiplatform/projects/PROJECT_NUMBER`
  - Whole org: `principalSet://agents.global.org-ORG_ID.system.id.goog/attribute.platform/aiplatform` (the Agent Registry docs also show `principalSet://agents.global.org-ORG_ID.system.id.goog/*`)
  - Pattern: broad baseline roles to the principal set (`serviceusage.serviceUsageConsumer`, `logging.logWriter`, `monitoring.metricWriter`, `aiplatform.expressUser`, `cloudapiregistry.viewer`) and **narrow data roles to the individual agent principal**.
- **Deny and boundaries.** IAM **deny policies** on agent principal sets, and **Principal Access Boundary (PAB)** policies bound to the org, limit which resources agents can *ever* reach regardless of allow policies. VPC-SC supports agent identities in ingress and egress rules. Legacy bucket roles can't be granted to agent identities.
- **Lifecycle gotchas (very testable).**
  - Deleting an agent **does not remove** its IAM bindings (stale grants remain), so clean them up.
  - Redeploying as a *new* resource, even with the same name, creates a **new principal**, so the old bindings don't apply. Update the existing resource instead of delete-and-recreate.
- **Outbound to third parties: Agent Identity auth manager.** A credential vault and broker holding "auth providers": API key, OAuth client credentials, or **user-delegated OAuth** with consent flow and revocation. The agent authenticates to it with its SPIFFE ID, and every end-user access is attributed to the agent's identity in audit logs.
- **Signals**
  - "Least privilege per agent, auditable, not shared SA" → **Agent Identity**.
  - "All agents in project need logging/model access" → **principalSet** grant.
  - "Guarantee agents can never touch resources outside folder X" → **PAB**.
  - "Agent calls Salesforce on behalf of user" → **auth manager** OAuth delegation.

#### 3.2.3 Agent Registry and Google Cloud MCP servers

**Agent Registry** is the governance catalog of **Agents, MCP servers, Endpoints and Skills** (resources: `Agent`, `McpServer`, `Endpoint`, `Skill`, `SkillRevision`, `Publisher`).
- **Read vs write split:** you discover through the read-only `Agent`/`McpServer`/`Endpoint` resources, and you register or modify through the writable **`Service`** resource (`gcloud agent-registry services delete ...`).
- **Automatic registration** covers supported resources such as Agent Runtime agents and Google Cloud remote MCP servers. It is **single-project scope**. Use **manual registration** for external or custom components and for cross-project central catalogs. To remove an auto-registered MCP server, delete the server or disable its API.
- **Identifiers:** URNs (`urn:agent:...`, `urn:mcp:googleapis.com:projects:N:locations:global:SERVER`, `urn:skill:...`) are for **inventory and lookup only**. Policies use the **agent principal**, not the URN.
- **Bindings** connect a source agent to a target agent, MCP server or endpoint, or to an **auth provider** (delegated access).
- The registry supports A2A specification versions **0.3 and 1.0**.
- Cross-project egress through a central Agent Gateway needs `roles/iap.egressor` for the agent principal on the target.
- **Registry MCP server:** `agentregistry.googleapis.com/mcp` with `search_agents`, `search_mcp_servers`, `get_*`, `list_*`, `create_service`, and `create_binding`. `tools/list` needs no auth.
- **ADK:** `pip install "google-adk[a2a,agent-identity]"`.

```python
from google.adk.integrations.agent_registry import AgentRegistry
reg = AgentRegistry(project_id=PROJECT, location="global")
tools = reg.get_mcp_toolset(mcp_server_name=f"projects/{PROJECT}/locations/global/mcpServers/crm")
remote = reg.get_remote_a2a_agent(agent_name=f"projects/{PROJECT}/locations/global/agents/billing")
root_agent = LlmAgent(name="orchestrator", model="gemini-flash-latest",
                      tools=[tools], sub_agents=[remote])
```

**Google Cloud (remote) MCP servers.** These are Google-hosted, Streamable-HTTP MCP endpoints per product. Examples: BigQuery `https://bigquery.googleapis.com/mcp`, AlloyDB, Cloud SQL (`sqladmin`), Spanner, Firestore, Bigtable, **Memorystore** (`redis.googleapis.com/mcp`), Cloud Run, GKE, Cloud Storage, Pub/Sub, Logging, Monitoring, Trace, IAM, Agent Registry, Agent Search (`discoveryengine`), Apigee API hub, Agent Platform toolsets (`REGION-aiplatform.googleapis.com/mcp/{generate,retrieval,evaluation,...}`), and Developer Knowledge. Workspace (Drive, Gmail, Calendar, Chat) is in Developer Preview.
- **Enable** the product's MCP server in the project. **Callers need `roles/mcp.toolUser` (`mcp.tools.call`) plus the product's own data roles.**
- Auth is OAuth 2.0 + IAM: ADC, an OAuth client ID and secret, or a bearer token. **Dynamic Client Registration is not supported.** API keys are accepted only by some servers (for example Monitoring).
- Google recommends a **separate identity for agents** using MCP tools.
- ADK access: `ApiRegistry(api_registry_project_id=..., header_provider=...)` with `.get_toolset(mcp_server_name=...)` (needs `cloudapiregistry` + `apihub` APIs and `apiregistry.viewer`; `gcloud beta api-registry mcp enable bigquery`), or `McpToolset(connection_params=StreamableHTTPConnectionParams(url=...))`.

**Custom integration layers: when to use which**

| Need | Tool |
|---|---|
| Governed SQL/NoSQL access with your own curated, parameterised queries; connection pooling; auth; many DBs (AlloyDB, Cloud SQL, Spanner, BigQuery, Firestore, Bigtable, Postgres, MySQL, Mongo, Redis, Neo4j, Snowflake…) | **MCP Toolbox for Databases** (open source, self-hosted; `tools.yaml` or `--prebuilt alloydb-postgres`; ADK built-in support) |
| Generic admin or ad-hoc access to a Google DB/service with zero hosting | **Google Cloud remote MCP server** (for example the AlloyDB or Cloud SQL MCP) |
| SaaS/ERP (Salesforce, SAP, ServiceNow, Jira) or reuse of existing integration flows | **Application Integration / Integration Connectors** → `ApplicationIntegrationToolset` |
| Existing REST APIs managed in Apigee, with quota/security/analytics | **Apigee API hub** → `APIHubToolset`, or expose Apigee proxies as MCP; Apigee API hub MCP server |
| Any REST API with an OpenAPI spec | `OpenAPIToolset` |
| Third-party SaaS that ships an MCP server (Atlassian, Asana…) | `McpToolset` to its remote server, registered in Agent Registry and governed through Agent Gateway |

**Signals**
- "Agents should discover approved tools at runtime rather than hard-code URLs" → **Agent Registry**.
- "Curated, least-privilege DB queries as tools" → **MCP Toolbox** (not raw `execute_sql`).
- "No infra to host, just let agent query BigQuery" → **BigQuery remote MCP server** + `mcp.toolUser`.
- "Salesforce/SAP" → **Application Integration connectors**.

---

### 3.3 Orchestrating and coordinating agentic workflows

#### 3.3.1 Agentic protocols: MCP vs A2A

| | **MCP** | **A2A (Agent2Agent)** |
|---|---|---|
| Connects | Agent ↔ **tools/data/resources** | Agent ↔ **agent** (opaque, autonomous peer) |
| Unit | Tool call (function + schema), resources, prompts | **Task** with lifecycle, messages, **artifacts**; streaming and long-running |
| Discovery | `tools/list`; Agent Registry `McpServer` | **Agent Card** (name, description, url, version, capabilities, `skills[]`, input/output modes, `securitySchemes`) at `/.well-known/agent-card.json` |
| Transports | stdio, Streamable HTTP (SSE legacy) | JSON-RPC 2.0, gRPC, HTTP+JSON (`preferredTransport`) |
| Use when | Deterministic capability, your agent keeps control of reasoning | Capability owned by another team, framework or language; needs its own reasoning and state; formal contract across org boundaries |
| ADK | `McpToolset`; also expose ADK tools *as* an MCP server | `to_a2a(root_agent)` (auto-generates the card) or `adk api_server --a2a` to **expose**; `RemoteA2aAgent(name, agent_card=URL)` to **consume** |

**ADK guidance: prefer *local* sub-agents over A2A when** you only need code organisation, the path is latency-critical, agents need shared in-memory state, or the logic is a simple helper. Serialisation and network overhead make A2A counterproductive there.

**A2A on Google Cloud**
- **Agent Runtime** hosts A2A agents (`A2aAgent`, `AgentCard`, `AgentExecutor`). The operations are `on_message_send`, `on_get_task`, `on_cancel_task` and `handle_authenticated_agent_card` (Preview).
- **Cloud Run:** `gcloud run deploy ... --no-allow-unauthenticated --functional-type=agent --identity-type=agent-identity`. Callers need `run.invoker`. For a public agent, declare `securitySchemes` in the card.
  - The **TaskStore** is in-memory by default, so tasks are lost on scale-down. Use **AlloyDB** for persistent tasks and horizontal scaling.
- Test with **a2a-inspector**.
- ADK's A2A extension (`RemoteA2aAgent(use_legacy=False)`, `X-A2A-Extensions` header) preserves reasoning traces, long-running tools and artifacts across agents.

**Signals**
- "Different team's Java agent must be called from our Python ADK orchestrator" → **A2A**.
- "Give agent access to a database/API" → **MCP**.
- "Remote agent's tasks lost when Cloud Run scales in" → persistent **AlloyDB TaskStore**.
- "Discover partner agent capabilities" → **Agent Card** / Agent Registry `search_agents` by A2A skill.

#### 3.3.2 Multi-agent handoffs and workflows

> 📊 **Infographic:** ADK orchestration patterns
>
> [![ADK orchestration patterns](../infographics/06-adk-orchestration-patterns.png)](../infographics/06-adk-orchestration-patterns.png)

**Coordination primitive decision table**

| Primitive | Control | Context/state | Best for | Trade-off |
|---|---|---|---|---|
| `SequentialAgent` | Fixed order, code-driven | Shared state (`output_key`) | Pipelines: validate → process → report | Rigid |
| `ParallelAgent` | Concurrent | Separate branches, **shared state** (distinct keys) | Independent lookups, fan-out research | Race conditions on shared keys. Cost multiplies |
| `LoopAgent` | Repeat until `escalate` / `max_iterations` | Same context each iteration | Generator-critic, refinement, polling | Runaway loops without a cap |
| **Graph `Workflow`** (ADK 2.0) | Explicit nodes, edges, routes, `JoinNode`, `RequestInput` | Typed node input/output | Deterministic business processes mixing code, LLM, tools, HITL | More upfront design. Some integrations incompatible |
| **LLM-driven transfer** (`sub_agents`, `transfer_to_agent`) | LLM decides; control *moves* to the target | Target takes over the conversation | Open-ended routing: coordinator / dispatcher | Non-deterministic. Depends on good `description`s. `chat` mode needs manual transfer back |
| **`AgentTool`** | Parent LLM calls the child like a function; **parent keeps control** | Child's final response (and state/artifact deltas) returned as the tool result | Specialist "consultant" calls; wrapping built-in tools; hierarchical decomposition | Extra LLM hop. Child can't converse with the user |
| Collaboration **modes** (ADK 2.0: `chat` / `task` / `single_turn` on sub-agents) | `task` returns automatically via `finish_task`; `single_turn` returns immediately and **can run in parallel** | `task`/`single_turn` run in isolated session branches | Coordinators delegating bounded jobs | `task` agents must be leaf agents. Don't set `mode` on the root |
| `RemoteA2aAgent` | Like a sub-agent, but over the network | Remote agent owns its state | Cross-team or cross-framework delegation | Latency, auth, contract versioning |

Rule of thumb from the ADK docs: prompt-based multi-step procedures get *less reliable as instructions grow*. Move the fixed skeleton into workflow or graph agents and keep LLM judgement only at genuine decision points.

**Where the Google Cloud governance pieces fit into orchestration**
- **Agent Runtime** hosts the agents, with managed Sessions, Memory Bank, Code Execution sandbox, per-agent identity and auto-registration.
- **Agent Registry** lets the orchestrator discover the sub-agents (A2A) and toolsets (MCP) it binds to at runtime.
- **Agent Identity** gives each hop its own principal, so the orchestrator's permissions ≠ the sub-agent's.
- **Agent policies** are enforced at **Agent Gateway**:
  - **IAM allow/deny policies**, enforced through IAP, decide which agents, MCP servers and endpoints each agent principal may call (egress).
  - **Semantic governance policies** are **Natural Language Constraints**, evaluated by an LLM policy engine (PDP) on each *proposed* tool call against the user prompt, chat history and tool manifest. The verdict is ALLOW or DENY with a rationale. They can be agent-wide or tool-specific, and can reference parameters: "refund amount ≤ $80".
  - Semantic governance **complements**, never replaces, IAM and Model Armor. Policies change without redeploying code. Denial rationales may reach end users, so keep secrets out of the constraints.
  - The policies page does **not** support VPC-SC.

**Signals**
- "Coordinator should hand the conversation to billing specialist who then talks to user" → `sub_agents` transfer.
- "Coordinator must stay in control and combine specialist outputs" → `AgentTool`.
- "Specialist asks clarifying questions then returns automatically" → `mode="task"`.
- "Run three specialists concurrently from an LLM coordinator" → `single_turn` sub-agents, or a `ParallelAgent`.
- "Block agent from issuing refunds above policy even though IAM allows the tool" → **semantic governance policy**.
- "Restrict which MCP servers each agent can reach" → **IAM allow policy on Agent Gateway**.

**Common distractors**
- Using A2A between modules of one codebase.
- Using `ParallelAgent` for dependent steps.
- A `LoopAgent` without `max_iterations`.
- Relying on the LLM coordinator for a compliance-mandated order of steps.
- Granting the orchestrator a union of every sub-agent's permissions.
- Using registry URNs in IAM bindings.

---

### Practice questions

**1.** A retail company runs an ADK customer-service agent on Cloud Run with 3–20 instances. Users report the agent "forgets" what they said two messages earlier, intermittently. The code uses `Runner(session_service=InMemorySessionService())`. What is the best fix with the least operational overhead?
A. Enable Cloud Run session affinity
B. Switch to `VertexAiSessionService` pointing at an Agent Runtime instance
C. Store the conversation in `temp:` state
D. Increase the model context window

**Answer: B.** In-memory sessions are per-instance and lost on scale events. Agent Platform Sessions is managed and needs only an Agent Runtime instance (no code deploy). Affinity is best-effort and still loses data on restart. `temp:` is never persisted.

**2.** A travel agent must remember across months that a user prefers aisle seats. It must update that fact when the user later says "actually, window seats now", without keeping contradictory facts. Which approach fits?
A. Write `user:seat_pref` state from a tool
B. `VertexAiRagMemoryService` over transcripts
C. Memory Bank via `VertexAiMemoryBankService`, generating memories from sessions
D. `memories.create` for every utterance

**Answer: C.** Memory Bank extracts and **consolidates**, resolving contradictions per scope. `user:` state works for one explicit key but gives no semantic recall or extraction. RAG memory returns raw transcripts, contradictions included. `memories.create` skips consolidation.

**3.** A bank must host an open-weight model with its own fine-tuned weights. Traffic is predictable and high volume, and regulators prohibit multi-tenant inference services. The team wants to choose GPU types. Which serving option is best?
A. Model Garden MaaS
B. Self-deployed Model Garden endpoint with custom weights on a dedicated endpoint
C. Gemini Flash-Lite with Provisioned Throughput
D. LiteLLM pointing at a public API

**Answer: B.** Self-deployment fits custom weights, single-tenant or VPC data paths, hardware choice and lower TCO at steady volume. MaaS is serverless and multi-tenant, with no custom weights.

**4.** Your ADK agent calls a Gemma model served by vLLM on GKE through `LiteLlm(model="openai/...", api_base=...)`. It answers in text but never invokes its tools. What is the most likely cause?
A. Gemma cannot run on GKE
B. The vLLM server wasn't started with tool calling enabled (`--enable-auto-tool-choice` and a tool-call parser)
C. LiteLLM doesn't support tools
D. `output_key` is missing

**Answer: B.** The ADK docs call out enabling OpenAI-compatible tool calling on the serving side. Without it, the model returns plain text.

**5.** An underwriting process must (1) extract fields with an LLM, (2) run a deterministic Python risk score, (3) route to "auto-approve" or "manual review" by score, and (4) pause for a human underwriter on manual review. Auditors need predictable paths. What should you build?
A. One `LlmAgent` with a detailed instruction
B. A coordinator `LlmAgent` with sub-agents using `transfer_to_agent`
C. An ADK graph `Workflow` with a function node, a router `Event(route=...)`, and a `RequestInput` node
D. A `LoopAgent` with `max_iterations=4`

**Answer: C.** Graph workflows mix code and LLM nodes, give explicit routing, and support deterministic HITL through `RequestInput`. LLM-driven transfer is non-deterministic.

**6.** A research agent needs weather, news and stock data, which are independent calls, and then one summary. Latency matters. Which composition is correct?
A. `SequentialAgent([weather, news, stocks, summarizer])`
B. `SequentialAgent([ParallelAgent([weather, news, stocks]), summarizer])`, each fetcher with a distinct `output_key`
C. `ParallelAgent([weather, news, stocks, summarizer])`
D. `LoopAgent([weather, news, stocks])`

**Answer: B.** Fan-out/gather: the parallel children share state, so each needs a distinct key, and the summariser must run after all three. Option C runs the summariser concurrently with the fetchers.

**7.** An orchestrator must consult a "tax specialist" agent, combine its answer with other findings, and keep talking to the user itself. Which mechanism fits?
A. Add the specialist to `sub_agents`
B. Wrap the specialist in `AgentTool` and add it to `tools`
C. Expose the specialist through A2A even though it's in the same codebase
D. Put the specialist in a `ParallelAgent`

**Answer: B.** `AgentTool` keeps control with the parent and returns the child's result as a tool output. Transfer through `sub_agents` hands the conversation to the specialist. A2A adds needless network overhead for in-process code.

**8.** Security requires that each deployed agent have its own auditable principal, not a shared service account, and that IAM grants exist before the code ships. What should you do?
A. Create one custom service account per agent and set it at deploy
B. Create the Agent Runtime instance with only `identity_type=AGENT_IDENTITY`, grant roles to its `principal://…/reasoningEngines/ID`, then `runtimes.update` with code
C. Use the Reasoning Engine Service Agent and add roles
D. Grant roles to `allUsers` temporarily

**Answer: B.** Agent Identity is per-agent, SPIFFE-based and lifecycle-bound. The docs show creating the identity-only instance first so that IAM can be set before deployment. The service agent is shared across agents.

**9.** After a platform team deleted and redeployed an Agent Runtime agent with the same display name and code, it gets `PERMISSION_DENIED` on a BigQuery dataset it could previously read. Why?
A. Context-Aware Access blocked it
B. The new resource has a new resource ID and therefore a new principal; the old bindings reference the deleted identity
C. BigQuery doesn't support agent identities
D. The display name must be unique

**Answer: B.** An agent identity derives from the resource ID. Old bindings stay as inactive grants and must be re-created for the new principal, and cleaned up.

**10.** Every Agent Runtime agent in a project needs log writing and metric writing. Only the "claims" agent may read the claims dataset. What is the most maintainable IAM design?
A. Grant `logging.logWriter`, `monitoring.metricWriter` and dataset access to the project principal set
B. Grant logging/metrics roles to `principalSet://…/attribute.platformContainer/aiplatform/projects/NUM`, and dataset read only to the claims agent's `principal://` identifier
C. Grant everything to each agent individually
D. Use a shared custom service account

**Answer: B.** The docs recommend broad baseline roles on the principal set and sensitive data roles on the individual agent.

**11.** Engineers search a technical knowledge base by exact part numbers (for example "XR-7741-B"). Semantic-only retrieval in Vector Search 1.0 misses them, but conceptual queries work well. What should they implement?
A. Switch to COSINE distance
B. Hybrid index with dense and sparse (BM25) embeddings, merged by RRF with `rrf_ranking_alpha` around 0.5
C. Increase `chunk_size`
D. Use the `SEMANTIC_SIMILARITY` task type

**Answer: B.** Sparse embeddings capture exact tokens and RRF fuses them with semantic results. An alpha of 1 would mean dense only.

**12.** A RAG Engine corpus must use customer-managed encryption keys. The team also wants the managed vector store without operating any database. Which configuration?
A. Serverless mode (Vector Search 2.0 backend)
B. Spanner mode with RagManagedDb
C. Weaviate backend
D. Vector Search 1.0 backend managed by the team

**Answer: B.** The docs state that CMEK is supported by RagManagedDb (Spanner mode). Serverless mode and the Vector Search backends don't support CMEK through RAG Engine.

**13.** An ADK agent must let analysts query BigQuery with no extra infrastructure to host, using Google-managed MCP. The agent runs with its own identity. Which permissions does that identity need at minimum?
A. `roles/mcp.toolUser` plus BigQuery data and job roles on the relevant resources
B. `roles/owner`
C. Only `roles/bigquery.dataViewer`
D. An API key for the MCP endpoint

**Answer: A.** Remote MCP servers need `mcp.tools.call` (MCP Tool User) *and* the product's own permissions. BigQuery's MCP uses OAuth and IAM, not API keys.

**14.** A company has 400 internal skills (SKILL.md packages). Loading them all into every agent's prompt is slow and costly. They also want versioned, centrally governed skills. What should the ADK agent use?
A. Put all skills in the instruction
B. `SkillToolset` backed by `GCPSkillRegistry`, so the agent calls `search_skills`/`load_skill` on demand
C. One `AgentTool` per skill
D. Agent Config YAML

**Answer: B.** Skill Registry gives on-demand, targeted retrieval with immutable revisions and a default revision, and keeps the context window small.

**15.** A refund agent is IAM-authorised to call `issue_refund`. Policy says refunds over $100 need a manager, and the rule changes quarterly. Prompt-injection through customer emails is a concern. What is the best control?
A. Add the rule to the system instruction
B. A semantic governance policy (natural language constraint) on the `issue_refund` tool, enforced at Agent Gateway
C. Remove the tool's IAM permission
D. Lower the model temperature

**Answer: B.** Semantic governance evaluates each proposed tool call against user intent and constraints, can reference parameters, resists context poisoning, and changes without redeploying. Instructions can be overridden by injected text, and removing IAM access breaks legitimate refunds.

---

### Key doc links

- Agent Platform overview: https://docs.cloud.google.com/gemini-enterprise-agent-platform/overview
- Google models (Gemini line-up): https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/google-models
- Choose open-model serving option (MaaS vs self-deploy vs vLLM): https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/open-models/choose-serving-option
- Self-deployed models: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models
- Deploy open models from Model Garden: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/open-models/deploy-model-garden
- ADK models (LiteLLM, vLLM, Gemma, Apigee, routing): https://adk.dev/agents/models/ · https://adk.dev/agents/models/litellm/ · https://adk.dev/agents/models/vllm/ · https://adk.dev/agents/models/routing/ · https://adk.dev/agents/models/apigee/
- ADK graph workflows / routes / human input: https://adk.dev/graphs/ · https://adk.dev/graphs/routes/
- ADK workflow agents: https://adk.dev/agents/workflow-agents/
- ADK collaborative teams (modes): https://adk.dev/workflows/collaboration/
- ADK Agent Config: https://adk.dev/agents/config/ (syntax: https://adk.dev/api-reference/agentconfig/)
- ADK callbacks / plugins: https://adk.dev/callbacks/ · https://adk.dev/plugins/
- ADK sessions / state / memory: https://adk.dev/sessions/session/ · https://adk.dev/sessions/state/ · https://adk.dev/sessions/memory/
- ADK tool limitations: https://adk.dev/tools/limitations/
- ADK MCP tools: https://adk.dev/tools-custom/mcp-tools/
- ADK A2A intro/quickstarts: https://adk.dev/a2a/ · https://adk.dev/a2a/quickstart-exposing/ · https://adk.dev/a2a/quickstart-consuming/
- ADK Agent Registry / Skill Registry / API Registry / Agent Search / Application Integration / MCP Toolbox: https://adk.dev/integrations/ (pages: agent-registry, skill-registry, api-registry, agent-search, application-integration, mcp-toolbox-for-databases)
- ADK express mode: https://adk.dev/integrations/express-mode/
- Agents CLI: https://google.github.io/agents-cli/ · https://google.github.io/agents-cli/cli/ · https://google.github.io/agents-cli/guide/getting-started/ · https://adk.dev/get-started/agents-cli/
- Agent Platform Sessions: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sessions · https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sessions/manage-with-adk
- Memory Bank: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank · …/memory-bank/setup · …/memory-bank/generate-memories · …/memory-bank/fetch-memories · …/memory-bank/api-quickstart
- Agent Identity (Agent Runtime): https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity
- Agent Identity overview (IAM): https://docs.cloud.google.com/iam/docs/agent-identity-overview
- Agent Runtime identity setup / access: https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/runtime/setup · https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/manage-agent-access
- Agent Registry: https://docs.cloud.google.com/agent-registry/overview · https://docs.cloud.google.com/agent-registry/concepts · https://docs.cloud.google.com/agent-registry/manage-mcp-tools · https://docs.cloud.google.com/agent-registry/use-agentregistry-mcp · https://docs.cloud.google.com/agent-registry/register-skills · https://docs.cloud.google.com/agent-registry/manage-skill-revisions
- Skill Registry: https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/skill-registry
- Google Cloud MCP servers (supported products): https://docs.cloud.google.com/mcp/supported-products · auth: https://docs.cloud.google.com/mcp/set-up-authentication-mcp-servers
- Policies (IAM + semantic governance): https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/overview · https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/semantic-governance-overview
- RAG Engine: https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/rag-engine/rag-overview · vector DB choices: …/rag-engine/vector-db-choices · deployment modes: …/rag-engine/deployment-modes · VS 2.0 backend: …/rag-engine/use-rag-managed-vertex-ai-vector-search · embeddings: …/rag-engine/use-embedding-models · transformations: …/rag-engine/fine-tune-rag-transformations · API: https://docs.cloud.google.com/gemini-enterprise-agent-platform/reference/models/rag-api
- Vector Search 1.0: configuring indexes https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/vector-search/configuring-indexes · hybrid search …/vector-search/about-hybrid-search · updates …/vector-search/update-rebuild-index
- Agent Retrieval (Vector Search 2.0): https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/vector-search-2/overview · reranking …/vector-search-2/query-search/reranking
- Ranking API: https://docs.cloud.google.com/generative-ai-app-builder/docs/ranking
- Agent Search chunking: https://docs.cloud.google.com/generative-ai-app-builder/docs/parse-chunk-documents
- Grounding with Agent Search: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/grounding/grounding-with-vertex-ai-search
- Embeddings: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/embeddings/get-text-embeddings · task types …/embeddings/task-types
- AlloyDB AI vector search: https://docs.cloud.google.com/alloydb/docs/ai/vector-search-overview · index choice …/ai/choose-index-strategy · hybrid …/ai/run-hybrid-vector-similarity-search
- BigQuery vector search: https://docs.cloud.google.com/bigquery/docs/vector-search · vector index https://docs.cloud.google.com/bigquery/docs/vector-index · `VECTOR_SEARCH` https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/search_functions
- A2A on Agent Runtime: https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/runtime/create-an-a2a-agent · https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/use-an-a2a-agent
- A2A on Cloud Run: https://docs.cloud.google.com/run/docs/deploy-a2a-agents
- A2A spec: https://a2a-protocol.org/latest/specification/
