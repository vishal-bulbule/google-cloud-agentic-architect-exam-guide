# Google Cloud Professional Agentic Architect — Study Guide

> Built 2026-09-25 from the official exam guide, Google Developer Knowledge corpus (docs.cloud.google.com, adk.dev, antigravity.google), and ADK `llms-full.txt`. Product names are the **2026 Agent Platform names** — the exam uses these, not the Vertex AI names.

## 0.1 Exam blueprint and where to spend time

> 📊 **Infographic:** Exam blueprint & platform map
>
> [![Exam blueprint & platform map](../infographics/00-exam-blueprint.png)](../infographics/00-exam-blueprint.png)

| # | Section | Weight | Est. questions (of ~50–60) | Priority |
|---|---|---|---|---|
| 3 | Developing custom agents (ADK, models, sessions/memory, RAG, Agent Identity, Registry, MCP/A2A, multi-agent) | **~33%** | 17–20 | Highest — breadth is huge |
| 4 | Evaluating and deploying (evalsets, Gen AI eval, Agent Runtime vs Cloud Run vs GKE, troubleshooting, observability) | **~22%** | 11–13 | High |
| 2 | Coding agents (Antigravity, Claude Code on GCP, MCP/skills/hooks/subagents, sandboxes, Agents CLI) | **~17%** | 8–10 | High — newest product surface, least muscle memory |
| 5 | Security and governance (OAuth/Auth Manager, PAB, Agent Gateway, Model Armor, HITL, identity propagation) | **~15%** | 7–9 | Medium-high — many new products in preview |
| 1 | Low-code (Agent Designer, CX Agent Studio flows/pages/routes, Gemini Enterprise data connectors, multimodal ingestion) | **~13%** | 6–8 | Medium — easy points if you learn the vocabulary |

**Read of the blueprint:** 70% of the exam is sections 3+4+2 = *build it in code, evaluate it, ship it, and use coding agents to do so*. Section 5 is threaded through everything — expect security to appear as the "deciding constraint" in section 3/4 scenarios too (e.g. "…and the CISO requires that the agent never holds long-lived credentials").

## 0.2 The 2026 rename map (memorize — questions will use new names)

| Old name | Exam name |
|---|---|
| Vertex AI / Vertex AI Platform | **Gemini Enterprise Agent Platform** ("Agent Platform") |
| Vertex AI Agent Engine | **Agent Runtime** |
| Agent Engine Sessions | **Agent Platform Sessions** (managed sessions) |
| Agent Engine Memory Bank | **Agent Platform Memory Bank** |
| Agent Engine Code Execution | **Agent Platform Code Execution** |
| Vertex AI Search | **Agent Search** |
| Vertex AI Vector Search (1.0) | **Vector Search** ("Vector Search 1.0" in exam scope) |
| Vertex AI Vector Search 2.0 | **Agent Retrieval** |
| Vertex AI RAG Engine | **RAG Engine** |
| Vertex AI Studio | **Agent Studio** |
| Vertex AI Studio App Builder | **App Builder in Agent Studio** |
| Vertex AI Conversation | **Agent Conversation** |
| Gen AI evaluation service on Vertex AI | **Gemini Enterprise Agent Platform Evals** (exam text: "Agent Platform Gen AI evaluation service") |
| Vertex AI Model Garden | **Model Garden** |
| Agentspace | **Gemini Enterprise** (the end-user/employee agent app; IAM role still `roles/discoveryengine.agentspaceUser`) |
| Gemini Enterprise Agent Designer | **Workflow Builder** (GA 2026-09-03) — exam text still says "Agent Designer"; Agent Studio's canvas URL is also `/studio/agent-designer` |
| Dialogflow CX | **CX Agent Studio** (GA 2026-02-04, ADK-based; Dialogflow CX now "legacy" — pages/routes/event handlers live in CX *flows*, wrapped as flow-based agents) |
| NotebookLM Enterprise | **Gemini Notebook Enterprise** |
| Agent Builder | folded into **Agent Platform** |

Source: https://docs.cloud.google.com/gemini-enterprise-agent-platform/vertex-ai-name-changes

**Don't confuse:** *Gemini Enterprise* (the SaaS employee-facing app with Agent Designer, connectors, agent gallery) ≠ *Gemini Enterprise Agent Platform* (the developer platform, ex-Vertex AI). Questions exploit this.

## 0.3 Mental model: the Agent Platform in four pillars

The docs are organised as `build / scale / govern / optimize`. Map every in-scope tool to one:

```
BUILD        ADK · Agents CLI · Antigravity (CLI/SDK/App) · Agent Studio / Agent Designer
             CX Agent Studio · Model Garden · Gemini LLMs · Skill Registry · MCP servers
             RAG Engine · Agent Retrieval · Vector Search 1.0 · Agent Search

SCALE        Agent Runtime (ex-Agent Engine) · Cloud Run · GKE (incl. Agent Sandbox)
             Agent Platform Sessions · Memory Bank · Code Execution
             Data: BigQuery · Cloud SQL · Firestore · Memorystore for Redis · Cloud Storage

GOVERN       Agent Identity (SPIFFE, per-agent principal) · Auth Manager (OAuth 2.0 broker)
             Agent Registry (catalog of agents / MCP servers / endpoints / skills)
             Agent Gateway (network enforcement via IAP; DRY_RUN → ENFORCE)
             IAM allow/deny · Principal Access Boundary · VPC-SC · Model Armor
             Sensitive Data Protection

OPTIMIZE     Agent evaluation (ADK evalsets, Agent Platform Evals, autoraters)
             Agent Observability · Cloud Logging · Cloud Trace (OpenTelemetry)
```

**The governance chain in one line** (the single most testable integration story):
> Agent Registry = *what exists* (inventory) → Agent Identity = *who the agent is* (principal) → IAM / PAB / Access policies = *what it may do* → Agent Gateway = *where it's enforced on the wire* (IAP) → Model Armor = *what content passes* → Auth Manager = *how it gets 3rd-party / user credentials* → Cloud Audit Logs / Observability = *proof*.

## 0.4 Facts worth memorizing early (verified in docs)

- **Agent Runtime**: supports long-running operations up to **7 days**, sub-second cold starts, provisioning < 1 min, **bring-your-own custom container** supported; you can specify your own session ID when creating a Session.
- **Memory Bank**: continuous event streaming with automatic memory generation triggered by event count or idle time; immutable **memory revisions** (version history).
- **Agent Identity**: **GA**. Per-agent SPIFFE-style principal; `principalSet://agents.global.org-ORG_ID.system.id.goog/...` to grant to all agents in a project/org. Credentials protected by Google-managed **Context-Aware Access (mTLS + DPoP token binding)** so tokens can't be replayed outside the runtime. Integrates with IAM allow/deny, **PAB**, VPC-SC (APIs `agentidentity.googleapis.com`, `agentidentitycredentials.googleapis.com`; restricted VIP).
- **Auth Manager** (Agent Identity auth manager): centralized credential vault + auth broker; API key, OAuth client ID/secret, or **OAuth delegation on behalf of a user**; handles consent dialog; access revocation; all access attributable to the agent's SPIFFE ID.
- **Agent Gateway**: networking component that governs user→agent, agent→tool, agent→agent traffic; enforces IAM (Unified) Access policies via **IAP**; run **DRY_RUN** first (logs violations to Cloud Audit Logs, doesn't block) then **ENFORCE**. IAM agent-policy pages state **no VPC Service Controls support**; perimeter enforcement of gateway traffic exists only for gateways created after 2026-09-08 using an agent connectivity template in `ALL_TRAFFIC` mode (see §5). Two separate dry-runs: `iamEnforcementMode: DRY_RUN` (access) and `INSPECT_ONLY` (Model Armor). Launch stage: Private Preview at announcement.
- **Agent Identity principal changes on redeploy** (new `reasoningEngines` ID ⇒ new principal ⇒ old IAM grants orphaned) — grant baseline roles to the project `principalSet`, re-bind sensitive roles post-deploy via `spec.effectiveIdentity`.
- **Agent Registry**: catalog of **Agent, McpServer, Endpoint, Skill, SkillRevision, Publisher** resources; auto-registration from supported runtimes + manual registration; keyword/prefix/**semantic** search; `gcloud agent-registry mcp-servers list|describe`; Terraform `google_agent_registry_*`; console gives ADK code snippets per tool; Observability tab (latency, traffic, errors, token spend).
- **Skill Registry** (Preview): skill = zip with **SKILL.md** (YAML front matter `name` ≤ 64 chars lowercase/hyphen, `description` ≤ 1024 chars); zip ≤ 10 MB, ≤ 500 MB unzipped, ≤ 10k items, ≤ 8 levels deep, no symlinks; Skill (mutable) vs **SkillRevision** (immutable); `skills:retrieve` = semantic search; built-in `gcp-skill-registry` skill; IDs can't start with `gcp-`; **no VPC-SC, no CMEK**; regions us-central1, europe-west4, us-east5.
- **Agent Platform remote MCP server** is GA; Gemini Embedding 2 (`gemini-embedding-2`) GA; Deep Research Agent (prebuilt) runs on Gemini 3.1 Pro.

## 0.5 How Google Professional exams are written — answer heuristics

1. **Managed over self-built** unless the scenario states a constraint that rules managed out (custom runtime, GPU, sidecar, data residency region not supported, existing GKE platform team).
2. **Least privilege + per-agent identity** beats shared service accounts. "One SA for all agents" is almost always the wrong option.
3. **Dry-run before enforce**, **staging before prod**, **evaluate before deploy**. Options that skip the safe rollout step are distractors.
4. **Deterministic control where you can, LLM where you must.** Workflow agents (Sequential/Parallel/Loop/graph) over "let the LLM decide" when the order is known; callbacks/Model Armor/policies over "add a line to the system prompt" for safety.
5. **Read the constraint words**: *lowest latency*, *minimum operational overhead*, *most cost-effective*, *without code changes*, *auditable*, *on behalf of the user*. Exactly one option satisfies the named constraint; the others are technically valid but miss it.
6. **Preview vs GA**: if two options work and one depends on a preview feature for a "regulated production" workload, prefer the GA path.

## 0.6 4-week study plan (≈ 1.5–2 h/day)

| Week | Focus | Hands-on (sandbox project, cheapest tiers, tear down after) |
|---|---|---|
| 1 | §3.1 + §3.3 — ADK depth: LlmAgent, workflow agents, **graph workflows** (new), transfer vs AgentTool, callbacks, plugins, session state prefixes, Memory Bank, model routing; MCP + A2A | Build a 3-agent graph workflow with one MCP tool + one A2A remote agent; wire Agent Platform Sessions + Memory Bank |
| 2 | §3.2 + §5 — RAG (Vector Search / Agent Retrieval / RAG Engine / Agent Search), Agent Identity, Registry, Auth Manager, PAB, Agent Gateway, Model Armor, HITL | Deploy to Agent Runtime with Agent Identity; register in Agent Registry; add Model Armor template + tool-confirmation HITL; read (don't apply) a PAB policy |
| 3 | §4 — ADK evalsets, `adk eval`, criteria; Agent Platform Evals (trajectory metrics, autoraters); runtime selection; troubleshooting; Trace/Logging | Write evalset with golden trajectories; run in Cloud Build as CI gate; compare Agent Runtime vs Cloud Run deploy of same agent |
| 4 | §2 + §1 — Antigravity (skills/rules/hooks/subagents/plugins), Agents CLI, Claude Code on Agent Platform, GKE Agent Sandbox / Workstations; Agent Designer, CX Agent Studio pages/routes/event handlers, Gemini Enterprise connectors | Configure Antigravity with an MCP server + custom skill; build a small CX Agent Studio flow; do all practice questions; re-read every "Exam signals" block |

Final 3 days: only the **Exam signals**, **decision tables**, and **practice question explanations** from each chapter + the §6 cheat sheet.
