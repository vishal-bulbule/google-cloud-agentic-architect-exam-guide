## Section 6 — Last-72-hours cheat sheet

Cross-section distillation. Every line traces back to a chapter above; go there for the why.

### 6.1 The decision trees that decide most questions

**Where does the agent run?**
```
Need managed Sessions + Memory Bank + Agent Identity with least ops, Python/ADK?  → Agent Runtime
  └─ but must host a custom MCP server / non-Python / scale-to-zero containers?  → Cloud Run
      └─ untrusted generated code, gVisor, GPUs, co-located OSS model, platform team? → GKE (Agent Sandbox)
Task > 60 min on Cloud Run request?  → not a request: GKE / Cloud Run Jobs / async. Agent Runtime LRO up to 7 days.
```

**How is the multi-agent flow controlled?**
```
Order known & fixed ................ SequentialAgent / graph Workflow
Independent work, cut latency ...... ParallelAgent (distinct output_key each)
Repeat until good .................. LoopAgent + max_iterations + escalate
Branch on classification + code steps + human approval mid-flow → graph Workflow (Event route, JoinNode, RequestInput)
Open-ended delegation, specialist keeps the conversation → sub_agents (LLM transfer)
Specialist as a function, caller keeps control → AgentTool
Agent in another team / framework / org → A2A (agent card) ; tools & data → MCP
```

**Where does state live?**
```
Within one turn, scratch ........ temp:
Per session ..................... (no prefix) session.state
Per user across sessions ........ user:
Whole app ....................... app:
Semantic long-term facts, dedup/consolidated → Memory Bank   (not raw RAG over transcripts)
Must be in our own Postgres ..... DatabaseSessionService (Cloud SQL/AlloyDB)
Prod anything ................... never InMemory*
```

**Which retrieval backend?**
```
Enterprise docs + connectors + ACLs, least build ........ Agent Search (GE data stores)
Managed RAG pipeline, pluggable vector DB, code-first ... RAG Engine
Custom ANN at scale, you own embeddings ................. Vector Search 1.0  (batch vs streaming fixed at create)
Next-gen managed retrieval with built-in rerank ......... Agent Retrieval (ex-Vector Search 2.0)
Data already relational/transactional ................... AlloyDB / Cloud SQL pgvector
Analytics data already in BQ ............................ BigQuery VECTOR_SEARCH
```

**Which security control?** (they stack — the question asks which one *fits the phrase*)
```
"who is the agent"                      → Agent Identity (per-agent principal, not shared SA)
"never outside folder X regardless"     → PAB  (limits eligibility, never grants)
"block permission P for all agents"     → IAM deny
"read-only tools OK, destructive not"   → Agent Gateway + Access policy (CEL on tool annotations)
"enforce business rule in English"      → Semantic Governance Policies
"prompt injection / PII / URLs, central"→ Model Armor (floor settings = org baseline)
"3rd-party SaaS / act as the user"      → Auth Manager (OAuth delegation, 3LO)
"irreversible action needs a human"     → HITL: tool confirmation / RequestInput / Workflow Builder Approval
"test before blocking"                  → DRY_RUN (IAM) / INSPECT_ONLY (Model Armor)
```

**Which eval tool / metric?**
```
Every PR, deterministic, cheap ........ adk eval (tool_trajectory_avg_score, response_match_score) / adk conformance test
Same meaning, different words ......... final_response_match_v2
No reference answer ................... rubric-based / adaptive rubrics
Ungrounded claims ..................... hallucinations_v1
Extra tool calls / missed step / any order → trajectory_precision / recall / any_order_match
A/B two prompts ....................... pairwise model-based metric
Judge disagrees with SMEs ............. calibrate custom autorater vs human labels
Production drift ...................... online monitors → Cloud Monitoring alerts
```

**Which coding-agent customization?**
```
"enforce/guarantee/block" → hook or permission deny     "convention/guideline" → rule
"procedure with scripts"  → skill                        "separate context/parallel/restricted tools" → subagent
"ship to 200 devs"        → plugin                       "untrusted code isolation" → GKE Agent Sandbox
"no exfiltration from dev env" → Cloud Workstations private + VPC-SC
```

### 6.2 Numbers worth memorizing

| Fact | Value |
|---|---|
| Agent Runtime long-running ops | up to 7 days |
| Agent Runtime `min_instances` / `max_instances` | 0–10 (default 1) / 1–1000 (100 with VPC-SC or PSC-I) |
| Agent Runtime default concurrency | 9 per container (async ADK: multiple of 9, e.g. 36) |
| Agent Runtime default quota | 90 QPM per region |
| Agent Runtime PSC interface subnet | ≥ /28, immutable after create |
| Sessions default TTL | 365 days |
| Memory Bank similarity search default | top 3; scope ≤ 5 keys; no TTL unless configured |
| RAG Engine default chunking | 1024 tokens, 256 overlap; hybrid search = Weaviate only; CMEK = Spanner mode only |
| OCR parser | first 500 pages only |
| GE blended search | ≤ 50 data stores per app; ACLs creation-time only |
| Skill Registry skill | zip ≤ 10 MB (≤ 500 MB unzipped), SKILL.md required, `name` ≤ 64, `description` ≤ 1024; no VPC-SC/CMEK |
| Antigravity rules | 24 KB/file, 20k tokens total budget |
| Online eval monitors cadence | ~every 10 min |
| ADK eval defaults | trajectory 1.0 (exact), response_match 0.8 |
| Gen AI eval judge `sampling_count` | default 4 |
| Agent Gateway blocked-for-missing-allowlist code | HTTP 498 |

### 6.3 Traps that catch experienced people

1. Redeploy/re-create an Agent Runtime agent ⇒ **new Agent Identity principal**; old IAM bindings orphaned.
2. ADK `ModelArmorPlugin` **does not screen tool outputs** — indirect injection via tools needs gateway egress Model Armor.
3. ADK Tool Confirmation **doesn't work with `VertexAiSessionService` or `DatabaseSessionService`** (experimental).
4. Model Armor template + Gemini safety settings **in the same call = error**.
5. `adk deploy cloud_run` **without `--session_service_uri`** ⇒ in-memory sessions.
6. Agent Runtime **cannot host custom MCP servers**; no internet egress by default.
7. Periodic BigQuery ingestion into GE **ignores ACLs**; source IAM never carries over.
8. Media data stores index **metadata only**; unstructured stores reject audio/video.
9. Antigravity remote MCP uses **`serverUrl`** (not `url`); camelCase rule trigger silently dropped.
10. Agent Registry URNs are **inventory identifiers**, IAM bindings use the **agent principal**.
11. VPC-SC: Agent Runtime project must be **in the perimeter before deploy**; Agent Gateway/IAM agent policies have limited VPC-SC support.
12. Changing GE identity provider ⇒ **recreate ingested data stores**, users lose chat history.

### 6.5 Heavily tested — quick recall

**Antigravity customization** (full detail: Section 2.2.b)
```
Workspace  .agents/rules/*.md · .agents/skills/<name>/SKILL.md · .agents/hooks.json · .agents/mcp_config.json · .agents/agents/*.md
Global     ~/.gemini/config/{rules,skills,hooks.json,mcp_config.json}   (CLI skills: ~/.gemini/antigravity-cli/skills/)
Rule trigger   always_on | model_decision (needs description) | glob (needs globs) | manual (@-mention)   — camelCase = silently dropped
Rule limits    24 KB/file · 20k-token budget for global + always_on rules · rules/ scanned flat (nested → rules.json)
Hook events    PreToolUse · PostToolUse (matcher = tool-name regex) · PreInvocation · PostInvocation · Stop
Hook gate      PreToolUse deny = block · PostToolUse can't block · Stop → decision "continue" = "don't finish until green"
Hook timeout   seconds (default 30)  ≠ Gemini CLI milliseconds
Skill          only description required · progressive disclosure: metadata → SKILL.md body → scripts/references on demand
Workflows      deprecated → skills (/migrate-workflows), retired 2026-11-01
Enforce/block → hook · convention → rule · procedure + scripts → skill · separate context → subagent · ship to devs → plugin
```

**Agent Registry — MCP servers** (full detail: Section 3.2.4)
```
Enable        gcloud services enable agentregistry.googleapis.com   (also turns on its own MCP server)
Auto (same project only)  Google remote MCP servers (global, on API enable) · Cloud Run --functional-type=mcp-server
                          · GKE label registry.gke.io/functional-type: MCP_SERVER · Agent Runtime / Gemini Enterprise agents
Manual        gcloud agent-registry services create NAME --location=L --mcp-server-spec-type=tool-spec
                --mcp-server-spec-content=@toolspec.json --interfaces=url=URL,protocolBinding=jsonrpc
              tool spec = tools/list shape · ≤10 KB · ≤100 tools · never re-scanned → update the spec yourself
Write via Service, read via McpServer/Agent/Endpoint views · search = keyword/prefix (semantic = skills only)
Bind auth     gcloud agent-registry bindings create … --auth-provider=…   (needs roles/agentregistry.admin)
Govern        Agent Gateway only reaches REGISTERED destinations · CEL on mcp.tool.isReadOnly · roles/iap.egressor
Traps         us/eu multi-regions: no manual registration or bindings · URN ≠ IAM principal · don't give agents editor/admin
ADK           AgentRegistry(project, location).get_mcp_toolset(name, continue_uri=…) · get_remote_a2a_agent(…)
```

**Agent-to-tool auth** (full detail: Section 5.1.1b)
```
Google APIs / remote MCP   Agent Identity (ADC) + product role + roles/mcp.toolUser
Custom MCP on Cloud Run    ID token, aud = run.app URL · roles/run.invoker · X-Serverless-Authorization wins if both headers
3rd-party API key / 2LO    Auth Manager auth provider (vault) — never in prompt/state/code
Act as the user (3LO)      Auth Manager OAuth delegation · consent · redirect = …/authProviders/NAME/oauthcallback · continue_uri = back to your app
Gemini Enterprise agents   authorization resource → toolAuthorizations · user token via external_access_token_key
GKE self-hosted            Workload Identity Federation (Agent Identity not on GKE)
Grant                      roles/agentidentity.user on the auth provider · register provider in set_up() when deploying
Tokens                     CAA mTLS + DPoP binding → stolen token useless outside runtime
```

### 6.4 Items flagged unverified by research (don't over-invest)

- "Agent vs human mode" in Agents CLI — best mapping is interactive/agent-assisted vs `--yes`/manual/`--json`; no literal flag found.
- Whether Antigravity CLI terminal sandbox is on by default (docs conflict).
- Exact Agent Runtime / Sessions / Memory Bank pricing.
- Which models support Gemini automatic model-routing; ADK built-in retrieval-recall metric (likely none — use custom metric / Agent Platform Evals).
- ADK `RoutedLlm`/`RoutedAgent` documented only as experimental TypeScript.
