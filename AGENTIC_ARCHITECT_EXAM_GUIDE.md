# Google Cloud Professional Agentic Architect — Study Guide

> Built 2026-09-25 from the official exam guide, the Google Developer Knowledge corpus (docs.cloud.google.com, adk.dev, antigravity.google), and ADK `llms-full.txt`. Product names are the **2026 Agent Platform names** — the exam uses these, not the Vertex AI names.

## 0.1 Exam blueprint and where to spend time

> 🧠 **Visual memory map:** Exam blueprint & platform map
>
> [![Exam blueprint & platform map](visual-memory/00-exam-blueprint.png)](visual-memory/00-exam-blueprint.png)

| # | Section | Weight | Est. questions (of ~50–60) | Priority |
|---|---|---|---|---|
| 3 | Developing custom agents (ADK, models, sessions/memory, RAG, Agent Identity, Registry, MCP/A2A, multi-agent) | **~33%** | 17–20 | Highest — the biggest section and the broadest |
| 4 | Evaluating and deploying (evalsets, Gen AI eval, Agent Runtime vs Cloud Run vs GKE, troubleshooting, observability) | **~22%** | 11–13 | High |
| 2 | Coding agents (Antigravity, Claude Code on GCP, MCP/skills/hooks/subagents, sandboxes, Agents CLI) | **~17%** | 8–10 | High — newest product surface, least familiar to most candidates |
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
- **Agent Gateway**: networking component that governs user→agent, agent→tool, agent→agent traffic; enforces IAM (Unified) Access policies via **IAP**; run **DRY_RUN** first (logs violations to Cloud Audit Logs, doesn't block) then **ENFORCE**. IAM agent-policy pages state **no VPC Service Controls support**; perimeter enforcement of gateway traffic exists only for gateways created after 2026-09-08 using an agent connectivity template in `ALL_TRAFFIC` mode (see Section 5). Two separate dry-runs: `iamEnforcementMode: DRY_RUN` (access) and `INSPECT_ONLY` (Model Armor). Launch stage: Private Preview at announcement.
- **Agent Identity principal changes on redeploy** (new `reasoningEngines` ID ⇒ new principal ⇒ old IAM grants orphaned) — grant baseline roles to the project `principalSet`, re-bind sensitive roles post-deploy via `spec.effectiveIdentity`.
- **Agent Registry**: catalog of **Agent, McpServer, Endpoint, Skill, SkillRevision, Publisher** resources; auto-registration (same project only) from Agent Runtime, Cloud Run `--functional-type`, labelled GKE workloads and Google remote MCP servers + manual registration for everything else; keyword/prefix search (**semantic search only for skills**); `gcloud agent-registry mcp-servers list|describe`; Terraform `google_agent_registry_*`; console gives ADK code snippets per tool; Observability tab (latency, traffic, errors, token spend).
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
| 1 | Section 3.1 + Section 3.3 — ADK depth: LlmAgent, workflow agents, **graph workflows** (new), transfer vs AgentTool, callbacks, plugins, session state prefixes, Memory Bank, model routing; MCP + A2A | Build a 3-agent graph workflow with one MCP tool + one A2A remote agent; wire Agent Platform Sessions + Memory Bank |
| 2 | Section 3.2 + Section 5 — RAG (Vector Search / Agent Retrieval / RAG Engine / Agent Search), Agent Identity, Registry, Auth Manager, PAB, Agent Gateway, Model Armor, HITL | Deploy to Agent Runtime with Agent Identity; register in Agent Registry; add Model Armor template + tool-confirmation HITL; read (don't apply) a PAB policy |
| 3 | Section 4 — ADK evalsets, `adk eval`, criteria; Agent Platform Evals (trajectory metrics, autoraters); runtime selection; troubleshooting; Trace/Logging | Write evalset with golden trajectories; run in Cloud Build as CI gate; compare Agent Runtime vs Cloud Run deploy of same agent |
| 4 | Section 2 + Section 1 — Antigravity (skills/rules/hooks/subagents/plugins), Agents CLI, Claude Code on Agent Platform, GKE Agent Sandbox / Workstations; Agent Designer, CX Agent Studio pages/routes/event handlers, Gemini Enterprise connectors | Configure Antigravity with an MCP server + custom skill; build a small CX Agent Studio flow; do all practice questions; re-read every "Exam signals" block |

Final 3 days: only the **Exam signals**, **decision tables**, and **practice question explanations** from each chapter + the Section 6 cheat sheet.

## Contents

1. [Section 1 — Low-code agents (~13%)](#section-1--building-agents-using-low-code-tools-13)
2. [Section 2 — Coding agents (~17%)](#section-2--using-coding-agents-for-application-development-17)
3. [Section 3 — Custom agents (~33%)](#section-3--developing-custom-agents-33)
4. [Section 4 — Evaluate & deploy (~22%)](#section-4--evaluating-and-deploying-agentic-workflows-22)
5. [Section 5 — Security & governance (~15%)](#section-5--securing-and-governing-agentic-workflows-15)
6. [Section 6 — Cheat sheet](#section-6--last-72-hours-cheat-sheet)

Visual memory maps: see the [README](README.md#visual-memory-maps)

---

## Section 1 — Building agents using low-code tools (~13%)

**What the exam is really testing**
1. Can you pick the right low-code surface (Gemini Enterprise Workflow Builder / Agent Designer, Agent Studio on Agent Platform, CX Agent Studio, legacy Dialogflow CX flows) for a given persona and determinism requirement, and do you know where determinism actually comes from (flows, handoff rules, callbacks), as opposed to instructions?
2. Can you shape agent behavior with instructions, XML-structured prompts, few-shot examples, variables and global instructions, and do you know when each one helps or hurts (overfitting, prompt-cache invalidation, context loss)?
3. Can you connect enterprise data safely: federated vs ingested connectors, identity provider choice (Google Identity vs Workforce Identity Federation), ACL propagation, and parser choices for multimodal and unstructured content?

---

### Product name map (verified in docs as of Sep 2026)

| Old name | Current name | Evidence |
|---|---|---|
| Google Agentspace | **Gemini Enterprise** | GE release note, Oct 09 2025: "Google Agentspace is part of Gemini Enterprise". IAM role is still `roles/discoveryengine.agentspaceUser` (displayed as "Gemini Enterprise User"). |
| Agent Designer (Gemini Enterprise no-code agents) | **Workflow Builder** | GE release note, Sep 03 2026: "General availability of Workflow Builder (formerly Agent Designer)". The exam guide still says "Agent Designer". |
| Vertex AI Search | **Agent Search on Gemini Enterprise Agent Platform** | Vertex AI name-changes page |
| Vertex AI Studio | **Agent Studio on Gemini Enterprise Agent Platform** (its console canvas URL is still `/agent-platform/studio/agent-designer`) | name-changes page and Agent Studio docs |
| Vertex AI Platform | **Gemini Enterprise Agent Platform** | name-changes page |
| Vertex AI Agent Engine | **Agent Runtime** | name-changes page |
| Vertex AI Vector Search 2.0 | **Agent Retrieval** | name-changes page |
| Vertex AI Conversation | **Agent Conversation on Gemini Enterprise Agent Platform** | name-changes page |
| Dialogflow CX (generative / playbooks) | **Customer Experience Agent Studio (CX Agent Studio)**, described as "the evolution of Dialogflow CX". Dialogflow CX and ES docs are now listed as "Legacy conversational agents". | CX Agent Studio overview. GA on Feb 04 2026. |
| Contact Center AI / CCAI umbrella | **Gemini Enterprise for Customer Experience** (CX Agent Studio + Agent Assist + CX Insights + Commerce agents) | GE for CX overview |
| NotebookLM Enterprise | **Gemini Notebook Enterprise** | GE release note, Jul 16 2026 |
| Generic search apps | **Custom search** | GE release note, Jun 16 2025 |

**Exam trap:** The exam guide says "Gemini Enterprise Agent Designer". The same tool now appears as **Workflow Builder**, and Agent Platform has its own low-code canvas, **Agent Studio**, whose URL path is `agent-designer`. Map the words in a question to the persona: business users working in the Gemini Enterprise web app means Workflow Builder; developers in the Cloud console who deploy to Agent Runtime means Agent Studio.

---

## 1.1 Configuring agentic workflows and behavior using low-code tools

> 🧠 **Visual memory map:** Which low-code surface?
>
> [![Which low-code surface?](visual-memory/01-low-code-surfaces.png)](visual-memory/01-low-code-surfaces.png)

### 1.1.a State-based workflows (pages, transition routes, event handlers)

#### Core concepts: the four low-code surfaces

| Surface | Who uses it | Where it runs | Determinism mechanism | Output |
|---|---|---|---|---|
| **Workflow Builder** (formerly Agent Designer), Gemini Enterprise web app | Employees and business users (non-admin), within the connectors and tools the admin provisions | Gemini Enterprise app, invoked by `@mention`, on a schedule, or by an event | **Workflows**: triggers → steps → variables, with Condition branching, loops, filters and HITL steps. **Chat agents**: mostly non-deterministic. | Published or shared agent inside GE |
| **Agent Studio** (Agent Platform, Cloud console) | Developers and architects | Deploys to **Agent Runtime** (optionally "Deploy as A2A") | Main agent plus local or Agent Registry subagents. "Get code" exports ADK Python. | ADK app on Agent Runtime |
| **CX Agent Studio** (`ces.cloud.google.com`) | Customer-experience teams building chat and voice agents | Managed, with channels: web widget, telephony, WhatsApp/Instagram, CCaaS | Root/steering agent plus sub-agents. **Handoff rules**, **callbacks**, **flow-based agents** (imported Dialogflow CX flows). | Agent application ("app") |
| **Dialogflow CX flows** (legacy) | Existing CCAI estates | Dialogflow CX | **Pages = states**, routes, event handlers, forms. This is a full state machine. | Reused from CX Agent Studio via flow-based agents |

#### Dialogflow CX state machine (still tested via "pages, transition routes, event handlers")
- A **flow** is a conversation topic. Every agent has a **Default Start Flow**. A **page** is a state, exactly one page is active at a time, and each flow has a **start page**.
- **Page lifecycle:** entry fulfillment → form pre-fill (session params with the same name, then intent params) → state-handler evaluation → form-parameter prompt → send the response queue → wait for input.
- **State handlers** have three parts: *requirements* (intent and/or condition, or event), *fulfillment* (static responses or a webhook), and an optional *transition target*.
  - **Routes**: an intent route (intent requirement) or a condition route (condition only).
  - **Event handlers**: built-in events (`sys.no-match-default`, `sys.no-match-1..6`, `sys.no-input-default`, `sys.no-input-1..6`, `sys.invalid-parameter` at parameter level only, `sys.long-utterance` over 256 chars, `webhook.error`, `webhook.error.timeout`, `.bad-request`, `.rejected` for 401/403, `.unavailable` for 503) and custom events.
- **Scope rules (common in exam questions):**
  - Flow-level routes are always in scope on the flow **start page**. On any other page, flow-level routes are in scope **only if they have an intent requirement**. This is why a flow-level *condition* route "doesn't fire" on a mid-flow page.
  - Page routes, page and flow event handlers, and parameter-level (reprompt) handlers for the parameter currently being filled are all in scope.
- **Evaluation order:** intent routes (page → page route groups → flow → flow route groups), then condition routes (the same order, with flow-level only on the start page), then event handlers (parameter → page → flow). A handler *with* a transition target ends evaluation. A handler without one lets evaluation continue, which means multiple fulfillments can queue messages. Intents and events are *consumed* by the first match. Conditions are *not* consumed.
- **Webhook-error nuance:** if the handler that called the failing webhook has a transition target, the webhook **fails silently** and `webhook.error` is *not* raised.
- **Symbolic targets:** `START_PAGE`, `END_FLOW`, `END_FLOW_WITH_CANCELLATION`, `END_FLOW_WITH_FAILURE`, `END_FLOW_WITH_HUMAN_ESCALATION`, `END_SESSION`, `PREVIOUS_PAGE`, `CURRENT_PAGE`. The **flow stack limit is 25**. `END_FLOW` returns to the calling page and resumes that page's remaining handlers, because the handler call stack is preserved.
- Conditions: AND, OR, or a custom expression such as `$sys.func.rand() < 0.1`. Parameters are referenced as `$session.params.x` or `$intent.params.x`.
- **Generative options inside flows:** *Generators* (an LLM prompt called from fulfillment, with placeholders `$conversation` and `$last-user-utterance`, excluded from the Dialogflow CX SLA), *generative fallback* (runs on no-match; adds `$flow-description` and `$route-descriptions`), and *banned phrases* (which apply to both). **Playbooks** are the fully generative, instruction-plus-examples unit.

#### CX Agent Studio: how "state" is expressed now
- **Agent application**: a root (steering) agent plus sub-agents, with an agent tree structure built on ADK.
- **Variables** (Text, Number, Yes/No, Custom Object, List). The agent *cannot* set variables. Only tools and callbacks can, for example with `context.state["var"] = value`.
- **Handoff rules**: deterministic parent→child (forward) or child→parent (backward) transfers based on variable conditions or a Python `should_trigger_transfer_callback`. You can *force* a transfer or *block* it until a condition holds. Limits: a single AND/OR operator across conditions in the UI; only text, number and boolean variables; block rules accept variables only, not code; complex API-built rules become read-only in the UI.
- **Callbacks** (`before_agent`, `before_model`, `after_model`, `before_tool`, `after_tool`, and so on) use ADK semantics: returning a value overrides the step. They run in a sandbox with **no private network access**. Use OpenAPI or MCP tools when you need to reach private resources.
- **Flow-based agents**: import an existing Dialogflow CX flow as a sub-agent. Control passes to the flow until `END_SESSION`, then returns. You need to grant the **Customer Engagement Suite Service Agent** the **Dialogflow API Client** role. Map input variables to session parameters and output session parameters back to variables.
- **Guardrails**: prompt guard, blocklist (whole word, any mention, or regex), safety level (Relaxed, Balanced, Strict), and rules (natural language or `after_model_callback` code). Outcomes are *Say exactly*, *Handoff to an agent*, or *Generate a response*. **Supervisor agents** (Sep 2026) cover audio quality and missed tool calls, in blocking or non-blocking mode.
- **Tools**: Python, OpenAPI, MCP, data store, file search, Integration Connectors, Google Search and Maps, client function, system, widget, agent-as-tool, A2A remote agent, plus Jira, Confluence, SharePoint, Salesforce and ServiceNow. Tools run **sync or async** (default timeout 30s sync, 60s async).
- Models include `composite-v1` (voice: listener, thinking and speaker models), `gemini-3.1-flash-live`, and 3.x/2.5 Flash for text.

#### Workflow Builder: how "state" is expressed
- **Triggers**: Manual (`@workflow` with input fields), Schedule (interval or cron plus time zone), and Event (a data change in a connected app, with a filter).
- **Steps**: Gemini Agent (instructions, knowledge, connected apps and actions, model **Gemini 3.0+ only**, plain text or JSON-schema output), Flow control (Condition with a variable, operator and value, AND/OR, plus loops and filtering), Human-in-the-loop (**Request info** or **Approval**; the run pauses with status "Needs Review"), MCP servers as steps or tools, and existing agents as steps.
- Includes versioning (save, compare, restore), execution logs with step traces, and sharing with users or groups.
- **Admin gating**: the admin must enable the Workflow Builder toggle in *Manage web app features*. Agents can only use the connectors, tools and permissions the GE admin provisioned.

#### Decision table

| Requirement | Use |
|---|---|
| Business users automating a cross-app process (Gmail → Jira → approval) inside Gemini Enterprise | **Workflow Builder workflow** with an Approval HITL step |
| Open-ended employee Q&A agent over Drive or Jira | **Workflow Builder chat agent** |
| Developer prototypes a multi-agent app visually, then needs code, Agent Runtime deploy, A2A or Agent Registry subagents | **Agent Studio** (Get code → ADK) |
| Customer-facing voice or chat agent with telephony, web widget, barge-in, low latency | **CX Agent Studio** |
| Strict regulated sequence (authenticate → collect 5 fields → validate) that must never be skipped | Dialogflow CX **flow** (forms plus condition routes) wrapped as a **flow-based agent**, or CX Agent Studio **handoff rules / callbacks** |
| "Route authenticated users to Agent B deterministically" in CX Agent Studio | **Handoff rules** (not instructions) |
| Canned response or fixed business rule | **Callback** (CX Agent Studio best practice: not a flow) |
| Intent classification | **LLM** (the docs flag flows for intent detection as a *bad* use) |

#### Trade-offs
- **Flows vs LLM agents**: flows buy determinism and auditability at the cost of design time and brittleness, because session parameters are implicit globals and upstream changes break downstream flows. Treat flows as **black boxes with explicit input and output parameters**.
- **Instructions vs handoff rules vs callbacks**: instructions cost nothing to write but are not deterministic. Handoff rules are deterministic and need no code, but are limited in expressiveness. Callbacks are fully deterministic and flexible but require Python and run in a sandbox without PNA.
- **Tools vs callbacks**: a tool's internals are deterministic, but the *orchestration* (deciding to call it, choosing arguments, interpreting the result) is LLM-driven and can hallucinate. Callbacks run outside the model's control.
- **Async tools** avoid "dead air" in voice, but the agent keeps talking before results arrive. Design responses so they don't depend on results that haven't arrived yet.

#### Limits and gotchas
- Flow stack limit is 25. `sys.long-utterance` fires above 256 characters for non-generative intents.
- CX Agent Studio steering agents cannot transfer between Dialogflow CX agents until `END_SESSION` hands control back.
- Workflow Builder: event triggers are *not* canvas step nodes. If the full Drive tool is enabled on the same Gemini Agent node, it **overrides** restricted "Add from Drive" knowledge. The workaround is two separate agent nodes.
- Workflow Builder chat agents built from a conversational prompt **cannot connect Cloud-source data stores** (BigQuery, Cloud Storage, Cloud SQL). Use designer (flow builder) mode for those.
- Agent Studio stores all agent config in **`us-west1`**, so an org policy that restricts resource locations can block agent creation. You must **Save** before Preview, Deploy or knowledge upload, because an unsaved agent has no identity.
- Agent Studio **deprecated** direct Vertex AI Search data-store tools and direct MCP URLs in favor of **MCP servers from Agent Registry**. Agent Search appears there as `discoveryengine.googleapis.com`. Legacy tools are read-only.

**Exam signals**
- "Business analyst, no code, Gemini Enterprise, approval before sending" → Workflow Builder + Approval step.
- "Must deterministically route / cannot rely on the model" → handoff rules or callbacks (CX Agent Studio), or a condition route (Dialogflow CX).
- "Reuse existing Dialogflow CX flows while migrating" → flow-based agent, with CX Agent Studio as the steering agent.
- "Flow-level condition route not triggering on page X" → scope rule (only intent routes are in scope off the start page).
- "Webhook fails but no error event fires" → the calling route had a transition target.
- "Visual design, then hand to developers as code" → Agent Studio "Get code".

**Common distractors:** "Add more few-shot examples to force routing" (not deterministic). "Use Dialogflow ES contexts" (legacy). "Build a custom ADK app on GKE" when the scenario says low-code. "Use a flow for intent classification".

---

### 1.1.b System instructions and in-console prompt templates (few-shot, chain-of-thought)

#### CX Agent Studio instructions
- **Reference syntax**: `{var}` for a **dynamic variable**, `{{var}}` for a **static variable**, `{@AGENT: Name}`, and `{@TOOL: tool_name}`. In the editor, typing `@` or `{` opens chip pickers.
- **Static vs dynamic variables**:

| | Static `{{x}}` | Dynamic `{x}` |
|---|---|---|
| How it's injected | Compiled into the prompt text before the model call | Appended to history as `<state_update>` events |
| Instruction adherence | Highest | Slightly lower, and adds latency |
| Changes mid-session? | No (config, business rules, large catalogs) | Yes (tool outputs, user-extracted data) |
| Gotcha | **Updating it invalidates prompt caching**, which raises latency | Can be **lost when history is trimmed** past the context window |

- **Restructure instructions** button: converts prose to the recommended XML with `<role>`, `<persona>`/`<primary_goal>`, `<constraints>`, `<taskflow>`/`<subtask>`/`<step>`/`<trigger>`/`<action>`, and `<examples>`. **Refine** rewrites a selected span using AI, guided by a Requirements field.
- **Inline few-shot format**: `<examples> ... [user] ... [model] ```tool_code ...``` ```tool_outputs ...``` [model] ... </examples>`. Guidance: use few-shot to fix specific failures, complex formats or nuanced logic. **Use sparingly** (to avoid overfitting), be descriptive rather than exhaustive, and **start with instructions first**.
- **Global instructions** (advanced app settings) are inherited by every agent on every turn. Good uses: brand tone, DOs and DON'Ts, shared variables, customer profile.
- **Modular components** (restricted access): channel blocks such as `{@startChannelTELEPHONY}…{@endChannelTELEPHONY}`, modality blocks such as `{@startModalityVOICE}`, and Jinja-style `{% if _session.channel == 'WEB_UI' %}`. For evals, set the `_session` scenario variable.
- **Language**: write instructions in English. The agent auto-detects the user's language and responds in it unless instructed otherwise.

#### Dialogflow CX playbooks (legacy, still documented)
- A playbook has a Goal, Instructions, **Examples** (the few-shot examples) and Parameters. The docs recommend **at least 1 example, ideally 4 or more**, and say that *example quality and quantity matter more than instruction precision*.
- Example end states: `OK`, `CANCELLED`, `FAILED`, `ESCALATED`, `PENDING`. Input and output *summaries* pass context between playbooks. Parameters are the **only** way to exchange data between flows and playbooks.

#### Agent Studio (Agent Platform) prompts
- **System instructions** field plus prompt. **Prompt templates** use `{variable}` placeholders (curly braces, no spaces). Limits: **system instructions cannot be a template variable**, and **templates don't support multimodal prompts**.
- **Prompt management**: saved prompts are versioned, available in the SDK (`types.Prompt`, `PromptData` with `system_instruction` and `variables`), support CMEK and VPC-SC, and are shareable. Anyone with `roles/aiplatform.user` sees all saved prompts in the project.
- The **prompt optimizer** (Preview) and "Build with code > Get code" are also available.

#### Workflow Builder prompting
- Build by natural-language prompt (the best-practice advice is to give context, set boundaries, and tell Gemini to *ask you questions*) or in the flow builder. Starter prompts appear as a random 3 at a time. GE admins **cannot override the core assistant's LLM system instructions** (removed Jul 2025). Customize through agents or skills instead.

#### Chain-of-thought (verified guidance)
- CoT example ordering: never show the final structured answer **before** the reasoning steps in examples.
- For **Thinking** models, first try *without* explicit step-by-step reasoning instructions and rely on Thinking (adjustable thinking levels). Keep explicit CoT only if it measurably helps.
- For multi-step tasks, **chain prompts** (sequential) or **aggregate** them (parallel). In low-code terms, that means splitting into sub-agents or workflow steps instead of writing one mega-prompt.

#### Trade-offs
- More few-shot examples improve format fidelity but cost tokens and risk overfitting and poor generalization.
- XML structure improves adherence but makes prompts longer and less editable by non-technical owners.
- Explicit CoT improves transparency on non-thinking models, but adds latency and output tokens, and is often redundant with Thinking.
- Static variables give the best adherence but break the prompt cache when changed.

**Exam signals:** "agent ignores rule buried in long prose" → Restructure instructions (XML `<constraints>`). "Output must follow exact JSON/format" → few-shot (or structured output / JSON schema in Workflow Builder). "Same rule across all sub-agents" → global instructions. "Value changes each turn from a tool" → dynamic variable. "Large rarely-changing catalog, maximize adherence" → static variable. "Different greeting for voice vs chat" → modality conditions.

**Distractors:** "Fine-tune the model" to fix formatting (overkill). "Put the system instruction in a prompt-template variable" (not supported). "Let the agent set variables from instructions" (only tools and callbacks can).

---

## 1.2 Connecting enterprise data to Gemini Enterprise

> 🧠 **Visual memory map:** Gemini Enterprise — connecting data
>
> [![Gemini Enterprise — connecting data](visual-memory/02-ge-enterprise-data.png)](visual-memory/02-ge-enterprise-data.png)

### 1.2.a Securely connecting and querying proprietary data (Gemini Enterprise, Agent Search)

#### Core model
- **App (engine)** ↔ **data stores** is many-to-many. Multiple stores on one app is **blended search**, with a **50 data stores per search app** limit. If one store uses CMEK, **all** must use the same CMEK config. Unstructured data imported from BigQuery isn't supported in blended search. **Website data stores cannot be connected** to GE search/assistant apps.
- Each third-party source creates **one data store per entity** (for example Jira issues, attachments, comments, worklogs).
- **Actions** (write-back: create a Jira issue, send Gmail, upload to SharePoint) are enabled per data store. Users must authorize them in chat ("Enable Actions"). Best practice: associate only **one actions-enabled data store per connector type** per app.
- Regions: `global`, `us`, `eu`. US and EU stores are encrypted, with Google-managed keys by default or CMEK.

#### Federated vs ingested

| | **Federation** | **Ingestion (indexing)** |
|---|---|---|
| Data copy | None. Queries go live to the source API. | Copied into the GE index |
| Freshness | Real time | Depends on sync (full and incremental schedules, or **real-time sync** via webhooks, Preview) |
| Search quality | Can be lower (no index; complex PDFs are under-parsed) | Higher (full layout parsing, knowledge graph) |
| Cost / time | No storage | Storage plus ingestion time |
| Privacy | **Query string (and possibly LLM-rewritten session history) goes to the third party**, under *their* ToS | Stays in Google Cloud |
| ACLs | The source enforces them through the user's OAuth | Synced ACLs, enforced by GE through your IdP |
| M365 identity | No Entra/WIF requirement | **Requires Microsoft Entra ID + Workforce Identity Federation** |

Choose federation for rapidly changing data, data you don't want copied, or quick setup. Choose ingestion for search quality, complex documents, cross-source relevance, or when data must stay inside the Google Cloud boundary.

#### First-party Google Cloud sources
- **BigQuery / Cloud Storage**: *one-time* ingestion (GA, manual refresh, **supports ACLs**, console or API) vs *periodic* ingestion (Preview, every 1, 3 or 5 days, console only, **ACLs NOT respected**). Tables can't be changed after creation.
- **Cloud SQL**: exports through an intermediate Cloud Storage location (optional serverless export at extra cost). Spanner and others are also listed.
- **Cross-project import**: grant the `service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com` agent the roles `bigquery.jobUser` + `bigquery.dataEditor`, or `storage.objectViewer`/`objectAdmin`.
- **Critical gotcha**: *source IAM permissions are not imported*. Without ACL metadata, anyone with GE access can see ingested BigQuery or Cloud Storage data. Fix it with `acl_info.readers.principals` (`user_id`/`group_id`, or `idp_wide: true` for public) in the metadata, plus **"This data store contains access control information"** (`"aclEnabled": true`) **at creation time**. You **can't toggle ACLs later**.
- **Google Drive**: federated through the Drive API. Needs a Workspace customer ID (no @gmail.com), smart features ON, and the Google-managed OAuth app allowlisted. Searches only domain-owned docs. **Service-account credentials can't search Workspace stores**; they need user credentials. Drive index limits: 1 MB of extracted text, 10 MB files for most types, PDF OCR up to 80 pages. CMEK and data residency don't extend to Drive data itself. Admin folder filters are no longer supported for new Drive stores.

#### Third-party sources
- The docs list Jira Cloud/DC, Confluence Cloud/DC, SharePoint, OneDrive, Outlook, Entra ID, ServiceNow, Box, Dropbox, Zendesk, Linear, Slack (in workflows) and more. Many new ones are Preview.
- **SharePoint federated**: Graph delegated `Sites.Search.All` + `AllSites.Read`, results filtered by the user's own permissions. **Ingested**: application `Sites.FullControl.All` or `Sites.Selected` (plus `GroupMember.Read.All` and so on) with federated credentials or an OAuth refresh token. Supports GCC, GCC High and DoD.
- **VPC-SC**: you **can't enforce a perimeter on existing** third-party stores (Box, Linear, Confluence DC and others). You must delete and recreate them.
- **Static IP egress** is available on supported connectors, for allowlisting on the source system.
- A **Sensitive Data Protection policy** can be attached to a data store (for example Drive) to inspect and de-identify retrieved data.
- **Custom connector**: Fetch → Transform (to Discovery Engine `Document` with `aclInfo`) → Sync (incremental or full). Use **identity mapping** (an Identity Mapping Store bound to the data store; nested groups must be flattened; use `external_group:` prefixes in document ACLs) when the source uses non-IdP identities.

#### Identity and access
- **Identity provider (one per location)**: **Google Identity** is recommended. It's *required* for Workspace sources, and third-party IdPs federate into it via OIDC/SAML. **Workforce Identity Federation** is for third-party IdP setups, and is *mandatory* for **M365 sources via ingestion** (Entra ID groups). Configure it in Console → Gemini Enterprise → Settings → Authentication.
- **WIF attribute mapping**: `google.subject=assertion.email.lowerAscii()` (lowercase, because license assignment is case-sensitive), `google.groups`, and `attribute.as_user_identifier_1..50` for alternate IDs. Configure SCIM for Entra with large group counts and for sharing autocomplete.
- **Changing the IdP** (Google ↔ third-party, or a new WIF pool) means you must **delete and recreate ingestion data stores**, and users **lose chat history**. Editing attribute mapping or switching providers within the same pool does not require recreation.
- ACL limit: **3,000 readers per document**, where each group or user counts as one.
- **Users need `roles/discoveryengine.agentspaceUser`** (Gemini Enterprise User). App-level IAM (GA Feb 2026) scopes access to individual apps. Creating data stores needs `roles/discoveryengine.editor`.
- **Model Armor** can screen prompts and responses in GE apps.

#### Agents querying the data
- Workflow Builder / GE agents: admin-provisioned data stores and actions only.
- **CX Agent Studio data store tool**: Website and Cloud Storage stores can be created inline. A tool can hold multiple stores **from one region only**. Settings include a filter (Never/Always/Auto), per-modality rewriter and summarization models and prompts, and a grounding score.
- **Agent Studio**: the Agent Search **MCP server via Agent Registry** (`discoveryengine.googleapis.com`), with auth resolved via IAM on the **agent identity** principal (`principal://agents.global.org-…/reasoningEngines/ID`). Knowledge-file uploads get `roles/storage.objectViewer` on the `{projectNumber}_{location}_agent_studio_files` bucket, and anyone with project or bucket access can read those files.
- **Grounding in Agent Studio prompts**: "Your data" → RAG Engine, Agent Search (data store path, **max 10 data stores**) or Elasticsearch. Requires `discoveryengine.servingConfigs.search`.
- ADK equivalent: `VertexAiSearchTool(data_store_id=...)`. This requires Google Cloud auth, not an AI Studio key. Subclass it and override `_build_vertex_ai_search_config` for per-user filters.

**Exam signals**
- "Results must respect per-user SharePoint permissions, data can't leave M365" → **federated** SharePoint.
- "Best search quality over complex scanned PDFs in SharePoint" → **ingestion** + layout parser, which requires Entra ID + WIF.
- "Okta shop, only third-party sources" → Google Identity (recommended) or WIF.
- "Users see BigQuery rows they can't access in BigQuery" → ACLs weren't enabled at creation, or periodic ingestion was used (which ignores ACLs).
- "Legacy app with its own group names" → custom connector + identity mapping.
- "Enforce VPC-SC on existing Confluence DC store" → recreate the store.

**Distractors:** "Grant BigQuery IAM and GE will inherit it" (it won't). "Enable ACLs on the existing data store" (creation-time only). "Use a service account to search Drive" (not supported). "Periodic BigQuery sync with ACLs" (not respected).

---

### 1.2.b Ingesting and processing unstructured multimodal data (video, audio, images)

#### Options by layer

| Need | Mechanism | Key facts |
|---|---|---|
| Images, tables and charts *inside documents* for search/RAG | **Agent Search / GE layout parser** (default in GE) with **image annotation** and **table annotation**, plus **Gemini layout parsing** (Preview, PDFs) | Supported file types: PDF, HTML, DOCX, PPTX, XLSX, XLSM. Detected image types: BMP, GIF, JPEG, PNG, TIFF. Available only with **document chunking** turned on. Configured via `documentProcessingConfig.defaultParsingConfig.layoutParsingConfig`. HTML exclusion uses `excludeHtmlElements`, `excludeHtmlClasses`, `excludeHtmlIds`. |
| Scanned PDFs / text in images | **OCR parser** (`ocrParsingConfig.useNativeText=true` for mixed PDFs) | **First 500 pages only.** For complex scanned layouts, Google recommends the layout parser over OCR. |
| Default when nothing is specified | **Digital parser** | Also the fallback for unsupported types (for example TXT under the layout parser) |
| Already-parsed content | Bring your own parsed document (Preview) | |
| Unstructured data store document types | TXT, PDF, HTML, DOCX, PPTX, XLSX, XLSM | **No native audio or video documents** in unstructured stores. **Media data stores** index *metadata* (title, uri, categories, duration, available time), not the media content. |
| RAG over images and diagrams (DIY) | **RAG Engine LLM parser** | Accepts PDF, PNG, JPEG, WEBP, HEIC, HEIF. **Gemini 2.x only (3.x not supported)**. Custom parsing prompt and `max_parsing_requests_per_min`. Cost is roughly files × pages × tokens. Alternatively the **Document AI layout parser** (`LAYOUT_PARSER_PROCESSOR`; 20 MB and 500 pages max per file; recommended chunk size 1024 with overlap 256). |
| Video and audio *understanding* in the workflow | **Gemini multimodal** (`fileData.fileUri` from Cloud Storage, YouTube or inline) | Video: default **1 FPS** sampling (lower for lectures, higher for fast motion) and `videoMetadata` clipping offsets. **Agentic video understanding** (`media_processing="agentic"`, Preview) navigates long videos with fewer tokens, but you must pass `step_list` back each turn. With VPC-SC on, HTTP media URLs aren't supported, so use Cloud Storage. |
| Semantic search over image corpora | BigQuery object table + `AI.GENERATE_EMBEDDING` (multimodal embedding / `gemini-embedding-2`) + `VECTOR_SEARCH`, or **Agent Retrieval** multimodal embeddings | Batch limit of 25,000 rows per `AI.GENERATE_EMBEDDING` call (per the tutorial). Retry `RESOURCE_EXHAUSTED` rows. |
| Ad-hoc user uploads in GE chat | Assistant upload | Images (.png/.jpeg) up to 30 MB, **.mp4 video up to 200 MB, .mp3 audio up to 200 MB**, PDF up to 100 MB, DOCX up to 3 MB. The **max upload/download file size for connector actions is 200 MB**. |
| Voice agents | CX Agent Studio native audio (`gemini-3.1-flash-live`, `composite-v1`, Chirp 3 voices) | Real-time audio, not ingestion |

#### Recommended pattern for "index our training videos / call recordings for agent Q&A" (a design pattern I've assembled from the pieces above; there's no single documented product)
1. Land the media in Cloud Storage. 2. Run Gemini (batch) to produce transcripts, timestamped chapters and scene descriptions as text or JSON. 3. Ingest those derived documents, with a `uri` back to the media and `acl_info`, into an Agent Search or GE data store (or a RAG Engine corpus). 4. Agents cite the chapter and timestamp. The trade-off is added cost and latency at ingest in exchange for cheap, fast, ACL-aware retrieval. Sending raw video to Gemini on every query is simple but expensive per query and can't be ACL-filtered across a corpus.

#### Trade-offs
- The **layout parser** gives better chunking and answers but is billed as a Document AI feature. The **digital parser** is cheaper but ignores tables and headings.
- The **Gemini layout / LLM parser** gives the best quality on charts and flowcharts, at a per-page token cost plus quota and rate tuning.
- **Higher video FPS** captures more detail but costs more tokens. Agentic video processing uses fewer tokens but is Preview and requires carrying `step_list` state.
- Federated connectors use the layout parser too, but real-time querying can prevent full analysis of complex PDFs. **Choose ingestion for deep parsing**.

#### Limits and gotchas
- OCR: 500 pages. Drive OCR: 80 pages. RAG Engine layout parser: 20 MB and 500 pages. Gemini API `fileUri`: one video, one audio file and up to 10 images per request in the REST sample, with a 15 MB cap for HTTP-URL media.
- Parsing and chunking settings are chosen **at data store creation**. Re-importing into an existing RAG corpus doesn't re-parse files imported without a parser, so delete and re-import them.
- Agent Studio knowledge files: **PDF or plain text only, up to 10 files, 2 MB each, parent node only**.

**Exam signals:** "scanned PDFs with infographics and tables" → layout parser (+ image/table annotation, Gemini enhancement). "long lecture videos, reduce token cost" → lower FPS or agentic video understanding. "search images by text description" → multimodal embeddings + vector search. "diagrams in slide decks for RAG Engine" → LLM parser or Document AI layout parser.

**Distractors:** "Upload MP4s to an unstructured Agent Search data store" (unsupported type). "Use a media data store to answer questions about what is said in videos" (metadata only). "OCR parser for a 900-page scanned manual" (processes only 500 pages).

---

### Practice questions

**Q1.** A retail bank's HR team (no developers) wants an automation in Gemini Enterprise. When a new-hire email arrives, it should draft onboarding tasks in Jira and schedule calendar invites. A manager must sign off before anything is created. What should you recommend?

- **A.** A Workflow Builder workflow with an event trigger, a Gemini Agent step, and an Approval human-in-the-loop step
- **B.** A Workflow Builder chat agent with instructions to "always ask the manager first"
- **C.** An ADK agent deployed to Agent Runtime and registered in Gemini Enterprise
- **D.** A CX Agent Studio app with a handoff rule to a "manager" sub-agent

**Answer: A.** Workflows support event triggers and native Approval HITL steps that pause the run ("Needs Review"). B relies on non-deterministic instructions. C needs developers. D is a customer-experience conversational tool, not internal cross-app automation.

**Q2.** A Dialogflow CX agent has a flow-level condition route `$session.params.vip = true → VIP page`. It works on the start page but never fires once the user is on the "Collect address" page. Why?

- **A.** Conditions are consumed after the first evaluation
- **B.** Flow-level routes with only a condition requirement are in scope only on the flow start page
- **C.** Event handlers are evaluated before routes
- **D.** The flow stack limit of 25 was exceeded

**Answer: B.** Off the start page, only flow-level *intent* routes are in scope. A is wrong because conditions are not consumed. C is wrong because routes are evaluated before event handlers. D doesn't match the symptom.

**Q3.** In CX Agent Studio, unauthenticated callers must never reach the "Payments" sub-agent, even if they ask convincingly. What is the lowest-effort way to guarantee this?

- **A.** Add a `<constraints>` rule in the root agent's XML instructions
- **B.** Add four few-shot examples showing refusals
- **C.** Configure a handoff rule that blocks transfer to Payments until `is_authenticated == true`
- **D.** Set the safety guardrail to Strict

**Answer: C.** Handoff rules are the deterministic, no-code control. A and B are non-deterministic. D targets harmful content, not authorization.

**Q4.** A CX agent references a 40 KB product policy on every turn. It rarely changes, and adherence is poor when it's supplied through a tool-updated variable. What should you do?

- **A.** Make it a static variable `{{policy}}`
- **B.** Keep it a dynamic variable `{policy}` and add few-shot examples
- **C.** Move it into a callback that rewrites the user message
- **D.** Put it in a data store tool

**Answer: A.** Static variables compile into the prompt and give the best adherence (at the cost of prompt-cache invalidation when updated). B: dynamic variables sit in history, can be trimmed, and have lower adherence. C is hacky. D adds retrieval non-determinism for content that should always be present.

**Q5.** An enterprise uses Okta for SSO and Microsoft 365. It wants the highest-quality Gemini Enterprise answers over complex SharePoint PDFs, with document-level permissions enforced. What is required?

- **A.** Federated SharePoint connector with Google Identity
- **B.** Ingestion-mode SharePoint connector, with Workforce Identity Federation configured against Microsoft Entra ID
- **C.** Ingestion-mode connector with Okta WIF only
- **D.** Export SharePoint to Cloud Storage and use periodic ingestion

**Answer: B.** M365 ingestion requires Entra ID groups via WIF, even if another IdP handles SSO. Ingestion gives full layout parsing. A: federation gives lower quality on complex PDFs. C: Okta alone doesn't satisfy the Entra requirement. D: periodic Cloud Storage ingestion doesn't respect ACLs.

**Q6.** After ingesting a BigQuery table into a Gemini Enterprise data store, analysts see rows they can't query in BigQuery. What is the fix?

- **A.** Grant `roles/bigquery.dataViewer` more narrowly
- **B.** Recreate the data store with ACLs enabled and `acl_info` in the data, using one-time ingestion
- **C.** Toggle "access control" on the existing data store
- **D.** Switch to periodic ingestion so permissions stay in sync

**Answer: B.** Source IAM isn't imported, ACLs are set only at creation time, and only one-time ingestion honors ACLs. A has no effect on GE. C can't be done after creation. D ignores ACLs.

**Q7.** A legal team needs to query 3,000 scanned contracts, each 50–900 pages long, with tables and stamps, through Agent Search. Which configuration is best?

- **A.** Digital parser (the default)
- **B.** OCR parser with `useNativeText=false`
- **C.** Layout parser with chunking, table annotation and Gemini layout parsing enabled at data store creation
- **D.** Upload the files to the Gemini Enterprise chat

**Answer: C.** The layout parser is recommended for complex scanned PDFs with tables, and chunking is required. A misses scanned text. B processes only the first 500 pages and ignores structure. D is ad hoc with no index or ACLs.

**Q8.** A company wants its agent to answer questions about hour-long recorded town halls stored in Cloud Storage, while minimizing per-query token cost. What should it do?

- **A.** Put the MP4 files in an unstructured Agent Search data store
- **B.** Create a media data store
- **C.** Pre-process with Gemini (a low-FPS or agentic video pass) into timestamped transcripts and chapters, then ingest those text documents with ACLs into a data store
- **D.** Send the full video at 5 FPS with every question

**Answer: C.** Derived text is cheap to retrieve and ACL-aware. A: MP4 isn't a supported unstructured type. B indexes metadata only. D maximizes token cost.

**Q9.** A developer builds an agent in Agent Studio on Agent Platform that must search an existing Agent Search data store. The legacy "Vertex AI Search Data Store" tool is read-only. What should they do?

- **A.** Recreate the agent in Workflow Builder
- **B.** Add the Agent Search MCP server (`discoveryengine.googleapis.com`) from Agent Registry, and make sure the agent identity has the required roles
- **C.** Upload the documents as knowledge files
- **D.** Add the data store's URL as a direct MCP endpoint

**Answer: B.** Agent Studio deprecated direct data store and direct MCP tools in favor of Agent Registry MCP servers, with auth through the agent identity. C is limited to 10 files of 2 MB, PDF or text. D is also deprecated. A switches surface for no reason.

**Q10.** Your team's CX Agent Studio callback must look up a customer in an on-prem CRM reachable only over Private Service Connect. The callback times out. What is the best fix?

- **A.** Increase the callback timeout
- **B.** Move the lookup into an OpenAPI or MCP tool that supports private network access, and set the result into a variable
- **C.** Configure Service Directory for the callback
- **D.** Use a static variable containing the CRM data

**Answer: B.** Callbacks run in a sandbox without private network access (even with Service Directory configured). OpenAPI and MCP tools support PNA. A doesn't address reachability. C is explicitly not supported for callbacks. D gives stale, non-scalable data.

---

### Key doc links
- CX Agent Studio overview: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio
- CX Agent Studio release notes: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/resources/release-notes
- CX Agent Studio instructions: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/instruction
- CX Agent Studio variables: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/variable
- CX Agent Studio agents and models: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/agent
- CX Agent Studio handoff rules: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/handoff
- CX Agent Studio callbacks: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/callback
- CX Agent Studio flow-based agents: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/flow
- CX Agent Studio guardrails: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/guardrail
- CX Agent Studio best practices: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/best-practices
- CX Agent Studio tools and data store tool: https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/tool , https://docs.cloud.google.com/gemini-enterprise-cx/cx-agent-studio/tool/data-store
- Gemini Enterprise for CX overview: https://docs.cloud.google.com/gemini-enterprise-cx
- Dialogflow CX basics, pages, handlers: https://docs.cloud.google.com/dialogflow/cx/docs/basics , https://docs.cloud.google.com/dialogflow/cx/docs/concept/page , https://docs.cloud.google.com/dialogflow/cx/docs/concept/handler
- Dialogflow CX generators, generative fallback, generative vs deterministic: https://docs.cloud.google.com/dialogflow/cx/docs/concept/generators , https://docs.cloud.google.com/dialogflow/cx/docs/concept/generative-fallback , https://docs.cloud.google.com/dialogflow/cx/docs/generative-deterministic
- Dialogflow CX playbooks and examples: https://docs.cloud.google.com/dialogflow/cx/docs/concept/playbook , https://docs.cloud.google.com/dialogflow/cx/docs/concept/playbook/example , https://docs.cloud.google.com/dialogflow/cx/docs/concept/playbook/best-practices
- Workflow Builder: https://docs.cloud.google.com/gemini/enterprise/docs/workflow-builder , https://docs.cloud.google.com/gemini/enterprise/docs/workflow-builder/workflow-agents , https://docs.cloud.google.com/gemini/enterprise/docs/workflow-builder/create-chat-agent , https://docs.cloud.google.com/gemini/enterprise/docs/workflow-builder/use-hitl-steps
- Gemini Enterprise release notes (renames): https://docs.cloud.google.com/gemini/enterprise/docs/release-notes
- Gemini Enterprise agents overview: https://docs.cloud.google.com/gemini/enterprise/docs/agents-overview
- Agent Studio (Agent Platform) overview and design agents: https://docs.cloud.google.com/gemini-enterprise-agent-platform/agent-studio , https://docs.cloud.google.com/gemini-enterprise-agent-platform/agent-studio/design-agents
- Agent Platform agents paths: https://docs.cloud.google.com/gemini-enterprise-agent-platform/agents
- Vertex AI → Agent Platform name changes: https://docs.cloud.google.com/gemini-enterprise-agent-platform/vertex-ai-name-changes
- Prompt templates, prompt sharing and management, strategies, break-down prompts, few-shot: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/prompts/prompt-templates , https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/prompt-sharing , https://docs.cloud.google.com/gemini-enterprise-agent-platform/reference/models/prompt-classes , https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/prompts/prompt-design-strategies , https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/prompts/break-down-prompts , https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/prompts/few-shot-examples
- GE connectors and data stores: https://docs.cloud.google.com/gemini/enterprise/docs/connectors/introduction-to-connectors-and-data-stores , https://docs.cloud.google.com/gemini/enterprise/docs/apps-data-stores , https://docs.cloud.google.com/gemini/enterprise/docs/concepts
- GE BigQuery, Cloud Storage, Cloud SQL connectors: https://docs.cloud.google.com/gemini/enterprise/docs/connectors/connect-bigquery , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/connect-cloud-storage , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/connect-cloud-sql
- GE Drive, SharePoint connectors: https://docs.cloud.google.com/gemini/enterprise/docs/connectors/gdrive , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/gdrive/set-up-data-store , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/ms-sharepoint , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/ms-sharepoint/set-up-data-store , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/ms-sharepoint/third-party-config
- GE identity provider and ACLs: https://docs.cloud.google.com/gemini/enterprise/docs/configure-identity-provider , https://docs.cloud.google.com/gemini/enterprise/docs/identity , https://docs.cloud.google.com/gemini/enterprise/docs/identity-mapping
- GE custom connector: https://docs.cloud.google.com/gemini/enterprise/docs/connectors/custom-connector , https://docs.cloud.google.com/gemini/enterprise/docs/connectors/create-custom-connector
- GE parse and chunk: https://docs.cloud.google.com/gemini/enterprise/docs/parse-chunk-documents
- GE assistant uploads and limits: https://docs.cloud.google.com/gemini/enterprise/docs/assistant-chat
- Agent Search product page and parse/chunk: https://cloud.google.com/products/gemini-enterprise-agent-platform/agent-search , https://docs.cloud.google.com/generative-ai-app-builder/docs/parse-chunk-documents , https://docs.cloud.google.com/generative-ai-app-builder/docs/media-documents
- Grounding with Agent Search: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/grounding/grounding-with-vertex-ai-search
- RAG Engine LLM parser and layout parser: https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/rag-engine/llm-parser , https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/rag-engine/layout-parser-integration
- Video understanding: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/video-understanding
- BigQuery multimodal embeddings: https://docs.cloud.google.com/bigquery/docs/generate-multimodal-embeddings
- ADK grounding with Agent Search: https://adk.dev/grounding/grounding_with_search/ ; ADK callbacks: https://adk.dev/callbacks/


---

## Section 2 — Using coding agents for application development (~17%)

**What the exam is really testing**
1. Do you know *where* each customization lives (file name, path, scope) in Antigravity and Claude Code on Google Cloud, and which primitive fits the need: rule, skill, hook, subagent, plugin, MCP server, or extension?
2. Can you pick the right isolation boundary for agent-run code (Antigravity terminal sandbox, Cloud Workstations, GKE Agent Sandbox with gVisor) and pair it with least-privilege credentials and egress control?
3. Can you connect the inner loop (coding agent) to the outer loop (Agents CLI, then Agent Runtime / Cloud Run / GKE, then Skill Registry / Agent Registry, then Gemini Enterprise) without hand-rolling glue?

> Naming note (2026): Vertex AI is now **Gemini Enterprise Agent Platform** ("Agent Platform"). Agent Engine is now **Agent Runtime**. The API is still `aiplatform.googleapis.com`. Antigravity has three surfaces: **Antigravity 2.0** (standalone desktop app, successor of Agent Manager), **Antigravity CLI** (`agy`), and **Antigravity IDE** (standalone editor) plus **IDE extensions** (VS Code, Visual Studio, JetBrains, Zed, Xcode). There is also the **Antigravity SDK** (Python, `pip install google-antigravity`).

---

### 2.1 Using coding agents effectively

#### 2.1.a Configuring coding agents with MCP servers, custom skills, and tool access

**Core concepts**
- **MCP** gives the agent live context (schemas, logs) and actions (create a ticket, run a query). **Skills** give it procedural know-how, loaded on demand. **Permissions** decide what the agent may do without asking.
- Antigravity uses **one config layout across all three surfaces**. Workspace scope lives under `.agents/`. Global scope lives under `~/.gemini/config/`. The CLI also has its own home at `~/.gemini/antigravity-cli/`.

**Antigravity MCP configuration**

| Scope | File |
|---|---|
| Global | `~/.gemini/config/mcp_config.json` |
| Workspace | `.agents/mcp_config.json` |
| Plugin | `<plugin>/mcp_config.json` |
| SDK | `LocalAgentConfig(mcp_servers=[McpStdioServer(...)/McpStreamableHttpServer(...)])`; the SDK also auto-discovers `.agents/mcp_config.json` |

```json
{
  "mcpServers": {
    "sqlite-explorer": {
      "command": "node",
      "args": ["/usr/local/bin/sqlite-mcp-server.js"],
      "env": { "SQLITE_DB_PATH": "/var/data/app.db" }
    },
    "bigquery": {
      "serverUrl": "https://bigquery.googleapis.com/mcp",
      "authProviderType": "google_credentials",
      "disabledTools": ["execute_sql_write"]
    },
    "drive": {
      "serverUrl": "https://drivemcp.googleapis.com/mcp/v1",
      "oauth": { "clientId": "…", "clientSecret": "…" }
    }
  }
}
```
(The BigQuery `serverUrl` and tool name in this example are illustrative and unverified. Every field name comes from the Antigravity MCP docs.)

- **Transport.** Use `command` for stdio or `serverUrl` for Streamable HTTP or SSE. **Gotcha:** the legacy `url` and `httpUrl` fields are **not supported** for remote servers, so `serverUrl` is mandatory.
- **Optional fields:** `args`, `env`, `cwd`, `headers`, `authProviderType`, `oauth`, `disabled`, `disabledTools`.
- **Auth has three modes:**
  - **`authProviderType: "google_credentials"`** uses ADC. Run `gcloud auth application-default login` first, and set a quota project with `gcloud auth application-default set-quota-project`. This is the right answer for Google Cloud remote MCP servers.
  - **OAuth with DCR** needs zero config. Without DCR, supply `oauth.clientId` and `oauth.clientSecret`, and register the redirect URI `https://antigravity.google/oauth-callback`. Tokens are stored in `~/.gemini/antigravity/mcp_oauth_tokens.json`.
  - **Static `headers`** (API key or bearer token) is the least preferred option.
- **UI and TUI.** Antigravity 2.0 uses Settings → Customizations → Installed MCP Servers → **Add MCP** (the MCP Store). The IDE uses "…" → MCP Servers → Manage → *View raw config*. The CLI uses `/mcp`, the interactive MCP Manager, which shows status, lets you authenticate, and lists tools.
- **The MCP Store** lists Google servers (BigQuery, AlloyDB, Cloud SQL, Spanner, Bigtable, Dataplex, MCP Toolbox for Databases, GKE OneMCP, Cloud CLI Execution (gcloud), Apigee, Firebase, Cloud Audit Manager, Quotas) and third-party servers (GitHub, SonarQube, Wiz, CrowdStrike, and others).

**Antigravity tool access (permissions engine)**
- Every sensitive operation is a resource of the form `action(target)`. The actions are `read_file`, `write_file`, `read_url`, `execute_url`, `command`, `mcp` (plus `unsandboxed` on Windows and in the CLI).
- **Precedence is Deny > Ask > Allow.** If `command(*)` is in Ask and `command(git)` is in Allow, every command still prompts.
- **MCP tools default to Ask.** Grant them with `mcp(server/tool)`, `mcp(server/*)` or `mcp(*)`.
- The CLI stores rules in `~/.gemini/antigravity-cli/settings.json`:
```json
{ "permissions": {
    "allow": ["command(git)", "command(regex:npm run (build|lint|test))", "write_file(src/)", "read_url(google.com)", "mcp(linter/*)"],
    "deny":  ["command(rm -rf)", "command(sudo)", "write_file(.git/)", "write_file(/home/user/.ssh)"],
    "ask":   ["execute_url(aws.amazon.com)", "mcp(sql/execute_mutation)"] } }
```
- **Implicit rules:** allowing write also allows read on the same path, and denying read also denies write.
- **Anti-smuggling:** prefix matching is disabled when the command contains `$(…)`, backticks, `<(…)`, brace expansion, or tool flags such as `git -c core.pager=…`. These need an exact full-line match, or they fall back to Ask. `git status && git log` still prefix-matches.
- **SDK equivalent:** a declarative policy engine, `policies=[deny("*"), allow("view_file"), ask_user("run_command", handler=…)]`. `policy.safe_defaults()` allows read-only tools and asks before writes. You can restrict tools with `CapabilitiesConfig(enabled_tools=BuiltinTools.read_only())`.

**Custom skills (Agent Skills open standard, agentskills.io)**
- A skill is a folder: `SKILL.md` (YAML frontmatter plus body), with optional `scripts/`, `examples/`/`references/`, and `resources/`/`assets/`.
- Only `description` is required in Antigravity. `name` defaults to the folder name.
- **Progressive disclosure:** only the name and description are indexed up front. The body is read on activation.
- You can invoke a skill **autonomously** (the model matches it by description) or by **slash command** (`/<skill-name>`). The CLI turns every skill into a slash command automatically.

| Surface | Workspace | Global |
|---|---|---|
| Antigravity 2.0 / IDE | `.agents/skills/<name>/` | `~/.gemini/config/skills/<name>/` (IDE also reads legacy `~/.gemini/antigravity/skills/`) |
| Antigravity CLI | `.agents/skills/<name>/` | `~/.gemini/antigravity-cli/skills/<name>/`; plugin skills in `~/.gemini/antigravity-cli/plugins/<p>/skills/` |
| Gemini CLI (legacy) | `.gemini/skills/` or `.agents/skills/` | `~/.gemini/skills/` or `~/.agents/skills/` |
| SDK | `LocalAgentConfig(skills_paths=[...])` | n/a |

- **Gotcha:** `.agent/skills` (singular) is still read for backward compatibility. The default is `.agents/skills`.
- **Migrating from Gemini CLI:** workspace `.gemini/skills/` must be moved to `.agents/skills/` by hand. `agy plugin import gemini` converts extensions into plugins.

**Claude Code on Google Cloud (Claude via Agent Platform)**

| Setting | Value / purpose |
|---|---|
| `CLAUDE_CODE_USE_VERTEX=1` | Route Claude Code to Agent Platform instead of the Anthropic API |
| `CLOUD_ML_REGION` | `global`, a multi-region (`us` or `eu`, which use `aiplatform.{us,eu}.rep.googleapis.com`), or a region such as `us-east5`. If unset, it falls back to `us-east5` |
| `ANTHROPIC_VERTEX_PROJECT_ID` | Billing and quota project. **It wins over** `GOOGLE_CLOUD_PROJECT` and the project in the credentials file |
| `VERTEX_REGION_CLAUDE_*` | Per-model region override, used when `global` isn't supported for a model (for example `VERTEX_REGION_CLAUDE_HAIKU_4_5=us-east5`) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` / `_SONNET_MODEL` / `_HAIKU_MODEL`, `ANTHROPIC_MODEL` | **Pin models** for fleet rollouts. Aliases otherwise resolve to Claude Code's built-in defaults |
| `ANTHROPIC_VERTEX_BASE_URL` | Custom endpoint or gateway |
| `gcpAuthRefresh` (settings.json) | Command run when ADC has expired, for example `gcloud auth application-default login` |
| `DISABLE_PROMPT_CACHING=1` / `ENABLE_PROMPT_CACHING_1H=1` | Prompt caching is on by default with a 5-minute TTL. The 1-hour TTL costs more |
| IAM | `roles/aiplatform.user` (the key permission is `aiplatform.endpoints.predict`). You can use a custom role with only that permission |

- **Setup:** `gcloud services enable aiplatform.googleapis.com`, then enable the Claude models in **Model Garden**. Alternatively, run the in-product wizard: `claude` → 3rd-party platform → Google Vertex AI, or `/setup-vertex`. The wizard writes the `env` block of `~/.claude/settings.json`.
- **Auth** uses the standard ADC chain, including X.509 Workload Identity Federation through `GOOGLE_APPLICATION_CREDENTIALS`. `/logout` is unavailable in Vertex mode.
- **Verify** with `/status`, which should show `API provider: Google Vertex AI`.
- **Troubleshooting:**
  - A 404 "model not found" means the model isn't enabled in Model Garden or isn't available in that location or on `global`.
  - A 429 means you should check that both the primary and the small/fast model exist in the region, or switch to `global`.
- **Best practice:** use a dedicated GCP project for Claude Code, which simplifies cost tracking and IAM.
- **Claude Code config files:**
  - MCP: `.mcp.json` (project) or `claude mcp add --scope project|user …`. The ADK docs example is `claude mcp add adk-docs --transport stdio -- uvx --from mcpdoc …`.
  - Skills: `.claude/skills/<name>/SKILL.md`.
  - Subagents: `.claude/agents/*.md`.
  - Hooks: `settings.json`.
  - Enterprise enforcement: managed settings.
- The Anthropic docs note that MCP **tool search** is on by default for Claude 4.5-generation and later models on Agent Platform.

**Trade-offs**
- **ADC vs OAuth vs headers for MCP.** ADC reuses GCP IAM and needs no new secret, but it runs with *your* identity, which is broad. OAuth is per-user and revocable, but it needs a client registration. Static headers are simplest, but they are a long-lived secret on disk.
- **`global` vs regional endpoint for Claude.** `global` gives better availability and fewer 429s. Regional or multi-region gives data residency, but some models aren't available there.
- **Unpinned model aliases** mean zero maintenance, but upgrades are uncontrolled and can shift cost (for example, the default moving from Sonnet to Opus). Pinning gives predictable cost and behavior, but you have to bump versions yourself.

**Exam signals**

| Keyword in the question | Likely answer |
|---|---|
| "Connect Antigravity to a Google Cloud remote MCP server without storing secrets" | `serverUrl` plus `authProviderType: "google_credentials"` (ADC) |
| "Share MCP config with the whole repo team" | `.agents/mcp_config.json`, committed |
| "Hide a destructive tool from the model but keep the server" | `disabledTools` (config), or a `deny` rule `mcp(server/tool)` |
| "Use Claude Code but keep billing, quota and data in our GCP org" | `CLAUDE_CODE_USE_VERTEX=1` plus `ANTHROPIC_VERTEX_PROJECT_ID` plus `roles/aiplatform.user` |
| "Consistent model across 500 developers" | Pin `ANTHROPIC_DEFAULT_*_MODEL` (Claude Code), or `--model` (agy headless fails loudly on an unknown model) |
| "Model not available on global endpoint" | `VERTEX_REGION_CLAUDE_<MODEL>` override |

**Distractors**
- The `url` or `httpUrl` field in Antigravity (not supported).
- Putting API keys in rules or `AGENTS.md`.
- Granting `roles/aiplatform.admin` to developers.
- Using the Anthropic API key when the requirement is "stay within GCP governance".

---

#### 2.1.b Using coding agents in secure sandboxes (GKE, Cloud Workstations, Antigravity)

> 🧠 **Visual memory map:** Sandbox isolation tiers
>
> [![Sandbox isolation tiers](visual-memory/04-sandbox-tiers.png)](visual-memory/04-sandbox-tiers.png)

**Layer 1: the Antigravity terminal sandbox (local, per command)**
- **OS primitives, no VM or Docker, no startup delay:**
  - Linux uses kernel **namespaces**.
  - macOS uses **`sandbox-exec`** (Seatbelt/SBPL).
  - The CLI docs also list `nsjail` on Linux and AppContainer on Windows.
- **Default boundary:** read-write on the workspace, temp dirs and build caches. Read-only on system dirs (`/usr`, `/etc`). **`~/.ssh` and `.env` are blocked.** **No network access.**
- **Permissions expand the boundary:**
  - `read_file` paths become read-only mounts.
  - `write_file` paths become read-write mounts.
  - **`read_url` domains become the sandbox's outbound network allowlist.** This is how you express egress control.
- **Presets (macOS and Linux)** are set under Settings → General → Permission Settings, with per-project overrides under Settings → Projects (default "Inherit General"):

| Preset | Sandbox | Terminal commands | File access | MCP & web |
|---|---|---|---|---|
| **Default** | On | Allowed in sandbox; ask outside | Workspace + temp | Ask |
| **Request Review** | Off | Always ask | Workspace only | Ask |
| **Turbo** | Off | Allowed, unrestricted | Full filesystem | Allowed |

- **Escape hatch:** the agent can ask to run a command unsandboxed, and approval is always required unless a rule covers it. On macOS and Linux that rule is `command(git push)`. On Windows and in the CLI it is `unsandboxed(git push)`. An "always allow" on a bypass prompt records an `unsandboxed(...)` rule.
- **CLI:** set `"enableTerminalSandbox": true` in `~/.gemini/antigravity-cli/settings.json` (default `false` in the CLI features doc; unverified whether the newer macOS/Linux default preset supersedes this).
- **Windows** still uses the legacy model. Sandbox Mode is Preview, and *none* of the presets turn it on. Enabling it switches the preset to Custom.
- **Headless (`agy -p`):**
  - Tools needing approval are **soft-denied**: the run exits 0 and prints a notice to stderr.
  - Pre-grant tools with `permissions.allow`.
  - `--dangerously-skip-permissions` approves everything. It is a distractor unless the environment is itself disposable and isolated.
- **Claude Code equivalent:**
  - Enable with `/sandbox` or `"sandbox": {"enabled": true}`. It uses Seatbelt on macOS and bubblewrap plus socat on Linux/WSL2.
  - `network.allowedDomains`, `allowUnsandboxedCommands: false` (strict mode), `strictAllowlist`, and `allowManagedDomainsOnly` (managed-settings lockdown) control the boundary.

**Layer 2: Cloud Workstations (managed dev VM, per developer)**
- Workstation images include Gemini CLI. Google's Data Agent Kit is preinstalled in Cloud Shell and Cloud Workstations.
- **Agent-optimized development:**
  - `IdleAction.SUSPEND` suspends the VM (RAM, processes and agent context persisted, compute billing stopped) instead of stopping and deleting it.
  - A lifecycle **keep-alive hook** runs `/google/scripts/keep_alive.sh` while the agent works and kills it when the agent waits for input.
  - Samples ship under `/google/samples/agents/keepalive/{claude,gemini}/`.
  - Claude Code uses the `UserPromptSubmit` hook to start and the `Notification` hook to stop. The docs name `BeforeAgent`/`Notification` for Antigravity CLI.
- **Security controls:**
  - A **private cluster** (Private gateway plus a PSC endpoint, reached over VPN or Interconnect).
  - **VPC Service Controls:** restrict the Compute Engine API whenever you restrict the Cloud Workstations API. Make Cloud Storage and Artifact Registry VPC-accessible. Public clusters are blocked inside a perimeter.
  - **Disable public IPs** (org policy `constraints/compute.vmExternalIpAccess`). Then use Private Google Access or Cloud NAT.
  - **Secure Web Proxy** for auditable, identity-aware egress allowlisting with TLS inspection. Set `https_proxy` in a custom image.
  - `gcloud workstations configs update … --disable-ssh-to-vm` forces access through the IAM-enforced gateway.
  - The workstation config's **service account** is the agent's cloud identity, so scope it minimally.

**Layer 3: GKE Agent Sandbox (server-side, per agent session, untrusted LLM-generated code)**
- A managed GKE add-on based on the OSS `kubernetes-sigs/agent-sandbox` project. It provides **gVisor kernel-level isolation** (Kata Containers is supported but not a Google product), **sub-second provisioning** through warm pools, and **Default Deny networking**. There is no extra charge beyond GKE resources.
- **CRDs** (`extensions.agents.x-k8s.io/v1beta1`, with `Sandbox` in `agents.x-k8s.io/v1beta1`):
  - `SandboxTemplate` is the blueprint.
  - `SandboxWarmPool` holds pre-warmed pods (`replicas`).
  - `SandboxClaim` is the request. In v1beta1 it **must** set `spec.warmPoolRef`. A cold start uses a pool with `replicas: 0`.
  - `Sandbox` is a single stateful pod with a stable hostname and storage, and no warm pool.
  - A **Sandbox Router** provides the stable endpoint.
  - It integrates with **Pod snapshots** for suspend and resume.
- **Enable it:**
  - Autopilot: `gcloud beta container clusters create-auto … --enable-agent-sandbox`.
  - Standard: create a gVisor node pool (`--image-type=cos_containerd --sandbox=type=gvisor`), then `clusters update --enable-agent-sandbox`.
  - Verify with `addonsConfig.agentSandboxConfig.enabled`.
  - Version: 1.36.3-gke.1767000 or later for v1beta1. The concept doc says 1.35.2 or later for snapshots.
- **Required pod spec**, enforced by a Validating Admission Policy (non-compliant manifests are **rejected**):
```yaml
spec:
  runtimeClassName: gvisor
  automountServiceAccountToken: false      # least privilege: no KSA token in untrusted code
  securityContext: { runAsNonRoot: true }
  nodeSelector: { sandbox.gke.io/runtime: gvisor }
  tolerations: [{ key: sandbox.gke.io/runtime, value: gvisor, effect: NoSchedule }]
  containers:
  - resources: { limits: { cpu: "500m", memory: "1Gi" } }   # limits required (DoS)
    securityContext: { capabilities: { drop: ["ALL"] } }
```
- **Prohibited settings:** `hostNetwork`, `hostPID`, `hostIPC`, `privileged`, `hostPath`, `capabilities.add`, `hostPort`, custom sysctls, and projected SA-token volumes.
- **Programmatic access:** the Agentic Sandbox Python client. `kubectl port-forward` to the router is dev-only.
- **Storage:** ephemeral PVC via `volumeClaimTemplates`, or Filestore agent volumes (RWX) for persistent high-density workspaces.

**Sandbox options compared**

| | Antigravity terminal sandbox | Cloud Workstations | GKE Agent Sandbox |
|---|---|---|---|
| Protects | Developer laptop from the agent's shell commands | Corporate code/data; standardizes the dev env | Platform/cluster from **untrusted LLM-generated code** at scale |
| Isolation unit | Per command (OS namespaces / Seatbelt) | Per developer VM (container on COS VM) | Per session pod (gVisor user-space kernel) |
| Egress control | `read_url` allowlist | VPC-SC, no public IP, Secure Web Proxy, Cloud NAT | Default-deny NetworkPolicy in `SandboxTemplate` |
| Identity | Developer's local creds / ADC | Workstation config service account | No KSA token by default; add Workload Identity deliberately |
| Startup | Zero | Minutes (image streaming helps); suspend/resume | <1 s with warm pool |
| Best for | Interactive local coding | Regulated enterprises, "code never leaves the perimeter", long-running agents | Code-interpreter tools, agent runtimes, multi-tenant execution |
| Trade-off | Weakest boundary (shares the host kernel) | VM cost, network setup (PSC, DNS) | Cluster ops burden, warm-pool idle cost, gVisor syscall overhead |

**Limits and gotchas**
- Under the Default preset, the sandbox has **no network** access. `npm install` fails until you add a `read_url(registry.npmjs.org)` grant or approve an unsandboxed run.
- Antigravity's permission system for macOS and Linux differs from the one on Windows.
- In a Workstations VPC-SC perimeter with image streaming, you must add `containerfilesystem.googleapis.com` plus an egress rule.
- Agent Sandbox v1alpha1 → v1beta1 migration uses `migrate.sh`, run as bootstrap → control-plane upgrade → migrate. Phase 3 is the point of no return.
- Skill Registry and Vertex AI calls from a sandbox with no egress **fail**. ADK then falls back to local filesystem skills.

**Exam signals**

| Keyword in the question | Likely answer |
|---|---|
| "execute untrusted model-generated code", "kernel-level isolation", "sub-second" | GKE Agent Sandbox (gVisor + `SandboxWarmPool`) |
| "prevent source-code exfiltration", "developers' cloud IDE", "no public internet" | Cloud Workstations private cluster + VPC-SC + no public IP (+ Secure Web Proxy for an allowlist) |
| "long agent task, stop paying while waiting for human" | Workstations `IdleAction.SUSPEND` + keep-alive hooks |
| "let agent run tests without prompting but protect `~/.ssh`" | Antigravity Default preset (terminal sandbox) |
| "allow the sandbox to reach only the internal package mirror" | `read_url(mirror.corp)` → sandbox egress allowlist |
| "pod must not access Kubernetes API" | `automountServiceAccountToken: false` (required) + default-deny network |

**Distractors**
- "Turbo mode for productivity".
- `privileged: true` "to allow Docker-in-Docker".
- "Run untrusted code on Cloud Run with the default SA" (no kernel sandbox story, and the default SA is over-privileged).
- `--dangerously-skip-permissions` on a developer laptop.
- Sole-tenant nodes (they isolate tenants from each other, not code from the kernel).

---

#### 2.1.c Using coding agents to refactor code, optimize runtimes, and patch app-layer vulnerabilities

**Core patterns (Antigravity)**
- **Plan first for risky change.** Use Planning Mode (Antigravity 2.0), `/plan`, or `agy --mode=plan`. The agent investigates with read-only tools and produces an **Implementation Plan artifact** you can comment on.
- The **Artifact Review Policy** can be "Request Review" (recommended) or "Always Proceed".
- Use **Fast Mode** only for localized edits such as a rename or a small refactor.
- **`--mode=accept-edits`** auto-approves file edits (subagents inherit it). It suits long refactors *after* the plan has been approved.
- **Parallelize safely.** `invoke_subagent` with the workspace option **`branch`** creates an isolated **Git worktree** per subagent. The other options are `inherit` and `share`. Subagents start with a clean context window.
- **`/boost`** is a three-tier multi-agent orchestrator (Orchestrator → DeepCoder / DeepInvestigator → workers) for hard concurrency bugs and non-trivial refactors, with independent verification loops. It requires a paid tier.
- **`/teamwork-preview`** coordinates multiple agents for large multi-file projects, with milestone decomposition and verification.
- **Recovery:** `/rewind` (`/undo`), `/fork`, `/resume`, and `Esc` to interrupt a turn.
- **Deterministic quality gates:** a `PostToolUse` hook on `write_to_file`/`replace_file_content` runs a linter or formatter. A `PreToolUse` hook on `run_command` can `deny`. A `Stop` hook returning `decision: "continue"` keeps the agent working until tests pass — the "don't finish until green" gate (see 2.2.b). `PostInvocation` + `force_continue` also works but fires after every model call.
- **Evidence from tools via MCP:**
  - SonarQube, Wiz, CrowdStrike and Splunk MCP servers for security findings.
  - Chrome DevTools MCP for front-end performance traces.
  - Cloud Trace/Logging data (via the gcloud or observability MCP) for runtime hotspots.
- **CodeMender agent on Agent Platform** is Google's managed code-security agent. It finds vulnerabilities, **validates exploitability by building and running a PoC exploit in a customer-managed environment**, and generates verified patches. It can ingest findings from tools such as Wiz, and a lightweight CLI keeps source code in your environment.
- **CI automation:** `agy -p "…" --output-format json|stream-json --model <slug>`. The status field is `SUCCESS`/`ERROR`/…, and the default timeout of 5 minutes is overridable with `--print-timeout`. Headless mode fails loudly on an unknown model, whereas the UI silently falls back.

**Trade-offs**
- **Planning mode** costs latency and tokens but buys correctness on multi-file changes. **Fast mode** is the reverse.
- **Parallel subagents in worktrees** are faster but need merge and conflict resolution. **A single agent** is slower but coherent.
- **`/boost`** gives higher accuracy on hard problems at far higher cost and time. Don't use it for a rename.
- **LLM patching without exploit validation** is fast but risks false positives. CodeMender-style validation costs compute but removes alert fatigue.

**Exam signals**
- "Ensure every edit is linted/tests pass before the agent continues": **hooks** (`PostToolUse` for per-edit lint, `Stop` → `continue` for the tests-green gate). Rules alone don't enforce anything.
- "Large refactor across 40 services without agents stepping on each other": subagents with `branch` (worktree) isolation, or `/teamwork-preview`.
- "Prioritize only exploitable vulnerabilities and auto-generate verified fixes": **CodeMender**.
- "Human must approve the approach before files change": Planning Mode + Request Review.

**Distractors**
- Tuning the model temperature to "make patches safer".
- Trusting a rule saying "always run tests" as enforcement (it is advisory; a hook enforces).
- Turbo preset on production credentials.

---

### 2.2 Customizing coding agents for enterprise workflows

> 🧠 **Visual memory map:** Coding-agent customization primitives
>
> [![Coding-agent customization primitives](visual-memory/03-coding-agent-primitives.png)](visual-memory/03-coding-agent-primitives.png)

#### 2.2.a Creating skills, plugins, extensions, hooks, rules, and subagents using Antigravity

**Rules (persistent constraints and invariants)**
- **Files:**
  - `AGENTS.md` and `GEMINI.md` have no frontmatter and are always on.
  - `.agents/rules/*.md` needs frontmatter.
  - Global rules live in `~/.gemini/AGENTS.md`, `~/.gemini/GEMINI.md`, `~/.gemini/config/{AGENTS,GEMINI}.md`, and `~/.gemini/config/rules/*.md`. The CLI also reads `~/.gemini/antigravity-cli/rules/`.
  - Plugin rules live in `plugins/<p>/rules/`.
- **Directory scoping:** when the agent touches a file, it walks up to the workspace root and loads `AGENTS.md`/`GEMINI.md`/`.agents/rules/` at each level. Rules are **cumulative**, and the more specific rule wins on conflict.
- **Frontmatter:**
```yaml
---
trigger: model_decision   # always_on | model_decision | glob | manual
description: "Apply when writing DB migrations"   # required for model_decision
globs: "*.proto, **/*.pb.go"                        # required for glob
---
```
- **Gotchas:**
  - A missing frontmatter or an invalid trigger (camelCase `alwaysOn`, `modelDecision`) means the rule is **silently discarded**.
  - `.agents/rules/` is scanned **flat**. Nested dirs are ignored unless listed in `.agents/rules.json` (`inherits`, `entries`, `include_only`, `exclude`), which is also how you share rules across repos.
  - `@[label](path)` inlines a file, while `@file` only canonicalizes a path reference.
- **Limits:** 24,000 bytes per rule file (truncated beyond that), and a **20,000-token aggregate budget** for always-on and global rules. When the budget is exceeded, the largest rules are demoted to pointers (`path: description`).

**Skills (multi-step, on-demand procedures)**
- Covered in 2.1.a.
- **Legacy workflows** (`.agents/workflows/*.md`, 12,000-character limit, whole file loaded) are **deprecated and retired on 2026-11-01**. Migrate with `/migrate-workflows`, which scaffolds `.agents/skills/<name>/SKILL.md` and renames the original to `.bak`. When names collide, the skill wins over the workflow.

**Hooks (deterministic interception)**
- **Files:** `.agents/hooks.json` (workspace) and `~/.gemini/config/hooks.json` (global). The CLI also reads `~/.gemini/antigravity-cli/settings.json` and plugin `hooks.json`. Inspect them with `/hooks` (CLI) or Customizations → Hooks.
- **Events:** `PreToolUse`, `PostToolUse` (both take a `matcher` on the tool name, such as `run_command`, `write_to_file`, `replace_file_content`, `invoke_subagent`, …), `PreInvocation`, `PostInvocation`, `Stop`.
- **Handler:** `{"type":"command","command":"./scripts/x.sh","timeout":30}`. The default timeout is 30 s and only `command` is supported. Add `"enabled": false` to disable a hook.
```json
{ "safety-gate": { "PreToolUse": [ { "matcher": "run_command",
    "hooks": [ { "command": "./scripts/safety-check.sh", "timeout": 10 } ] } ] } }
```
- **I/O contract:** JSON on stdin and stdout, camelCase. The common input fields are `conversationId`, `workspacePaths`, `transcriptPath`, `artifactDirectoryPath`, `modelName`.
  - `PreToolUse` output: `decision` = `allow` | `deny` | `ask` | `force_ask` | `deny_unless_prior_grant`, plus `reason` and `permissionOverrides`.
  - `PreInvocation`/`PostInvocation` can return `injectSteps` (`userMessage`, `ephemeralMessage`, `toolCall`).
  - `PostInvocation` can also return `terminationBehavior: force_continue|terminate`.
  - `PostToolUse` returns `{}` and receives `error` when the tool failed.
- The transcript lives at `<app_data_dir>/brain/<conversationId>/.system_generated/logs/transcript.jsonl`, where `<app_data_dir>` is `~/.gemini/antigravity`, `~/.gemini/antigravity-cli`, or `~/.gemini/antigravity-ide`. It is useful for audit export.

**Subagents (context-isolated specialists)**
- **Files:**
  - `.agents/agents/<name>.md` or `.agents/agents/<name>/agent.md`.
  - `~/.gemini/config/agents/`.
  - `plugins/<p>/agents/`.
  - Created at runtime via the `define_subagent` tool.
- **Frontmatter:** `name`, `description` (both required), `tools` (explicit allowlist), `mainAgent` (default true), `subagent` (default true), `model` (`inherit`|`flash`|`pro`), `commandExecutionPolicy` (`off`|`auto`|`eager`|**`sandbox`** default), `mcpServers`, `skills`/`plugins`. The body is the system prompt.
- **Built-ins:** `research`, `browser` (only via `/browser`, sandboxed Chrome), `self` (a clone).
- **Behavior:**
  - Each subagent runs in its own context window and does **not** inherit the parent's history.
  - Workspace can be `inherit`, `branch` (git worktree), or `share`.
  - Monitor with `/agents` or Alt+J (CLI).
- **Known issue:** a misspelled tool name in `tools` can **hang** the subagent.
- **SDK:** `CapabilitiesConfig(enable_subagents=True)` gives dynamic self-cloning. `types.SubagentConfig(...)` defines static subagents. Custom tools given to a subagent **must also be registered on the parent**.

**Plugins (the distribution unit)**
- **Layout:** `plugin.json` (required; `name` must match `^[a-zA-Z0-9-_]+$`, required for the CLI; `$schema: https://antigravity.google/schemas/v1/plugin.json`), plus optional `mcp_config.json`, `hooks.json`, `skills/`, `agents/`, `rules/`.
- **Install:**
  - Workspace: `.agents/plugins/`. Global: `~/.gemini/config/plugins/`.
  - CLI: `agy plugin list|install <path/url>|enable|disable|uninstall`, staged under `~/.gemini/antigravity-cli/plugins/<name>/`.
  - Antigravity 2.0 → Customizations → **Build with Google** curated plugins, such as Data Agent Kit (`agy plugin install https://github.com/gemini-cli-extensions/data-agent-kit-starter-pack`).

**Extensions (two meanings; the exam may use either)**
1. **Antigravity IDE extensions** put the Antigravity agent inside VS Code (≥1.90), Visual Studio 2026, JetBrains (2026.2.1+), Zed, or Xcode. They auto-install the local `agy` backend. **Enterprise sign-in** (Gemini Enterprise) is supported on Antigravity 2.0, the CLI, and IDE extensions, with JetBrains/Zed/Xcode in Preview. The **standalone Antigravity IDE is not supported for enterprise**.
2. **Gemini CLI extensions** (`gemini-extension.json` with `mcpServers`, `contextFileName`, `excludeTools`, `settings[]` with `envVar`/`sensitive` stored in the keychain, and `commands/*.toml`, `hooks/hooks.json`, `skills/`, `agents/`). In Antigravity these are **plugins**. Convert them with `agy plugin import gemini`, which turns legacy commands into skills.

- Separately, **Agents CLI extensions** (`agents-cli-extension.yaml`, `agents-cli extension add|list|remove|update`) override or add agents-cli commands, for example an org deploy policy or another framework. Experimental.

**When to use which**

| Primitive | Loaded | Enforced? | Use it for | Don't use it for |
|---|---|---|---|---|
| **Rule** (`AGENTS.md`, `.agents/rules`) | Always / glob / model decision / manual | Advisory (prompt) | Invariants: "use zod", "never import internal/" | Multi-step procedures (bloats every turn) |
| **Skill** (`SKILL.md`) | On demand (progressive disclosure) or `/name` | Advisory | Repeatable workflows with scripts/assets: deploy, migrate, review | Hard security guarantees |
| **Hook** (`hooks.json`) | Every matching event | **Deterministic** (runs code, can deny) | Lint/format after write, block commands, audit, keep-alive | Teaching the model "how" |
| **Subagent** (`.agents/agents/*.md`) | When delegated | Tool allowlist + execution policy | Context isolation, parallelism, least-privilege specialist (read-only reviewer) | Tiny tasks (spawn overhead) |
| **MCP server** | Tools listed; called on demand | Permission engine (`mcp(...)`) | Live data and external actions | Static knowledge (use a skill) |
| **Plugin** (`plugin.json`) | Bundles the above | Inherits component semantics | Distributing a team or org standard kit as one unit | A single one-off rule |
| **Extension** | IDE host / legacy Gemini CLI bundle | n/a | Bringing the agent into an existing IDE; migrating Gemini CLI assets | New Antigravity packaging (use plugins) |

**Exam signals**
- "Guarantee" / "enforce" / "block" means a **hook** (or permission deny). "Guideline" / "convention" means a **rule**. "Procedure with scripts" means a **skill**. "Separate context / restricted tools / parallel" means a **subagent**. "Distribute to 200 devs as one install" means a **plugin**. "Our devs use VS Code" means an **IDE extension**.
- "Rule isn't being applied" usually means missing frontmatter, a camelCase trigger, or a nested folder not listed in `rules.json`.
- "Rules suddenly partially ignored in a large monorepo" means the 20k-token budget demoted them to pointers.

#### 2.2.b Deep dive — Antigravity rules, hooks and skills: paths, formats, when to use which

> 🧠 **Visual memory map:** Antigravity customization map
>
> [![Antigravity customization map](visual-memory/16-antigravity-customization.png)](visual-memory/16-antigravity-customization.png)

This subsection goes one level deeper than 2.2.a. It covers exact paths, file formats, activation semantics and the hook I/O contract, because exam questions often turn on one path or one field value. Everything below comes from the antigravity.google docs (rules, skills, hooks, subagents, plugins, MCP, permissions, CLI settings, workflows migration, Gemini CLI migration) unless it is marked **(unverified)**.

**Surface availability (a detail that can decide a question)**

| Feature | Antigravity 2.0 (app) | Antigravity CLI (`agy`) | Antigravity IDE |
|---|---|---|---|
| Rules, skills, hooks, MCP, plugins | Yes | Yes | Yes |
| Custom subagents (`.agents/agents/`) | Yes | Yes | Not listed on the subagents page |
| New permission engine (`action(target)`) | Yes (macOS/Linux) | Yes | Not listed on the permissions page |
| `/migrate-workflows` | Yes (the docs say "Open Antigravity 2.0") | — | Workflows originated here |

**1. File-system map**

Workspace scope. Commit it to the repo and the whole team shares it.

```text
<repo>/
├── AGENTS.md                  # rule, no frontmatter, always on (directory scope)
├── GEMINI.md                  # same semantics as AGENTS.md
├── .agents/                   # default customization dir (legacy .agent/ still read)
│   ├── AGENTS.md | GEMINI.md  # also discovered here
│   ├── rules/                 # FLAT scan: only immediate *.md children
│   │   ├── typescript.md      # YAML frontmatter with `trigger:` REQUIRED
│   │   └── frontend/react.md  # IGNORED unless listed in rules.json
│   ├── rules.json             # inherits / entries / include_only / exclude
│   ├── skills/
│   │   └── <skill-name>/
│   │       ├── SKILL.md       # required (frontmatter: description required)
│   │       ├── scripts/       # optional executables
│   │       ├── references/    # optional docs (Antigravity docs also show examples/)
│   │       └── assets/        # optional templates/data (Antigravity docs: resources/)
│   ├── hooks.json             # workspace hooks
│   ├── agents/
│   │   ├── <name>.md          # custom subagent (frontmatter + system prompt)
│   │   └── <name>/agent.md    # folder form (what the CLI /agents panel suggests)
│   ├── plugins/
│   │   └── <plugin>/plugin.json …
│   ├── mcp_config.json        # workspace MCP servers (also auto-discovered by the SDK)
│   └── workflows/<name>.md    # LEGACY: deprecated, retired 2026-11-01
└── services/payments/
    ├── AGENTS.md              # directory-scoped rule, loaded when the agent touches files here
    └── .agents/rules/*.md     # directory-scoped modular rules
```

Global (user) scope. It applies to every workspace on the machine. The shared home `~/.gemini/config/` is used by all three surfaces. The CLI adds its own home.

```text
~/.gemini/
├── AGENTS.md | GEMINI.md          # global always-on rules (no frontmatter)
├── config/                        # shared by 2.0, CLI and IDE
│   ├── AGENTS.md | GEMINI.md      # also global always-on
│   ├── rules/*.md                 # modular global rules (frontmatter required)
│   ├── skills/<name>/SKILL.md     # global skills: 2.0 + IDE (CLI uses its own path, below)
│   ├── hooks.json                 # global hooks (all surfaces)
│   ├── agents/<name>.md | <name>/agent.md   # global subagents
│   ├── plugins/<plugin>/          # global plugins: 2.0 + IDE (manual drop-in)
│   ├── mcp_config.json            # global MCP servers (all surfaces)
│   └── workflows/<name>.md        # LEGACY global workflows
├── antigravity-cli/               # CLI home = CLI <app_data_dir>
│   ├── settings.json              # permissions{allow,deny,ask}, toolPermission,
│   │                              #   enableTerminalSandbox, hooks (also allowed here) …
│   ├── keybindings.json
│   ├── rules/*.md                 # extra global rules (CLI only)
│   ├── skills/<name>/SKILL.md     # global skills for the CLI
│   ├── plugins/<plugin>/          # where `agy plugin install` stages plugins
│   │   └── rules/ skills/ agents/ hooks.json mcp_config.json
│   └── brain/<conversationId>/.system_generated/logs/transcript.jsonl
├── antigravity/                   # 2.0 <app_data_dir>
│   ├── mcp_oauth_tokens.json      # MCP OAuth tokens
│   ├── skills/                    # LEGACY global skills (IDE still reads it)
│   └── brain/<conversationId>/…   # transcripts + artifacts
└── antigravity-ide/               # IDE <app_data_dir> (transcripts)
```

Plugin layout. A plugin is the distribution unit, and it has the same shape wherever it is installed.

```text
<plugin-name>/
├── plugin.json        # REQUIRED marker/manifest: {"$schema","name","description"}
├── mcp_config.json    # optional
├── hooks.json         # optional
├── skills/<skill>/SKILL.md
├── agents/<agent>.md
└── rules/<rule>.md
```
- `plugin.json` allows only `name` and `description` (`additionalProperties: false`). `name` matches `^[a-zA-Z0-9-_]+$`. It is **required for the CLI**. In 2.0 and the IDE it defaults to the folder name. `$schema` is `https://antigravity.google/schemas/v1/plugin.json`.
- **Permissions location.** The CLI keeps them in `~/.gemini/antigravity-cli/settings.json`. Antigravity 2.0 sets them in **Settings → General → Permission Settings**, with per-project overrides under **Settings → Projects**. The backing file path for 2.0 is not documented **(unverified)**. No workspace-level permissions file is documented.

**Precedence and merge rules between scopes**

| Primitive | How scopes combine | Conflict resolution |
|---|---|---|
| Rules | **Cumulative.** Global, workspace and directory rules are all combined into the prompt | More specific directory rules win. Documented precedence order: directory-scoped, then global |
| Skills | Workspace and global are both listed | A skill beats a legacy workflow with the same name. Same-named skills in two scopes: not documented **(unverified)** |
| Hooks | Workspace, global, CLI `settings.json` and plugin `hooks.json` all load (`/hooks` shows the effective set) | Ordering between multiple matching hooks is not documented **(unverified)** |
| MCP | Global plus workspace `mcp_config.json` (plus plugin) | Same server name in two scopes: not documented **(unverified)** |
| Permissions | Preset (Default / Request Review / Turbo), with explicit rules layered on top | **Deny > Ask > Allow**. Explicit rules always beat preset defaults. A subagent inherits its parent's scopes and sandbox |

**2. Rules (in depth)**

- **Two file types:**
  - `AGENTS.md` / `GEMINI.md` are plain Markdown with **no frontmatter**. They are always on for their directory scope.
  - `rules/*.md` must **start with YAML frontmatter** that declares a valid `trigger`. A missing frontmatter or an invalid value (`alwaysOn`, `modelDecision`) means the rule is **silently discarded**.
- **Frontmatter fields:**

| Field | Required | Notes |
|---|---|---|
| `trigger` | Yes | `always_on` \| `model_decision` \| `glob` \| `manual` (snake_case, exact) |
| `description` | Required for `model_decision`, recommended for all | For `model_decision`, it becomes the index entry the model sees. It is also the fallback pointer text when an `always_on` rule is demoted by the budget |
| `globs` (singular `glob` also accepted) | Required for `glob` | A comma-separated string, e.g. `"*.proto, **/*.pb.go"`. **Quote patterns that start with `*`**, because YAML treats `*` as an alias anchor |

- **Activation modes: what is loaded, and when:**

| `trigger` | Loaded up front | Full body enters context | Best for |
|---|---|---|---|
| `always_on` | Full content, **every turn** | Always | Short, universal invariants. The docs prefer putting these in `AGENTS.md` |
| `model_decision` | Only **path + `description`** (progressive disclosure) | When the agent judges the task matches the description | Long domain guides (migrations, API style) that are only sometimes relevant |
| `glob` | Nothing until triggered | When the agent **interacts with a file matching `globs`** | Language- or file-type-specific conventions (`*.proto`, `*.tf`) |
| `manual` | Nothing | **Only when you `@`-mention it in chat** | Release checklists, audit rubrics, one-off playbooks |

- **Nesting and discovery.** When Antigravity reads or edits a file, it walks **up from that file's folder to the workspace root**. At each level it loads `<dir>/AGENTS.md|GEMINI.md`, `<dir>/.agents/AGENTS.md|GEMINI.md`, and `<dir>/.agents/rules/*.md` (legacy `<dir>/.agent/rules/*.md`). A monorepo can therefore give `services/payments/` its own rules, and they appear only when the agent works there.
- **Flat scan.** `.agents/rules/frontend/react.md` is ignored unless it is registered in `.agents/rules.json`:
```json
{ "inherits": [ { "path": "../shared-config/.agents/rules.json" } ],
  "entries":  [ { "path": "../shared-rules", "exclude": ["deprecated_rules.md", "legacy/"] },
                { "path": "rules", "include_only": ["frontend/react.md"] } ] }
```
  `rules.json` is also the documented way to **share rules across repos** (`inherits`), keeping their frontmatter and triggers intact.
- **Includes:**
  - `@[label](path)` **inlines** the target file before prompt evaluation and size checks. The path is relative to the rule file, and `~/` expands to home. Frontmatter in the included file is stripped.
  - `@path` does **not** inline anything. It rewrites the reference to a canonical absolute workspace path (`@/workspace/path`).
- **Limits:**
  - **24,000 bytes per rule file**, measured after includes are expanded. Anything beyond that is truncated.
  - A **20,000-token aggregate budget** for all active **global + `always_on`** rules. This is separate from the "customization budget" used by skills and MCP. When it is exceeded, the **largest** files are demoted to pointers (`- <path>: <description>`) that the agent reads on demand.
- **UI.** 2.0: Customizations → **Rules** tab → **+ Global** / **+ Workspace**. IDE: **…** → Customizations → Rules. CLI: edit the files directly.
- **Example (`.agents/rules/db-migrations.md`):**
```markdown
---
trigger: model_decision
description: "Apply whenever writing or reviewing database migrations or SQL schema changes."
---
# Database migration rules
1. Never drop or rename a column in one deployment — use expand-and-contract.
2. Always create indexes on existing PostgreSQL tables with CONCURRENTLY.
@[Schema conventions](../../docs/schema-conventions.md)
```

**3. Skills (in depth)**

- **Standard.** Skills follow the open Agent Skills standard (agentskills.io), which Antigravity adopted in May 2026. A skill is a **directory** containing `SKILL.md`.
- **Frontmatter:**

| Field | Antigravity docs | Open spec (agentskills.io) |
|---|---|---|
| `name` | Optional. Lowercase with hyphens. **Defaults to the folder name** | Required. 1–64 chars, `a-z0-9-`, no leading, trailing or double hyphens, **must match the folder name** |
| `description` | **Required.** It is what the agent sees when deciding. Write it in third person with trigger keywords | Required, 1–1024 chars |
| `license`, `compatibility` (≤500 chars), `metadata` (string map), `allowed-tools` (experimental) | Not documented by Antigravity | Optional. Whether Antigravity honors `allowed-tools` is **(unverified)** |

  **Portability tip:** always set `name` equal to the folder name. Antigravity doesn't require it, but other skill-compliant tools and Skill Registry validation (≤64 chars, lowercase/hyphen, `description` ≤1024) do.
- **Progressive disclosure (three layers):**
  1. **Discovery.** At conversation start, only `name` + `description` for every skill are in context (the spec budgets about 100 tokens per skill).
  2. **Activation.** When the task matches, the agent reads the whole `SKILL.md` body (the spec recommends under 5,000 tokens and under 500 lines).
  3. **Resources.** Files in `scripts/`, `references/`, `assets/` (or `examples/`, `resources/`) are read or run **only when the body tells the agent to**. The docs' best practice is to treat scripts as **black boxes**: run `script --help` rather than reading the source, which saves context. Scripts run through the normal `run_command` tool, so they go through **permissions, the terminal sandbox and `PreToolUse` hooks** like any other command.
- **Invocation:**
  - **Autonomous:** the model matches on the description.
  - **Explicit:** `/<skill-name>` in 2.0 and the CLI. The CLI auto-creates a slash command for every skill, and `/skills` lists them. You can also mention the skill by name in the prompt.
- **Locations:** workspace `.agents/skills/<name>/`. Global `~/.gemini/config/skills/<name>/` (2.0, IDE, plus the legacy `~/.gemini/antigravity/skills/` in the IDE). CLI global `~/.gemini/antigravity-cli/skills/<name>/`. Plugin skills in `<plugin>/skills/`.
- **Sharing and distribution, from narrowest to widest:**
  1. **Commit** `.agents/skills/` to the repo.
  2. Ship a **plugin**: drop it in `.agents/plugins/` or `~/.gemini/config/plugins/`, or run `agy plugin install <path|git-url>`. For example, `agy plugin install https://github.com/GoogleChrome/modern-web-guidance`.
  3. Use **Build with Google** curated bundles: Settings → Customizations → Build with Google Plugins (for example Firebase, Modern Web Guidance, Antigravity SDK).
  4. **Agents CLI:** `uvx google-agents-cli setup` installs the `google-agents-cli-*` skills into every detected coding agent (Antigravity, Claude Code, Codex, Cursor, …). The default is global, and `--workspace` installs project-level. `npx skills add google/agents-cli` installs the skills only.
  5. **Skill Registry / Agent Registry** hold governed skills for **ADK agents at runtime**, through `SkillToolset` + `GCPSkillRegistry`, not for loading into the Antigravity IDE (see 2.2.c). **Exam trap:** the registry answers "govern and version skills for deployed agents". A plugin answers "distribute skills to developers' coding agents".
- **Example (`.agents/skills/deploy-staging/SKILL.md`):**
```markdown
---
name: deploy-staging
description: Deploys the current build to the staging Cloud Run service and smoke-tests it. Use when asked to deploy, release to staging, or verify a PR preview.
---
# Deploy to staging
1. Run `scripts/preflight.sh --help`, then `scripts/preflight.sh` (lint + unit tests). Stop on failure.
2. Build and deploy with `scripts/deploy.sh staging` — do not hand-write gcloud flags.
3. Run the smoke checks listed in `references/smoke-checks.md` and report results.
```

**4. Hooks (in depth)**

- **Files:**
  - Workspace: `.agents/hooks.json`.
  - Global: `~/.gemini/config/hooks.json`.
  - The CLI also reads a `hooks` section in `~/.gemini/antigravity-cli/settings.json`.
  - Plugins can ship their own `hooks.json`.
  - To inspect them, use `/hooks` (CLI), Settings → Customizations → Hooks (2.0), or **…** → Customizations → Hooks (IDE).
- **Schema.** The top-level keys are **named hooks**. Each named hook maps event names to handler arrays. `"enabled": false` disables a hook without deleting it.
```json
{
  "block-secret-writes": {
    "PreToolUse": [
      { "matcher": "write_to_file|replace_file_content|multi_replace_file_content",
        "hooks": [ { "type": "command", "command": "./scripts/hooks/deny-secrets.sh", "timeout": 5 } ] }
    ]
  },
  "tests-must-pass": {
    "Stop": [ { "type": "command", "command": "./scripts/hooks/tests-gate.sh", "timeout": 120 } ]
  },
  "lint-reminder": {
    "enabled": false,
    "PreInvocation": [ { "type": "command", "command": "./scripts/hooks/reminder.sh" } ]
  }
}
```
  - **Tool events** (`PreToolUse`, `PostToolUse`) take a list of `{ "matcher": <regex>, "hooks": [handlers] }`. The matcher is a **regex on the tool name**: `""` or `"*"` means all tools, `"run_command|view_file"` matches either, and `"browser_.*"` matches a prefix.
  - **Lifecycle events** (`PreInvocation`, `PostInvocation`, `Stop`) take a **plain list of handlers**, and any matcher is ignored.
  - A **handler** is `type` (optional, only `"command"`), `command` (required), and `timeout` (**seconds**, default **30**).
  - Matchable tool names include `view_file`, `write_to_file`, `replace_file_content`, `multi_replace_file_content`, `list_dir`, `find_by_name`, `grep_search`, `search_web`, `read_url_content`, `run_command`, `manage_task`, `schedule`, `list_permissions`, `ask_permission`, `invoke_subagent`, `define_subagent`, `send_message`, `manage_subagents`, `ask_question`, `generate_image`. The docs don't give the matcher name format for MCP tools **(unverified)**.
- **Events and contract.** Input arrives as JSON on **stdin** and output goes as JSON to **stdout**, all in **camelCase**. Every input carries `conversationId`, `workspacePaths`, `transcriptPath`, `artifactDirectoryPath` and `modelName`.

| Event | Fires | Extra input | Output (stdout) | What it can do |
|---|---|---|---|---|
| `PreToolUse` | Before a tool executes | `toolCall{name,args}`, `stepIdx` | **`decision` (required)**: `allow` \| `deny` \| `ask` \| `force_ask` \| `deny_unless_prior_grant`. Optional: `reason`, `permissionOverrides` (e.g. `["command(npm test)"]`) | **Block** (hard deny), auto-approve, force a prompt even if "Always Allow" was cached, or only allow what a user granted before |
| `PostToolUse` | After a tool completes | `toolCall`, `stepIdx`, `error` (set if the tool failed) | `{}` | **Observe only**: lint/format, audit logs, notify. It can't undo or hide the result |
| `PreInvocation` | Before each model call | `invocationNum`, `initialNumSteps` | `injectSteps`: `[{userMessage}\|{ephemeralMessage}\|{toolCall}]` | Inject reminders or context, or force a tool call before the model thinks |
| `PostInvocation` | After each model call | Same as `PreInvocation` | `injectSteps`, `terminationBehavior`: `force_continue` \| `terminate` \| `""` | Keep the loop going, or kill it |
| `Stop` | When the execution loop terminates | `executionNum`, `terminationReason` (`model_stop`, `max_steps_exceeded`, `error`), `error`, `fullyIdle` | **`decision` (required)**: `"continue"` re-enters the loop, and `reason` is injected as a system message. Any other value allows the stop | **Definition-of-done gate**: don't let the agent finish while tests fail |

- **Exit codes.** Unlike Gemini CLI and Claude Code (where **exit code 2 = block**), the Antigravity docs define blocking **only through the JSON `decision`**. Behavior on a non-zero exit, a timeout or malformed JSON is **not documented (unverified)**, so design hooks to **fail closed**: always print a valid decision, and treat a script error as `deny`.
- **What hooks can and cannot enforce.** They **can** deterministically gate any tool call (commands, file writes, subagent spawns, web fetches), run formatters and linters, write audit logs (they get `transcriptPath` for export), inject steps, and stop the agent from declaring "done". They **cannot** rewrite tool arguments (there is no documented equivalent of Gemini CLI's `tool_input` override), redact a tool's output, or filter the model's tool list. Hooks also run with **your user privileges**, so a committed workspace `hooks.json` is code execution: review it like CI config.
- **Example: deny writes to secrets** (`scripts/hooks/deny-secrets.sh`):
```bash
#!/usr/bin/env bash
# PreToolUse on file-write tools: hard-deny secrets, allow everything else.
set -euo pipefail
payload="$(cat)"
target="$(jq -r '.toolCall.args.TargetFile // empty' <<<"$payload")"
if [[ "$target" =~ (^|/)(\.env[^/]*|secrets/|.*\.pem$|.*id_rsa) ]]; then
  echo "blocked write to $target" >> .agents/hook-audit.log   # audit trail
  printf '{"decision":"deny","reason":"Writes to secret files are blocked by policy (%s)."}\n' "$target"
else
  echo '{"decision":"allow"}'
fi
```
  **Design note:** the same "never write secrets" requirement can also be met without code by `deny: ["write_file(.env)", …]` in permissions. Use the hook when you need **logic, logging or dynamic decisions**. Use a permission rule when a static pattern is enough (principle 2: the simpler control wins).
- **Example: tests must pass before "done"** (`scripts/hooks/tests-gate.sh`, on `Stop`):
```bash
#!/usr/bin/env bash
payload="$(cat)"
# Only gate a normal, fully idle stop; cap retries to avoid an infinite loop.
[[ "$(jq -r '.fullyIdle' <<<"$payload")" == "true" ]] || { echo '{"decision":"stop"}'; exit 0; }
[[ "$(jq -r '.executionNum' <<<"$payload")" -lt 4 ]]  || { echo '{"decision":"stop"}'; exit 0; }
if out="$(npm test --silent 2>&1)"; then
  echo '{"decision":"stop"}'
else
  jq -n --arg r "Tests are failing — fix them before finishing: ${out: -1500}" '{decision:"continue", reason:$r}'
fi
```
  **Precision vs 2.1.c:** `PostInvocation` + `force_continue` also keeps the loop alive, but it fires after **every** model call. The `Stop` hook with `decision: "continue"` is the event built for the "don't finish until green" gate.

**5. Subagents, plugins, workflows, MCP, permissions (the essentials, with paths)**

- **Subagents** live in `.agents/agents/<name>.md` or `<name>/agent.md`, in `~/.gemini/config/agents/`, or in `<plugin>/agents/`. They can also be created at runtime by `define_subagent`.
  - **Frontmatter:** `name` and `description` are required. Optional fields are `tools` (explicit allowlist; a misspelled name can **hang** the subagent), `mainAgent`/`subagent` (default `true`), `model` (`inherit`\|`flash`\|`pro`), `commandExecutionPolicy` (`off`\|`auto`\|`eager`\|`sandbox`, default `sandbox`), `mcpServers`, and `skills`/`plugins`. The body is the system prompt.
  - **Behavior:** a subagent gets a clean context (it does not inherit history) and can use workspace `inherit`\|`branch` (git worktree)\|`share`. Nesting is capped at **10 levels**. It inherits the parent's command prefixes, file scopes and sandbox, and permission prompts **bubble up** to the parent.
  - **Managing them:** `/agents` (CLI) with **Alt+J** to jump to the next subagent awaiting approval. The built-ins are `research`, `browser` (`/browser` only) and `self`.
- **Plugins:** see the layout above. The CLI commands are `agy plugin list | install <path|url> | enable | disable | uninstall <name>` and `agy plugin import gemini`, which converts Gemini CLI extensions and turns their legacy `commands` into skills.
- **Workflows (deprecated):**
  - **Paths:** `.agents/workflows/<name>.md` and `~/.gemini/config/workflows/<name>.md`. A workflow is a single file of up to **12,000 characters**, loaded **in full**, and invoked as `/<workflow-name>`. Workflows can call other workflows.
  - **Retirement:** they are **retired on 2026-11-01**, after which they are no longer indexed or executable.
  - **Migration:** `/migrate-workflows` scans both paths, scaffolds `.agents/skills/<name>/SKILL.md`, and renames each original to `.md.bak`. On a name clash, the **skill wins**.
- **MCP:** use `mcp_config.json` (global `~/.gemini/config/`, workspace `.agents/`, or a plugin). The file holds `mcpServers`, and each server has `command` or `serverUrl` (never `url`/`httpUrl`), with optional `args`, `env`, `cwd`, `headers`, `authProviderType: "google_credentials"`, `oauth`, `disabled` and `disabledTools`. Details are in 2.1.a.
- **Permissions:** rules take the form `action(target)` with the actions `read_file`, `write_file`, `read_url`, `execute_url`, `command`, `mcp` (plus `unsandboxed` on Windows and in the CLI). They are evaluated **Deny > Ask > Allow**, and a hook's `permissionOverrides` can supply extra grants per call. Details are in 2.1.a and 2.1.b.

**6. When to use which**

| Requirement shape | Use | Exact artifact | Why not the others |
|---|---|---|---|
| Invariant that should shape *every* answer ("use zod", "no `internal/` imports") | **Rule**, `always_on` or `AGENTS.md` | `AGENTS.md` / `.agents/rules/x.md` | A skill might not trigger. A hook can't teach style |
| Long guide that's only sometimes relevant | **Rule**, `model_decision` | `trigger: model_decision` + good `description` | `always_on` burns the 20k budget every turn |
| Conventions for one file type | **Rule**, `glob` | `trigger: glob`, `globs: "*.tf"` | Loads only when those files are touched |
| Checklist used only on request | **Rule**, `manual` (`@`-mention), or a skill invoked by `/name` | `trigger: manual` | Pick the skill if it needs scripts or steps |
| Repeatable multi-step procedure, especially with scripts or templates | **Skill** | `.agents/skills/<n>/SKILL.md` + `scripts/` | A rule bloats context and has no bundled assets. A workflow is deprecated |
| "Must never happen", "always", "audit every…" | **Hook** (`PreToolUse` deny) or **permission deny** | `.agents/hooks.json` / `permissions.deny` | Rules and skills are advisory prompt text |
| "Don't finish until tests pass" | **Hook** (`Stop` → `continue`) | `Stop` handler | A rule saying "run tests" is only a request |
| Auto-format or lint after edits | **Hook** (`PostToolUse`) | matcher `write_to_file\|replace_file_content\|multi_replace_file_content` | — |
| Separate context, least-privilege tools, parallel work | **Subagent** | `.agents/agents/<n>.md` (`tools`, `commandExecutionPolicy`) | A skill runs inside the main context with the main agent's tools |
| Live system data or external actions | **MCP server** | `mcp_config.json` + `mcp(server/tool)` perms | A skill carries knowledge, not live access |
| Ship rules + skills + hooks + MCP + agents to many devs as one unit | **Plugin** | `plugin.json` + component folders | Copying files per repo drifts |
| Share only rules across repos | `rules.json` `inherits` / `entries` | `.agents/rules.json` | A plugin is heavier if you only need rules |

**Keyword → primitive signals**

| Phrase in the question | Answer |
|---|---|
| "coding standard", "convention", "always follow", "project context" | Rule / `AGENTS.md` |
| "only for `*.sql` files", "when editing Terraform" | Rule with `trigger: glob` + `globs` |
| "only when relevant", "reduce tokens", "detailed guide" | Rule `model_decision`, or a skill (progressive disclosure) |
| "only when explicitly requested", "audit rubric" | Rule `trigger: manual` (`@`-mention) |
| "step-by-step procedure", "bundle a script", "reusable across tools", "slash command" | Skill |
| "guarantee", "block", "prevent", "enforce", "regardless of the prompt", "log every" | Hook (`PreToolUse` → `deny`) and/or a `deny` permission |
| "before the agent declares done", "keep working until" | `Stop` hook (`decision: "continue"`) |
| "inject a reminder every turn" | `PreInvocation` hook (`injectSteps` → `ephemeralMessage`) |
| "read-only reviewer", "restricted tools", "own context", "parallel" | Subagent |
| "one install", "distribute to the org", "bundle" | Plugin |
| "legacy workflow", "12,000 characters", "Nov 2026" | Migrate to skills (`/migrate-workflows`) |

**Common distractors**
- `trigger: alwaysOn` / `modelDecision`. camelCase is **silently dropped**, so use `always_on` / `model_decision`.
- An **unquoted** `globs: *.ts`. YAML reads `*` as an alias, so write `globs: "*.ts"`.
- Rules in nested folders under `.agents/rules/` without `rules.json`.
- Putting a hook in `.agents/settings.json` (not a documented location). Hooks live in `hooks.json`. Only the CLI's `~/.gemini/antigravity-cli/settings.json` also accepts them.
- Blocking in Antigravity with **exit code 2**, or `decision: "block"`. Those are Gemini CLI and Claude Code conventions. Antigravity uses JSON `decision: "deny"`.
- Expecting `PostToolUse` to block. It returns `{}`, and the tool has already run.
- A hook `timeout` in **milliseconds**. Antigravity uses **seconds** (default 30). Gemini CLI uses ms (default 60000).
- Putting CLI global skills in `~/.gemini/config/skills/`. The CLI docs list `~/.gemini/antigravity-cli/skills/`. The `.gemini/skills/` folder from Gemini CLI must be **moved** to `.agents/skills/`.
- Skill Registry as the way to give **developers' IDE agents** a skill. It serves ADK/runtime agents. Use a plugin or the repo instead.
- `url` / `httpUrl` in `mcp_config.json`. Use `serverUrl`.

**Antigravity vs Gemini CLI vs Claude Code (paths and semantics)**

| Concern | Antigravity | Gemini CLI | Claude Code (Anthropic docs) |
|---|---|---|---|
| Always-on project context | `AGENTS.md` / `GEMINI.md` (any dir, walked up) | `GEMINI.md` (hierarchical) | `CLAUDE.md` (+ `CLAUDE.local.md`) |
| Global context | `~/.gemini/{AGENTS,GEMINI}.md`, `~/.gemini/config/rules/` | `~/.gemini/GEMINI.md` | `~/.claude/CLAUDE.md` |
| Scoped / modular rules | `.agents/rules/*.md`, `trigger` + `globs` | Subdirectory `GEMINI.md` files | `.claude/rules/*.md` with `paths:` frontmatter |
| Skills | `.agents/skills/` · `~/.gemini/config/skills/` · CLI `~/.gemini/antigravity-cli/skills/` | `.gemini/skills/` · `~/.gemini/skills/` | `.claude/skills/` · `~/.claude/skills/` |
| Custom slash commands | Skills (`/name`); workflows deprecated | `.gemini/commands/*.toml` | Skills (legacy `.claude/commands/*.md`) |
| Hooks config | `.agents/hooks.json` · `~/.gemini/config/hooks.json` | `hooks` in `.gemini/settings.json` / `~/.gemini/settings.json` | `hooks` in `.claude/settings.json` / `~/.claude/settings.json` / managed settings |
| Hook events | `PreToolUse`, `PostToolUse`, `PreInvocation`, `PostInvocation`, `Stop` | `BeforeTool`, `AfterTool`, `BeforeAgent`, `AfterAgent`, `BeforeModel`, `AfterModel`, `BeforeToolSelection`, `SessionStart`, `SessionEnd`, `PreCompress`, `Notification` | `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `SessionStart`, `SessionEnd`, `Notification`, `PreCompact` (among others) |
| Block semantics | JSON `decision: "deny"` | Exit 2 or `decision: "deny"`/`"block"` | Exit 2 or JSON permission decision `deny` |
| Hook timeout unit | Seconds (default 30) | Milliseconds (default 60000) | Seconds |
| Subagents | `.agents/agents/<n>.md` | `.gemini/agents/*.md` **(unverified)** | `.claude/agents/*.md` · `~/.claude/agents/` |
| MCP | `.agents/mcp_config.json` · `~/.gemini/config/mcp_config.json` (`serverUrl`) | `mcpServers` in `settings.json` (`url`/`httpUrl`) | `.mcp.json` (project) · `claude mcp add --scope user` |
| Packaging | Plugin (`plugin.json`) | Extension (`gemini-extension.json`) | Plugin (`.claude-plugin/plugin.json`) |
| Permissions | `permissions.{allow,deny,ask}`, `action(target)` | Policy/settings **(not covered here)** | `permissions.{allow,ask,deny}`, e.g. `Bash(npm test)` |

#### 2.2.c Augmenting Antigravity with Agents CLI (build, scale, govern, optimize deployed agents)

**What Agents CLI is**
- It is the **"Agents CLI in Agent Platform"** (`google-agents-cli`, GitHub `google/agents-cli`). It is **not itself a coding agent**. It is a machine-readable CLI plus **skills** that give any coding agent (Antigravity, Claude Code, Codex, Gemini CLI, Cursor) expert ADK, eval and deploy knowledge.
- **Install** with `uvx google-agents-cli setup`, which is the *only* command you run yourself. It installs `agents-cli`, the ADK packages, and the skills into detected coding agents. `--workspace` installs project-level (default is global), `--agent` targets specific agents, and `agents-cli update` refreshes. Alternatives: `pipx`/`pip install google-agents-cli`, or skills only with `npx skills add google/agents-cli`.
- **Bundled skills:** `google-agents-cli-workflow`, `-adk-code`, `-scaffold`, `-eval`, `-deploy`, `-publish` (Gemini Enterprise registration), `-observability`.

**Lifecycle mapped to commands**

| Stage | Commands | Notes |
|---|---|---|
| Build | `agents-cli create <name> [--prototype] [--deployment-target agent_runtime\|cloud_run\|gke\|none] [--session-type in_memory\|cloud_sql\|agent_platform_sessions] [--cicd-runner google_cloud_build\|github_actions\|skip]`, `scaffold enhance\|upgrade`, `install`, `lint --fix`, `playground` (ADK web UI, :8080), `run "msg" [--mode adk\|adk_live\|a2a] [--url]` | The agent writes `DESIGN_SPEC.md` first. Project files: `app/agent.py`, `fast_api_app.py`, `tests/eval/`, `agents-cli-manifest.yaml`, `Dockerfile`, `GEMINI.md`, `.env` |
| Evaluate / optimize | `eval run` (= `generate` + `grade`), `eval compare`, `eval dataset synthesize`, `eval metric list`, `eval analyze\|optimize` (experimental) | Evalset at `tests/eval/evalsets/*.evalset.json`, LLM-as-judge config in `tests/eval/eval_config.json`; results as JSON under `artifacts/` |
| Deploy / scale | `deploy [--deployment-target …] [--min-instances --max-instances --concurrency --memory --cpu] [--dry-run] [--no-wait]`, `build` | Reads `deployment_target` from `pyproject.toml`. Network flags: `--network-attachment`, `--dns-peering-domain`, `--agent-gateway-egress/ingress` |
| Infra / CI-CD | `infra single-project [--apply]` (SA, bucket, BQ dataset for telemetry), `infra setup-cicd` | Terraform-based; **review generated IaC** before production |
| Govern / publish | `publish gemini-enterprise` | Registers the deployed agent in Gemini Enterprise |
| Observe | Cloud Trace on by default; "set up observability" provisions prompt-response logging to GCS/BQ | |

- **IAM:** **Agent Platform User** is enough to deploy to Agent Runtime. **Owner** is needed for the full production setup (Terraform, CI/CD, IAM). The docs recommend an empty project.
- **Modes:** `-i/--interactive` (required for `login`) or `-y/--yes/--auto-approve` for non-interactive agent-driven runs. `cmd-info --json` gives machine-readable output. The exam guide's phrase "agent vs. human mode" (Section 3.1) most likely maps to this non-interactive, auto-approve, JSON-output path for agents versus the interactive prompts for humans (**unverified**, since no doc uses the term).
- **Where Skill Registry fits in governance:**
  - **Skill Registry** (Agent Platform, Preview) is a private, low-latency repository of skills. A **Skill** is mutable. A **Skill revision** is an immutable snapshot.
  - Create with `POST https://LOCATION-aiplatform.googleapis.com/v1beta1/projects/P/locations/L/skills?skillId=…` and body `{displayName, description, zippedFilesystem: base64}`, or with `agentplatform.Client(...).skills.create(...)`.
  - `SKILL_ID` must be 1–63 lowercase characters, cannot start with `gcp-`, and is reserved forever.
  - **Payload validation** is async. It requires a zip with a root `SKILL.md`. Limits: ≤10 MB zipped, ≤500 MB unzipped, ≤10,000 entries, depth ≤8, compression ratio ≤100, no symlinks or `..`. The `name` must be ≤64 characters, lowercase and hyphenated. The `description` must be ≤1024 characters.
  - The built-in skill **`gcp-skill-registry`** appears after the first API call.
  - **ADK consumption:** `SkillToolset(skills=[], registry=GCPSkillRegistry(project_id, location))` adds `search_skills` and `load_skill`, and loaded skills are cached in session state.
  - **Agent Registry** can also govern standalone skills: `gcloud alpha agent-registry skills create ID --payload=zip` or `--gcs-source-uri`. It prefixes `private-` to the ID. It needs `roles/agentregistry.user`, and the service agent `service-PROJECT_NUMBER@gcp-sa-agentregistry.iam.gserviceaccount.com` needs `storage.objects.get` for the GCS import. **Its limits differ:** 500 KB zipped, 10 MB uncompressed, 1 MB per file.

**Trade-offs**
- **Agents CLI scaffold vs hand-rolled ADK.** The scaffold gives an opinionated, production-shaped project with evals, CI/CD and Terraform, but it is more files and more opinion. For learning or throwaway work, `adk create` is simpler (per principle 2).
- **`--prototype`** is fast but has no CI/CD or IaC. The full scaffold is audit-ready but needs Owner and more setup.
- **Skills bundled in the repo vs Skill Registry.** Repo skills are versioned with the code and work offline. Registry skills are centrally governed, revisioned and discoverable across agents, but they need network egress to Agent Platform and IAM.

**Exam signals**
- "Let the team's coding agent build, eval and deploy ADK agents using Google best practices": `uvx google-agents-cli setup`, then prompt the agent.
- "Add CI/CD and Terraform to an existing ADK project": `agents-cli scaffold enhance` (or `infra setup-cicd`).
- "Regression-check an agent change": `agents-cli eval run` then `eval compare`.
- "Make the deployed agent available to business users": `agents-cli publish gemini-enterprise`.
- "Centrally version and share skills across many agents; load only when relevant": **Skill Registry** plus ADK `GCPSkillRegistry` (or Agent Registry skills for org-wide governance).

**Distractors**
- "Agents CLI replaces Antigravity" (it augments any coding agent).
- "Run every agents-cli command manually" (the agent drives it).
- "Upload skills as a tar.gz" (zip only, with `SKILL.md` at the root).
- "Deploy with the `--prototype` scaffold to prod with Terraform" (prototype has none).

---

### Practice questions

**Q1.** Your platform team wants every developer's Antigravity agent to query BigQuery and Spanner through Google's remote MCP servers. Security prohibits storing any long-lived secrets on laptops, and access must respect each developer's IAM grants. What should you configure?

- **A.** `headers: {"Authorization": "Bearer <SA key token>"}` in `~/.gemini/config/mcp_config.json`
- **B.** `serverUrl` with `authProviderType: "google_credentials"` in a committed `.agents/mcp_config.json`, with developers using `gcloud auth application-default login`
- **C.** `url` with an OAuth client ID and secret per server
- **D.** A shared service account key referenced through `env` in a stdio MCP wrapper

**Answer: B.** ADC reuses each developer's identity and IAM with no stored secret, and the workspace file distributes the config. A and D put long-lived credentials on disk and collapse everyone onto one identity. C uses the unsupported `url` field, and OAuth client secrets are unnecessary for Google Cloud servers that accept ADC.

**Q2.** A fintech runs an ADK agent that includes a "code interpreter" tool executing Python the LLM writes. The tool must start in under a second per session, must not reach the Kubernetes API, and must isolate the host kernel. What is the best design?

- **A.** Cloud Run jobs with the default compute service account
- **B.** GKE Agent Sandbox: a gVisor `SandboxTemplate` with `automountServiceAccountToken: false`, a `SandboxWarmPool`, and a `SandboxClaim` per session
- **C.** GKE Standard pods with `privileged: true` inside a dedicated namespace
- **D.** Cloud Workstations with Turbo preset

**Answer: B.** gVisor provides kernel-level isolation, warm pools deliver sub-second claims, and the admission policy *requires* no SA token. A lacks a warm, sandboxed claim model and uses an over-privileged default SA. C violates the prohibited-config list. D is a developer environment, not a multi-tenant execution runtime.

**Q3.** A team added `.agents/rules/frontend/react.md` with `trigger: alwaysOn`. The agent ignores it. What are the two root causes?

- **A.** The rule exceeds 12,000 characters, and rules need a `name` field
- **B.** The file is nested in a subdirectory (flat scan) and `alwaysOn` is an invalid trigger (it must be `always_on`)
- **C.** Rules only load from `~/.gemini/config/rules/`, and require `/rules reload`
- **D.** React rules must be skills

**Answer: B.** `.agents/rules/` is scanned flat unless the file is registered in `.agents/rules.json`, and an invalid camelCase trigger is silently discarded. The 12,000-character limit belongs to legacy workflows, and rules don't need `name`. Workspace rules are fully supported.

**Q4.** Compliance requires that no agent in the repo can run `terraform apply` or `gcloud … delete`, regardless of prompts or model behavior, and that every blocked attempt is logged to an internal endpoint. What should you use?

- **A.** An `AGENTS.md` rule stating "never run terraform apply"
- **B.** A skill named `safe-infra` describing the approved process
- **C.** A `PreToolUse` hook on `run_command` in `.agents/hooks.json` that logs and returns `{"decision":"deny"}` for those patterns (optionally with matching `deny` permission rules)
- **D.** A subagent with `model: pro`

**Answer: C.** Hooks and permission denies are deterministic and can run logging code, while rules and skills are advisory prompt content. A model tier doesn't enforce anything.

**Q5.** 300 engineers will use Claude Code through Agent Platform. Finance wants predictable per-token cost and platform wants controlled upgrades. Some models aren't served on the `global` endpoint. Which configuration fits best?

- **A.** `CLAUDE_CODE_USE_VERTEX=1`, `CLOUD_ML_REGION=global`, `ANTHROPIC_VERTEX_PROJECT_ID`, pinned `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL`, and `VERTEX_REGION_CLAUDE_<MODEL>` for regional-only models
- **B.** Use the `opus` alias so users always get the newest model
- **C.** An Anthropic API key per developer with a spending cap
- **D.** `CLOUD_ML_REGION=us-east5` and grant `roles/aiplatform.admin`

**Answer: A.** Pinning controls both cost and upgrade timing, `global` improves availability, and per-model region overrides cover the gaps. B drifts cost and model. C leaves GCP governance and billing. D is over-privileged (`aiplatform.user` suffices) and forgoes global availability.

**Q6.** A regulated bank wants developers to use AI coding agents, but source code must never leave its perimeter, direct internet egress is forbidden except for an approved package mirror, and SSH to VMs must be auditable through IAM. What should you choose?

- **A.** Antigravity 2.0 on laptops with the Default preset
- **B.** Cloud Workstations: private cluster with PSC, a VPC-SC perimeter (restricting both the Workstations and Compute Engine APIs), public IPs disabled, Secure Web Proxy allowlisting the mirror, and `--disable-ssh-to-vm`
- **C.** GKE Agent Sandbox for each developer
- **D.** Cloud Shell with Gemini CLI

**Answer: B.** Each listed control maps to a documented Workstations security practice. A keeps code on endpoints outside the perimeter. C is a code-execution runtime, not a developer IDE platform with gateway SSH controls. D offers no perimeter or egress control.

**Q7.** Your team's coding agent must build a new ADK agent, create an evalset, deploy it to Cloud Run with CI/CD and Terraform, and make it available in Gemini Enterprise. What should you do with the least custom tooling?

- **A.** Write a custom MCP server that wraps `gcloud run deploy`
- **B.** Install Agents CLI (`uvx google-agents-cli setup`) and let the agent drive `create`, `eval run`, `scaffold enhance --deployment-target cloud_run`, `infra setup-cicd`, `deploy`, and `publish gemini-enterprise`
- **C.** Use `adk create` and deploy manually with a Dockerfile
- **D.** Use `agents-cli create --prototype` and deploy to production

**Answer: B.** Agents CLI skills and commands cover the full lifecycle, including publishing. A reinvents existing tooling. C lacks evals, CI/CD and publishing. D's `--prototype` omits CI/CD and Terraform.

**Q8.** An org has 400 internal skills. Loading them all into every agent's context is too expensive, and security wants immutable, versioned snapshots with central management. ADK agents run on Agent Runtime. What should you use?

- **A.** Put all skills in `AGENTS.md`
- **B.** Skill Registry with ADK `SkillToolset(registry=GCPSkillRegistry(...))`, so agents call `search_skills` and `load_skill` on demand, backed by immutable skill revisions
- **C.** Bake all skills into the container image under `skills/`
- **D.** One MCP server per skill

**Answer: B.** The registry provides on-demand discovery with progressive loading, and revisions are immutable snapshots. A loads everything on every turn. C has no central governance and requires a redeploy per change. D misuses MCP for procedural knowledge and multiplies operational overhead.

**Q9.** A nightly CI job runs `agy -p "fix lint errors and run tests"`. It exits 0, but no tests ran and stderr mentions a tool being denied. What is the best fix?

- **A.** Add `--dangerously-skip-permissions`
- **B.** Add scoped rules such as `command(regex:npm run (lint|test))` under `permissions.allow` in `~/.gemini/antigravity-cli/settings.json` for the CI runner
- **C.** Switch to Turbo preset
- **D.** Increase `--print-timeout`

**Answer: B.** In headless mode, unapproved tools are soft-denied and the run still exits 0, so pre-granting the exact commands is the least-privilege fix. A and C remove all guardrails. D doesn't address the denial.

**Q10.** Developers on Cloud Workstations run Claude Code and Antigravity CLI for multi-hour refactors, often pausing for review. Workstations keep timing out and losing agent state, and finance objects to raising idle timeouts. What should you do?

- **A.** Set the idle timeout to 24 hours
- **B.** Configure `IdleAction.SUSPEND` and install the sample keep-alive hooks (for example Claude Code `UserPromptSubmit` to start and `Notification` to stop `/google/scripts/keep_alive.sh`)
- **C.** Move developers to GKE Agent Sandbox
- **D.** Run agents with `nohup` on the workstation

**Answer: B.** Suspend preserves RAM and agent context while stopping compute billing, and the hooks keep the VM alive only during active work. A burns compute while idle. C is the wrong tool for interactive dev environments. D doesn't prevent the idle shutdown.

**Q11.** A team is moving from Gemini CLI to Antigravity CLI. Their repo has ten custom skills in `.gemini/skills/`, and their `GEMINI.md` files are at the repo root and in several service folders. After switching, the rules still apply but none of the skills appear as slash commands. What should they do?

- **A.** Run `agy plugin import gemini` to convert the skills
- **B.** Move `.gemini/skills/` to `.agents/skills/` in the repo. The `GEMINI.md` files need no change
- **C.** Copy the skills into `~/.gemini/config/rules/` so they load globally
- **D.** Rename every `SKILL.md` to `AGENTS.md`

**Answer: B.** The migration guide says workspace skills must be moved by hand from `.gemini/skills/` to `.agents/skills/`, while `GEMINI.md`/`AGENTS.md` context files work unchanged. A converts *extensions* into plugins, not a repo's skill folder. C turns procedures into rules (and rule files need `trigger` frontmatter). D turns on-demand skills into always-on context.

**Q12.** A platform team wants protobuf conventions (never reuse a field number, always mark deleted fields `reserved`) applied whenever the agent edits `.proto` or generated `.pb.go` files. The conventions must not consume context in other tasks. Which rule file is correct?

- **A.** `.agents/rules/proto.md` with `trigger: glob` and `globs: "*.proto, **/*.pb.go"`
- **B.** `.agents/rules/proto.md` with `trigger: always_on`
- **C.** `.agents/rules/proto/conventions.md` with `trigger: glob` and `globs: *.proto`
- **D.** `AGENTS.md` at the repo root with a `globs:` frontmatter block

**Answer: A.** A `glob` rule activates only when the agent touches matching files, and the quoted, comma-separated `globs` string is the documented format. B costs tokens on every turn. C is nested (ignored without `rules.json`) and has an unquoted `*` that YAML parses as an alias. D is wrong because `AGENTS.md` takes no frontmatter and is always on.

**Q13.** Internal audit has a 15-page security review rubric. Auditors want the agent to use it only when they explicitly ask for an audit, and it must never load automatically, even if a task looks security-related. What should you configure?

- **A.** A rule with `trigger: model_decision` and a description mentioning security audits
- **B.** A rule with `trigger: manual`, which auditors pull in by `@`-mentioning it in chat
- **C.** A `PreInvocation` hook that injects the rubric as an `ephemeralMessage`
- **D.** Add the rubric to the global `~/.gemini/GEMINI.md`

**Answer: B.** `manual` rules are never loaded automatically, only on an explicit `@` mention, and the docs cite audit rubrics as the use case. A lets the model decide to load it. C injects it every turn. D makes it always on across every project and eats into the 20k-token budget.

**Q14.** Agents in a repo often stop and report "done" while unit tests are failing. There is already an `AGENTS.md` rule saying "always run tests before finishing". The team wants a deterministic gate that sends the agent back to work with the failure output, with minimal extra machinery. What should you add?

- **A.** A `PostToolUse` hook on `write_to_file` that returns `{"decision":"deny"}` when tests fail
- **B.** A `Stop` hook in `.agents/hooks.json` that runs the tests and, on failure, returns `{"decision":"continue","reason":"<failures>"}`, with a retry cap
- **C.** Change the rule to `trigger: always_on` with stronger wording
- **D.** A `PreToolUse` hook on `run_command` that returns `force_ask`

**Answer: B.** A `Stop` hook with `decision: "continue"` re-enters the loop and injects the reason as a system message, which makes it a deterministic definition-of-done gate. A fails because `PostToolUse` only returns `{}` and cannot block. C is still advisory. D only adds prompts and never checks the test result.

**Q15.** You are porting a Gemini CLI `BeforeTool` hook (defined in `.gemini/settings.json`, `"timeout": 5000`, blocks by exiting with code 2) to Antigravity. Which set of changes is correct?

- **A.** Keep the file and event name. Antigravity reads Gemini CLI `settings.json` hooks
- **B.** Move it to `.agents/hooks.json` as `PreToolUse` under a named hook, set `"timeout": 5` (seconds), and block by printing `{"decision":"deny","reason":"…"}` to stdout
- **C.** Move it to `.agents/rules/hooks.md` with `trigger: always_on`
- **D.** Move it to `.agents/hooks.json` as `PreInvocation` with a `matcher` and keep `"timeout": 5000`

**Answer: B.** Antigravity uses `hooks.json`, the `PreToolUse` event with a regex tool matcher, a timeout in seconds (default 30), and a JSON `decision` for blocking. A is wrong because Gemini CLI hook config is not a documented Antigravity location. C turns enforcement into advisory text. D ignores the matcher on lifecycle events, and 5000 would mean 5000 seconds.

**Q16.** A central platform team must roll out the same four skills, two `glob` rules, a secrets-blocking hook, and a read-only BigQuery MCP server to 300 developers using Antigravity 2.0 and the CLI across 150 repos. Updates must ship as one versioned unit. What is the best approach?

- **A.** Publish the skills to Skill Registry and ask developers to copy the rest by hand
- **B.** Package everything as a plugin (`plugin.json` plus `skills/`, `rules/`, `hooks.json`, `mcp_config.json`) and install it globally (`~/.gemini/config/plugins/`, or `agy plugin install <git-url>` for the CLI)
- **C.** Add a `.agents/rules.json` with `inherits` pointing at a shared repo
- **D.** Put all the content into one large `AGENTS.md` in every repo

**Answer: B.** Plugins are the distribution unit for skills, rules, hooks, MCP servers and agents. A misuses Skill Registry, which serves ADK and runtime agents rather than IDE customization, and it leaves most of the bundle manual. C shares only rules. D can't carry hooks or MCP config and bloats the always-on budget.

**Q17.** You need a security reviewer that can only read code and search, runs in its own context so it doesn't pollute the main conversation, uses the stronger model tier, and runs any shell commands only in the sandbox. The main agent should delegate to it automatically. What should you create?

- **A.** A skill `.agents/skills/security-review/SKILL.md` describing the review steps
- **B.** A subagent `.agents/agents/security-reviewer.md` with `description`, `tools: [view_file, grep_search]`, `model: pro`, `commandExecutionPolicy: sandbox`, and `subagent: true`
- **C.** A rule with `trigger: model_decision` about security reviews
- **D.** A plugin containing only `plugin.json`

**Answer: B.** Only a subagent provides context isolation plus a tool allowlist, a model tier and an execution policy, and the planner delegates to it based on `description`. A and C run inside the main agent's context with its full toolset. D carries no behavior. Spell tool names exactly, because a misspelled tool can hang the subagent.

**Q18.** A team has 25 legacy workflows in `.agents/workflows/` and `~/.gemini/config/workflows/`, several of them near the 12,000-character limit, and they want them to keep working after the retirement date. Some workflow names match skills that already exist. What should they do?

- **A.** Nothing. Workflows remain supported indefinitely
- **B.** Run `/migrate-workflows` in Antigravity 2.0 to scaffold `.agents/skills/<name>/SKILL.md` for each one (originals are renamed `.bak`), then move embedded scripts into `scripts/`. Where names collide, the existing skill already takes precedence
- **C.** Convert each workflow to an `always_on` rule
- **D.** Split each workflow into two files under 6,000 characters

**Answer: B.** Workflows retire on 2026-11-01, and `/migrate-workflows` is the documented path. Skills win name collisions, and moving scripts into the skill bundle keeps `SKILL.md` lean. C loads procedures on every turn. D keeps a deprecated format.

---

### Key doc links
- Antigravity MCP: https://antigravity.google/docs/mcp
- Antigravity skills: https://antigravity.google/docs/skills
- Workflows → skills migration: https://antigravity.google/docs/migration/workflows-to-skills
- Antigravity rules: https://antigravity.google/docs/rules
- Antigravity hooks: https://antigravity.google/docs/hooks
- Antigravity subagents: https://antigravity.google/docs/subagents
- Antigravity plugins: https://antigravity.google/docs/plugins
- Antigravity permissions: https://antigravity.google/docs/permissions
- Antigravity terminal sandbox: https://antigravity.google/docs/sandbox
- Antigravity agent settings / presets: https://antigravity.google/docs/agent-settings
- Antigravity CLI features (terminal sandbox, plugins): https://antigravity.google/docs/cli/features
- Antigravity CLI headless mode: https://antigravity.google/docs/cli/headless
- Antigravity CLI modes: https://antigravity.google/docs/cli/modes
- Antigravity CLI best practices: https://antigravity.google/docs/cli/best-practices
- Migrating from Gemini CLI: https://antigravity.google/docs/cli/gcli-migration
- Antigravity in Gemini Enterprise: https://antigravity.google/docs/enterprise
- IDE extensions: https://antigravity.google/docs/ide/extensions
- Artifacts / artifact review / implementation plan: https://antigravity.google/docs/artifacts, https://antigravity.google/docs/artifact-review, https://antigravity.google/docs/implementation-plan
- Antigravity SDK: https://antigravity.google/docs/sdk/overview, https://antigravity.google/docs/sdk/tools, https://antigravity.google/docs/sdk/mcp, https://antigravity.google/docs/sdk/policies, https://antigravity.google/docs/sdk/subagents
- Claude Code on Agent Platform: https://code.claude.com/docs/en/google-vertex-ai
- Claude Code sandboxing: https://code.claude.com/docs/en/sandboxing
- GKE Agent Sandbox concepts: https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox
- Enable GKE Agent Sandbox: https://docs.cloud.google.com/kubernetes-engine/docs/how-to/how-install-agent-sandbox
- Agent Sandbox isolation / storage: https://docs.cloud.google.com/kubernetes-engine/docs/how-to/agent-sandbox, https://docs.cloud.google.com/kubernetes-engine/docs/how-to/agent-sandbox-storage
- Filestore agent volumes with Agent Sandbox: https://docs.cloud.google.com/filestore/docs/agent-sandbox
- Cloud Workstations agent-optimized development: https://docs.cloud.google.com/workstations/docs/agent-optimized-development
- Cloud Workstations security best practices: https://docs.cloud.google.com/workstations/docs/set-up-security-best-practices
- Cloud Workstations VPC-SC and private clusters: https://docs.cloud.google.com/workstations/docs/configure-vpc-service-controls-private-clusters
- Gemini CLI in Cloud Workstations: https://docs.cloud.google.com/workstations/docs/ai-agent-assisted-coding-gemini-cli
- Agents CLI quickstart (Agent Platform): https://docs.cloud.google.com/gemini-enterprise-agent-platform/agents/quickstart-adk
- Agents CLI (ADK docs): https://adk.dev/get-started/agents-cli/, https://adk.dev/deploy/agent-runtime/agents-cli/, https://adk.dev/tutorials/coding-with-ai/
- Agents CLI docs / CLI reference / extensions / repo: https://google.github.io/agents-cli/, https://google.github.io/agents-cli/cli/, https://google.github.io/agents-cli/guide/extensions/, https://github.com/google/agents-cli
- Skill Registry: https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/skill-registry, https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/skill-registry/create-manage
- ADK Skill Registry integration / ADK skills: https://adk.dev/integrations/skills-registry/, https://adk.dev/skills/
- Agent Registry skills: https://docs.cloud.google.com/agent-registry/register-skills
- Gemini CLI extensions / skills: https://geminicli.com/docs/extensions/reference/, https://geminicli.com/docs/cli/skills/
- CodeMender: https://cloud.google.com/security/codemender
- Antigravity IDE workflows (legacy): https://antigravity.google/docs/ide/workflows
- Antigravity CLI `/agents` command (custom agent paths): https://antigravity.google/docs/cli/commands/agents
- Antigravity CLI settings / reference (settings.json keys): https://antigravity.google/docs/settings, https://antigravity.google/docs/cli/reference
- Build with Google plugins: https://antigravity.google/docs/build-with-google
- Agent Skills open specification: https://agentskills.io/specification
- Gemini CLI hooks / hooks reference: https://geminicli.com/docs/hooks/, https://geminicli.com/docs/hooks/reference/
- Claude Code hooks / memory (CLAUDE.md, rules) / skills / subagents / plugins: https://code.claude.com/docs/en/hooks, https://code.claude.com/docs/en/memory, https://code.claude.com/docs/en/skills, https://code.claude.com/docs/en/sub-agents, https://code.claude.com/docs/en/plugins
- Data Agent Kit: https://cloud.google.com/products/data-agent-kit


---

## Section 3 — Developing custom agents (~33%)

**What the exam is really testing**
1. Whether you can pick the *right-sized* building block (model, workflow primitive, session and memory backend, vector store, protocol) for a stated requirement, and name the trade-off you accept.
2. Whether you know the managed Agent Platform services (Agent Runtime, Sessions, Memory Bank, Agent Retrieval, RAG Engine, Agent Search, Agent Registry, Agent Identity) well enough to know their defaults, limits and how they wire into ADK.
3. Whether you can keep a multi-agent system deterministic, governable and least-privilege: graph and workflow agents instead of prompt spaghetti, per-agent identity instead of shared service accounts, registries instead of hard-coded endpoints.

> **2026 naming map.** Vertex AI is now **Gemini Enterprise Agent Platform** ("Agent Platform"). Agent Engine is now **Agent Runtime**; the REST resource is still `reasoningEngines`. Vertex AI Search is now **Agent Search** (API: `discoveryengine`). Vector Search 2.0 is now **Agent Retrieval**, and the original index/endpoint product is **Vector Search 1.0**. ADK class names still carry the old prefix: `VertexAiSessionService`, `VertexAiMemoryBankService`, `VertexAiSearchTool`, `VertexAiRagMemoryService`. The ADK docs call RAG Engine "Knowledge Engine" in places, while the Cloud docs still say **RAG Engine**. If an answer option uses either name, treat them as the same product. (That the product itself has been renamed is unverified.)

---

### 3.1 Designing and building agentic workflows in code

#### 3.1.1 Selecting and configuring the model

> 🧠 **Visual memory map:** Model selection map
>
> [![Model selection map](visual-memory/05-model-selection.png)](visual-memory/05-model-selection.png)

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

> 🧠 **Visual memory map:** State, sessions & memory
>
> [![State, sessions & memory](visual-memory/07-state-memory.png)](visual-memory/07-state-memory.png)

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

> 🧠 **Visual memory map:** RAG pipeline & backend chooser
>
> [![RAG pipeline & backend chooser](visual-memory/08-rag-pipeline.png)](visual-memory/08-rag-pipeline.png)

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

> 🧠 **Visual memory map:** Protocols, registry & agent identity
>
> [![Protocols, registry & agent identity](visual-memory/09-protocols-identity-registry.png)](visual-memory/09-protocols-identity-registry.png)

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
- **Automatic registration** covers Agent Runtime agents, built-in Workspace and Gemini Enterprise agents, Google Cloud remote MCP servers (registered when you enable the product API), GKE Deployments labelled `registry.gke.io/functional-type`, and Cloud Run workloads deployed with `--functional-type=agent|mcp-server`. It is **single-project scope**. Use **manual registration** for external or custom components and for cross-project central catalogs. To remove an auto-registered MCP server, delete the server or disable its API.
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

#### 3.2.4 Deep dive — Registering MCP servers in Agent Registry

> 🧠 **Visual memory map:** Agent Registry — registering MCP servers
>
> [![Agent Registry — registering MCP servers](visual-memory/17-agent-registry-mcp.png)](visual-memory/17-agent-registry-mcp.png)

Registering an MCP server does three jobs. It makes the server and its tools **discoverable** (console, gcloud, the registry's own MCP server, ADK, Agent Studio). It gives **bindings** a target to point at (resource links and auth providers). And it makes the server a **governable destination**: an egress Agent Gateway denies anything that isn't registered, and IAM egress policies can only be bound to registered resources. If an MCP server isn't in the registry, the platform can't govern it.

**A. Resource model**

| Resource | Access | What it is | Resource name |
|---|---|---|---|
| `Service` | **Writable** | Manual registration of an agent, MCP server or endpoint. The spec you set decides which read view it becomes. The output-only `registryResource` field holds the projected name | `projects/P/locations/L/services/ID` |
| `McpServer` | Read-only | Discovery view of an MCP server and its tools | `projects/P/locations/L/mcpServers/ID` |
| `Agent` | Read-only | Discovery view of an agent. A2A skills are indexed from its Agent Card | `…/agents/ID` |
| `Endpoint` | Read-only | Discovery view of a target URL, usually a REST API | `…/endpoints/ID` |
| `Skill`, `SkillRevision`, `Publisher` (Preview) | Managed directly, **not** through `Service` | Standalone `SKILL.md` packages with immutable revisions and a default revision. Your skills sit under the `private` publisher | `…/skills/private-ID` |
| `Binding` | Writable | Source agent → target (agent, MCP server, endpoint), or source agent → **auth provider** | `…/bindings/ID` |

- **Write through `Service`, read through the typed views.** You never create or patch an `McpServer` directly. To change a manually registered server, you update its `Service`.
- **The spec flag picks the collection.** The three flag pairs are mutually exclusive. If you use the wrong pair, the entry lands in the wrong collection and type-specific policies may not apply to it.

  | gcloud flags | REST field | Becomes | Valid spec types |
  |---|---|---|---|
  | `--mcp-server-spec-type` / `--mcp-server-spec-content` | `mcpServerSpec` | `McpServer` | `tool-spec` (REST enum also has `NO_SPEC`) |
  | `--agent-spec-type` / `--agent-spec-content` | `agentSpec` | `Agent` | `a2a-agent-card`, `no-spec` |
  | `--endpoint-spec-type` | `endpointSpec` | `Endpoint` | `no-spec` |
- **Interfaces** hold the connection details: `url` plus `protocolBinding`, which is `jsonrpc`, `http-json` or `grpc` (in Terraform, `JSONRPC`, `HTTP_JSON` or `GRPC`). MCP servers normally use `jsonrpc`. With an `A2A_AGENT_CARD` spec, `interfaces` must be empty, because the card carries its own URLs.
- **REST:** `POST https://agentregistry.googleapis.com/v1/projects/P/locations/L/services?serviceId=ID`. `serviceId` is 4–63 characters of `[a-z0-9-]`. The call returns a long-running **Operation**. The optional `requestId` (a UUID) makes retries idempotent for at least 60 minutes. Create needs `agentregistry.services.create` and the `cloud-platform` or `agentregistry.read-write` scope.

**Three identifiers you must not confuse**

| Identifier | Example | Used for |
|---|---|---|
| **URN** (logical, immutable) | Google remote MCP: `urn:mcp:googleapis.com:projects:PROJECT_NUMBER:locations:global:SERVER_NAME`<br>Manual MCP: `urn:mcp:projects-PROJECT_NUMBER:projects:PROJECT_NUMBER:locations:REGION:agentregistry:services:SERVER_ID` | Inventory, lookup, `--filter="mcpServerId='urn:mcp:…'"`, binding source/target |
| **Resource URI** (runtime reference) | The Cloud Run service, GKE Deployment or Agent Runtime instance that actually runs it (`agentregistry.googleapis.com/system/RuntimeReference`) | Topology graph queries |
| **Agent principal** (IAM) | `principal://agents.global.org-ORG_ID.system.id.goog/resources/…` | **All** access policies. URNs never go in IAM |

**Locations**
- **Global**, multi-regions **`us`** and **`eu`**, and about 40 regions. The registry is **project-scoped**: enable the API per project. If you move to another project, nothing migrates; you recreate every entry.
- **`us`/`eu` restriction:** in the multi-regions you **can't** manually register agents, MCP servers or endpoints, and you can't create bindings. Use a region or `global`. Standalone skills *are* supported in `global` and the multi-regions.
- **Google-managed remote MCP servers live in `global`**, so IAM bindings on them must use `--region=global`. A regional flag such as `--region=us-central1` returns `NOT_FOUND`.
- **Cross-project governance:** the registry, the Agent Gateway and the agent endpoints must be in the **same region or `global`**. Gemini Enterprise alignment:

  | Gemini Enterprise app | Agent Gateway | Agent Registry |
  |---|---|---|
  | `global` | `us-central1` | `us-central1`, `us` or `global` |
  | `us` | `us-central1` | `us-central1` or `us` |
  | `eu` | `europe-west1` | `europe-west1` or `eu` |

**B. Setup**

```bash
gcloud services enable agentregistry.googleapis.com --project=PROJECT_ID   # also turns on the registry's own MCP server
gcloud services enable iap.googleapis.com --project=PROJECT_ID             # only if Agent Gateway will enforce policy
# Cloud Run auto-registration also needs: run, iam, agentregistry and App Hub APIs
```

| Role | Grants | Typical holder |
|---|---|---|
| `roles/agentregistry.viewer` | get, list and search agents, MCP servers, endpoints and skills; view bindings | Developers, **and every agent identity that resolves tools at runtime** |
| `roles/agentregistry.editor` | Viewer plus `services.create/update/delete` (manual registration, tool-spec updates) and skills. It **can't** create bindings | Platform engineers who onboard servers |
| `roles/agentregistry.admin` | `agentregistry.*`, including **`bindings.create/update/delete`** | Registry administrators |
| `roles/agentregistry.user` | Skills and skill revisions CRUD; read-only on everything else | Skill authors |
| `roles/serviceusage.serviceUsageAdmin`, `roles/resourcemanager.projectIamAdmin` | Enable the API and grant the roles above | Project setup |
| `roles/mcp.toolUser` (`mcp.tools.call`) | Call tools on Google MCP servers, including `agentregistry.googleapis.com/mcp` | Any caller of Google remote MCP servers |
| `roles/iap.egressor` | Egress through Agent Gateway to a registered target (granted to the **agent principal** on the target) | Agent identities |
| `roles/iap.admin` | Manage IAP egress policies on registry resources | Security admins |

> ⚠️ **Don't grant `agentregistry.editor` or `.admin` to agents.** Those roles can edit tool annotations such as `readOnlyHint` and `destructiveHint`, and policies trust those annotations. They can also enroll a malicious third-party agent. Agents get **viewer** only.

**C. Automatic registration (same project only)**

| Source | How to opt in | What lands in the registry | How to remove it |
|---|---|---|---|
| **Google and Google Cloud remote MCP servers** (BigQuery, Compute Engine, Cloud SQL, Agent Search …) | **Enable the product's API** in the project (for example `gcloud services enable compute.googleapis.com`) | The server **and its tools**, immediately, in the **`global`** location. No tool spec to upload | **Disable the product API** (or delete the underlying server) |
| **Apigee API hub** | Turn on sync of MCP-style APIs to Agent Registry | Imported MCP APIs. Keep the sync enabled or the data goes stale | Stop the sync |
| **GKE** | Deployment **label** `registry.gke.io/functional-type: "MCP_SERVER"`, plus the **annotations** `modelcontextprotocol.info/urls` (endpoint URLs) and `modelcontextprotocol.info/capabilities` (card: endpoint, protocol) | The GKE controller **introspects** the server, gets its tool spec and registers the tools. The Deployment name becomes the display name | Delete the Deployment (the docs say to delete the underlying server; whether removing the label deregisters it is unverified) |
| **Cloud Run** (Preview) | `gcloud beta run deploy SVC --image=IMG --functional-type=mcp-server [--identity-type=agent-identity\|service-account]` | Server name and type `MCP_SERVER` under `/mcpServers`. The identity defaults to a **service account** if you don't set one | Delete the service (whether tools are introspected for Cloud Run servers is unverified) |

- **GKE backward compatibility:** the old annotation `apphub.cloud.google.com/functional-type` still works, but the label is recommended.
- **Only Cloud Run *services* can be MCP servers.** Jobs support only `--functional-type=agent`.
- **Agents, for contrast:**
  - Agent Runtime agents are registered with no flag, and updates and deletes sync automatically.
  - On Cloud Run, `--functional-type=agent` **requires** `--identity-type=agent-identity`; any other identity type is an error.
  - GKE needs the label `registry.gke.io/functional-type: "AGENT"` plus the annotation `a2a-protocol.org/agent-card`.
  - Built-in Workspace and Gemini Enterprise agents appear with no setup.
- **Automatic registration never crosses projects.** A central governance project must register workload-project components manually.

**D. Manual registration of an MCP server, step by step**

Use manual registration for third-party or SaaS MCP servers, on-premises or other-cloud servers, unsupported runtimes, and servers in another project that a central registry and gateway must govern.

1. **Prerequisites.** The API is enabled, you hold `roles/agentregistry.editor`, and the location is a **region or `global`** (not `us`/`eu`).
2. **Write `toolspec.json`.** Its payload is exactly the shape of an MCP `tools/list` response. The file limit is **10 KB** and a service can hold at most **100 tools**. **Manual registration does not introspect the server.** The registry records the endpoint and only the tools you declare.
   ```json
   {"tools": [
     {"name": "get_customer_info", "description": "Retrieves customer details.",
      "inputSchema": {"type": "object", "properties": {"email": {"type": "string"}}},
      "annotations": {"title": "Get Customer Info", "readOnlyHint": true, "idempotentHint": true}},
     {"name": "create_support_ticket", "description": "Creates a support ticket.",
      "annotations": {"destructiveHint": true, "idempotentHint": false, "openWorldHint": true}}
   ]}
   ```
   The annotation defaults follow the MCP spec: `readOnlyHint=false`, **`destructiveHint=true`**, `idempotentHint=false`, **`openWorldHint=true`**. If you leave a tool unannotated, policies treat it as a potentially destructive, open-world tool.
3. **Register it.**
   - **Console:** Agent Registry → **MCP servers** tab → **Add MCP server** → enter the display name, description and region → under **Tool specification**, enter the endpoint URL and paste the tool spec, or click **Import tools** (this only works for **publicly reachable** URLs) → **Next** → select the tools to include → **Save**.
   - **gcloud:**
     ```bash
     gcloud agent-registry services create crm-mcp \
       --project=PROJECT_ID --location=us-central1 \
       --display-name="CRM MCP" \
       --mcp-server-spec-type=tool-spec \
       --mcp-server-spec-content=@toolspec.json \
       --interfaces=url=https://crm.example.com/mcp,protocolBinding=jsonrpc
     ```
   - **Terraform:**
     ```hcl
     resource "google_agent_registry_service" "crm_mcp" {
       location     = "us-central1"
       service_id   = "crm-mcp"
       display_name = "CRM MCP"
       interfaces {
         url              = "https://crm.example.com/mcp"
         protocol_binding = "JSONRPC"
       }
       mcp_server_spec {
         type    = "TOOL_SPEC"
         content = file("toolspec.json")
       }
     }
     # registry_resource output = projects/…/locations/…/mcpServers/…
     ```
   - **REST body:** `{"displayName": "...", "interfaces": [{"url": "...", "protocolBinding": "JSONRPC"}], "mcpServerSpec": {"type": "TOOL_SPEC", "content": {"tools": [...]}}}`. This body is assembled from the REST reference; the enum spelling of `protocolBinding` in REST is unverified.
   - **From an agent or IDE:** the `create_service` tool on `https://agentregistry.googleapis.com/mcp`.
4. **Verify.**
   ```bash
   gcloud agent-registry mcp-servers list --project=PROJECT_ID --location=us-central1 \
     --filter="displayName='CRM MCP'"          # or mcpServerId='urn:mcp:…'
   gcloud agent-registry mcp-servers describe crm-mcp --project=PROJECT_ID --location=us-central1
   ```
   The server's details page has these tabs:
   - **Overview:** URN, location and an ADK snippet.
   - **Tools:** schema and **annotations** per tool, with a per-tool ADK snippet.
   - **Observability:** latency, traffic, errors and token spend.
   - **Security:** Security Command Center findings for that resource.

   In Terraform, use the data source `google_agent_registry_mcp_server`.
5. **Update when the server changes.** The registry never re-scans a manually registered server, so new tools stay invisible until you upload the spec again:
   ```bash
   gcloud agent-registry services update crm-mcp --project=PROJECT_ID --location=us-central1 \
     --mcp-server-spec-content=new-toolspec.json
   ```
   The upload **replaces** the whole tool list; it doesn't merge. The display name and description can be edited in the console under Overview → **Edit**.
   - **Docs inconsistency:** the *Manage MCP tools* page says Terraform supports only `NO_SPEC` for MCP servers and can't carry tool specs, while *Register MCP servers* shows `mcp_server_spec { type = "TOOL_SPEC" }`. For tool-spec changes, use gcloud, the console or the API.
6. **Delete.**
   - **Manual entries:** `gcloud agent-registry services delete crm-mcp --project=PROJECT_ID --location=us-central1`. In the console, you type `DELETE` to confirm; in Terraform, remove the resource and apply. The server disappears from search and discovery.
   - **Auto-registered Google servers:** you can't delete the entry. Disable the product API or delete the underlying server.
   - **Clean up the dependents yourself.** Deleting the entry does **not** delete bindings or policies that reference it. **Manual cross-project entries** also never auto-update when the remote server changes or is deleted.

**E. Endpoints, A2A agents and custom ADK agents (same `Service` pattern)**

```bash
# External REST API as a governable destination
gcloud agent-registry services create payments-api --location=us-central1 --display-name="Payments API" \
  --endpoint-spec-type=no-spec --interfaces=url=https://api.example.com/v1,protocolBinding=http-json

# Composite "core Google APIs" endpoint: one IAP binding for many hostnames (hub-and-spoke gateway)
gcloud agent-registry services create core-gapi-services --project=CENTRAL_PROJECT --location=us-central1 \
  --display-name="Core Google APIs" --endpoint-spec-type=no-spec \
  --interfaces=protocolBinding=jsonrpc,url=https://telemetry.googleapis.com \
  --interfaces=protocolBinding=jsonrpc,url=https://iamcredentials.googleapis.com \
  --interfaces=protocolBinding=jsonrpc,url=https://agentregistry.googleapis.com

# A2A agent (card <= 10 KB, A2A 0.3 or 1.0; card skills are indexed for search)
gcloud agent-registry services create billing-agent --location=global --display-name="Billing" \
  --agent-spec-type=a2a-agent-card --agent-spec-content=@agent-card.json

# Non-A2A REST agent: discoverable by name/description only, no searchable skills
gcloud agent-registry services create travel-agent --location=global --display-name="Travel" \
  --agent-spec-type=no-spec --interfaces=url=https://travel.example.com/v1,protocolBinding=http-json
```
- **Endpoint connection tests:** the console's **Test connection** works only for public URLs. You can still register private URLs.
- **Custom ADK agent on your own infrastructure:** expose it over A2A (`to_a2a(root_agent)` serves `/.well-known/agent-card.json`), save the generated card, then register it with `--agent-spec-type=a2a-agent-card`.
- **Skills (Preview):**
  - Create one with `gcloud alpha agent-registry skills create SKILL_ID --location=global --display-name=… --payload=./skill.zip`, or with `--gcs-source-uri=gs://…`. For the GCS option, grant `storage.objects.get` to `service-PROJECT_NUMBER@gcp-sa-agentregistry.iam.gserviceaccount.com`.
  - The ID becomes `private-SKILL_ID`.
  - Limits: ZIP ≤ 500 KB compressed, ≤ 10 MB uncompressed, ≤ 1 MB per file.

**F. Consuming registered MCP servers**

| Search mode | Agents | MCP servers | Skills |
|---|---|---|---|
| Keyword (`AND`/`OR`/`NOT`) | ✅ metadata, description, **A2A skills** | ✅ description and **tools** | ✅ metadata only |
| Prefix (`displayName:Prod_*`) | ✅ | ✅ | ✅ |
| **Semantic** | ❌ | ❌ | ✅ indexes the whole `SKILL.md` (`--search-type=semantic`) |

```bash
gcloud agent-registry mcp-servers search --project=P --location=L --search-string="database"
gcloud agent-registry agents search      --project=P --location=L --search-string="flight OR booking"
gcloud alpha agent-registry skills search --project=P --location=L --query="manage relational databases" --search-type=semantic
```

**The registry as an MCP server.** It lives at `https://agentregistry.googleapis.com/mcp` over Streamable HTTP. It accepts OAuth and IAM credentials, never API keys, and `tools/list` needs no auth. Its tools:
- **Discovery:** `search_agents`, `search_mcp_servers`, `get_agent`, `get_mcp_server`, `get_endpoint`, `get_service`, `list_*`, `list_bindings`, `get_binding`, `fetch_available_bindings`, `get_operation`.
- **Admin:** `create_service`, `update_service`, `delete_service`, `create_binding`, `update_binding`, `delete_binding`.

**ADK (Python) — resolve at runtime instead of hard-coding URLs.** Requirements: `pip install "google-adk[a2a,agent-identity]"` (the module imports both extras at load time, so a core-only install raises `ImportError`), `google-adk>=1.29.0`, and ADC.

```python
from google.adk.agents.llm_agent import LlmAgent
from google.adk.auth.credential_manager import CredentialManager
from google.adk.integrations.agent_identity import GcpAuthProvider
from google.adk.integrations.agent_registry import AgentRegistry

CredentialManager.register_auth_provider(GcpAuthProvider())      # lets ADK resolve auth-provider bindings
registry = AgentRegistry(project_id=PROJECT, location="us-central1")  # optional header_provider=callable

crm_tools = registry.get_mcp_toolset(
    mcp_server_name="mcpServers/crm-mcp",                         # short form; full: projects/P/locations/L/mcpServers/ID
    continue_uri="https://app.example.com/oauth/continue")        # only for 3-legged OAuth (user consent)
billing = registry.get_remote_a2a_agent(
    agent_name="agents/billing-agent",
    httpx_client=authed_httpx_client)                             # A2A calls are NOT auto-authenticated

root_agent = LlmAgent(name="orchestrator", model="gemini-flash-latest",
                      tools=[crm_tools], sub_agents=[billing])
```
- **Client methods:** `list_mcp_servers(filter_str, page_size, page_token)`, `get_mcp_server(name)`, `get_mcp_toolset(mcp_server_name)` (returns an ADK `McpToolset`), `list_agents(...)`, `get_agent_info(name)`, `get_remote_a2a_agent(agent_name)` (returns a `RemoteA2aAgent`).
- **Go:** `agentregistry.New(ctx, agentregistry.Config{ProjectID, Location})`, then `MCPToolset(ctx, name, WithMCPHTTPClient/WithMCPHeaders)` and `RemoteAgent(ctx, name, WithA2AHTTPClient/WithA2AHeaders)`.
- **Fetch once at startup,** not per invocation, to avoid extra latency. An agent can have only one parent, so reuse a fetched remote agent under several orchestrators with `.clone()`.
- **Production traffic** to the resolved endpoints should go through **Agent Gateway**. The registry supplies the endpoints; the gateway enforces policy.

**Agent Studio (low-code)**
- Add a tool with **Add (+) → "MCP Server from Agent Registry"**. You choose a **Location**, an **MCP Server**, and **Auth Config**, where `None` means access resolves through IAM. The option appears only **after the agent is saved**, and the agent can use **all** tools on that server.
- **Direct MCP-URL connections and the Vertex AI Search data-store tool are deprecated,** and existing ones are read-only. To migrate:
  - A direct MCP URL: register the server first, then re-add it from the registry.
  - The data-store tool: switch to the **Agent Search MCP server** (`discoveryengine.googleapis.com`). The data-source selection doesn't carry over automatically.

**Gemini Enterprise**
- Admins can **import MCP servers from Agent Registry** as data stores. This requires an Agent Gateway in a region aligned with the app, with the registry associated to that gateway.
- **Gap:** direct communication between Gemini Enterprise agents and Gemini Enterprise data connectors does **not** trigger gateway enforcement.

**G. Authenticating to registered MCP servers**

| Model | When | How |
|---|---|---|
| **Agent's own identity** (ADC = Agent Identity or a service account) | Google Cloud MCP servers and tools | The agent identity needs `agentregistry.viewer`, **plus** the product's own roles (for example Compute Instance Admin for a Compute Engine MCP tool), **plus** `mcp.toolUser` for Google remote MCP servers. ADK attaches Google auth headers to Google MCP servers automatically. For remote A2A agents, pass an authenticated `httpx.AsyncClient` |
| **Auth manager: API key or 2-legged OAuth** | Custom or external tools called with the agent's own credentials | Create an **auth provider** in the Agent Identity auth manager, then **bind** it to the agent. No credentials in code |
| **Auth manager: 3-legged OAuth** | The tool must act **on behalf of the user** (consent, delegated permission) | Create an auth provider with redirect URIs and bind it. The client app must handle the **`adk_request_credential`** function call, and the code passes **`continue_uri`** to `get_mcp_toolset`. The platform prompts for consent, stores the token, then resumes |
| **Custom headers** (`header_provider`) | External toolsets that don't support the auth manager, or for extra context | Headers go **only to the target MCP server**, never to the Agent Registry API (which always uses ADC) and never to A2A agents |

```bash
# Auth-provider binding (needs roles/agentregistry.admin; the auth-provider path must use the project ID)
gcloud agent-registry bindings create crm-oauth \
  --project=PROJECT_ID --location=us-central1 --display-name="CRM OAuth" \
  --source-identifier="urn:agent:projects-123:projects:123:locations:us-central1:aiplatform:reasoningEngines:456" \
  --auth-provider="projects/PROJECT_ID/locations/us-central1/connectors/crm-oauth-provider"
# Resource binding (agent -> MCP server), used to map orchestrator-to-tool relationships
gcloud agent-registry bindings create orch-to-crm --project=PROJECT_ID --location=us-central1 \
  --display-name="Orchestrator to CRM" --source-identifier="urn:agent:…" --target-identifier="urn:mcp:…"
# also: gcloud agent-registry bindings list | describe | update | delete
```
In Terraform, `google_agent_registry_binding` takes `source`, `target` and `auth_provider_binding { auth_provider, scopes, continue_uri }`. Bindings aren't available in `us`/`eu`.

**H. Governance: what registration unlocks**
- **Agent Gateway (egress, `AGENT_TO_ANYWHERE`)** attaches up to two registries: one global and one regional or multi-regional. Destinations must be **registered** or matched by an explicit unregistered-host rule; otherwise the default is **deny**. One gateway governs up to 5,000 registered resources. If the registry is regional, policies apply only to resources in that region (see Section 5.1).
- **IAM egress (allow) policies through IAP**, bound to registry resources:
  ```bash
  gcloud iap web set-iam-policy policy.json --project=PROJECT_ID \
    --resource-type=agent-registry --mcp-server=crm-mcp --region=us-central1
  # --agent=ID | --endpoint=ID | no resource flag = the whole registry; --folder/--organization also accepted
  # Google-managed MCP servers: --region=global
  ```
  ```json
  {"policy": {"bindings": [{"role": "roles/iap.egressor",
    "members": ["principal://agents.global.org-ORG/resources/aiplatform/projects/NUM/locations/us-central1/reasoningEngines/support"],
    "condition": {"title": "read-only CRM tools",
      "expression": "api.getAttribute('iap.googleapis.com/mcp.tool.isReadOnly', false) == true"}}]}}
  ```
  The console condition builder offers **Name, ReadOnly, Destructive, Idempotent and Open World**. In Unified Access Policies, the equivalent attributes are `destination.agent_registry.mcp_server.tool.annotations.read_only_hint` / `destructive_hint` / `idempotent_hint` / `open_world_hint` and `…mcp_server.tool.name`.
- **Policies are validated when you bind them, not at runtime.** Binding to an unregistered resource fails immediately with `NOT_FOUND`, so the docs recommend defining policies in CI/CD.
- **Annotations are declarations, not proofs.** For manual servers, whoever writes `toolspec.json` asserts `readOnlyHint`. Annotation-based CEL is only as trustworthy as the people who can edit the registry, which is another reason to keep editor and admin roles away from agents.
- **Content and semantics:**
  - **Semantic governance policies** and **Model Armor** run at the gateway on top of IAM.
  - For Google MCP servers, Model Armor floor settings can scan all traffic: `gcloud model-armor floorsettings update --full-uri=projects/P/locations/global/floorSetting --add-integrated-services=GOOGLE_MCP_SERVER --google-mcp-server-enforcement-type=INSPECT_AND_BLOCK`. If the agent and server are in different projects, floor settings in both projects mean Model Armor is invoked twice.
- **Visibility:**
  - Each MCP server's **Observability** tab. For the gateway's own Observability tab, see Section 5.1.
  - The **topology** graph, keyed by resource URI.
  - Agent Platform **Security** tab and per-resource **Security** tabs, which surface Security Command Center findings such as excessive permissions and toxic combinations.

**I. Quotas and limits**

| Limit | Value |
|---|---|
| Agents, MCP servers, endpoints, bindings, A2A skills, standalone skills per project | **100 each** (global and per region; quota can be raised) |
| Skill revisions per skill | 100 |
| API rate | 12,000/min aggregate (200 QPS); 1,200/min global (20 QPS); 1,200/min per region |
| Agent spec (Agent Card) and MCP tool spec content | **10 KB** each |
| A2A skills or tools per service | **100** |
| Display name / description | 63 / 2,048 characters |
| Page size | 100 |

**J. End-to-end flow**

```text
1 REGISTER   Google MCP ── enable product API ──────────┐
             GKE label / Cloud Run --functional-type ────┼──► Service ──projects──► McpServer
             gcloud agent-registry services create ──────┘    (writable)            (read-only: tools + annotations)
               --mcp-server-spec-type=tool-spec                                          │
                                                                                         ▼
2 DISCOVER   gcloud … mcp-servers search · registry MCP search_mcp_servers
             ADK AgentRegistry.get_mcp_toolset() · Agent Studio "MCP Server from Agent Registry"
                                                                                         │
                                                                                         ▼
3 BIND/AUTH  own identity: agentregistry.viewer + product roles + mcp.toolUser
             or bindings create --auth-provider=…/connectors/X  (API key · 2LO · 3LO + continue_uri)
                                                                                         │
                                                                                         ▼
4 CALL       Agent ──► Agent Gateway (AGENT_TO_ANYWHERE)
                         a. IAP: iap.egressor on the registered target? (CEL: tool name, read-only hint)
                         b. destination registered? otherwise DENY
                         c. Model Armor · semantic governance
                       ──► MCP server  tools/call
```

**K. Decision tables**

| Choose | When |
|---|---|
| **Automatic** registration | A Google remote MCP server (just enable the API), GKE with the label and annotations, Cloud Run with `--functional-type=mcp-server`. The server is in the **same project** as the registry. You want lifecycle sync and zero tool-spec maintenance |
| **Manual** (`services create`) | External, SaaS, on-premises or unsupported runtimes. A server in **another project** that a central gateway must govern. You want to publish only a **curated subset** of tools. Accept the cost: you maintain the tool spec and the entry's lifecycle |
| Register as an **Endpoint** instead of an MCP server | The destination is a plain REST API, or you only need host-level allow or deny. Tool-level CEL and tool discovery need an **MCP server** entry |

| Registry-resolved (`get_mcp_toolset`) | Hard-coded `McpToolset(url=…)` |
|---|---|
| Central discovery and reuse; gateway can enforce; bindings supply credentials without code; URL changes need no redeploy | Fewer moving parts for a single prototype. **Nothing to govern**: under an enforcing egress gateway, an unregistered host is denied anyway |
| Cost: registry dependency at startup (fetch once and cache), plus IAM and binding setup | Cost: URLs and credentials in code, no inventory, no annotation-based policy |

| Agent Registry skills (`Skill` resources, Preview) | Skill Registry (Agent Platform, Preview) |
|---|---|
| Governed **inside the registry** next to agents and MCP servers. `gcloud alpha agent-registry skills …`. Semantic search. ZIP ≤ 500 KB | Separate Agent Platform service consumed in ADK through `GCPSkillRegistry` + `SkillToolset` (`search_skills` / `load_skill`). Its own payload validation (ZIP ≤ 10 MB). The Skill Registry docs point to Agent Registry for central governance |

Also distinguish **ADK `ApiRegistry`** (Cloud API Registry, Section 3.2.3), which is a toolset factory for Google-managed MCP servers. It is **not** the governance catalog that Agent Gateway enforces against.

**L. Gotchas (high-yield)**
- **No introspection on manual entries.** New tools are invisible until you run `services update --mcp-server-spec-content=…`, which is a **full replace**.
- **Update the `Service`, not the `McpServer`.** The typed resources are read-only.
- **Google MCP servers are global:** `--region=global` on policy bindings, otherwise `NOT_FOUND`.
- **No manual registration or bindings in `us`/`eu`.**
- **Auto-registration is single-project;** cross-project entries must be registered manually and you own their lifecycle.
- **Remove an auto-registered Google server** by disabling its API. There's no "delete entry" for it.
- **Bindings need `agentregistry.admin`;** editor can register but can't bind.
- **Unannotated tools default to destructive and open-world.** CEL on `read_only_hint` then denies them.
- **"Import tools" in the console works only for public URLs.** For private servers, paste the spec.
- **The registry MCP server rejects API keys.** `tools/list` is anonymous; `tools/call` needs `mcp.toolUser`.
- **`header_provider` never authenticates to the registry API,** and it doesn't apply to A2A agents.
- **URNs are for lookup only.** IAM always uses the agent **principal**.

**Signals**
- "Make the partner's hosted MCP server discoverable and governable" → **manual `services create --mcp-server-spec-type=tool-spec`** plus a `toolspec.json`.
- "BigQuery MCP appears in the registry without any action" → it was **auto-registered when the BigQuery API was enabled** (`global`).
- "Custom MCP server on Cloud Run should self-register" → **`--functional-type=mcp-server`**.
- "MCP server on GKE should self-register with its tools" → **`registry.gke.io/functional-type: "MCP_SERVER"`** plus the `modelcontextprotocol.info/*` annotations.
- "Registry shows old tool list" → **`services update --mcp-server-spec-content`** (no re-scan).
- "Agent should call tool as the signed-in user" → **auth manager 3LO + binding + `continue_uri`**.
- "Allow only read-only tools from this server" → **IAP egress policy with a CEL condition on the read-only annotation**.
- "Central security project governs agents in many projects" → **manual registration in the central registry** (same region or global) plus `iap.egressor`.
- "Low-code agent needs a registered MCP tool" → **Agent Studio → MCP Server from Agent Registry**.

**Distractors**
- Expecting the registry to re-scan a manually registered server.
- Patching the `McpServer` resource.
- Using a URN as an IAM member.
- Registering in `us`/`eu` to satisfy residency, then trying to bind.
- Expecting `--region=us-central1` to work for Google MCP servers.
- Using semantic search for MCP servers (it exists only for skills).
- Granting agents `agentregistry.editor` "so they can self-register".
- Relying on auto-registration across projects.
- Registering an MCP server as a plain Endpoint and expecting tool-level policies.
- Giving Cloud Run `--functional-type=agent` without `agent-identity`.

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

> 🧠 **Visual memory map:** ADK orchestration patterns
>
> [![ADK orchestration patterns](visual-memory/06-adk-orchestration-patterns.png)](visual-memory/06-adk-orchestration-patterns.png)

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

**Q1.** A retail company runs an ADK customer-service agent on Cloud Run with 3–20 instances. Users report the agent "forgets" what they said two messages earlier, intermittently. The code uses `Runner(session_service=InMemorySessionService())`. What is the best fix with the least operational overhead?

- **A.** Enable Cloud Run session affinity
- **B.** Switch to `VertexAiSessionService` pointing at an Agent Runtime instance
- **C.** Store the conversation in `temp:` state
- **D.** Increase the model context window

**Answer: B.** In-memory sessions are per-instance and lost on scale events. Agent Platform Sessions is managed and needs only an Agent Runtime instance (no code deploy). Affinity is best-effort and still loses data on restart. `temp:` is never persisted.

**Q2.** A travel agent must remember across months that a user prefers aisle seats. It must update that fact when the user later says "actually, window seats now", without keeping contradictory facts. Which approach fits?

- **A.** Write `user:seat_pref` state from a tool
- **B.** `VertexAiRagMemoryService` over transcripts
- **C.** Memory Bank via `VertexAiMemoryBankService`, generating memories from sessions
- **D.** `memories.create` for every utterance

**Answer: C.** Memory Bank extracts and **consolidates**, resolving contradictions per scope. `user:` state works for one explicit key but gives no semantic recall or extraction. RAG memory returns raw transcripts, contradictions included. `memories.create` skips consolidation.

**Q3.** A bank must host an open-weight model with its own fine-tuned weights. Traffic is predictable and high volume, and regulators prohibit multi-tenant inference services. The team wants to choose GPU types. Which serving option is best?

- **A.** Model Garden MaaS
- **B.** Self-deployed Model Garden endpoint with custom weights on a dedicated endpoint
- **C.** Gemini Flash-Lite with Provisioned Throughput
- **D.** LiteLLM pointing at a public API

**Answer: B.** Self-deployment fits custom weights, single-tenant or VPC data paths, hardware choice and lower TCO at steady volume. MaaS is serverless and multi-tenant, with no custom weights.

**Q4.** Your ADK agent calls a Gemma model served by vLLM on GKE through `LiteLlm(model="openai/...", api_base=...)`. It answers in text but never invokes its tools. What is the most likely cause?

- **A.** Gemma cannot run on GKE
- **B.** The vLLM server wasn't started with tool calling enabled (`--enable-auto-tool-choice` and a tool-call parser)
- **C.** LiteLLM doesn't support tools
- **D.** `output_key` is missing

**Answer: B.** The ADK docs call out enabling OpenAI-compatible tool calling on the serving side. Without it, the model returns plain text.

**Q5.** An underwriting process must (1) extract fields with an LLM, (2) run a deterministic Python risk score, (3) route to "auto-approve" or "manual review" by score, and (4) pause for a human underwriter on manual review. Auditors need predictable paths. What should you build?

- **A.** One `LlmAgent` with a detailed instruction
- **B.** A coordinator `LlmAgent` with sub-agents using `transfer_to_agent`
- **C.** An ADK graph `Workflow` with a function node, a router `Event(route=...)`, and a `RequestInput` node
- **D.** A `LoopAgent` with `max_iterations=4`

**Answer: C.** Graph workflows mix code and LLM nodes, give explicit routing, and support deterministic HITL through `RequestInput`. LLM-driven transfer is non-deterministic.

**Q6.** A research agent needs weather, news and stock data, which are independent calls, and then one summary. Latency matters. Which composition is correct?

- **A.** `SequentialAgent([weather, news, stocks, summarizer])`
- **B.** `SequentialAgent([ParallelAgent([weather, news, stocks]), summarizer])`, each fetcher with a distinct `output_key`
- **C.** `ParallelAgent([weather, news, stocks, summarizer])`
- **D.** `LoopAgent([weather, news, stocks])`

**Answer: B.** Fan-out/gather: the parallel children share state, so each needs a distinct key, and the summariser must run after all three. Option C runs the summariser concurrently with the fetchers.

**Q7.** An orchestrator must consult a "tax specialist" agent, combine its answer with other findings, and keep talking to the user itself. Which mechanism fits?

- **A.** Add the specialist to `sub_agents`
- **B.** Wrap the specialist in `AgentTool` and add it to `tools`
- **C.** Expose the specialist through A2A even though it's in the same codebase
- **D.** Put the specialist in a `ParallelAgent`

**Answer: B.** `AgentTool` keeps control with the parent and returns the child's result as a tool output. Transfer through `sub_agents` hands the conversation to the specialist. A2A adds needless network overhead for in-process code.

**Q8.** Security requires that each deployed agent have its own auditable principal, not a shared service account, and that IAM grants exist before the code ships. What should you do?

- **A.** Create one custom service account per agent and set it at deploy
- **B.** Create the Agent Runtime instance with only `identity_type=AGENT_IDENTITY`, grant roles to its `principal://…/reasoningEngines/ID`, then `runtimes.update` with code
- **C.** Use the Reasoning Engine Service Agent and add roles
- **D.** Grant roles to `allUsers` temporarily

**Answer: B.** Agent Identity is per-agent, SPIFFE-based and lifecycle-bound. The docs show creating the identity-only instance first so that IAM can be set before deployment. The service agent is shared across agents.

**Q9.** After a platform team deleted and redeployed an Agent Runtime agent with the same display name and code, it gets `PERMISSION_DENIED` on a BigQuery dataset it could previously read. Why?

- **A.** Context-Aware Access blocked it
- **B.** The new resource has a new resource ID and therefore a new principal; the old bindings reference the deleted identity
- **C.** BigQuery doesn't support agent identities
- **D.** The display name must be unique

**Answer: B.** An agent identity derives from the resource ID. Old bindings stay as inactive grants and must be re-created for the new principal, and cleaned up.

**Q10.** Every Agent Runtime agent in a project needs log writing and metric writing. Only the "claims" agent may read the claims dataset. What is the most maintainable IAM design?

- **A.** Grant `logging.logWriter`, `monitoring.metricWriter` and dataset access to the project principal set
- **B.** Grant logging/metrics roles to `principalSet://…/attribute.platformContainer/aiplatform/projects/NUM`, and dataset read only to the claims agent's `principal://` identifier
- **C.** Grant everything to each agent individually
- **D.** Use a shared custom service account

**Answer: B.** The docs recommend broad baseline roles on the principal set and sensitive data roles on the individual agent.

**Q11.** Engineers search a technical knowledge base by exact part numbers (for example "XR-7741-B"). Semantic-only retrieval in Vector Search 1.0 misses them, but conceptual queries work well. What should they implement?

- **A.** Switch to COSINE distance
- **B.** Hybrid index with dense and sparse (BM25) embeddings, merged by RRF with `rrf_ranking_alpha` around 0.5
- **C.** Increase `chunk_size`
- **D.** Use the `SEMANTIC_SIMILARITY` task type

**Answer: B.** Sparse embeddings capture exact tokens and RRF fuses them with semantic results. An alpha of 1 would mean dense only.

**Q12.** A RAG Engine corpus must use customer-managed encryption keys. The team also wants the managed vector store without operating any database. Which configuration?

- **A.** Serverless mode (Vector Search 2.0 backend)
- **B.** Spanner mode with RagManagedDb
- **C.** Weaviate backend
- **D.** Vector Search 1.0 backend managed by the team

**Answer: B.** The docs state that CMEK is supported by RagManagedDb (Spanner mode). Serverless mode and the Vector Search backends don't support CMEK through RAG Engine.

**Q13.** An ADK agent must let analysts query BigQuery with no extra infrastructure to host, using Google-managed MCP. The agent runs with its own identity. Which permissions does that identity need at minimum?

- **A.** `roles/mcp.toolUser` plus BigQuery data and job roles on the relevant resources
- **B.** `roles/owner`
- **C.** Only `roles/bigquery.dataViewer`
- **D.** An API key for the MCP endpoint

**Answer: A.** Remote MCP servers need `mcp.tools.call` (MCP Tool User) *and* the product's own permissions. BigQuery's MCP uses OAuth and IAM, not API keys.

**Q14.** A company has 400 internal skills (SKILL.md packages). Loading them all into every agent's prompt is slow and costly. They also want versioned, centrally governed skills. What should the ADK agent use?

- **A.** Put all skills in the instruction
- **B.** `SkillToolset` backed by `GCPSkillRegistry`, so the agent calls `search_skills`/`load_skill` on demand
- **C.** One `AgentTool` per skill
- **D.** Agent Config YAML

**Answer: B.** Skill Registry gives on-demand, targeted retrieval with immutable revisions and a default revision, and keeps the context window small.

**Q15.** A refund agent is IAM-authorised to call `issue_refund`. Policy says refunds over $100 need a manager, and the rule changes quarterly. Prompt-injection through customer emails is a concern. What is the best control?

- **A.** Add the rule to the system instruction
- **B.** A semantic governance policy (natural language constraint) on the `issue_refund` tool, enforced at Agent Gateway
- **C.** Remove the tool's IAM permission
- **D.** Lower the model temperature

**Answer: B.** Semantic governance evaluates each proposed tool call against user intent and constraints, can reference parameters, resists context poisoning, and changes without redeploying. Instructions can be overridden by injected text, and removing IAM access breaks legitimate refunds.

**Q16.** A logistics company uses a partner's hosted MCP server at `https://mcp.partner.example/mcp`. Orchestrator agents must discover the server and its tools at runtime, and security wants tool-level egress policies at Agent Gateway. What should the platform team do?

- **A.** Deploy a Cloud Run proxy with `--functional-type=mcp-server` so the partner server is auto-registered
- **B.** Run `gcloud agent-registry services create` with `--mcp-server-spec-type=tool-spec`, `--mcp-server-spec-content=@toolspec.json` and `--interfaces=url=…,protocolBinding=jsonrpc` in a supported region
- **C.** Register the URL as an Endpoint with `--endpoint-spec-type=no-spec`
- **D.** Hard-code the URL in `McpToolset` and rely on the gateway's unregistered-host allow rule

**Answer: B.** An external server needs manual registration, and the registry does not introspect it, so you must supply the tool spec for tools to be discoverable and policy-addressable. An Endpoint entry gives only host-level control. An unregistered-host rule can't express tool-level conditions.

**Q17.** The team added three tools to a manually registered MCP server last week. Agents still can't find them with `search_mcp_servers`, and the console's **Tools** tab shows the old list. What is the fix?

- **A.** Wait for the registry's nightly re-scan
- **B.** Patch the `McpServer` resource with the new tools
- **C.** Run `gcloud agent-registry services update SERVER --mcp-server-spec-content=new-toolspec.json` with the complete tool list
- **D.** Delete and recreate the auth-provider binding

**Answer: C.** Manual entries are never re-introspected. You update the writable `Service`, and the uploaded spec **replaces** the existing tool definitions, so it must contain every tool. `McpServer` is read-only.

**Q18.** An engineer binds an IAP egress policy to the auto-registered BigQuery MCP server with `gcloud iap web set-iam-policy … --resource-type=agent-registry --mcp-server=… --region=us-central1`. It fails with `NOT_FOUND`, although the server appears in the registry. Why?

- **A.** The BigQuery API isn't enabled
- **B.** Google-managed remote MCP servers are registered in the `global` location, so the binding must use `--region=global`
- **C.** Policies can't reference auto-registered servers
- **D.** The engineer lacks `roles/agentregistry.editor`

**Answer: B.** Google remote MCP servers are auto-registered globally. Regional bindings on them aren't supported and return `NOT_FOUND`. The server is visible, so its API is already enabled.

**Q19.** A bank runs a central governance project with an egress Agent Gateway in `us-central1`. MCP servers are deployed to Cloud Run in 12 workload projects with `--functional-type=mcp-server`. Agents calling them through the central gateway are denied as unregistered destinations. What is the best fix?

- **A.** Enable the Agent Registry API in the central project and wait for cross-project auto-discovery
- **B.** Manually register each MCP server in the central project's registry (in `us-central1` or `global`), grant `roles/iap.egressor` to the agent principals on those entries, and manage the entries' lifecycle
- **C.** Attach all 12 workload-project registries to the gateway
- **D.** Switch the gateway to `CLIENT_TO_AGENT` mode

**Answer: B.** Automatic registration is single-project, so cross-project components must be registered manually in the central registry, aligned by region, and their entries don't auto-update. A gateway attaches at most two registries (one global and one regional). Cross-project governance is egress-only.

**Q20.** An ADK agent must create Jira issues **as the signed-in employee**, with user consent. No OAuth secrets may appear in code. The Jira MCP server is registered in Agent Registry. What should you implement?

- **A.** Store a Jira API key in Secret Manager and send it through `header_provider`
- **B.** Create a 3-legged OAuth auth provider in Agent Identity auth manager, bind it to the agent with `gcloud agent-registry bindings create … --auth-provider=…`, handle `adk_request_credential` in the client, and pass `continue_uri` to `get_mcp_toolset`
- **C.** Grant the agent identity `roles/mcp.toolUser`
- **D.** Create a resource binding with `--target-identifier` set to the Jira server's URN

**Answer: B.** Delegated user access is 3LO through the auth manager. The binding lets ADK resolve the provider automatically, and `continue_uri` is where the user returns after consent. An API key acts as the agent, not the user. `mcp.toolUser` only covers Google MCP servers.

**Q21.** To "speed up onboarding," a platform team grants every agent identity `roles/agentregistry.editor` so that agents can self-register the tools they build. The security review flags this. What is the main risk?

- **A.** Editors can't search the registry
- **B.** An agent could modify tool annotations such as `readOnlyHint` or `destructiveHint`, which egress policies rely on, and could enroll a malicious third-party agent or server
- **C.** Editor exceeds the 100-bindings quota
- **D.** Editor grants `iap.egressor` implicitly

**Answer: B.** The docs explicitly warn against giving admin or editor roles to agents, because annotation and metadata tampering can make destructive tools look safe. Agents should hold `agentregistry.viewer`. Editor can't even create bindings (that needs admin).

**Q22.** A low-code team's Agent Studio agent connects directly to an internal MCP server by URL. After a platform update, the tool is read-only and can't be edited. What should they do?

- **A.** Recreate the agent in ADK
- **B.** Register the MCP server in Agent Registry, remove the legacy direct connection, then add it through **Add (+) → MCP Server from Agent Registry** (Location, server, Auth Config)
- **C.** Re-enter the same URL as a new direct MCP connection
- **D.** Convert the server to an A2A agent

**Answer: B.** Agent Studio has deprecated direct MCP-server connections, and existing ones become read-only, in favor of registered servers. The server must be registered first. The agent then gets all of that server's tools, with access resolved through IAM when Auth Config is `None`.

**Q23.** A support agent may call any **read-only** tool on the registered `crm-mcp` server, including tools added in the future, but no write tools. The server's tool spec carries accurate MCP annotations. What is the most precise and maintainable control?

- **A.** List the allowed tool names in the agent's system instruction
- **B.** Bind an IAP egress allow policy on `crm-mcp` (`gcloud iap web set-iam-policy … --mcp-server=crm-mcp`), granting `roles/iap.egressor` to the agent principal with a CEL condition on the tool's read-only attribute
- **C.** A Model Armor template with prompt-injection filtering
- **D.** Remove the write tools from the MCP server's code

**Answer: B.** Egress IAM policies at Agent Gateway can condition on registry tool annotations, so new read-only tools are covered automatically and write tools are denied. Instructions are bypassable. Model Armor inspects content, not tool permissions. Removing tools breaks other consumers.

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
- Agent Registry deep dive (MCP servers): data model https://docs.cloud.google.com/agent-registry/data-model · setup https://docs.cloud.google.com/agent-registry/setup · register MCP servers https://docs.cloud.google.com/agent-registry/register-mcp-servers · JSON schemas (toolspec / Agent Card) https://docs.cloud.google.com/agent-registry/json-schemas · locations https://docs.cloud.google.com/agent-registry/locations · roles https://docs.cloud.google.com/agent-registry/roles-permissions · quotas https://docs.cloud.google.com/agent-registry/quotas
- Agent Registry registration: automatic https://docs.cloud.google.com/agent-registry/automatic-registration · manual https://docs.cloud.google.com/agent-registry/manual-registration · agents https://docs.cloud.google.com/agent-registry/register-agents · endpoints https://docs.cloud.google.com/agent-registry/register-endpoints · custom ADK agents https://docs.cloud.google.com/agent-registry/register-custom-adk-agents
- Agent Registry consumption and auth: search https://docs.cloud.google.com/agent-registry/search-agents-and-tools · resolve endpoints / build orchestrators https://docs.cloud.google.com/agent-registry/resolve-endpoints-and-build-orchestrators · authenticate toolsets https://docs.cloud.google.com/agent-registry/authenticate-toolsets · bindings https://docs.cloud.google.com/agent-registry/manage-bindings · REST `Service` https://docs.cloud.google.com/agent-registry/reference/rest/v1/projects.locations.services
- ADK Agent Registry client: https://adk.dev/integrations/agent-registry/
- Cloud Run functional types (`--functional-type=agent|mcp-server`): https://docs.cloud.google.com/run/docs/ai/agent-platform-features
- IAM egress policies on registry resources (`gcloud iap web set-iam-policy --resource-type=agent-registry`): https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/configure-iam-policies
- Agent Studio (MCP Server from Agent Registry, legacy-tool migration): https://docs.cloud.google.com/gemini-enterprise-agent-platform/agent-studio/design-agents
- Gemini Enterprise: import MCP servers from Agent Registry: https://docs.cloud.google.com/gemini/enterprise/docs/connectors/custom-mcp-server/import-govern-mcp-server-agent-registry
- Google Cloud MCP servers, enable and manage: https://docs.cloud.google.com/mcp/enable-disable-mcp-servers · https://docs.cloud.google.com/mcp/manage-mcp-servers
- Agent Platform security findings: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/view-security-findings
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


---

## Section 4 — Evaluating and deploying agentic workflows (~22%)

**What the exam is really testing**
1. Can you pick the *right evaluation instrument* for the question being asked (deterministic trajectory match vs LLM-judge vs rubric vs online monitor) and wire it into CI/CD and production, not just run it once in a notebook?
2. Can you pick the *right runtime* (Agent Runtime vs Cloud Run vs GKE) from requirements — language, state, networking, isolation, cost profile, ops burden — and configure it (min instances, concurrency, sessions/memory URIs, PSC-I, revisions)?
3. Can you diagnose a misbehaving agent in production from telemetry (Cloud Trace spans, `gen_ai.*` metrics, BigQuery agent analytics, online eval scores) and apply the *specific* ADK/platform knob that fixes it (`max_llm_calls`, `max_iterations`, `ReflectAndRetryToolPlugin`, context caching, concurrency)?

> Naming (2026): Vertex AI → **Gemini Enterprise Agent Platform** ("Agent Platform"); Vertex AI Agent Engine → **Agent Runtime** (API resource is still `reasoningEngines`, SDK still exposes `AdkApp`, ADK CLI target is still `adk deploy agent_engine`, env var is still `GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY`). Expect both names in questions.

---

### 4.1 Evaluating agents in development and in production

> 🧠 **Visual memory map:** Continuous evaluation lifecycle
>
> [![Continuous evaluation lifecycle](visual-memory/10-eval-lifecycle.png)](visual-memory/10-eval-lifecycle.png)

#### 4.1.1 Why agent eval differs from model eval

Agent eval has **two axes**:
- **Trajectory / tool use** — did the agent call the right tools, with the right args, in an acceptable order? (the "how")
- **Final response** — is the answer correct, grounded, safe, well-formed? (the "what")

A correct answer produced by a wrong trajectory (e.g., skipped the authorization lookup, hit the payments API twice) is a production defect. Exam questions that mention "the agent gives correct answers but calls the refund API twice" are testing whether you evaluate the trajectory, not just the response.

#### 4.1.2 Creating test sets (golden data, prompts, edge cases)

| Test-set component | What goes in it | Why |
|---|---|---|
| **Golden dataset** | `user_content` + expected `tool_uses` (name + args) + reference `final_response` + optional `intermediate_responses` (sub-agent outputs) + `session_input.state` | Enables reference-based metrics (trajectory match, ROUGE, semantic match) |
| **Prompt coverage** | Happy paths per critical task / intent, paraphrases, multi-turn flows | Measures task coverage, not one lucky prompt |
| **Edge cases** | Ambiguous requests, missing params (agent must ask), out-of-scope, adversarial/prompt-injection, tool errors (503, empty results), long context, multilingual | Where agents actually fail in prod |
| **Conversation scenarios** (no fixed script) | `starting_prompt`, `conversation_plan`, `user_persona` | For multi-turn agents whose dialogue path legitimately varies |
| **Human-rated subset** | Human scores / pairwise choices | Calibrate an LLM judge (autorater) before trusting it |

Sources of golden data, in order of exam preference:
1. **Real sessions** captured in `adk web` → Eval tab → **"Add current session"** (then edit expected responses with the pencil icon).
2. **Production traces** (Cloud Trace / BigQuery agent analytics table) curated into eval cases — closes the loop from prod failures to regression tests.
3. **Synthetic generation**: ADK user simulation; Agent Platform `client.evals.generate_conversation_scenarios(agent_info=..., config={"count": 5, "generation_instruction": ...})`.
4. **Environment simulation** (ADK Python ≥ 1.24, experimental): intercept tool calls via `before_tool_callback` or `EnvironmentSimulationPlugin` to inject errors / mock responses deterministically — hermetic, offline eval of error handling without hitting real backends.

**ADK file formats**

| | `*.test.json` (test file) | `*.evalset.json` (eval set) |
|---|---|---|
| Scope | **One session** (may be multi-turn) | **Many sessions** ("eval cases"), long and complex |
| Role | Unit test, fast, run on every commit | Integration test, run less often |
| Criteria | `test_config.json` in the same folder | `--config_file_path` on `adk eval` (or `test_config.json`) |
| Runner | `pytest` + `AgentEvaluator.evaluate(...)` | `adk eval` CLI, `adk web` UI |
| Schema | Pydantic `EvalSet` / `EvalCase` | same |

Minimal eval case (comments removed; `.test.json` and `.evalset.json` share the schema):

```json
{
  "eval_set_id": "home_automation_light_set",
  "eval_cases": [{
    "eval_id": "turn_off_bedroom_device",
    "conversation": [{
      "invocation_id": "inv-1",
      "user_content": {"role": "user", "parts": [{"text": "Turn off device_2 in the Bedroom."}]},
      "final_response": {"role": "model", "parts": [{"text": "I have set the device_2 status to off."}]},
      "intermediate_data": {
        "tool_uses": [{"name": "set_device_info",
                       "args": {"location": "Bedroom", "device_id": "device_2", "status": "OFF"}}],
        "intermediate_responses": []
      }
    }],
    "session_input": {"app_name": "home_automation_agent", "user_id": "test_user", "state": {}}
  }]
}
```

Legacy test files: migrate with `AgentEvaluator.migrate_eval_data_to_new_schema`.

#### 4.1.3 ADK evaluation criteria — metric → what it catches

| Criterion | Type | Reference needed | LLM judge | Works with user sim | Catches |
|---|---|---|---|---|---|
| `tool_trajectory_avg_score` | Deterministic | Yes (expected `tool_uses`) | No | No | Wrong tool, wrong args, wrong order, extra/missing calls. `match_type`: `EXACT` (default), `IN_ORDER`, `ANY_ORDER` |
| `response_match_score` | ROUGE-1 unigram overlap | Yes | No | No | Gross answer drift; cheap CI gate. Penalizes valid paraphrase |
| `response_evaluation_score` | Agent Platform coherence score | Yes | Yes | No | Coherence |
| `final_response_match_v2` | Semantic equivalence to reference | Yes | Yes (valid/invalid, majority vote over `num_samples`) | No | Wrong facts while tolerating phrasing/format differences |
| `rubric_based_final_response_quality_v1` | Rubric yes/no per rubric | No (rubrics) | Yes | Yes | Tone, conciseness, policy adherence when no single reference exists |
| `rubric_based_tool_use_quality_v1` | Rubric over tool use | No (rubrics) | Yes | Yes | "Tool A must be called before Tool B", "never call delete without confirm" |
| `rubric_based_multi_turn_trajectory_quality_v1` | Rubric over whole conversation | No | Yes | Yes | Workflow hand-offs in multi-agent / graph workflows |
| `hallucinations_v1` | Segment → validate each sentence vs context (instructions, prompt, tool defs, tool results) | No | Yes | Yes | **Groundedness**: claims unsupported by / contradicting tool outputs. `evaluate_intermediate_nl_responses: true` also checks sub-agent text |
| `safety_v1` | Delegated to Agent Platform Eval SDK | No | Yes | Yes | Harmful content. **Needs `GOOGLE_CLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION`** |
| `multi_turn_task_success_v1` | Eval SDK | No | Yes | Yes | Did the conversation achieve its goal (outcome, not path) |
| `multi_turn_trajectory_quality_v1` | Eval SDK | No | Yes | Yes | Efficiency/logic of the path across turns |
| `multi_turn_tool_use_quality_v1` | Eval SDK | No | Yes | Yes | Tool selection/args across turns, reference-free |
| `per_turn_user_simulator_quality_v1` | Judge on the simulator | No | Yes | Yes | Whether the *simulated user* stayed on plan/persona (validates your test harness) |

Defaults when no criteria supplied: **`tool_trajectory_avg_score: 1.0`** and **`response_match_score: 0.8`**.

`test_config.json` / `EvalConfig` examples:

```json
{
  "criteria": {
    "tool_trajectory_avg_score": {"threshold": 1.0, "match_type": "IN_ORDER"},
    "final_response_match_v2": {
      "threshold": 0.8,
      "judge_model_options": {"judge_model": "gemini-flash-latest", "num_samples": 5}
    },
    "hallucinations_v1": {"threshold": 0.8, "evaluate_intermediate_nl_responses": true},
    "safety_v1": 0.8,
    "rubric_based_tool_use_quality_v1": {
      "threshold": 0.8,
      "rubrics": [{"rubric_id": "auth_first",
                   "rubric_content": {"text_property": "The agent calls verify_customer before any refund tool."}}]
    }
  }
}
```

ADK's official recommendations (memorize — they map 1:1 to exam answers):
- **CI/CD / regression** → `tool_trajectory_avg_score` + `response_match_score` (fast, predictable, cheap).
- **Trusted reference exists, phrasing varies** → `final_response_match_v2`.
- **No reference, but you can describe "good"** → `rubric_based_final_response_quality_v1`.
- **Validate tool-use logic/order** → `rubric_based_tool_use_quality_v1`.
- **Grounded in context / tool outputs** → `hallucinations_v1`.
- **Harmful content** → `safety_v1`.
- **Multi-turn goal completion** → `multi_turn_task_success_v1`.

**User simulation** (ADK Python ≥ 1.18; personas ≥ 1.26): `ConversationScenario{starting_prompt, conversation_plan, user_persona}`; prebuilt personas `EXPERT`, `NOVICE`, `EVALUATOR`; custom `UserPersona` with `behaviors[]` and `violation_rubrics`. **Gotcha:** reference-based criteria (`tool_trajectory_avg_score`, `response_match_score`, `final_response_match_v2`) are **not supported** with user simulation — use hallucinations/safety/rubric/multi-turn criteria.

```bash
adk eval_set create my_agent eval_set_with_scenarios
adk eval_set add_eval_case my_agent eval_set_with_scenarios \
  --scenarios_file my_agent/conversation_scenarios.json \
  --session_input_file my_agent/session_input.json
adk eval my_agent --config_file_path my_agent/eval_config.json eval_set_with_scenarios --print_detailed_results
```

**Custom metrics (ADK code-based autorater)**: a Python (sync or async) function `(eval_metric, actual_invocations, expected_invocations, conversation_scenario) -> EvaluationResult`, registered under `custom_metrics.<name>.code_config.name` = import path, with the threshold under `criteria.<name>`. Use for domain checks (PII leakage, SQL validity, policy lookups, calling an external classifier).

#### 4.1.4 Running ADK eval

| Mode | Command / API | Use |
|---|---|---|
| Web UI | `adk web <agents_dir>` → Eval tab → Run Evaluation (sliders for trajectory/response thresholds); **Trace** tab (Event/Request/Response/Graph) | Author cases, debug failures side-by-side (Actual vs Expected) |
| pytest | `await AgentEvaluator.evaluate(agent_module="my_agent", eval_dataset_file_path_or_dir="tests/.../x.test.json")` | Unit/integration tests in CI |
| CLI | `adk eval <AGENT_DIR> <evalset.json|eval_set_id>[:eval_1,eval_2] [--config_file_path=...] [--print_detailed_results]` | CI step, nightly jobs. Agent dir must expose `agent.root_agent`. Cannot mix file paths and eval-set IDs in one command |
| Conformance | `adk web --extra_plugins=google.adk.cli.plugins.recordings_plugin.RecordingsPlugin` + `adk conformance record tests/cat/case none` → `adk conformance test [--generate_report --report_dir=reports]` | **Replay-mode regression**: compares live LLM requests/responses/tool calls to recorded baseline; non-zero exit blocks PR merge |
| Optimize | `adk optimize` (Python ≥ 1.24) | Eval-guided instruction optimization using a sampler over an eval set |
| Agents CLI | `agents-cli eval run` (evalsets in `tests/eval/evalsets/*.evalset.json`, criteria in `tests/eval/eval_config.json`) | Scaffolded projects; same ADK eval engine |

#### 4.1.5 Gen AI evaluation service (Agent Platform)

Two SDK surfaces — know which is which:

| | **GenAI Client** `from vertexai import Client; client.evals.*` | **Legacy** `from vertexai.evaluation import EvalTask` |
|---|---|---|
| Status | Recommended (Preview) | GA, maintained, not actively developed |
| Adaptive rubrics | Yes (`types.RubricMetric.GENERAL_QUALITY`, `INSTRUCTION_FOLLOWING`, `TEXT_QUALITY`, agent: `FINAL_RESPONSE_QUALITY`, `TOOL_USE_QUALITY`, `HALLUCINATION`, `SAFETY`; multi-turn: `MULTI_TURN_TASK_SUCCESS`, `MULTI_TURN_TOOL_USE_QUALITY`) | No |
| Flow | `generate_conversation_scenarios` → `run_inference(agent=<reasoningEngine resource>, src=df)` → `evaluate()` / `create_evaluation_run()` / `batch_evaluate()` → `generate_loss_clusters()` → `client.optimizer.optimize(targets=["system_prompt"])` | `EvalTask(dataset, metrics).evaluate(runnable=agent)` |
| Trajectory metrics | via agent rubric metrics | `trajectory_exact_match`, `trajectory_in_order_match`, `trajectory_any_order_match`, `trajectory_precision`, `trajectory_recall`, `TrajectorySingleToolUse(tool_name=...)` |

Trajectory metric semantics (dataset columns `predicted_trajectory`, `reference_trajectory` — list of `{tool_name, tool_input}`):

| Metric | Score | Passes when |
|---|---|---|
| `trajectory_exact_match` | 0/1 | Identical calls, identical order, nothing extra |
| `trajectory_in_order_match` | 0/1 | All reference calls present in order; extras allowed |
| `trajectory_any_order_match` | 0/1 | All reference calls present, any order; extras allowed |
| `trajectory_precision` | 0–1 | (# predicted calls also in reference) / (# predicted) — **penalizes extra/unnecessary calls** |
| `trajectory_recall` | 0–1 | (# reference calls also in predicted) / (# reference) — **penalizes missed essential calls** |
| `trajectory_single_tool_use` | 0/1 | A named tool was used at all (**no reference needed**) |

`latency` (seconds) and `failure` (1 = error) are added to agent eval results automatically.

Metric families:
- **Computation-based** — `exact_match`, `bleu`, `rouge_1`, `rouge_l`, trajectory metrics, `CustomMetric`/`types.Metric(custom_function=...)`. Ground truth usually required; cheap and fast.
- **Model-based (autorater / LLM-as-judge)** — **Pointwise** (score one response against criteria, e.g., 1–5) or **Pairwise** (judge picks better of candidate vs baseline; **Gemini judge only**). Ground truth optional; slower, costs judge tokens (default judge `gemini-2.5-flash`, consumes Gemini throughput quota; MetricX/COMET use other models).
- Datasets from **JSONL/CSV in Cloud Storage, BigQuery table, or pandas DataFrame**. Supported agent types: Agent Runtime ADK template, LangChain template, or a custom function returning `{response, predicted_trajectory}`.

Custom LLM judge (legacy API, still the documented judge-customization surface):

```python
from vertexai.evaluation import EvalTask, PointwiseMetric, PointwiseMetricPromptTemplate
from vertexai.preview.evaluation import AutoraterConfig

follows_traj = PointwiseMetric(
    metric="response_follows_trajectory",
    metric_prompt_template=PointwiseMetricPromptTemplate(
        criteria={"Follows trajectory": "Response reflects information gathered by the tool calls..."},
        rating_rubric={"1": "Follows trajectory", "0": "Does not"},
        input_variables=["prompt", "predicted_trajectory"]),
    system_instruction="You are an expert evaluator.")

EvalTask(dataset=df,
         metrics=["trajectory_in_order_match", "trajectory_precision", follows_traj],
         autorater_config=AutoraterConfig(sampling_count=6)).evaluate(runnable=remote_agent)
```

**Custom autorater** knobs (`AutoraterConfig`): `autorater_model` (publisher model or **tuned judge endpoint**), `sampling_count` (1–32, **default 4**; more = more consistent, slower/costlier), `flip_enabled` (pairwise only; swaps candidate/baseline in half the calls to cancel **position bias**), `generation_config`; `system_instruction` on the metric. **Validate the judge before trusting it**: `evaluate_autorater(...)` against a dataset with `{metric}/human_rating` (pointwise) or `{metric}/human_pairwise_choice` (pairwise) → balanced accuracy, F1, confusion matrix (AutoSxS reports Cohen's kappa). `tune_autorater(base_model=..., train_dataset=...)` produces a tuned judge.

#### 4.1.6 Choosing the framework — decision table

| Requirement | Pick | Why |
|---|---|---|
| Dev inner loop / unit tests on an ADK agent, in repo, per PR | **ADK eval** (`.test.json` + pytest / `adk eval`) | Local, framework-native, understands sessions/sub-agents/state |
| Regression gate that must be deterministic and cheap | ADK `tool_trajectory_avg_score` + `response_match_score`, or `adk conformance test` (replay) | No judge variance, no judge cost |
| Evaluate a *deployed* agent (Agent Runtime) at scale; compare two models/prompts; large datasets in BQ/GCS | **Gen AI evaluation service** (`client.evals.run_inference(agent=...)`, `batch_evaluate`) | Managed, scales, pairwise, adaptive rubrics, loss clusters, results viewable in console |
| Non-ADK agent (LangGraph, CrewAI, custom) | Gen AI evaluation service (custom function / Agent Runtime template) | ADK eval is ADK-specific |
| Domain-specific quality no built-in metric expresses (compliance, SQL correctness, brand voice) with human-agreement requirement | **Custom autorater** (custom `PointwiseMetric`/rubric + tuned judge, calibrated vs human ratings) or ADK custom metric | Built-in rubrics too generic; calibration proves judge validity |
| Production quality drift | **Online monitors** (Agent Platform) | Continuous, samples live traces |

Agent Platform frames three evaluation types: **Rapid evaluation** (dev, frequent), **Test case evaluation** (regression, scheduled CI/CD), **Online monitoring** (production, continuous).

#### 4.1.7 Continuous evaluation pipelines (CI/CD + production)

Reference pipeline (Agents CLI `scaffold enhance` generates `.cloudbuild/`, `deployment/` Terraform, `tests/`):

1. **PR trigger (Cloud Build)**: lint + unit tests + `adk eval` / `pytest` over `.test.json` with deterministic criteria (trajectory 1.0 `EXACT`/`IN_ORDER`, ROUGE ≥ 0.8) + `adk conformance test`. Fail build = block merge.
2. **Merge → deploy to staging** (`agents-cli deploy` / `adk deploy ...`), then run the **evalset** (multi-turn, LLM-judge criteria: `final_response_match_v2`, `hallucinations_v1`, `safety_v1`, rubrics) and/or Gen AI eval against the staging agent. Load test.
3. **Gate + promote**: manual approval or threshold gate → prod. On Agent Runtime create a new **revision** and shift traffic (`trafficSplitManual` canary → `trafficSplitAlwaysLatest`); on Cloud Run `gcloud run deploy --no-traffic` + `gcloud run services update-traffic --to-revisions`.
4. **Production**: **Online monitors** — scheduled loop (~every 10 min) that samples traces from Cloud Trace/Logging by filter (agent, duration, token usage), scores them with Evaluation Service metrics (safety, response quality, hallucination, tool-use quality, custom), writes results to Cloud Logging and numeric scores to **Cloud Monitoring** (alertable), visible in the agent's **Observability → Evaluation** dashboard and per-trace Evaluation tab. Controls: **sampling percentage** and **max samples per run** (cost cap).
5. **Close the loop**: failed/low-score traces → loss clusters → new golden cases → optimizer / prompt fix → back to step 1.

Online monitor telemetry prerequisites (common troubleshooting question — "monitor active, no scores"):
```
OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=EVENT_ONLY
```
Invoke-agent span needs `gen_ai.agent.name`, `gen_ai.agent.description`, `gen_ai.conversation.id`; inference event `gen_ai.client.inference.operation.details` must carry input/output messages, system instructions, tool definitions. Multimodal: upload to GCS via `OTEL_INSTRUMENTATION_GENAI_UPLOAD_*`. Security: online eval runs as the project-level service agent (P4SA) — restrict who can create `OnlineEvaluator`.

#### 4.1.8 Evaluating response AND retrieval quality against a golden dataset (ADK)

For a RAG / search-tool agent, split the question:

| Quality dimension | ADK instrument |
|---|---|
| Was the retriever invoked, with the right query/filters/corpus? | `tool_trajectory_avg_score` (`IN_ORDER`/`ANY_ORDER` if other calls allowed) with expected `tool_uses` args; `rubric_based_tool_use_quality_v1` for "search before answering" rules |
| Did retrieval return the right evidence? | Custom metric that inspects the tool response in `Invocation.intermediate_data` against golden doc IDs (recall@k / precision@k) (pattern; ADK has no built-in retrieval-recall metric — unverified that none exists) |
| Is the answer grounded in what was retrieved? | `hallucinations_v1` (validates each sentence against tool results) |
| Is the answer correct vs golden answer? | `final_response_match_v2` (semantic) or `response_match_score` (lexical) |
| Multi-hop / sub-agent retrieval | `intermediate_responses` + `evaluate_intermediate_nl_responses: true` |

Interpretation matrix: **low trajectory + low answer** → retrieval not invoked / wrong args (tool description or instruction problem). **High trajectory + high hallucination failures** → retrieved evidence irrelevant or model ignoring it (chunking/reranking/prompt). **High grounding + low `final_response_match_v2`** → corpus missing/outdated content (data problem, not model).

**Exam signals — 4.1**

| Keyword in stem | Likely answer |
|---|---|
| "fast, deterministic regression in CI", "every pull request" | `tool_trajectory_avg_score` + `response_match_score`; `adk eval`/pytest in Cloud Build; `adk conformance test` |
| "correct meaning, different wording" | `final_response_match_v2` (not ROUGE) |
| "no reference answer", "tone", "concise", "policy" | Rubric-based criteria / adaptive rubrics (`GENERAL_QUALITY` with guidelines) |
| "unsupported claims", "not grounded in tool output" | `hallucinations_v1` / `HALLUCINATION` |
| "extra unnecessary tool calls" | `trajectory_precision` (or `EXACT` match) |
| "missed a required step" | `trajectory_recall` / `IN_ORDER` |
| "order doesn't matter, e.g., 5 searches" | `ANY_ORDER` / `trajectory_any_order_match` |
| "just verify a tool was used" | `trajectory_single_tool_use` |
| "compare new prompt vs current (A/B) offline" | Pairwise model-based metric; `flip_enabled` |
| "judge disagrees with SMEs" | Calibrate autorater vs human ratings (`evaluate_autorater`), custom rubric, tuned judge, raise `sampling_count` |
| "conversation path varies" | User simulation (`ConversationScenario`) + reference-free criteria |
| "test tool failure handling without calling real APIs" | ADK Environment Simulation |
| "quality degrades weeks after launch" | Online monitors → Cloud Monitoring alert |

**Common distractors**: using ROUGE/BLEU to judge semantic correctness; using `EXACT` trajectory when the agent legitimately issues variable numbers of searches; using `tool_trajectory_avg_score` with user simulation (unsupported); running LLM-judge metrics on every commit (slow, flaky, costly); "fine-tune the model" as the first response to an eval failure; relying on Cloud Monitoring latency metrics to detect answer quality drift.

---

### 4.2 Deploying and scaling production workloads

#### 4.2.1 Runtime selection

> 🧠 **Visual memory map:** Agent Runtime vs Cloud Run vs GKE
>
> [![Agent Runtime vs Cloud Run vs GKE](visual-memory/11-runtime-selection.png)](visual-memory/11-runtime-selection.png)

Google's own guidance (Architecture Center "Choose your agentic AI architecture components"):

| Use case | Runtime |
|---|---|
| Python agent, fully managed, minimal ops | **Agent Runtime** |
| Containerized, serverless, event-driven scaling, any language | **Cloud Run** |
| Containerized, complex stateful needs, fine-grained infra control | **GKE** |
| Already standardized on Cloud Run or GKE | Stay on that platform |

Detailed comparison:

| Dimension | Agent Runtime | Cloud Run | GKE |
|---|---|---|---|
| Deploy | `adk deploy agent_engine`, `client.runtimes.create(agent=AdkApp(...))`, source-file deploy (≤ 8 MB `source_packages`), `agents-cli deploy` | `adk deploy cloud_run`, `gcloud run deploy` (custom FastAPI + Dockerfile), `agents-cli deploy` | `adk deploy gke` (Python only), `kubectl apply` manifests |
| Language | Python (ADK, LangChain, LangGraph, AG2, LlamaIndex, A2A, custom); Go via `adkgo deploy agentengine` | Any (container) | Any (container) |
| Scaling | `min_instances` 0–10 (default 1), `max_instances` 1–1000 (default 100; **1–100 with VPC-SC or PSC-I**), `container_concurrency` default 9 | 0→N autoscale, max concurrency up to 1,000/instance, default max 100 instances/revision, min instances, startup CPU boost | HPA/VPA/cluster autoscaler, node pools, GPUs/TPUs |
| Resources | `resource_limits` cpu ∈ {1,2,4,6,8}, memory 1–32 Gi; default 4 CPU / 4 Gi | Up to Cloud Run limits; GPUs available | Any machine type, accelerators |
| Request duration | Managed (streaming/bidi supported; bidi quota 10 concurrent/min) | **Max 60 min** request timeout (default 5 min) — constrains long agent tasks | Unbounded |
| Sessions / memory | **Built-in**: Agent Platform Sessions + Memory Bank (`VertexAiMemoryBankService` default when deployed) | External: `--session_service_uri agentengine://<id>` / SQLAlchemy URL (Cloud SQL), `--memory_service_uri agentengine://` or `rag://`, `--artifact_service_uri gs://`; Memorystore/Firestore | Same external options; in-cluster stores |
| Private networking | **PSC interface** (network attachment, ≥ /28 subnet, DNS peering); **no internet egress by default** → proxy VM (+ Cloud NAT with VPC-SC) | **Direct VPC egress** or Serverless VPC Access connector | Native VPC (Pod/node IPs) |
| Security features | VPC-SC, CMEK, DRZ, HIPAA, Access Transparency, Access Approval; per-agent Agent Identity / service account; Secret Manager env | IAM invoker, ingress controls, Agent Identity & Agent Registry integration, Secret Manager, sandboxed code execution | Workload Identity, network policies, **GKE Agent Sandbox (gVisor)** for untrusted code, Model Armor integration |
| Observability | Built-in: Cloud Trace, Logging, Monitoring on `aiplatform.googleapis.com/ReasoningEngine`; Observability tab (Overview, Evaluation, Models, Tools, Usage, Logs); Traces tab (session/trace/span views) | Cloud Run metrics + you set ADK OTel export (`--otel_to_cloud`) | You set OTel collector/exporters |
| Release mgmt | **Revisions + traffic split** (v1beta1): `trafficSplitManual` / `trafficSplitAlwaysLatest`; 950 revisions/agent, 6,000/project/region (not adjustable; auto GC) | Revisions + `update-traffic`, tags, gradual rollout | Deployments, Argo/Cloud Deploy canaries |
| Custom MCP server hosting | **Not supported** (can call MCP servers) | Yes (streamable HTTP) | Yes |
| Cost model | Compute billed on vCPU/memory time of instances (unverified exact SKUs); **idle time with `min_instances` not billed while in Preview** (subject to change); free tier; Sessions/Memory Bank billed separately (unverified units) | Request- or instance-based billing, scale to zero; min instances cost idle; WebSockets keep instance billed | Node/cluster costs; best for steady high volume with CUDs; Autopilot pay-per-Pod |
| Ops burden | Lowest | Low | Highest |
| Key quotas | 90 Query/StreamQuery per min per region, 100 agent resources, session writes 100/min, event appends 300/min, Memory Bank writes 100/min, reads 300/min (defaults) | Service limits | Cluster limits |

Cloud Run resource types for agents: **Services** (request-driven, scale to zero — chat APIs), **Instances** (always-on singleton agent loops), **Worker pools** (queue consumers — Pub/Sub/Kafka agent fleets, no public endpoint), **Jobs** (run-to-completion — **batch evaluations**, ingestion).

GKE specifics for agents: **GKE Agent Sandbox** add-on — `SandboxTemplate` (`runtimeClassName: gvisor`, `automountServiceAccountToken: false`, `runAsNonRoot`, drop ALL caps), `SandboxWarmPool` (sub-second claims), `SandboxClaim`; default-deny network; Pod snapshots for pause/resume. **Agent Substrate** (OSS, GKE Standard; private-GA for prod) suspends idle agents to snapshots for high-density long-lived agents. GKE is also where you co-host **open models** (Gemma/vLLM) next to the agent — Gemini itself cannot be self-hosted on Cloud Run/GKE.

**Trade-offs to state explicitly**
- Agent Runtime: fastest to production, managed sessions/memory/eval/observability — but Python-centric, less compute customization, PSC-I needed for private resources (no default internet egress), custom MCP servers must live elsewhere, per-region query quota (90/min default) must be raised for high-QPS.
- Cloud Run: any language, scale-to-zero, cheap for spiky traffic — but stateless (externalize sessions or lose them), 60-min request cap, cold starts unless min instances (which cost money).
- GKE: max control, isolation (gVisor), GPUs, long-running and stateful agents, best unit economics at steady scale with CUDs — but you own cluster ops, upgrades, autoscaling tuning, OTel wiring.

#### 4.2.2 Deployment commands and configuration

```bash
# Agent Runtime (ADK CLI)
adk deploy agent_engine --project=$PROJECT --region=us-central1 \
  --display_name="Support Agent" --otel_to_cloud ./support_agent
# Query endpoint: https://$LOC-aiplatform.googleapis.com/v1/projects/$P/locations/$LOC/reasoningEngines/$ID:query

# Cloud Run (ADK CLI) — externalize state; pass gcloud flags after "--"
adk deploy cloud_run --project=$PROJECT --region=us-central1 --service_name=support-agent \
  --session_service_uri=agentengine://$AGENT_RUNTIME_ID \
  --memory_service_uri=agentengine://$AGENT_RUNTIME_ID \
  --artifact_service_uri=gs://$BUCKET ./support_agent \
  -- --no-allow-unauthenticated --min-instances=2

# GKE (Python only; default ClusterIP; grant Agent Platform User to default KSA via Workload Identity)
adk deploy gke --project=$PROJECT --cluster_name=agents --region=us-central1 \
  --service_type=LoadBalancer ./support_agent

# Agents CLI (scaffold CI/CD + Terraform, then deploy target from pyproject.toml)
agents-cli scaffold enhance --deployment-target agent_runtime   # or cloud_run
agents-cli deploy
agents-cli infra single-project   # telemetry infra: SA, GCS bucket, BigQuery dataset
```

Agent Platform SDK with resource controls and private networking:

```python
remote = client.runtimes.create(agent=AdkApp(agent=root_agent), config={
    "min_instances": 2, "max_instances": 50,
    "resource_limits": {"cpu": "4", "memory": "8Gi"},
    "container_concurrency": 36,            # async ADK: multiple of 9
    "service_account": "support-agent@proj.iam.gserviceaccount.com",
    "env_vars": {"GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY": "true",
                 "OTEL_SEMCONV_STABILITY_OPT_IN": "gen_ai_latest_experimental",
                 "OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT": "EVENT_ONLY"},
    "psc_interface_config": {"network_attachment": "projects/p/regions/us-central1/networkAttachments/agents",
        "dns_peering_configs": [{"domain": "corp.internal.", "target_project": "host-proj", "target_network": "vpc"}]},
})
```

Scaling rules from the Agent Runtime "Optimize and scale" guide:
- **Cold start**: default `min_instances=1` → 300 concurrent requests averaged ~4.7 s cold vs ~0.4 s warm; `min_instances=10` → ~1.4 s. Max `min_instances` is 10; beyond that, smooth load with a **queue**.
- **Underutilized async workers**: default `container_concurrency=9` assumes sync code; each container runs 9 agent processes, so per-process concurrency = `container_concurrency / 9`. For async ADK agents set a **multiple of 9 (e.g., 36)** — max latency dropped from ~60 s to ~7 s in Google's test. Too high → OOM.
- Recommended starting point for sync agents: `2 * cpu + 1`.

Gotchas:
- `adk deploy cloud_run` without `--session_service_uri` / `--artifact_service_uri` falls back to **in-memory** — sessions vanish on instance recycle and are not shared across instances. Cloud Run session affinity is **best-effort** and does not fix this.
- `adk deploy` (Python) does **not** ship the dev web UI unless `--with_ui` — never ship `--with_ui` to prod.
- `adk deploy gke` uses the `default` KSA in `default` namespace — bind Workload Identity or model calls 403.
- PSC-I config cannot be changed on an existing agent without recreating it; subnets `100.64.0.0/20` and `240.0.0.0/4` not allowed; Shared VPC needs `roles/compute.networkUser` (and network attachment perms) for the Agent Platform Service Agent.
- Revisions/traffic splitting on Agent Runtime are **v1beta1**; queries addressed to a specific revision path bypass traffic rules. Changing a versioned field (`env`, `minInstances`, `containerConcurrency`, `pscInterfaceConfig`, package spec, `identityType`...) creates a new revision.
- Payload capture: `adk deploy agent_engine --otel_to_cloud` sets `ADK_CAPTURE_MESSAGE_CONTENT_IN_SPANS=false` by default; on Cloud Run/GKE set it explicitly (PII risk).

#### 4.2.3 Troubleshooting agent issues

> 🧠 **Visual memory map:** Troubleshooting agents in production
>
> [![Troubleshooting agents in production](visual-memory/12-troubleshooting.png)](visual-memory/12-troubleshooting.png)

| Symptom | Likely cause | Diagnostic signal | Fix |
|---|---|---|---|
| Invocation runs for minutes, token bill spikes, eventually errors | **Reasoning loop** (model re-calls same tool, critic/reviser never converge) | Trace: repeated `call_llm`→`execute_tool` spans; `gen_ai.invoke_agent.inference_calls` / `tool_calls` histograms high; BQ `v_tool_completed` same tool+args repeated | `RunConfig(max_llm_calls=N)` (default **500**; ≤0 = unlimited, not for prod); `LoopAgent(max_iterations=N)` + exit via `exit_loop` tool setting `tool_context.actions.escalate=True`; tighten tool descriptions; return structured "no results" instead of empty |
| LoopAgent always hits max_iterations | Exit condition never emitted | No event with `escalate=True` in trace | Give critic an explicit exit tool + instruction; check state key names |
| p95 latency high, LLM spans normal | **Tool invocation latency** (slow API/DB/MCP) | `gen_ai.execute_tool.duration` by `gen_ai.tool.name`; Observability → Tools tab p95; BQ `TOOL_COMPLETED` latency | Cache, parallelize (ParallelAgent), timeouts, move tool closer (region/VPC), async tools, long-running tool pattern |
| p95 high at traffic spikes only | Cold starts / under-provisioned concurrency | Request latency vs instance count; Agent Runtime Usage tab CPU/memory | Raise `min_instances`; `container_concurrency` multiple of 9 for async; Cloud Run min instances + startup CPU boost; queue to smooth |
| Latency grows with conversation length; cost per turn rising | Context bloat (full history every turn) | `gen_ai.client.token.usage` input tokens climbing per session; `adk.experimental.invoke_agent.input_tokens` | `GetSessionConfig(num_recent_events=...)`, `context_window_compression`, event compaction, context caching |
| Transient tool failures abort the run | No retry/reflection | `TOOL_ERROR` rows; `execute_tool` spans with `error.type` | `ReflectAndRetryToolPlugin(max_retries=3)` (per-tool, `TrackingScope.INVOCATION` default); subclass `extract_error_from_result` for tools returning `{"status":"error"}`; idempotent tools |
| 429 / RESOURCE_EXHAUSTED | Model quota or Agent Runtime query quota (90/min) | Observability → Models tab "quota failures"; LLM_ERROR rows | Quota increase, Provisioned Throughput, global endpoint, backoff, model routing to cheaper model |
| Answers subtly worse weeks after launch, no errors | **Drift** (user behavior shift, data/corpus staleness, model version change behind `-latest` alias) | Online monitor scores trending down in Cloud Monitoring; hallucination rate up | Pin model versions; refresh corpus; add drifted prod traces to golden set; rerun evalset; alert on monitor metrics |
| Confident wrong facts | Hallucination / ungrounded answer | `hallucinations_v1`/`HALLUCINATION` scores; trace shows answer without retrieval span | Force retrieval (instruction/rubric), grounding, lower temperature, `before_model_callback` checks, human-in-the-loop for high-risk actions |
| Sessions lost / user "forgotten" after deploy | In-memory session service on Cloud Run/GKE | Session IDs not found after restart | `--session_service_uri agentengine://` or DB URL; Memory Bank for long-term |
| Agent on Agent Runtime can't reach on-prem API / internet | No egress path | Connection timeouts in logs | PSC-I + network attachment + DNS peering; proxy VM (+ Cloud NAT under VPC-SC) |
| Online monitor shows no results | Missing GenAI telemetry | Traces lack `gen_ai.*` attributes | Set `OTEL_SEMCONV_STABILITY_OPT_IN` + `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`; check monitor filters; search Logging by `resource.labels.online_evaluator` |
| BigQuery analytics rows missing in EU dataset | Older plugin cross-region stream routing bug | `get_drop_stats()` counters | Upgrade plugin; check `roles/bigquery.dataEditor` on table + `bigquery.jobUser` |

#### 4.2.4 Monitoring and optimizing — performance, reliability, cost

**Telemetry stack (all OpenTelemetry GenAI semantic conventions)**
- **Traces** → Cloud Trace. ADK span names: `invoke_agent`, `call_llm`, `generate_content`, `execute_tool`. Attributes: `gen_ai.agent.name`, `gen_ai.conversation.id`, `gcp.vertex.agent.invocation_id`, `gcp.vertex.agent.session_id`, `gcp.vertex.agent.llm_request/llm_response/tool_call_args/tool_response` (redacted unless capture enabled). Enable via `--otel_to_cloud` (CLI), `GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true` (Agent Runtime; replaces legacy `enable_tracing=True`), or `google.adk.telemetry.google_cloud.get_gcp_exporters(enable_cloud_tracing=True)` + `maybe_set_otel_providers`. Requires Cloud Trace, Logging and **Telemetry (OTLP) API** enabled; viewers need `roles/cloudtrace.user`, `roles/logging.viewer`.
- **Metrics** (ADK): `gen_ai.invoke_agent.duration`, `gen_ai.invoke_workflow.duration`, `gen_ai.execute_tool.duration`, `gen_ai.invoke_agent.inference_calls`, `gen_ai.invoke_agent.tool_calls`, `gen_ai.client.operation.duration`, `gen_ai.client.token.usage` (split by `gen_ai.token.type`); experimental per-invocation token rollups incl. `cache_read.input_tokens`, `reasoning.output_tokens`. Agent Runtime built-ins on `aiplatform.googleapis.com/ReasoningEngine`: request count, request latencies, container CPU/memory allocation time.
- **Logs** → Cloud Logging. Python log levels: use INFO/WARNING in prod; DEBUG logs full prompts (sensitive). Prompt capture: `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` = `NO_CONTENT` | `EVENT_ONLY` | `SPAN_ONLY` | `SPAN_AND_EVENT` (span modes also need `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`).
- **BigQuery Agent Analytics plugin** (`BigQueryAgentAnalyticsPlugin(project_id, dataset_id)` on the `App`; `pip install "google-adk[bigquery-analytics]"`): async Storage Write API into `agent_events` (day-partitioned, clustered on `event_type, agent, user_id`), events such as `LLM_REQUEST`, `LLM_RESPONSE` (tokens, cache metadata, finish reason, TTFT), `TOOL_STARTING/COMPLETED/ERROR` (with provenance LOCAL/MCP/SUB_AGENT/A2A), `AGENT_TRANSFER`, `HITL_*`, `STATE_DELTA`; auto views (`v_llm_response`, `v_tool_error`...); rows carry `trace_id` to join with Cloud Trace; large payloads offloaded to GCS. IAM: `bigquery.jobUser` (project) + `bigquery.dataEditor` (table). Use for SQL-level cost/latency/tool-failure analytics, curating eval datasets, and dashboards (Looker Studio). Trade-off: Storage Write API cost + PII in BigQuery — use `max_content_length`, allowlists, and dataset IAM.
- **Dashboards**: Agent Platform agent → Observability tab: **Overview** (sessions, turns/session, invocations, token in/out, p50/p95/p99, errors), **Evaluation** (online monitor scores), **Models** (per-model p95, errors, quota failures, tokens), **Tools** (per-tool p95, errors, "no tool called" rate), **Usage** (container CPU/memory, tokens), **Logs**. Traces tab: session/trace/span views with DAG. Topology tab for dependencies. Cloud Monitoring **Application Monitoring** (App Hub) aggregates agents, MCP servers, Agent Gateway, Model Armor.

**Optimization levers and their trade-offs**

| Lever | Mechanism | Buys | Costs |
|---|---|---|---|
| Context caching | `App(context_cache_config=ContextCacheConfig(min_tokens=2048, ttl_seconds=600, cache_intervals=5))` (Gemini 2.0+) | Lower input-token cost + latency for large static instructions/docs | Cache storage; stale content until TTL/refresh; only above `min_tokens` |
| History trimming | `RunConfig(get_session_config=GetSessionConfig(num_recent_events=50))`, `context_window_compression` | Bounded tokens per turn | Loses older context — pair with Memory Bank |
| Model routing / tiering | Flash/Flash-Lite for routing/classification sub-agents, Pro only where reasoning needed; Gemini `routing_config.auto_mode.model_routing_preference` = `PRIORITIZE_COST` / `BALANCED` / `PRIORITIZE_QUALITY` (API field; availability per model unverified); API Gateway model routing (Preview) for OpenAI-compatible multi-model routing | Cost/latency | Possible quality drop → re-run evalset per route |
| Thinking budget | Lower reasoning tokens for simple steps | Latency/cost | Accuracy on hard tasks |
| Call caps | `max_llm_calls`, `max_iterations` | Bounded worst-case cost | Legit long tasks may truncate |
| Concurrency / min instances | Agent Runtime `container_concurrency`, `min_instances`; Cloud Run min instances | Latency under spikes | Idle cost (Cloud Run; Agent Runtime idle not billed in Preview) |
| Parallelism | `ParallelAgent`, parallel tool calls | Wall-clock latency | More concurrent tokens/quota |
| Retries | `ReflectAndRetryToolPlugin` | Reliability | Extra calls/latency; needs idempotent tools |
| Online eval sampling | Sampling % + max samples per run | Continuous quality signal | Judge tokens |

**Exam signals — 4.2**

| Keyword in stem | Likely answer |
|---|---|
| "minimal operational overhead", "Python ADK", "managed sessions and memory" | Agent Runtime |
| "any language / existing containers", "scale to zero", "spiky traffic", "host MCP server" | Cloud Run |
| "untrusted LLM-generated code", "gVisor", "GPU open model co-located", "fine-grained infra control", "steady high volume + CUDs" | GKE (Agent Sandbox) |
| "task runs > 60 minutes" | Not a single Cloud Run request → GKE, Cloud Run Jobs/worker pools, or async pattern |
| "Agent Runtime must reach on-prem / private IP" | PSC interface + network attachment + DNS peering |
| "Agent Runtime needs internet under VPC-SC" | Proxy VM in VPC + Cloud NAT via PSC-I |
| "latency spikes at burst with ADK agent on Agent Runtime" | Raise `container_concurrency` (multiple of 9) and `min_instances` |
| "sessions lost after Cloud Run scale-in" | `--session_service_uri` (agentengine:// or Cloud SQL) |
| "canary new agent version" | Agent Runtime revisions + `trafficSplitManual`; Cloud Run `update-traffic` |
| "agent stuck calling tool repeatedly" | `max_llm_calls` in `RunConfig`; `LoopAgent max_iterations` / escalate |
| "which tool is slow" | Cloud Trace `execute_tool` spans / `gen_ai.execute_tool.duration` / Tools dashboard |
| "SQL analysis of agent behavior, cost per user, tool error rates" | BigQuery Agent Analytics plugin |
| "reduce cost of large repeated system prompt" | Context caching |
| "transient tool errors" | `ReflectAndRetryToolPlugin` |

**Common distractors**: "Deploy to GKE for lowest ops"; "Cloud Run session affinity guarantees session persistence"; "set `max_llm_calls=0` to avoid truncation" (0 = unlimited); "increase `max_instances`" when the real issue is per-instance concurrency or cold starts; Cloud Logging text search instead of traces for latency attribution; "enable DEBUG logging in production"; VPC connector for Agent Runtime (it uses PSC-I, not Serverless VPC Access); building a custom proxy for model routing when a managed option exists.

---

### Limits & gotchas (quick list)

- ADK defaults: `tool_trajectory_avg_score` 1.0 (EXACT), `response_match_score` 0.8; `max_llm_calls` 500.
- `safety_v1` and `multi_turn_*` criteria require GCP project/location (they call the Agent Platform Eval SDK) — they fail in an offline CI runner without credentials.
- `final_response_match_v2`, rubric and hallucination criteria are non-deterministic; use `num_samples` majority voting and don't gate every commit on them.
- Gen AI eval: pairwise only with Gemini judge; `sampling_count` 1–32 (default 4); `EvalTask` has no adaptive rubrics.
- Agent Runtime: `min_instances` 0–10, `max_instances` 1–1000 (1–100 with VPC-SC/PSC-I), CPU {1,2,4,6,8}, memory 1–32 Gi, default concurrency 9; 90 queries/min/region default quota; source deploy ≤ 8 MB.
- Agent Runtime revisions: 950/agent, 6,000/project/region, not adjustable.
- Cloud Run: 60-min max request timeout, 1,000 max concurrency/instance, default 100 max instances/revision.
- PSC-I: min /28 subnet; can't modify in place.

---

### Practice questions

**Q1.** Your team adds a new tool to an ADK customer-service agent. After the change, answers are still correct, but in 20% of cases the agent calls `lookup_order` twice before responding. You want CI to catch this class of regression cheaply on every pull request. What should you do?

- **A.** Add `final_response_match_v2` with threshold 0.9 to `test_config.json`.
- **B.** Add expected `tool_uses` to the `.test.json` cases and gate on `tool_trajectory_avg_score` with `match_type: EXACT` at 1.0.
- **C.** Enable an online monitor with the Tool Use Quality metric.
- **D.** Add `response_match_score` at 0.95.

**Answer: B.** Duplicate calls are a trajectory defect; `EXACT` fails on extra calls and is deterministic and cheap for CI. Response metrics (A, D) pass because answers are correct; online monitors (C) find it only after production.

**Q2.** A research agent issues between three and seven web searches in varying order before calling `write_report`. You need a trajectory metric that verifies `search` and `write_report` were both called, without failing on extra searches or order. Which ADK configuration fits?

- **A.** `tool_trajectory_avg_score` with `EXACT`.
- **B.** `tool_trajectory_avg_score` with `IN_ORDER`.
- **C.** `tool_trajectory_avg_score` with `ANY_ORDER`.
- **D.** `response_match_score` at 0.8.

**Answer: C** (B is acceptable only if order `search`→`write_report` must be enforced; the stem says order doesn't matter). `ANY_ORDER` requires all expected calls present, tolerates extras and ordering. `EXACT` fails on variable search counts.

**Q3.** A multi-turn travel-booking agent asks for missing details in different orders depending on how the user phrases requests, so fixed scripted turns keep failing even when the booking succeeds. You want automated evaluation of goal completion. What should you do?

- **A.** Record more golden conversations and use `tool_trajectory_avg_score`.
- **B.** Use ADK user simulation with a `ConversationScenario` (starting prompt, conversation plan, persona) and evaluate with `multi_turn_task_success_v1` and `hallucinations_v1`.
- **C.** Use `final_response_match_v2` on the last turn only.
- **D.** Use `adk conformance test` in replay mode.

**Answer: B.** User simulation generates dynamic user turns; reference-free multi-turn criteria are supported with it. Reference-based criteria (A, C) aren't supported with user simulation, and replay (D) enforces the brittle fixed path.

**Q4.** Your LLM judge metric for "regulatory compliance of responses" frequently disagrees with your compliance SMEs. You must prove the judge is trustworthy before using it as a release gate. What should you do first?

- **A.** Raise `sampling_count` to 32.
- **B.** Switch to BLEU against SME-written references.
- **C.** Build a human-rated dataset (`compliance/human_rating`), run the custom pointwise metric, and compute agreement with `evaluate_autorater`; iterate the rubric or tune a judge model until agreement is acceptable.
- **D.** Replace the judge with `safety_v1`.

**Answer: C.** Autorater calibration against human ratings is the documented way to validate a judge; tuning the judge is the next step. More samples (A) reduce variance but not bias; BLEU (B) and safety (D) measure the wrong thing.

**Q5.** You're comparing a new system prompt against the production prompt on 2,000 historical queries stored in BigQuery, and you want the judge to pick the better response per query while controlling for position bias. Which approach is best?

- **A.** Pointwise `GENERAL_QUALITY` on each variant, compare averages.
- **B.** Pairwise model-based metric in the Gen AI evaluation service with `flip_enabled=True`, dataset loaded from BigQuery.
- **C.** ADK `response_match_score` between the two variants.
- **D.** Online monitors on both revisions.

**Answer: B.** Pairwise metrics are designed for candidate-vs-baseline comparison, and response flipping mitigates position bias; BigQuery is a supported dataset source. A is possible but less sensitive; C isn't a quality judgment; D requires production exposure.

**Q6.** An ADK agent deployed on Agent Runtime with default settings shows median latency of 4 s but max latency of 60 s during traffic bursts of ~300 concurrent requests. CPU usage per container is low. What should you change first?

- **A.** Increase `resource_limits` to 8 CPU / 32 Gi.
- **B.** Increase `container_concurrency` to a multiple of 9 (e.g., 36) and raise `min_instances`.
- **C.** Switch the model to Flash-Lite.
- **D.** Increase `max_instances` to 1000.

**Answer: B.** Default concurrency (9) assumes sync code; async ADK agents are underused, so requests queue while it scales out. Google's guidance is multiples of 9 plus enough `min_instances` for baseline load. Low CPU rules out A; D doesn't fix scale-out lag.

**Q7.** A regulated bank needs an ADK agent that calls an internal pricing API reachable only on a private RFC 1918 address in a Shared VPC. The agent must run with minimal ops overhead inside a VPC Service Controls perimeter. Which design fits?

- **A.** Agent Runtime with a Serverless VPC Access connector.
- **B.** Agent Runtime with a Private Service Connect interface (network attachment in the service project, ≥ /28 subnet), DNS peering to the private zone, and the Agent Platform Service Agent granted network permissions on the host project.
- **C.** Cloud Run with public ingress and an API key.
- **D.** Agent Runtime with `max_instances=1000`.

**Answer: B.** Agent Runtime reaches private networks through PSC-I with DNS peering; Shared VPC needs the service agent's network roles. Connectors (A) are for Cloud Run; C violates the requirement; with VPC-SC/PSC-I `max_instances` is capped at 100 (D is also invalid).

**Q8.** After moving an ADK agent from local testing to Cloud Run with `adk deploy cloud_run --project P --region R ./agent`, users report the agent forgets earlier turns intermittently, especially after quiet periods. What's the most likely cause and fix?

- **A.** Context window overflow; enable context caching.
- **B.** Sessions are held in the in-memory session service; redeploy with `--session_service_uri` pointing at Agent Platform Sessions (`agentengine://...`) or a Cloud SQL database URL.
- **C.** Session affinity is disabled; enable it.
- **D.** `max_llm_calls` is too low.

**Answer: B.** Without `--session_service_uri`, the container uses in-memory sessions that are lost on scale-in and not shared across instances. Session affinity (C) is best-effort and doesn't survive instance termination.

**Q9.** A LoopAgent with a critic and a reviser runs until `max_iterations=10` every time, doubling cost, even for documents the critic considers finished. What's the correct fix?

- **A.** Lower `max_llm_calls` to 50.
- **B.** Give the critic (or reviser) an `exit_loop` tool that sets `tool_context.actions.escalate = True`, and instruct it to call the tool when no further changes are needed.
- **C.** Convert the LoopAgent to a ParallelAgent.
- **D.** Increase the temperature of the critic.

**Answer: B.** LoopAgent stops at `max_iterations` or when a sub-agent emits an event with `escalate=True`; the loop has no exit signal. A only caps the damage; C changes semantics.

**Q10.** Your SRE team needs to find which of 12 tools causes a p95 latency regression in a production agent on Agent Runtime and wants per-tool trends alertable in Cloud Monitoring. What should you use?

- **A.** Search Cloud Logging for "tool" and eyeball timestamps.
- **B.** Enable `GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true`, use the Observability Tools view / `gen_ai.execute_tool.duration` by `gen_ai.tool.name`, and inspect `execute_tool` spans in Cloud Trace for slow traces.
- **C.** Run `adk eval` with `tool_trajectory_avg_score`.
- **D.** Enable DEBUG logging in production.

**Answer: B.** The ADK/OTel tool-duration metric and trace spans attribute latency per tool and can be alerted on. Eval (C) checks correctness, not latency; DEBUG (D) is noisy and risks PII.

**Q11.** Product leadership wants weekly SQL reports on cost per user, tool error rates by MCP server, and human-approval (HITL) turnaround time across all agents, joinable with Cloud Trace. What is the most direct solution?

- **A.** Export Cloud Monitoring metrics to CSV.
- **B.** Add `BigQueryAgentAnalyticsPlugin` to the ADK `App`, writing to an `agent_events` table, and query its event views (`v_llm_response`, `v_tool_error`, HITL events) joined on `trace_id`.
- **C.** Build a custom Pub/Sub logger in each tool.
- **D.** Use online monitors.

**Answer: B.** The plugin captures token usage, tool provenance (including MCP), HITL events, and trace IDs into partitioned BigQuery tables with ready views. C reinvents it; D measures quality, not usage analytics.

**Q12.** Four weeks after launch, a RAG support agent's answers are rated worse by customers, but error rates and latency are flat. Offline evalsets still pass. What's the best next step?

- **A.** Roll back to the previous revision.
- **B.** Configure online monitors on production traces with hallucination and response-quality metrics, alert on the resulting Cloud Monitoring metrics, and add low-scoring production conversations to the golden evalset for regression.
- **C.** Raise `min_instances`.
- **D.** Switch to `trajectory_exact_match` in CI.

**Answer: B.** This is quality drift from changing real-world inputs/data; static evalsets don't reflect it. Online monitoring detects it continuously, and feeding failures back into the golden set closes the loop. Rollback (A) assumes a code regression that isn't evidenced.

---

### Key doc links

- ADK — Why evaluate agents (evalset, test files, `adk eval`, `AgentEvaluator`, conformance): https://adk.dev/evaluate/index.md
- ADK — Evaluation criteria: https://adk.dev/evaluate/criteria/index.md
- ADK — User simulation: https://adk.dev/evaluate/user-sim/index.md
- ADK — Environment simulation: https://adk.dev/evaluate/environment_simulation/index.md
- ADK — Custom metrics: https://adk.dev/evaluate/custom_metrics/index.md
- ADK — Deploy overview / Cloud Run / GKE / Agent Runtime / Agents CLI: https://adk.dev/deploy/index.md, https://adk.dev/deploy/cloud-run/index.md, https://adk.dev/deploy/gke/index.md, https://adk.dev/deploy/agent-runtime/deploy/index.md, https://adk.dev/deploy/agent-runtime/agents-cli/index.md
- ADK — Observability (logging, metrics, traces): https://adk.dev/observability/index.md, https://adk.dev/observability/metrics/index.md
- ADK — Cloud Trace integration: https://adk.dev/integrations/cloud-trace/index.md
- ADK — BigQuery Agent Analytics plugin: https://adk.dev/integrations/bigquery-agent-analytics/index.md
- ADK — Reflect and Retry plugin: https://adk.dev/integrations/reflect-and-retry/index.md
- ADK — RunConfig: https://adk.dev/runtime/runconfig/index.md
- ADK — Context caching: https://adk.dev/context/caching/index.md
- Agent Platform — Evaluate a generative AI agent (trajectory metrics, EvalTask): https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/evaluation-agents
- Agent Platform — Agent evaluation with GenAI client: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/evaluation-agents-client
- Agent Platform — Evaluate your agents (evaluation types, quality flywheel): https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/evaluation/evaluate-agents
- Agent Platform — Agent evaluation overview (skills, simulation): https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/evaluation/agent-evaluation
- Agent Platform — Online monitors: https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/evaluation/evaluate-online
- Gen AI evaluation service overview / define metrics / run evaluation: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/evaluation-overview, https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/determine-eval, https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/run-evaluation
- Judge model: evaluate / configure; AutoraterConfig: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/evaluate-judge-model, https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/configure-judge-model, https://docs.cloud.google.com/gemini-enterprise-agent-platform/reference/rest/v1beta1/AutoraterConfig
- Agent Runtime — Deploy an agent (resource controls): https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/deploy-an-agent
- Agent Runtime — Optimize and scale: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/optimize-and-scale
- Agent Runtime — Revisions and traffic: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/manage-revisions-and-traffic
- Agent Runtime — PSC interface: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/private-service-connect-interface
- Agent Runtime — Tracing / Monitoring: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/tracing, https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/monitoring
- Agent Platform — Observability overview / Traces: https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/observability/overview, https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/observability/traces
- Agent Platform — Quotas: https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/agent-quotas
- Agent Platform — Scale (enterprise security matrix): https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale
- Agent Platform pricing: https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing
- Architecture Center — Choose agentic AI architecture components: https://docs.cloud.google.com/architecture/choose-agentic-ai-architecture-components
- Architecture Center — Multi-agent private networking patterns: https://docs.cloud.google.com/architecture/multi-agent-private-networking-patterns
- Cloud Run — Host AI agents: https://docs.cloud.google.com/run/docs/ai-agents ; session affinity: https://docs.cloud.google.com/run/docs/configuring/session-affinity ; traffic: https://docs.cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration
- GKE Agent Sandbox: https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox ; Agent Substrate: https://docs.cloud.google.com/kubernetes-engine/ai-ml/about-agent-substrate
- Cloud Observability — Agent observability: https://docs.cloud.google.com/stackdriver/docs/observability/agent-observability
- Agents CLI: https://google.github.io/agents-cli/


---

## Section 5 — Securing and governing agentic workflows (~15%)

**What the exam is really testing**
1. Can you pick the *right control at the right layer*: identity (Agent Identity, IAM allow/deny, PAB), network/egress (Agent Gateway + IAP Access policies, VPC-SC), content (Model Armor, Semantic Governance, Gemini safety settings), and behavior (ADK callbacks, plugins, HITL)?
2. Do you know how agent authority works, meaning **acting as the agent** (SPIFFE agent identity, 2LO, API key) versus **acting on behalf of the user** (3LO through auth manager or a Gemini Enterprise authorization resource), and where that shows up in audit logs?
3. Can you roll out governance safely: `DRY_RUN`/`INSPECT_ONLY` first, default-deny egress, allowlisting essential endpoints, and least privilege that survives agent re-creation?

> Naming (2026): Vertex AI Agent Engine is now **Agent Runtime** (the resource is still `reasoningEngines`). Vertex AI is now **Gemini Enterprise Agent Platform**. Docs live under `docs.cloud.google.com/gemini-enterprise-agent-platform/govern/...`. The console calls the governance pillar "Govern".

---

### 5.0 The governance model in one table

The "Govern" pillar answers four questions. Learn this table first, because most scenario questions map onto one row.

| Question | Component | Enforced where |
|---|---|---|
| *Who made the request?* | **Agent Identity**: a per-agent SPIFFE ID with an X.509 cert and cert-bound tokens | IAM, CAA (mTLS/DPoP), audit logs |
| *Which destinations are approved?* | **Agent Registry**: catalog of agents, MCP servers, endpoints, and skills (Preview) | Agent Gateway checks it before connecting |
| *Which actions are permitted?* | **Policies**: IAM **Unified Access Policies** (allow + deny rules with CEL), **Model Armor**, **Semantic Governance Policies**, custom authz extensions | Agent Gateway, through Service Extensions authz policies |
| *Where is it enforced?* | **Agent Gateway**: ingress (Client-to-Agent) and egress (Agent-to-Anywhere) | IAP evaluates Access policies; Model Armor/SGP are content callouts |
| *What happened?* | Agent Observability, Cloud Logging, Cloud Audit Logs, Cloud Trace | `networkservices.googleapis.com/Gateway` logs, IAP `data_access` audit logs |

Internally, every gateway policy type is an **authorization policy managed through Service Extensions**. IAP is a `REQUEST_AUTHZ` extension. Model Armor, Semantic Governance, and custom ext_proc engines are `CONTENT_AUTHZ` extensions.

Recommended configuration order (from the docs): **provision Agent Identities → register destinations in Agent Registry → IAM Access policies (DRY_RUN) → Model Armor templates (INSPECT_ONLY) → Semantic Governance (DRY_RUN) → deploy Agent Gateway and watch logs → switch to enforce.**

---

## 5.1 Configuring agent security and governance

### 5.1.1 Authentication and secure tool execution (agent-to-tool OAuth 2.0)

#### Agent Identity (the principal)
- Each agent on **Agent Runtime** or **Gemini Enterprise** (and on Cloud Run services configured with agent identity) gets a **SPIFFE ID**. The ID is strongly attested, tied to the agent's lifecycle, and derived from the hosting resource path:
  - SPIFFE: `spiffe://agents.global.org-ORG_ID.system.id.goog/resources/aiplatform/projects/PROJ_NUM/locations/us-central1/reasoningEngines/ENGINE_ID`
  - IAM principal: `principal://agents.global.org-ORG_ID.system.id.goog/resources/aiplatform/projects/PROJ_NUM/locations/REGION/reasoningEngines/ENGINE_ID`
  - Trust domain: `agents.global.org-ORG_ID.system.id.goog`, or `agents.global.proj-PROJ_NUM.system.id.goog` for projects without an org.
  - Gemini Enterprise agents use `.../resources/discoveryengine/projects/.../engines/...`.
  - `urn:agent:...` identifiers are for **inventory/catalog lookup only** and are **never** used in IAM bindings.
- **Credentials:** an auto-provisioned X.509 cert plus cert-bound access tokens. Calls straight to Google APIs use **mTLS**. When traffic goes through Agent Gateway, the gateway terminates mTLS and **DPoP** (RFC 9449) re-binds the token beyond the gateway. The docs call this "double-bound credentials".
- **Opting in on Agent Runtime.** Without the flag, the agent keeps using a service account for backward compatibility.
  ```python
  remote_app = client.runtimes.create(agent=AdkApp(agent=agent), config={
      "display_name": "my-agent",
      "identity_type": types.IdentityType.AGENT_IDENTITY,   # v1beta1
      "staging_bucket": "gs://BUCKET"})
  print(remote_app.api_resource.spec.effective_identity)
  ```
  Other ways to opt in: `agents-cli deploy --agent-identity`; for `adk deploy`, put `{"identity_type": "AGENT_IDENTITY"}` in `.agent_engine_config.json`; in Terraform, set `identity_type = "AGENT_IDENTITY"` on `google_vertex_ai_reasoning_engine`. You can also create the runtime instance with **only** `identity_type` and no code. That lets you set up IAM before the code ships.
- **Default roles:** `roles/aiplatform.agentContextEditor` and `roles/aiplatform.agentDefaultAccess`, which cover the agent's own logging, sessions, memories, and sandboxes. The docs also recommend `roles/aiplatform.expressUser`, `roles/serviceusage.serviceUsageConsumer`, and `roles/browser`. `roles/browser` is needed for `resourcemanager.projects.get` during SDK init. Without it you see the startup error "Failed to convert project number to project ID".
- **Grant to many agents** with a principal set: `principalSet://agents.global.org-ORG_ID.system.id.goog/attribute.platformContainer/aiplatform/projects/PROJ_NUM` covers all Agent Runtime agents in a project. `.../attribute.platform/aiplatform` covers the whole org.
- **Limitation:** agent identities can't get Legacy Bucket roles (`storage.legacyBucket*`).

**Service account vs Agent Identity**

| | Service account | Agent Identity |
|---|---|---|
| Granularity | Often shared by many agents or workloads | One per agent (per `reasoningEngines` resource) |
| Lifecycle | Independent. Keys can leak, and the account outlives the agent | Tied to the agent. Read-only and system-attested |
| Credential theft | Bearer tokens can be replayed anywhere | CAA forces mTLS/DPoP, so a stolen token fails outside the runtime (`401 "Context-Aware Access requirements are not met"`) |
| Audit | Logs show the SA. User attribution must be built by hand | Logs show the agent. For delegated flows they show **both user and agent** |
| Gateway, PAB, Access policies | Only partially usable as an agent principal | First-class principal in Access policies, PAB principal sets, and VPC-SC ingress/egress rules |
| Gotcha | — | **Delete and redeploy = new principal.** Old bindings become inactive grants and are not inherited. Blue-green, ephemeral, and Terraform re-create pipelines lose access silently |

Mitigate the redeploy gotcha in one of two ways. (a) Read `spec.effectiveIdentity` after deploy (`gcloud ai reasoning-engines describe ... --format="value(spec.effectiveIdentity)"`) and re-bind. (b) Put baseline, non-sensitive roles on the **project `principalSet`**, which survives re-creation, and keep individual `principal://` bindings for sensitive data only. Clean up stale bindings when you decommission an agent.

Opting out of CAA is possible but **strongly discouraged**. You would set `GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES=False`, and the only intended use is corner cases where agents share tokens.

#### Authentication models (which auth for which target)

| Authority | Method | Target | Solution |
|---|---|---|---|
| Agent's own | Agent Identity (IAM) | Google Cloud APIs | Grant roles to the `principal://` agent ID. ADC picks it up automatically |
| Agent's own | OAuth 2.0 **2-legged** (client credentials) | External SaaS (M2M) | Auth manager 2LO auth provider. **Recommended** for M2M with services that support OAuth |
| Agent's own | API key | External services | Auth manager API-key auth provider |
| Agent's own | HTTP basic | External | Stored like an API key. **Not recommended** |
| **User-delegated** | OAuth 2.0 **3-legged** | External tools (Jira, GitHub, Salesforce...) or Google APIs as the user | Auth manager 3LO auth provider: consent, token vault, refresh |

#### Agent Identity auth manager (the "Auth Manager (OAuth 2.0)" in-scope tool)
- A **centralized credentials vault and authentication broker** for *outbound* tool auth. It stores API keys, OAuth client secrets, and user tokens, and it runs the consent, code exchange, and refresh flows. Access to it is governed by IAM, with the agent authenticating using its **own SPIFFE ID**. All end-user access events are attributable to that SPIFFE ID, and admins get visibility into user delegations and can **revoke** them.
- Resources: **auth providers** (API key / 2LO / 3LO). The current v2 CLI is `gcloud agent-identity auth-providers ...`; the legacy v1 command is `gcloud alpha agent-identity connectors ...`, and its resource path uses `connectors/` instead of `authProviders/`.
  ```bash
  gcloud agent-identity auth-providers create jira-3lo --location=us-central1 \
    --three-legged-oauth-client-id=ID --three-legged-oauth-client-secret=SECRET \
    --three-legged-oauth-authorization-url=https://auth.atlassian.com/authorize \
    --three-legged-oauth-token-url=https://auth.atlassian.com/oauth/token
  gcloud agent-identity auth-providers add-iam-policy-binding jira-3lo --location=us-central1 \
    --role=roles/agentidentity.user --member="principal://agents.global.org-.../reasoningEngines/ENGINE_ID"
  ```
  The role is `roles/agentidentity.user`. It was `roles/iamconnectors.user` on the v1 connectors API. Also grant it to `user:you@` for local `adk web` testing.
- **Service caveats:** GitHub and Microsoft are **single-scope only**. ServiceNow grants only the scopes configured on the app side, so a missing scope causes a request loop.
- **Agent Registry bindings** connect an agent to an auth provider, so ADK resolves the credential without code config: `gcloud agent-registry bindings create ... --source-identifier=... --auth-provider=projects/.../connectors/ID`. Bindings aren't supported in the `us`/`eu` multi-regions.

#### ADK side
```python
# Agent Identity auth manager (Preview), pip install "google-adk[agent-identity]"
from google.adk.auth.credential_manager import CredentialManager
from google.adk.integrations.agent_identity import GcpAuthProvider, GcpAuthProviderScheme
CredentialManager.register_auth_provider(GcpAuthProvider())          # once
toolset = McpToolset(
    connection_params=StreamableHTTPConnectionParams(url="https://mcp.example.com"),
    auth_scheme=GcpAuthProviderScheme(
        name="projects/P/locations/L/authProviders/jira-3lo",
        continue_uri="https://myapp.example.com/oauth/continue"))    # 3LO only
# Or resolve from Agent Registry bindings:
# AgentRegistry(project_id, location).get_mcp_toolset(mcp_server_name="mcpServers/ID", continue_uri=...)
```
- **3LO consent loop.** ADK emits a `FunctionCall` named **`adk_request_credential`** carrying `auth_uri`. The client opens it in a popup. After consent, the provider redirects to `continue_uri`, and your handler POSTs to `https://agentidentitycredentials.googleapis.com/v1/{auth_provider}/credentials:finalize`. The client then resumes the agent by sending a `FunctionResponse`. With auth manager, **no auth code is needed** on resume, and you resume whether consent succeeded or failed.
- **Native ADK auth (no auth manager).** A tool takes `auth_scheme` (OpenAPI-style, e.g. `OAuth2(flows=...)`, `OpenIdConnect`, API key) and `auth_credential` (`AuthCredential(auth_type=AuthCredentialTypes.OAUTH2, oauth2=OAuth2Auth(client_id, client_secret))`). This works on `OpenAPIToolset`, `McpToolset`, and `AuthenticatedFunctionTool`. Inside a custom `FunctionTool`, you:
  1. check cached tokens in state,
  2. call `tool_context.get_auth_response(auth_config)`,
  3. if still missing, call `tool_context.request_credential(AuthConfig(auth_scheme=..., raw_auth_credential=...))` and return a "pending auth" result.

  **Trade-off:** you own token storage and refresh. Keep tokens in session state keyed per user, never in `app:` state.
- **Per-user headers for MCP:** `McpToolset(header_provider=callable(ReadonlyContext))` builds headers per invocation. Use it in multi-tenant apps that forward user JWTs.
- **Gemini Enterprise-hosted ADK/A2A agents acting as the user.** Create an OAuth web client with redirect URIs `https://vertexaisearch.cloud.google.com/oauth-redirect` and `.../static/oauth/oauth.html`. Then create a Discovery Engine **authorization resource** (`serverSideOauth2`) and reference it in `authorizationConfig.toolAuthorizations` when you register the agent. For A2A agents on Cloud Run, Gemini Enterprise sends a service-agent OIDC token in **`X-Serverless-Authorization`** (Cloud Run checks `roles/run.invoker`) and the **user's** OAuth token in `Authorization`, so the two never collide.
- **Leakage gotcha:** the BigQuery Agent Analytics plugin can write `client_secret`/`access_token` values into BigQuery, because `adk_request_credential` arguments are serialized in camelCase and the plugin's redaction misses that (issue #3845, open). Redaction is not DLP.

**Acting as the agent vs on behalf of the user**

| | Agent-auth (own identity: agent ID, 2LO, API key) | User-auth (3LO delegation) |
|---|---|---|
| Use when | Every user has the same access level, or the job is back-office or batch | Per-user data such as mailbox, repos, tickets, or row-level entitlements |
| Security property | IAM caps the blast radius (read-only role means no writes, whatever the LLM decides) | The agent can only do what the user could already do, which blocks confused-deputy escalation |
| Risk | Confused deputy: a low-privilege user steers a high-privilege agent. You must log user attribution yourself | OAuth scopes are often broader than needed, so still constrain with in-tool guardrails, Access policies, and SGP |
| Audit | Logs show the agent only | Logs show **agent + user** |
| Ops | No consent UX | Consent popup, `continue_uri` handler, token revocation |

### 5.1.1b Deep dive — Agent-to-tool authentication patterns

> 🧠 **Visual memory map:** Agent-to-tool authentication
>
> [![Agent-to-tool authentication](visual-memory/18-agent-to-tool-auth.png)](visual-memory/18-agent-to-tool-auth.png)

The exam tests this area heavily. Almost every agent-to-tool auth question comes down to three decisions:

1. **Whose authority?** The agent's own (Agent Identity, a service account, 2LO, or an API key), or the end user's (3LO or a platform-passed user token).
2. **What does the destination accept?** Google APIs accept a Google **access token**. Your own IAM-protected services (Cloud Run, IAP) accept a Google-signed **ID token** whose `aud` claim matches them. SaaS accepts **its own** OAuth token or API key. Maps-style Google services that aren't IAM-based accept a **Google API key**.
3. **Where does the secret live?** In the auth manager vault, in Secret Manager, or nowhere, because the metadata server mints short-lived tokens. It never belongs in code, prompts, or model-visible state.

**Token types in one table**

| Token | Proves | Audience / scope | Accepted by | Typical lifetime |
|---|---|---|---|---|
| Google **access token** (OAuth 2.0) | Permissions of a principal (agent, SA, WIF, user) | OAuth scopes (`cloud-platform`, `bigquery`…) + IAM roles | Google APIs and **Google remote MCP servers** (`bigquery.googleapis.com/mcp`, `run.googleapis.com/mcp`…) | 1 h by default (`gcloud auth print-access-token --lifetime` up to 12 h) |
| Google **ID token** (OIDC JWT) | Identity of the caller | `aud` = receiving service URL, or a configured custom audience | Cloud Run / Cloud Run functions (`run.invoker`), IAP (`iap.httpsResourceAccessor`), your own JWT validators | ~1 h. Fetched per request, no background refresh in ADK |
| **Third-party OAuth** token | Agent (2LO) or user (3LO) authority at the SaaS | SaaS scopes | Jira, GitHub, Salesforce, ServiceNow… | Set by the SaaS. Refresh token needed for 3LO |
| **API key** | "Caller has the key". No principal | API restrictions on the key | Services that don't need a principal (Maps Grounding Lite MCP, Translation v2), SaaS keys | Until rotated |
| **Agent X.509 cert** | The agent's SPIFFE identity (mTLS) | — | Google APIs over mTLS, Agent Gateway | 24 h, auto-rotated |

> Rule of thumb from the ADK docs: an **access token is your keycard** (it calls Google APIs), and an **ID token is your passport** (it calls your own IAM-secured services). Sending an access token to a private Cloud Run service gets a 401. So does sending an ID token to BigQuery.

#### Master matrix — destination → auth

| Destination | Recommended auth | Identity used | Where creds live | IAM / config needed | Exam keyword |
|---|---|---|---|---|---|
| **Google Cloud APIs** (BigQuery, GCS, Vertex/Gemini) | Agent Identity through ADC. Client libraries fetch a cert-bound access token | **Agent** (or user with 3LO: auth-manager 3LO provider with a Google authorization URL and scopes) | Nowhere. Minted by the metadata server | Resource-level role to `principal://agents.global.org-ORG.system.id.goog/resources/aiplatform/projects/NUM/locations/L/reasoningEngines/ID` (Cloud Run agents: `.../resources/run/projects/NUM/locations/L/services/NAME`) | "per-agent least privilege", "no keys", "ADC" |
| **Google remote MCP servers** (BigQuery, Cloud Run, Monitoring, IAM… MCP) | ADC/OAuth access token in `Authorization: Bearer`, plus `x-goog-user-project` | Agent/workload (production) or user | Nowhere (ADC), or auth-manager 3LO for user delegation | **`roles/mcp.toolUser`** (`mcp.tools.call`) **plus** the product role (e.g. `bigquery.dataViewer`). The MCP endpoint must be enabled. IAM **deny** on `mcp.googleapis.com/tools.call` with `tool.isReadOnly` blocks write tools | "MCP Tool User", "read-only MCP tools org-wide", "no DCR" |
| **Custom MCP server on Cloud Run** | Google-signed **ID token**, `aud` = service URL. Deploy `--no-allow-unauthenticated --functional-type=mcp-server` | **Agent** (Cloud Run IAM sees the caller only) | Nowhere. The metadata server mints the ID token | **`roles/run.invoker`** on the service for the calling agent's principal. For developer/CLI access: `--iap` + `roles/iap.httpsResourceAccessor` + a custom OAuth client allowlisted for programmatic access | "run.invoker", "audience", "X-Serverless-Authorization" |
| **MCP server on GKE** | Server validates a Google ID token (`aud` check), or put it behind IAP, a service mesh (mTLS), or Agent Gateway as a registered endpoint (pattern, unverified as a single documented recipe) | Agent (caller). The server's own calls use **Workload Identity Federation for GKE** | Nowhere (WIF), or Secret Manager for SaaS keys the server uses | Server → Google APIs: role to `principal://iam.googleapis.com/projects/NUM/locations/global/workloadIdentityPools/PROJECT.svc.id.goog/subject/ns/NS/sa/KSA` | "GKE", "no service account keys", "Workload Identity" |
| **SaaS with API key** | **Auth manager API-key provider**. ADK injects the header | Agent | Auth manager vault (Google-managed) | `roles/agentidentity.user` **on the auth provider** for the agent principal | "no hardcoded keys", "centralized vault" |
| **SaaS with OAuth client credentials (2LO)** | **Auth manager 2LO provider** (client ID/secret + token URL). No user, no consent | Agent | Vault holds the client secret. Access tokens minted and injected | `roles/agentidentity.user` on the provider | "machine-to-machine", "batch", "no user present" |
| **SaaS acting as the user (3LO)** | **Auth manager 3LO provider** + `continue_uri` handler in your frontend. In Gemini Enterprise: a Discovery Engine **authorization resource** | **User** (delegated), attributed to agent + user | Vault stores the user's tokens and refreshes them | `roles/agentidentity.user` on the provider. Redirect URI registered at the SaaS = the auth-manager `.../oauthcallback` URL | "on behalf of", "consent", "user's own permissions" |
| **Another agent via A2A** | Agent Runtime target: Google **access token** (ADC). Cloud Run target: **ID token** (`aud` = URL). Public target: whatever its **agent card `securitySchemes`** declare | Calling agent (or user token forwarded in `Authorization` when GE calls with `toolAuthorizations`) | Nowhere (ADC/metadata), or auth-manager binding | Agent Runtime: `roles/aiplatform.user` on the target reasoning engine + `roles/agentregistry.viewer` for discovery, **granted to the parent agent's identity**. Cloud Run: `roles/run.invoker`. Via Agent Gateway: an Access policy allowing `destination.agent_registry.agent.name` | "A2A", "agent card", "orchestrator identity" |
| **OpenAPI / REST tools** | `OpenAPIToolset`/`RestApiTool` with `auth_scheme` + `auth_credential` (API key, OAuth2, OIDC, service account, or `ServiceAccount(use_id_token=True, audience=...)` for IAM-protected APIs). Or `GcpAuthProviderScheme` via `AuthenticatedFunctionTool` | Depends on the scheme | Auth manager (preferred), or Secret Manager + session state (self-managed) | Depends on the target | "OpenAPI spec", "securitySchemes", "AuthCredential" |
| **Databases via MCP Toolbox** | Agent → Toolbox (on Cloud Run): ID token via `CredentialStrategy.workload_identity(target_audience=TOOLBOX_URL)`. Toolbox → DB: the server's own identity | Agent to Toolbox. **User** scoping through Toolbox **authenticated parameters** (bind `user_id` etc. from the user's OIDC token) and **authorized invocations** | Toolbox server config. DB passwords in Secret Manager, or IAM DB auth (unverified per engine) | `run.invoker` on Toolbox for the agent. DB-level grants for Toolbox's SA | "row-level per user", "SQL defined server-side, not by the LLM" |

> **Correction/nuance to "IAM can't see MCP tool names":** for **Google Cloud remote MCP servers**, IAM **deny** policies can condition on `mcp.googleapis.com/tool.isReadOnly`, `tool.name`, `resource.service`, and `request.auth.oauth.client_id` (the last one is deny-only). These attributes apply **only** to the `mcp.tools.call` permission. For **your own** MCP servers, IAM sees only `run.invoker`. Tool-level control there needs Agent Gateway Access policies (or in-server authz).

#### Identity choices compared

| | **Agent Identity** | **Attached service account** (Agent Runtime default, Cloud Run `--service-account`) | **Workload Identity Federation** (GKE / external) | **End-user OAuth** (3LO / GE token) |
|---|---|---|---|---|
| Where available | Agent Runtime, Gemini Enterprise, **Cloud Run** (`--functional-type=agent --identity-type=agent-identity`, Preview) | Everywhere | GKE pods (KSA principal); on-prem or other clouds through a WIF pool | Any agent, through the auth manager, GE authorizations, or ADK native OAuth |
| Principal | `principal://agents.global.org-ORG.system.id.goog/resources/...` (SPIFFE) | `serviceAccount:name@proj.iam.gserviceaccount.com` | `principal://iam.googleapis.com/.../workloadIdentityPools/PROJECT.svc.id.goog/subject/ns/NS/sa/KSA` | The human's account at Google or the SaaS |
| Keys / impersonation | **No SA keys. Can't be impersonated.** Not shared | Keys possible (a risk). Impersonation possible. Often shared | No keys. Short-lived federated tokens | Refresh tokens must be vaulted |
| Token theft | Cert-bound (CAA: mTLS, DPoP beyond the gateway) | Bearer, replayable | Bearer, replayable | Bearer at the SaaS. Vaulted by auth manager; with Agent Gateway + GE, **decrypted only at the gateway, so the agent never sees the raw credential** |
| Lifecycle | Tied to the resource. **Re-create means a new principal** (Cloud Run too, when switching SA → agent-identity) | Independent of the agent | Tied to the KSA name | Per user. Consent can be revoked |
| Audit shows | Agent SPIFFE ID (`principalSubject`). With delegation: **agent + user** | SA only. User attribution must be built by hand | Federated principal | User (+ agent when brokered by the auth manager) |
| Pick when | Default for new agents on supported runtimes | Legacy, unsupported runtime, or a shared back-office job | Self-hosted agents or MCP servers on GKE | Per-user data or entitlements. Blocks confused-deputy attacks |

**Acting as the agent vs on behalf of the user: audit attribution.** When the agent uses its own authority, Cloud Audit Logs show only the agent principal. Any "which user asked?" answer has to come from your own logging (session → user), and that link is the confused-deputy risk. With 3LO through the auth manager, the docs state that logs show **both the agent's and the user's identities**, and that all end-user access events are attributable to the agent's SPIFFE ID. With service-account impersonation, `serviceAccountDelegationInfo` shows the chain, but no end user appears.

#### 3LO sequence (Agent Identity auth manager + ADK)

```
User        Frontend (your app)        Agent (ADK on Agent Runtime)        Auth manager (agentidentitycredentials)        SaaS (Jira/GitHub/Google)
 |  "create a Jira issue"  |                         |                                   |                                           |
 |------------------------>|---- stream_query ------>|                                   |                                           |
 |                         |                         |-- retrieveCredentials (SPIFFE ID, needs roles/agentidentity.user) -->       |
 |                         |                         |<-- uri_consent_required: authorization_uri + consent_nonce ---|              |
 |                         |<-- FunctionCall adk_request_credential (auth_uri, nonce) --|                           |              |
 |                         |  save fc.id + auth_config + nonce in session                 |                           |              |
 |<-- popup auth_uri ------|                         |                                   |                                           |
 |---------------------------------------- sign in + consent (scopes) ------------------------------------------------------------->|
 |                         |                         |                                   |<-- code -> .../authProviders/NAME/oauthcallback (redirect URI registered at SaaS)
 |                         |                         |                                   |-- code exchange, store access+refresh token in vault
 |<-- redirect to continue_uri?user_id_validation_state=..&auth_provider_name=..&uuid=.. |                                   |
 |------------------------>| POST {auth_provider}/credentials:finalize {userId, userIdValidationState, consentNonce}  --->|   |
 |                         |<------------------------------------ 200 ------------------------------------------------|        |
 |                         |-- FunctionResponse(name=adk_request_credential, id=fc.id, response=auth_config) -->|              |
 |                         |      (no auth code needed; resume even if consent failed, ADK raises if it did)                    |
 |                         |                         |-- retrieveCredentials -------------->| returns user token (refreshed if needed)  |
 |                         |                         |------------------------ tool call, Authorization: Bearer <user token> ------->|
 |                         |<------- answer ---------|<----------------------------------------------------- data as the user -------|
 ... later: token expiry -> auth manager refreshes silently. Refresh token revoked/expired -> consent_required again.
 ... admin: disable/delete the auth provider (all agents stop), or revoke a user's delegation (auth manager provides revocation; exact CLI unverified)
```

**`continue_uri` vs redirect URI (a frequent trap).** The **redirect URI** registered in the SaaS/Google OAuth client is the auth manager's callback: `https://agentidentitycredentials.googleapis.com/v1/projects/PROJECT_ID/locations/LOCATION/authProviders/NAME/oauthcallback`. The **`continue_uri`** is **your app's** endpoint that the user lands on afterwards, where you call `credentials:finalize`. It can be defaulted on the provider with `--three-legged-oauth-default-continue-uri`. Mixing the two up gives `redirect_uri_mismatch`.

**Native ADK 3LO (no auth manager) differs as follows:**
- Your client appends **its own** `redirect_uri` to `auth_uri`, captures the full callback URL, and sets `auth_config.exchanged_auth_credential.oauth2.auth_response_uri` (and `redirect_uri`).
- It then sends a `FunctionResponse` named `adk_request_credential`. ADK performs the code exchange and **retries the tool**.
- You own token storage and refresh. With the ADK Resume feature, include the original `invocation_id`.

**Gemini Enterprise variant:**
- **Setup.** Create an OAuth web client with redirect URIs `https://vertexaisearch.cloud.google.com/oauth-redirect` and `https://vertexaisearch.cloud.google.com/static/oauth/oauth.html`. Create an authorization with `serverSideOauth2`. Build the auth URI with `access_type=offline&prompt=consent&include_granted_scopes=true`, and reference it in the agent's `authorizationConfig.toolAuthorizations`.
- **Runtime.**
  - For an ADK agent, GE runs consent and places the user's token in session state under the authorization ID. Read it with `external_access_token_key="AUTH_ID"` (BigQuery toolset) or from `tool_context.state`.
  - For an A2A agent on Cloud Run, the user token arrives in `Authorization` and the service-agent ID token in `X-Serverless-Authorization`.

#### ADK snippets (current API, Python)

**1. OpenAPI tool with OAuth2 (native ADK 3LO)**

```python
from fastapi.openapi.models import OAuth2, OAuthFlows, OAuthFlowAuthorizationCode
from google.adk.auth import AuthCredential, AuthCredentialTypes, OAuth2Auth
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

scheme = OAuth2(flows=OAuthFlows(authorizationCode=OAuthFlowAuthorizationCode(
    authorizationUrl="https://accounts.google.com/o/oauth2/auth",
    tokenUrl="https://oauth2.googleapis.com/token",
    scopes={"https://www.googleapis.com/auth/calendar.readonly": "read calendar"})))
cred = AuthCredential(auth_type=AuthCredentialTypes.OAUTH2,
    oauth2=OAuth2Auth(client_id=CLIENT_ID, client_secret=CLIENT_SECRET))  # from Secret Manager, never literals
calendar = OpenAPIToolset(spec_str=spec, spec_str_type="yaml", auth_scheme=scheme, auth_credential=cred)
# API key: token_to_scheme_credential("apikey", "header", "X-API-Key", key)
# OIDC:    OpenIdConnectWithConfig(authorization_endpoint=..., token_endpoint=..., scopes=["openid"])
```

**2. IAM-protected service on Cloud Run: ID token (OpenAPI) and MCPToolset header**

```python
# OpenAPI toolset -> private Cloud Run API: ID token, audience = service URL
from google.adk.auth.auth_credential import ServiceAccount
from google.adk.tools.openapi_tool.auth.auth_helpers import service_account_scheme_credential
scheme, cred = service_account_scheme_credential(ServiceAccount(
    use_default_credential=True, use_id_token=True, audience="https://mcp-x-123.us-central1.run.app"))
# use_id_token and audience must be set together; ADK raises otherwise.

# McpToolset -> custom MCP server on Cloud Run: mint a fresh ID token per call
import google.auth.transport.requests, google.oauth2.id_token
from google.adk.tools.mcp_tool import McpToolset, StreamableHTTPConnectionParams
MCP_URL = "https://mcp-x-123.us-central1.run.app"
def run_invoker_headers(ctx):                       # ReadonlyContext -> headers
    tok = google.oauth2.id_token.fetch_id_token(google.auth.transport.requests.Request(), MCP_URL)
    return {"X-Serverless-Authorization": f"Bearer {tok}"}   # keeps Authorization free for a user token
tools = McpToolset(connection_params=StreamableHTTPConnectionParams(url=f"{MCP_URL}/mcp"),
                   header_provider=run_invoker_headers)
```

The caller's principal needs `roles/run.invoker`. The `aud` must be the `run.app` URL or a configured custom audience. A custom domain doesn't work as `aud`, and a traffic-tag URL still uses the base service URL. On Cloud Run with **agent identity**, identity certificates are on by default, and Python `google-auth` then requests a **bound** ID token. Call the target's `*.mtls.run.app` URL while presenting `/var/run/secrets/workload-spiffe-credentials/certificates.pem` + `private_key.pem`, or opt out with `--no-identity-certificate`. Whether Agent Runtime's agent identity can mint ID tokens for arbitrary audiences through `fetch_id_token` is unverified. If it can't, use the SA-based runtime or Cloud Run for that caller.

**3. Custom `FunctionTool` with `request_credential` (self-managed tokens)**

```python
from google.adk.auth import AuthConfig
from google.adk.tools import ToolContext
def list_events(day: str, tool_context: ToolContext) -> dict:
    cfg = AuthConfig(auth_scheme=scheme, raw_auth_credential=cred)
    tokens = tool_context.state.get("user:cal_tokens")            # 1. cached (per user)
    if not tokens:
        exchanged = tool_context.get_auth_response(cfg)           # 2. just came back from consent?
        if not exchanged:
            tool_context.request_credential(cfg)                  # 3. emits adk_request_credential
            return {"status": "pending", "message": "Awaiting user authorization."}
        tokens = {"access_token": exchanged.oauth2.access_token,
                  "refresh_token": exchanged.oauth2.refresh_token}
        tool_context.state["user:cal_tokens"] = tokens            # 4. cache (see secret-handling caveat)
    # 5. call API; on 401/invalid_grant -> pop cache and request_credential again
```

**4. Agent Identity auth manager (Preview, `pip install "google-adk[agent-identity]"`)**

```python
from google.adk.auth.credential_manager import CredentialManager
from google.adk.integrations.agent_identity import GcpAuthProvider, GcpAuthProviderScheme
from google.adk.tools.authenticated_function_tool import AuthenticatedFunctionTool
from google.adk.auth.auth_tool import AuthConfig
from google.adk.tools.mcp_tool import McpToolset, StreamableHTTPConnectionParams
from vertexai.agent_engines import AdkApp

jira = McpToolset(connection_params=StreamableHTTPConnectionParams(url="https://mcp.jira.example"),
    auth_scheme=GcpAuthProviderScheme(
        name="projects/P/locations/us-central1/authProviders/jira-3lo",   # v1 API: .../connectors/...
        continue_uri="https://app.example.com/validateUserId"))           # 3LO only; optional if default set

async def spotify_search(credential, query: str):   # AuthenticatedFunctionTool injects AuthCredential
    token = credential.http.credentials.token        # 2LO / API key / 3LO token from the vault
    ...
search = AuthenticatedFunctionTool(func=spotify_search, auth_config=AuthConfig(
    auth_scheme=GcpAuthProviderScheme(name="projects/P/locations/us-central1/authProviders/spotify")))

class AuthenticatedAdkApp(AdkApp):                    # Python SDK deploy: register in set_up(),
    def set_up(self):                                 # not at import time, or you get
        CredentialManager.register_auth_provider(GcpAuthProvider())   # "No auth provider registered
        super().set_up()                              #  for custom auth scheme 'gcpAuthProviderScheme'"
# Local adk web / agents-cli deploy: module-level CredentialManager.register_auth_provider(GcpAuthProvider()) works.
# No scheme in code at all: AgentRegistry(project_id, location).get_mcp_toolset(mcp_server_name=..., continue_uri=...)
# resolves the auth provider from an Agent Registry binding (same region; auth providers are NOT in "global").
```

Deploy requirements from the docs: `"google-adk[agent-identity,mcp]>=2.7.1"` and `identity_type=AGENT_IDENTITY`.

#### Token binding, secrets, least privilege, lifetime

- **Token binding.** CAA is on by default for agent identities:
  - Calls to Google APIs use mTLS with the auto-rotated 24 h X.509 cert.
  - Beyond Agent Gateway, **DPoP** binds the token.
  - A token copied into a raw header, or shared with another process, fails with **401 UNAUTHENTICATED**.
  - The only escape hatch is `GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES=False`, which is **not recommended**. The exam answer is almost never "disable CAA".
- **Where secrets live (best → worst):**
  1. **Auth manager.** Vault + broker + refresh + revocation + audit, accessed with the agent's SPIFFE ID.
  2. **Secret Manager.** On Agent Runtime, use `secret_env`/`env_vars={"X": {"secret": ID, "version": V}}`. The secret must be in the **same project**. With agent identity, grant `roles/secretmanager.secretAccessor` to the **Agent Platform Service Agent** `service-NUM@gcp-sa-aiplatform.iam.gserviceaccount.com`, because it fetches secrets at deploy time. On Cloud Run, use `--update-secrets`.
  3. **Local `.env`.** Dev only, git-ignored.
  4. **`InMemorySessionService` state.** Dev only.
- **Never put secrets in** prompts or system instructions, model-visible tool arguments, SGP natural-language constraints (they can be quoted back to users), or logs.
- **Session state is persisted** by `DatabaseSessionService`/`VertexAiSessionService`. The ADK docs warn that refresh tokens in session state are risky, so keep only short-lived tokens there.
- **BigQuery analytics plugin redaction.** It redacts only `temp:` keys and a fixed list of key names. There is **no** special `secret:` prefix, and camelCase `clientSecret`/`accessToken` in `adk_request_credential` args can leak.
- **Least privilege:**
  - Grant resource-level roles (dataset, bucket, secret), not project-wide `Editor`.
  - For Google MCP, grant `roles/mcp.toolUser` plus the narrow product role, and add a deny policy on read-write tools for production projects.
  - Put baselines on the `principalSet`, and sensitive grants on the single `principal://`.
  - Add PAB to cap eligibility.
- **Scopes:**
  - Request the narrowest OAuth scope (e.g. `run.readonly`, `drive.readonly`, `bigquery`), not `cloud-platform`, for user delegation.
  - In the auth manager, GitHub and Microsoft support a **single scope** only.
  - ServiceNow grants only the scopes configured on its app, and a mismatch causes a consent loop.
- **Lifetime and refresh:**
  - Google access and ID tokens last about 1 h. Client libraries and the metadata server refresh them, so don't cache ID tokens past `exp`.
  - Auth manager refreshes 3LO tokens silently.
  - Native ADK requires you to refresh (`creds.refresh(Request())`) and, on `invalid_grant`, clear the cache and call `request_credential` again.
  - Google refresh tokens require `access_type=offline` (plus `prompt=consent` to reliably re-issue one).
  - Google refresh tokens for apps in "Testing" publishing status expire after 7 days (unverified here; Google OAuth policy).

#### Common failures and fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| **401** from Google API, "Request had invalid authentication credentials" / CAA not met | Agent's cert-bound token replayed outside the runtime (manual header injection, token shared) | Let client libraries make the call (ADC). Opting out of CAA is a last resort |
| **403** `agentidentity.authProviders.retrieveCredentials` denied | Principal lacks `roles/agentidentity.user` **on the auth provider** | Grant it to the agent's `principal://` (deployed) or to `user:you@` (local `adk web`) |
| `No auth provider registered for custom auth scheme 'gcpAuthProviderScheme'` | Registered at import time, then deployed with the Python SDK (serialized app) | Register in `set_up()` of an `AdkApp` subclass |
| `Location of auth provider does not match location of binding` | Registry client/binding in `global` | Use one region for auth provider, MCP server, and registry client. Bindings aren't supported in `us`/`eu` multi-regions either |
| `redirect_uri_mismatch` at the SaaS | Registered your app URL (or a wrong one) instead of the auth-manager callback | Register `.../authProviders/NAME/oauthcallback` exactly (see `gcloud ... describe`). Your URL is `continue_uri` |
| Consent loop / token rejected | ServiceNow app lacks the scope; GitHub/Microsoft asked for >1 scope | Align scopes with the app config. Use a single scope |
| Consent never completes (auth manager) | `continue_uri` handler doesn't POST `credentials:finalize`, or the nonce/user mismatch | Store `consent_nonce` + `user_id` per session and match on `uuid` for concurrent flows |
| Cloud Run **401** | No token, **access token instead of ID token**, or wrong `aud` (custom domain, tag URL) | Mint an ID token with `aud` = `run.app` URL (or configure a custom audience) |
| Cloud Run **403** | Caller principal lacks `roles/run.invoker`. For GE → A2A: the **Discovery Engine service agent** lacks it | Grant `run.invoker` to the right principal (agent identity, SA, or `service-NUM@gcp-sa-discoveryengine...`) |
| App's own JWT check breaks after enabling Cloud Run IAM | Both IAM and the app want `Authorization` | Send the IAM ID token in `X-Serverless-Authorization`. If both headers are present, Cloud Run checks only that one and strips its signature before the container |
| Google remote MCP **403** | Missing `roles/mcp.toolUser`, product role, MCP endpoint not enabled, or a read-only deny policy | Grant the roles, enable the MCP server, and check deny policies (`tools/list` still shows write tools) |
| API key rejected by a Google MCP server | Server requires an IAM principal | Use ADC/OAuth. API keys work only for non-IAM services (e.g. Maps) |
| `API_KEY_SERVICE_BLOCKED` / `API_KEY_INVALID` | API not enabled, or key restrictions / whitespace | Enable the API, fix the key's API restrictions, re-copy the key |
| Third-party MCP client can't log in to Google MCP | Client relies on **Dynamic Client Registration / CIMD** | Not supported. Pre-create an OAuth client ID/secret |
| `invalid_grant` / 401 after days of working | Refresh token expired or revoked; consent withdrawn | Clear cached tokens and re-trigger consent (the auth manager re-prompts) |
| 403s right after redeploy, or after Cloud Run SA → agent-identity switch | New principal, old grants don't apply | `principalSet` baselines, re-bind from `effectiveIdentity`, deploy `--no-traffic` first on Cloud Run |
| Agent Runtime deploy fails reading a secret | Agent Platform Service Agent lacks `secretAccessor`, or the secret is in another project | Grant it, and keep secrets in the agent's project |

#### Exam signals (auth-specific)

| Scenario says... | Pick |
|---|---|
| "agent calls BigQuery/GCS as itself, no keys" | **Agent Identity + ADC**, role on the resource |
| "agent on **GKE** needs Google APIs" | **Workload Identity Federation for GKE** (Agent Identity is only on Agent Runtime, GE, and Cloud Run) |
| "custom MCP server on Cloud Run, only our agents may call it" | `--no-allow-unauthenticated` + **`run.invoker`** to the agent principal + **ID token** with `aud` = URL |
| "developers' CLI/IDE must reach private Cloud Run MCP with OAuth login" | **IAP on Cloud Run** (`--iap`, `iap.httpsResourceAccessor`, custom OAuth client for programmatic access) or `gcloud run services proxy` |
| "app already uses the Authorization header" | **`X-Serverless-Authorization`** for the IAM token |
| "use Google's BigQuery/Cloud Run MCP server" | ADC/OAuth + **`roles/mcp.toolUser`** + product role. No API keys, no DCR |
| "block write MCP tools org-wide without a gateway" | **IAM deny** on `mcp.googleapis.com/tools.call` with `tool.isReadOnly == false` |
| "nightly job, SaaS supports OAuth, no user" | **2LO** auth provider |
| "SaaS only offers a static key" | **API-key auth provider** (not env vars, not code) |
| "as the signed-in user, their permissions, consent" | **3LO** auth provider (custom app) / **GE authorization resource + `toolAuthorizations`** (GE-hosted) |
| "agent must never see the user's raw credential" | Auth manager + **Agent Gateway** with GE (credential decrypted at the gateway) |
| "no auth code in code, bind provider declaratively" | **Agent Registry binding** + `registry.get_mcp_toolset(...)` |
| "orchestrator → sub-agent on Agent Runtime" | Roles on the **parent agent's identity** (`aiplatform.user` on the target engine, `agentregistry.viewer`) + gateway Access policy |
| "public A2A agent, callers need to know how to authenticate" | Declare **`securitySchemes` + `security`** in the agent card |
| "multi-tenant MCP, per-user JWT forwarded" | `McpToolset(header_provider=...)` |
| "private Cloud Run API from an OpenAPI tool" | `ServiceAccount(use_default_credential=True, use_id_token=True, audience=URL)` |

**Distractors to reject:**
- "Store the SaaS key in an environment variable / the system prompt / session state." Use the auth manager, or Secret Manager at minimum.
- "Use a service-account key file for the GKE/Cloud Run agent." Use WIF or the metadata server. Keys are the last resort for off-cloud callers with no WIF.
- "Grant the **user** `run.invoker` / `aiplatform.user` so the A2A call works." The **calling agent's** principal needs it.
- "Send the Google access token to the private Cloud Run MCP server." It needs an **ID token** with the right `aud`.
- "Set `--allow-unauthenticated` and rely on the MCP server's obscurity." Only acceptable for a genuinely public A2A/MCP endpoint that declares and enforces its own `securitySchemes`.
- "Register `continue_uri` as the OAuth redirect URI." Register the **auth-manager `oauthcallback`** URL.
- "Use 2LO so actions are attributed to each user." 2LO is the agent's authority. Use 3LO.
- "Disable CAA to fix a 401." Fix how the token is used instead.
- "Grant the agent `roles/agentidentity.user` on the whole project." The documented pattern grants it **on the specific auth provider** resource, which is least privilege: one agent, one provider.

---

### 5.1.2 Principal Access Boundary (PAB) with Agent Identity

> 🧠 **Visual memory map:** Identity & access — which control?
>
> [![Identity & access — which control?](visual-memory/14-identity-access-controls.png)](visual-memory/14-identity-access-controls.png)

- **What PAB is.** A policy attached to a **principal set** that defines which resources those principals are *eligible* to access. **It never grants access.** Access requires **allow AND not deny AND inside the PAB**. Use it to stop an agent from reaching resources outside a boundary (a folder, the org, a project) **regardless of what allow policies say**, for example an over-broad grant or a role granted in a foreign org.
- **Agent-relevant principal sets you can bind a PAB to:**
  - **Agent identities in a project's trust domain:** `//agents.global.org-ORG_ID.system.id.goog/attribute.container/projects/PROJ_NUM` (or the `proj-` form). The binding's parent is the project.
  - **Project principal set** `//cloudresourcemanager.googleapis.com/projects/PROJECT_ID`: service accounts, WIF pools, **and agent identities** in the project.
  - **Folder** and **org** principal sets include agent identities in all descendant projects. PAB is **not inherited like allow policies**. Instead, parent principal sets contain their descendants' principals.
- **Binding conditions** support only `principal.type` and `principal.subject`, with at most 10 logical operators. Use them to scope a broad binding to agents, or to exempt one principal.
- **No cross-org bindings.** Moving a project to another org leaves a cross-org binding, which IAM deletes automatically.
- **Roles:** `roles/iam.principalAccessBoundaryAdmin` to create policies (unverified exact name for the admin role). `roles/iam.principalAccessBoundaryUser` on the org to bind them. Binding also needs `roles/resourcemanager.projectIamAdmin`/`folderIamAdmin`/`organizationAdmin` on the target principal set.

```json
{ "displayName": "agents-stay-in-ai-folder",
  "details": { "rules": [ {
      "description": "Agents may only touch resources in the AI folder",
      "resources": ["//cloudresourcemanager.googleapis.com/folders/0123456789012"],
      "effect": "ALLOW" } ] } }
```
```bash
gcloud iam principal-access-boundary-policies create agents-stay-in-ai-folder \
  --organization=ORG_ID --location=global --details-rules=rules.json   # (flag names unverified)
gcloud iam principal-access-boundary-policies bindings create agents-pab-binding \
  --organization=organizations/ORG_ID --policy=agents-stay-in-ai-folder \
  --target-principal-set=cloudresourcemanager.googleapis.com/organizations/ORG_ID \
  --condition-expression="principal.type == 'iam.googleapis.com/AgentIdentity'"   # (type string unverified)
```
- **Agent Gateway + PAB:** "Agent Gateway can use IAP to enforce Principal Access Boundary policies." That means PAB applies both to Google API calls and to gateway-mediated egress.

**IAM allow vs IAM deny vs PAB vs VPC-SC vs Agent Gateway Access policy**

| Control | Attached to | Answers | Grants? | Typical agent use |
|---|---|---|---|---|
| **IAM allow** | Resource (project/bucket/dataset…) | Which principals may do X on this resource | Yes | Grant `roles/bigquery.dataViewer` to one agent's `principal://` |
| **IAM deny** | Resource hierarchy (org/folder/project) | Which permissions are forbidden for which principals, overriding allow | No | Deny `storage.buckets.delete` to all agent identities in the org |
| **PAB** | **Principal set** | Which resources these principals are *eligible* to touch | No | Keep all agents inside the AI folder, even if someone grants them a role elsewhere |
| **VPC-SC** | Service perimeter around projects/APIs | May data cross this perimeter (context: network, identity, access level)? | No | Stop exfiltration from BigQuery/GCS through Agent Runtime; agent identities can be used in ingress/egress rules |
| **IAM Unified Access Policy (Agent Gateway)** | Project (or gateway) that hosts Agent Gateway | May agent A reach destination B (MCP tool, agent, endpoint, URL), and which methods/tools? | Allow and deny rules in one policy; deny wins | Allow `getStatements`, deny `updateRecord` on the finance MCP server; allow the orchestrator to call one sub-agent |

Rule of thumb:
- **Resource-centric → allow/deny.**
- **Identity-centric boundary → PAB.**
- **Data-exfiltration perimeter → VPC-SC.**
- **Agent-to-tool traffic at the L7/MCP level (tool name, read-only hint, host/path/method) → Agent Gateway Access policy.**

IAM allow/deny can't see tool names on **your own** MCP servers. For Google Cloud remote MCP servers, only IAM **deny** conditions on `mcp.tools.call` can use `tool.isReadOnly`/`tool.name` (see 5.1.1b). Access policies can't protect a BigQuery dataset from a direct API call that bypasses the gateway. That is why you layer them.

---

### 5.1.3 Agent Gateway: monitor traffic and track agents

**What it is.** A Google-managed L7 gateway (`gcloud network-services agent-gateways`) that is the entry and exit point for agent traffic. It terminates mTLS, mediates protocols (MCP, A2A, REST, gRPC; any HTTP), applies policies, and emits telemetry. It is framework-agnostic.

| Mode (`governedAccessPath`) | Direction | Supported runtimes | Identity | Registry / Access policies | Content guards |
|---|---|---|---|---|---|
| `CLIENT_TO_AGENT` (ingress) | Client (Gemini CLI, Claude Code, Cursor, web app) to agent | **Agent Runtime only** | Client/user creds | **Not available** | Model Armor **or** SGP (only one CONTENT_AUTHZ policy; the two can't be combined) |
| `AGENT_TO_ANYWHERE` (egress) | Agent to MCP servers, agents, APIs, LLMs anywhere | Agent Runtime **and Gemini Enterprise** | Agent SPIFFE ID | Required. Up to 2 registries (1 global + 1 regional/multi-regional) | Model Armor, SGP, custom engines (max **4** custom authz policies per egress gateway) |

**Egress enforcement sequence** (a favorite exam topic):
1. The agent's outbound call is intercepted.
2. **IAP** checks for an Access policy granting `iap.resources.egressViaIAP` on the destination.
3. The destination must be **registered in Agent Registry**, or be matched by an explicit unregistered-host policy. **Default deny.**
4. Content checks run: Model Armor, SGP, custom engines.
5. The request is forwarded.

**Setup (egress):**
```yaml
# my-agent-gateway-egress.yaml
name: prod-egress
protocols: [MCP]
googleManaged:
  governedAccessPath: AGENT_TO_ANYWHERE
registries:
  - //agentregistry.googleapis.com/projects/PROJECT_ID/locations/us-central1
```
```bash
gcloud network-services agent-gateways import prod-egress --source=my-agent-gateway-egress.yaml --location=us-central1
```
Wire IAP in as a `REQUEST_AUTHZ` extension:
```yaml
# iap-request-authz-extension.yaml
name: iap-ext
service: iap.googleapis.com
failOpen: false
timeout: 1s
metadata:
  iapPolicyVersion: "V2"          # V2 = Unified Access Policy (recommended); V1 = legacy allow
  iamEnforcementMode: "DRY_RUN"   # remove it (or set ENFORCED) to enforce
```
```bash
gcloud service-extensions authz-extensions import iap-ext --source=iap-request-authz-extension.yaml --location=us-central1
# authz policy: target = agentGateways/prod-egress, policyProfile: REQUEST_AUTHZ, action: CUSTOM
gcloud network-security authz-policies import iap-authz --source=iap-request-authz-policy.yaml --location=us-central1
```
- **Prerequisite gotcha:** the managed org policy constraint **`constraints/iam.managed.disableAccessPolicyBindings`** is **on by default for new orgs**. It blocks binding Unified Access Policies until you turn it off.
- **Binding runtimes to the gateway:**
  - *Agent Runtime:* set the gateway config on the deployment spec. Verify with `.spec.deploymentSpec.agentGatewayConfig`; `null` means the binding failed.
  - *Gemini Enterprise:* use the app's **Security** tab, or PATCH `agentGatewaySetting.defaultEgressAgentGateway`. This **immediately routes all existing agent traffic**. You then still have to **import** MCP servers and A2A agents from Agent Registry into the GE app. Deploy the gateway in the region that maps to the GE multi-region.

**Unified Access Policies (IAM v3 "Access policies")**
- One policy holds several **rules**. Each rule has `principals` (agent `principal://` or `principalSet://`, WIF principals also allowed), an `effect` (`ALLOW`/`DENY`), `operation.permissions: ["iap.googleapis.com/resources.egressViaIAP"]`, and a CEL `conditions` block keyed `"iap.googleapis.com"`. **Deny rules are evaluated first and override allow.**
- You bind the policy to the **project** (every gateway in the project enforces it) or to a specific gateway: `gcloud iam policy-bindings create ... --target-resource=//cloudresourcemanager.googleapis.com/projects/P`.
- **CEL attributes:**

  | Destination | Attributes |
  |---|---|
  | Any registered resource | `destination.is_registered`, `destination.agent_registry.resource_type` (`AGENT`/`ENDPOINT`/`MCP_SERVER`/`SKILL`), `.location`, `.project_id` |
  | Agent | `...agent.name` |
  | MCP server | `...mcp_server.name`, `.method`, `.tool.name`, `.tool.annotations.read_only_hint`/`destructive_hint`/`idempotent_hint`/`open_world_hint`, `.prompt.name`, `.resource.name` |
  | Endpoint | `...endpoint.name` |
  | Unregistered | `destination.unregistered.host`/`.path`/`.method`. `startsWith`/`endsWith`/`contains` work **only** on host and path |

```json
[{ "description": "Read-only GitHub tool", "effect": "ALLOW",
   "principals": ["principal://agents.global.org-123/resources/aiplatform/projects/987/locations/us-central1/reasoningEngines/support-agent"],
   "operation": {"permissions": ["iap.googleapis.com/resources.egressViaIAP"]},
   "conditions": {"iap.googleapis.com": {"expression":
     "destination.agent_registry.mcp_server.tool.name == 'GitHubTool' && destination.agent_registry.mcp_server.tool.annotations.read_only_hint == true"}}},
 { "description": "Never mutate finance records", "effect": "DENY",
   "principals": ["principalSet://agents.global.org-123/attribute.platformContainer/aiplatform/projects/987"],
   "operation": {"permissions": ["iap.googleapis.com/resources.egressViaIAP"]},
   "conditions": {"iap.googleapis.com": {"expression":
     "destination.agent_registry.mcp_server.name == '/projects/p/locations/us-east1/mcpServers/finance' && destination.agent_registry.mcp_server.tool.name == 'updateRecord'"}}}]
```
```bash
gcloud iam access-policies create agent-egress --details-rules=policy.json --project=P --location=global
```

**DRY_RUN vs ENFORCE**
- In `DRY_RUN`, IAP **logs denials to Cloud Audit Logs but doesn't block**. The console calls this **"Audit-only"**. Use it in staging first.
- In `ENFORCE` (console: "Enforce policies"), anything without an explicit allow is blocked.
- **498 gotcha:** binding an agent to an *enforcing* gateway routes **all** outbound traffic through it, including platform calls such as the Sessions API. If the **essential endpoints** aren't registered and allowed, every invocation fails with **HTTP 498**. The essential endpoints are `agentregistry`, `logging`, `telemetry`, `cloudtrace`, `monitoring`, `secretmanager`, `cloudresourcemanager`, `iamcredentials`, and `aiplatform`, plus their `*.mtls.googleapis.com`, `REGION-aiplatform`, and `aiplatform.REGION.rep` variants. Hostname matching is **exact, with no wildcards**. Allowlist first, or stay in DRY_RUN until you have.
- Startup 403s under enforcement mean internal services are blocked. Switch to DRY_RUN, find the hostnames in logs, register them, and grant egress.

**Monitoring and audit**

| Log / surface | Use |
|---|---|
| `resource.type="networkservices.googleapis.com/Gateway"`, log `networkservices.googleapis.com%2Fgateway_requests` | Gateway allow/deny. `jsonPayload.authzPolicyInfo.policies.result`, `agentGatewayInfo.mcpInfo` (method + tool name), `agentRegistryResource`, `serviceExtensionsInfo` |
| `cloudaudit.googleapis.com%2Fdata_access`, `protoPayload.serviceName="iap.googleapis.com"`, permission `iap.resources.egressViaIAP` | IAP decisions. `authorizationInfo[].granted`, `authenticationInfo.principalSubject` (SPIFFE ID), `metadata.iamEnforcementMode="DRY_RUN"` |
| Label `iap.googleapis.com/audited_resource_name = unregisteredResource` | Destination hostname isn't in Agent Registry |
| Model Armor logs (`modelarmor.googleapis.com%2Fdetection` on `model-armor_managed_service`) | Content findings |
| Gateway **Observability tab** | Attempted authorizations, auth-failure %, RPS, egress logs (403s, unregistered endpoints). **Needs the `_Default` log bucket upgraded to Observability Analytics** |
| Agent Registry **topology** view | Live relationships and traffic flows between agents and MCP servers |

**Limits and gotchas**
- **"This feature does not support VPC Service Controls"** appears on the IAM agent policies and Access-policies pages. The only exception: VPC-SC perimeter enforcement for gateway traffic is supported **only** for gateways created **after 2026-09-08** that use an **agent connectivity template** in **`ALL_TRAFFIC`** egress mode. VPC egress settings are immutable, so you need a new template to change them.
- No **self-signed** destination cert chains.
- Each gateway governs up to **5,000** registered resources.
- Ingress mode isn't available for Gemini Enterprise.
- A regionalized registry means Access policies apply only to resources in that region.
- Registration is validated when you bind a policy, so referencing an unregistered service gives `NOT_FOUND`.
- Custom authz extensions target FQDNs only, over HTTP/2+TLS on 443. They **don't validate server certs**, so keep them inside your VPC with DNS peering. `CONTENT_AUTHZ` extensions must speak ext_proc in `FULL_DUPLEX_STREAMED` mode.
- The execution order of multiple custom policies with the same profile **isn't guaranteed**.

---

### 5.1.4 Agentic governance and policy enforcement: Agent Registry and Model Armor

#### Agent Registry
- **What it is:** the central catalog of **agents, MCP servers, endpoints, and skills (Preview)**. Agents and MCP servers are searchable by keyword or prefix (semantic search exists only for skills). **Auto-registration** (same project only) covers Agent Runtime and Gemini Enterprise agents, Google remote MCP servers, Cloud Run services deployed with `--functional-type=agent|mcp-server`, and GKE workloads labelled `registry.gke.io/functional-type`. Everything else, including cross-project entries, is registered manually (see Section 3.2.4). Registries can be global, multi-regional, or regional.
- **Governance role:** the gateway's allowlist, since unregistered destinations are denied unless a policy names them by host. Registry entries are also the targets of Access policies (`--agent`, `--endpoint`, `--mcp-server`), Semantic Governance policies, and auth-provider **bindings**.
- **IAM roles:** `roles/agentregistry.viewer` (discover), `.editor`, and `.admin` (bindings). For **A2A delegation**, the *parent agent's identity* (not your user) needs `roles/agentregistry.viewer` to resolve the sub-agent and `roles/aiplatform.user` on the sub-agent's reasoning engine.
- **Composite Google APIs endpoint:** register several core Google API hostnames in a single registry entry, which suits multi-project setups.

#### Model Armor (content security)
- **Filters (detections):**
  - **RAI:** hate speech, harassment, sexually explicit, dangerous. **CSAM is always on.**
  - **Prompt injection & jailbreak:** returns `NO_MATCH_FOUND` for inputs under 3 words. **High** confidence is recommended to limit false positives.
  - **Sensitive Data Protection:** *Basic* mode covers a fixed infoType set (credit card, US SSN, financial account, ITIN, GCP credentials, GCP API key) and **inspect only**. *Advanced* mode uses an SDP **inspect template**, plus an optional **de-identify template** for redaction. The SDP templates must be in the **same location** as the MA template. If they live in another project, grant the Model Armor service agent `roles/dlp.user` and `roles/dlp.reader`.
  - **Malicious URL:** scans the first 256 URLs.
  - Image modality screening (Preview), multi-language detection, and filter versions (floor settings default to Stable).
- **Confidence levels:** `HIGH` (few false positives, suits production), `MEDIUM_AND_ABOVE` (standard enterprise), `LOW_AND_ABOVE` (catches most, many false positives; only sensible for PI/jailbreak in high-stakes flows). They apply to RAI and PI/jailbreak. SDP uses its own likelihood scale.
- **Enforcement type:** `INSPECT_ONLY` logs findings. `INSPECT_AND_BLOCK` blocks. **Redaction happens only under `INSPECT_AND_BLOCK`, and only when SDP is the *sole* violated detector.** If several detectors fire, the whole payload is blocked. Logs are redacted per the de-identify template in either mode.
- **Templates vs floor settings:**
  - *Template* (regional, `projects/P/locations/L/templates/T`): per-call or per-integration config.
  - *Floor settings* (`projects|folders|organizations/.../locations/global/floorSetting`): the **minimum baseline**. Templates must meet it, and integrated services apply it even when a request specifies nothing.
  ```bash
  gcloud model-armor floorsettings update --full-uri=projects/P/locations/global/floorSetting \
    --enable-floor-setting-enforcement=true \
    --add-integrated-services=VERTEX_AI --vertex-ai-enforcement-type=INSPECT_AND_BLOCK \
    --pi-and-jailbreak-filter-settings-enforcement=ENABLED \
    --malicious-uri-filter-settings-enforcement=ENABLED \
    --add-rai-settings-filters='[{"confidenceLevel":"MEDIUM_AND_ABOVE","filterType":"DANGEROUS"}]'
  ```
- **Integration points:**

  | Integration | How | Notes |
  |---|---|---|
  | **Agent Gateway** (ingress + egress) | `CONTENT_AUTHZ` authz extension, `service: modelarmor.LOCATION.rep.googleapis.com`, `model_armor_settings` request/response template IDs. Or tick "Enable Model Armor" in the console | Templates in the **same region** as the gateway. Egress: grant the Service Extensions SA `service-PROJNUM@gcp-sa-dep.iam.gserviceaccount.com` `modelarmor.calloutUser` + `serviceusage.serviceUsageConsumer` (gateway project) and `modelarmor.user` (template project), **even for same-project templates**. Ingress: grant the Reasoning Engine service agent (`gcp-sa-aiplatform-re`) the equivalent. Use `httpRules` to send only JSON/text content types |
  | **Gemini on Agent Platform** (`generateContent`) | Per-request `model_armor_config{prompt_template_name, response_template_name}`, or floor setting `integratedServices: [AI_PLATFORM]` (gcloud `VERTEX_AI`) | Blocked calls return `promptFeedback.blockReason: MODEL_ARMOR`. Precedence: **request template > floor setting > Gemini safety filters**. Specifying **both** a template **and** Gemini safety settings in one request is an **error**. With a floor setting plus safety settings, both run and the **most restrictive** result wins. The floor-setting integration defaults to **INSPECT_ONLY** |
  | **Google / Google Cloud remote MCP servers** | Floor setting `--add-integrated-services=GOOGLE_MCP_SERVER --google-mcp-server-enforcement-type=INSPECT_AND_BLOCK` | Cross-project agent and MCP means MA is invoked **twice** (client and resource project floors). Don't enable PI/jailbreak unless the MCP traffic is natural language |
  | **ADK** | `ModelArmorPlugin(ModelArmorConfig(prompt_template_name, response_template_name))` on the `App` (ADK ≥ 2.8, `google-adk[gcp]`) | Screens the latest user turn and the model reply. It **skips `function_response` parts**, so tool output isn't screened; use the gateway or a callback for indirect injection. Both templates must be in one region. `block_on_screening_failure=True` by default (fail-closed). Blocks set `custom_metadata['model_armor_blocked']` |
  | Gemini Enterprise, direct `sanitizeUserPrompt`/`sanitizeModelResponse` API | Templates | Validate a template before deploying by calling `sanitizeUserPrompt` with test prompts |

#### Semantic Governance Policies (SGP)
- **What they are:** **natural-language constraints** (NLCs, up to 5,000 chars) evaluated by an **LLM judge** (the "policy engine", provisioned in your VPC) against each **proposed tool call**. Inputs are the user prompt, constraints, tool manifest, chat history, and suggested calls. Verdicts are `ALLOW`/`DENY`. A DENY removes the tool call and returns a **human-readable rationale**.
- **Scope:** agent-wide, or per MCP tool (`--mcp-tools`). Example: "refund `amount` ≤ $80".
- **CLI:** `gcloud beta ai semantic-governance-policies create --agent=projects/P/locations/L/agents/AGENT_ID --natural-language-constraint="..."`. Dry-run via authz-extension metadata `sgpEnforcementMode: DRY_RUN`.
- **Trade-off:** probabilistic, since an LLM judge can be wrong, and it adds latency. Always use dry-run first.
- **Gotcha:** constraints are Service Data, and the rationale may quote them to end users. Keep secrets out of NLCs, or intercept and redact the denial in the agent.
- **SGP vs IAM:** IAM and Access policies are *static* (can this principal call this tool?). SGP is *semantic* (does this call match the user's intent and the business rules?).

---

## 5.2 Implementing secure agent behavior and execution

### 5.2.1 Safety frameworks and guardrails: layered defense

> 🧠 **Visual memory map:** Guardrail layers
>
> [![Guardrail layers](visual-memory/15-guardrail-layers.png)](visual-memory/15-guardrail-layers.png)

The ADK safety guidance lists these layers: identity and authorization, in-tool guardrails, Gemini safety features, callbacks and plugins, Gemini-as-judge, sandboxed code execution, evaluation and tracing, and network controls/VPC-SC. Map each to the threat it handles.

| Control | Layer | Deterministic? | Catches | Misses / cost |
|---|---|---|---|---|
| **Gemini safety settings** (`SafetySetting(category, threshold, method)`) | Model | Classifier | Harmful *generated* content in 4 adjustable categories (+ civic integrity). Thresholds: `BLOCK_LOW_AND_ABOVE`, `BLOCK_MEDIUM_AND_ABOVE`, `BLOCK_ONLY_HIGH`, `BLOCK_NONE`, `OFF`. Method `SEVERITY` (default on Gemini API) or `PROBABILITY` | Not PI, PII, or URLs. **`OFF` is the default for gemini-3.5-flash and later**, so set it explicitly. Blocked output returns `finishReason: SAFETY` |
| **Model Armor** | Platform / gateway / plugin | Classifier | PI/jailbreak, RAI, PII/DLP (with redaction), malicious URLs. Central, cross-model, cross-agent, and **org/folder floor settings** | Per-call latency and cost. Doesn't know business rules. Short inputs are not flagged |
| **Callbacks** (`before_model_callback`, `after_model_callback`, `before_tool_callback`, `after_tool_callback`, `before/after_agent_callback`) | Agent code | Yes, unless the callback calls an LLM | Arg validation against session state (e.g. `user_id` must match the session), table allowlists, PII scrub, tool-output screening | Per-agent code to maintain. Return an `LlmResponse` from `before_model` to skip the model; return a `dict` from `before_tool` to skip the tool |
| **Plugins** (`BasePlugin` on the `App`/`Runner`) | Runner-global | Yes | Same hooks, applied **to every agent, tool, and model call** in the app. Plugin callbacks run **before** agent-level callbacks (verify precedence in ADK docs). ModelArmorPlugin, ATR guardrail plugin | Global scope means a bug hurts everything |
| **In-tool guardrails** | Tool | Yes | The policy lives in developer-set `ToolContext`/state (e.g. `select_only`, allowed tables), which the model can't change | Must be designed into each tool |
| **LLM-as-judge** (Gemini Flash-Lite in a callback) | Agent code | No | Novel or semantic attacks (e.g. "sympathy" social engineering that can get past Model Armor's prompt-injection filter), off-topic or brand risk | Extra latency and tokens. The judge itself can be injected |
| **Semantic Governance** | Gateway | No (managed LLM judge) | Tool call vs user intent and business rules, **without redeploying code** | Latency. Rationale leakage |
| **HITL** | Workflow | Human | Irreversible or high-value actions | Throughput and latency. Needs a UI and resumable state |
| **Sandboxed code exec** | Runtime | Yes | Model-generated code escaping (Agent Platform sandbox, GKE Sandbox, Cloud Workstations) | — |
| **IAM / PAB / Access policy / VPC-SC** | Identity & network | Yes | Blast radius, whatever the model decides | Can't judge content |

**Principle to cite in answers:** use deterministic controls (IAM, Access policies, in-tool policy, `before_tool_callback`) for anything that **must never** happen. Use probabilistic controls (Model Armor, SGP, LLM judge) to **reduce** risk. Use HITL for **irreversible** actions. Also, a prompt instruction is not a control.

#### Human-in-the-loop in ADK
- **Tool Confirmation (experimental):** `FunctionTool(reimburse, require_confirmation=True)`, or pass a predicate such as `async def f(amount, tool_context) -> bool: return amount > 1000`. For structured input, the advanced form calls `tool_context.request_confirmation(hint=..., payload={...})` and reads `tool_context.tool_confirmation.payload` on resume. The client sees an **`adk_request_confirmation`** event, and remote approval can come in through the ADK server REST API. **Limitation:** it doesn't work with `DatabaseSessionService` or `VertexAiSessionService` (the managed Agent Runtime sessions). This is a likely distractor. Go: `RequireConfirmationProvider`. Java/Kotlin: evaluate the condition inside the tool. TypeScript: manual.
- **Long-running tools:** `LongRunningFunctionTool` starts a job (e.g. a ticket sent for manager approval), returns a pending status, and the agent resumes when the client sends the final `FunctionResponse`. Use it for asynchronous or multi-hour approvals.
- **Graph workflows (ADK 2.x):** a node yields **`RequestInput(message=..., payload=..., response_schema=...)`** to pause the workflow deterministically, with no model involved. The reply becomes the node's output. With `rerunOnResume`, the node re-runs on resume (Go uses `ResumeOrRequestInput`). An `adk_request_input` event is used for free-form input.
- **MCP:** `McpToolset(require_confirmation=...)` gates destructive MCP tools. The MCP `elicitation_callback` handles server-initiated prompts.
- **Choosing:** use a sync yes/no in chat (Tool Confirmation), a structured approval form (advanced confirmation or `RequestInput`), an out-of-band or async approver (long-running tool), or a deterministic approval step in a pipeline (graph `RequestInput`).

### 5.2.2 Secure data access and identity propagation

**Identity propagation chain**
1. **User → Gemini Enterprise / app:** user signs in (Workforce/Cloud Identity). GE governs agent access through Workspace/GE admin.
2. **GE → agent:**
   - Cloud Run A2A: a service-agent OIDC token in `X-Serverless-Authorization`, with `run.invoker`.
   - Agent Runtime: IAM.
   - When configured, the user's OAuth token is also passed (GE authorization resource / `toolAuthorizations`).
3. **Agent → Google APIs as itself:** agent identity through ADC, protected by mTLS and CAA.
4. **Agent → Google APIs as the user:** a 3LO token from the auth manager or the GE authorization. Logs show user + agent.
5. **Agent → external SaaS:** auth manager (2LO/API key/3LO), with headers injected by ADK.
6. **Agent → other agent (A2A):**
   - The parent's agent identity is authorized by an Access policy (gateway) or IAM (`aiplatform.user` on the target reasoning engine).
   - Discovery goes through Agent Registry.

**Context-Aware Access (agent flavor).** It is on by default and Google-managed.
- mTLS proves the certificate-bound token is used from the provisioned runtime.
- DPoP: CAA verifies that the gateway-generated **resource DPoP proof** and the agent's **authorization DPoP proof** were produced with the same private key held in the agent identity pool.
- **Outcome:** stolen tokens are useless elsewhere, which defeats credential theft and account takeover and isolates Agent Runtime tenants from each other.

**VPC Service Controls**
- Add `aiplatform.googleapis.com` (which includes Agent Runtime and sandboxes) and the data services to the perimeter. You can also put `agentidentity.googleapis.com` and `agentidentitycredentials.googleapis.com` inside; clients then use `restricted.googleapis.com`. Agent identities can appear in ingress/egress rules.
- **The project must be in the perimeter *before* you deploy the agent.** Otherwise the agent isn't protected and keeps its public internet access.
- Inside VPC-SC, the agent has **no internet**. Egress goes through a **PSC interface**, then a proxy VM (RFC 1918) with Cloud NAT. Agent Runtime never provides direct internet egress, even without VPC-SC.
- URL-context live fetch is disabled, and request/response logging is unavailable.
- Agent Gateway's IAM Access policies **don't** support VPC-SC. Gateway traffic is VPC-SC-enforced only through the post-2026-09-08 connectivity template in `ALL_TRAFFIC` mode.

**Sensitive Data Protection (DLP)**
- *Inline, conversational:* use Model Armor with advanced SDP (inspect + de-identify templates) at the gateway, the model integration, or the plugin, and redact under `INSPECT_AND_BLOCK`.
- *Data-at-rest feeding RAG or agents:* run SDP discovery, inspection, and de-identification (masking, tokenization, FPE/crypto-hash) *before* indexing into Agent Search, RAG Engine, or BigQuery. Re-identification then needs the key, which lets you split roles.
- *In tools:* call the DLP API `deidentify_content` in an `after_tool_callback` so PII never reaches the model context.
- *Logs:* Model Armor logs are redacted per the de-identify template. ADK analytics plugin redaction is **not** DLP.

---

## Defense-in-depth reference architecture

> 🧠 **Visual memory map:** Defense in depth — the request path
>
> [![Defense in depth — the request path](visual-memory/13-defense-in-depth.png)](visual-memory/13-defense-in-depth.png)

```
[User] --SSO (Cloud Identity / Workforce IdF)--> [Gemini Enterprise app  |  custom web app / CLI client]
   |  (GE admin: which users see which agents; Workspace admin: data policies)
   |
   |  (1) INGRESS: Agent Gateway CLIENT_TO_AGENT  (Agent Runtime only)
   |        - Model Armor CONTENT_AUTHZ (PI/jailbreak, RAI, SDP, URLs)   OR  Semantic Governance
   v
[Agent on Agent Runtime]  identity_type=AGENT_IDENTITY  -> SPIFFE ID + X.509, CAA mTLS
   |  (2) IN-PROCESS (ADK)
   |     - Plugins on App: ModelArmorPlugin, logging/analytics (redaction != DLP)
   |     - before_model_callback (LLM judge, input policy) / Gemini SafetySettings
   |     - before_tool_callback (arg vs session state, allowlists) / in-tool ToolContext policy
   |     - HITL: require_confirmation / LongRunningFunctionTool / graph RequestInput
   |     - after_tool_callback (DLP scrub of tool output = indirect-injection & PII defense)
   |     - Credentials: auth manager (API key / 2LO / 3LO via adk_request_credential)
   |
   |  (3) EGRESS: Agent Gateway AGENT_TO_ANYWHERE  (mTLS terminates here, DPoP beyond)
   |     a. IAP REQUEST_AUTHZ: IAM Unified Access Policy (egressViaIAP; allow+deny; MCP tool CEL)
   |        + PAB on agent principal set
   |     b. Agent Registry check (default deny unregistered)
   |     c. CONTENT_AUTHZ: Model Armor (tool payloads/responses), Semantic Governance (intent), custom ext_proc
   |     d. Logs: gateway_requests + IAP data_access audit (DRY_RUN/ENFORCE) -> Observability dashboard
   v
[Tools / MCP servers (Google remote MCP w/ MA floor setting) | other agents via A2A | SaaS APIs]
   |
   |  (4) RESOURCE: IAM allow/deny on data; PAB boundary; VPC-SC perimeter (PSC-I + proxy egress);
   |      CMEK; SDP-de-identified corpora; audit logs show agent (and user if delegated)
   v
[Data: BigQuery, Cloud SQL, GCS, Firestore, Agent Search, RAG Engine]
```

---

## Exam signals (keyword → likely answer)

| Scenario says... | Pick |
|---|---|
| "per-agent least privilege", "credentials must not be replayable", "tied to agent lifecycle" | **Agent Identity** (not a shared SA). CAA mTLS/DPoP is on by default |
| "no matter what roles are granted, agents must never access resources outside folder X" | **PAB** bound to the agent principal set |
| "block a specific permission for all agents org-wide" | **IAM deny policy** |
| "allow agent to call read-only tools on an MCP server, block destructive ones" | **Agent Gateway + IAM Unified Access Policy** with `tool.annotations.read_only_hint` / a deny rule |
| "restrict which external hosts/paths agents may call" | Access policy with `destination.unregistered.host/path`, or register the endpoint in **Agent Registry** |
| "test policies without breaking traffic" | **DRY_RUN** (IAP), **INSPECT_ONLY** (Model Armor), `sgpEnforcementMode: DRY_RUN` |
| "all agent calls fail with 498 after attaching gateway" | Essential endpoints not allowlisted/registered (exact hostnames, incl. mTLS/regional variants) |
| "who called what tool", "track agents", "authorization failure rate" | Agent Gateway logs + IAP audit logs + Observability tab |
| "prompt injection / jailbreak / PII leakage / malicious URL, centrally across all agents and models" | **Model Armor** (floor settings for baseline, template at the gateway) |
| "org-wide minimum content-safety baseline developers can't weaken" | **Model Armor floor settings** at org/folder |
| "enforce business rules in plain English without redeploying", "tool call must match user intent" | **Semantic Governance Policies** |
| "access user's Jira/GitHub/Drive as the user", "consent" | **3LO auth provider** in the auth manager (`adk_request_credential`); in GE, an authorization resource |
| "machine-to-machine SaaS" | **2LO auth provider** |
| "no hardcoded secrets for third-party keys" | **Auth manager API-key provider** (a centralized vault) rather than env vars |
| "human approval before refunds > $1000" | ADK **`require_confirmation`** with a predicate |
| "approval takes hours / external system" | **LongRunningFunctionTool** |
| "deterministic pause in a graph workflow" | **`RequestInput`** node |
| "prevent data exfiltration from agent runtime" | **VPC-SC** (project in the perimeter *before* deploy) + PSC-I + proxy egress |
| "redact SSNs in prompts and responses" | Model Armor **advanced SDP** with a de-identify template + **INSPECT_AND_BLOCK** |
| "redeployed agent lost access" | New `reasoningEngines` ID means a new principal. Use principalSet baselines and re-bind from `effectiveIdentity` |

## Common distractors
- **"Put security rules in the system instruction."** That is not enforcement. Pick a deterministic or platform control.
- **"Use a shared service account with Owner/Editor for all agents."** This breaks least privilege and makes audit attribution impossible.
- **"Use PAB to grant the agent access."** PAB never grants access.
- **"Use VPC-SC to restrict which MCP tool an agent can call."** VPC-SC is perimeter/API-level; tool-level control belongs to Access policies.
- **"Rely on ADK `ModelArmorPlugin` for indirect prompt injection from tool results."** The plugin skips `function_response`. Use the gateway egress template or an `after_tool_callback`.
- **"Tool Confirmation on Agent Runtime managed sessions."** `VertexAiSessionService` is not supported (experimental limitation).
- **"Enable Client-to-Agent gateway for Gemini Enterprise."** Gemini Enterprise supports egress only.
- **"Wildcard `*.googleapis.com` allowlist."** Not supported; hostnames must match exactly.
- **"Model Armor template + Gemini safety settings in the same generateContent call."** That returns an error.
- **"Combine Model Armor and SGP on the same ingress gateway."** Only one `CONTENT_AUTHZ` policy is allowed on ingress.
- **"Grant A2A roles to your user account."** They must go to the **parent agent's identity**.

---

### Practice questions

**Q1.** A retailer runs 40 ADK agents on Agent Runtime. Security requires that no agent can ever access resources outside the `ai-prod` folder, even if a developer mistakenly grants an agent a role on a finance project. What should you configure?

- **A.** An IAM deny policy on the finance project denying all permissions to agents
- **B.** A Principal Access Boundary policy with an ALLOW rule for the `ai-prod` folder, bound to the agent identities' principal set
- **C.** A VPC-SC perimeter around `ai-prod`
- **D.** An Agent Gateway Access policy with a DENY rule for finance endpoints

**Answer: B.** PAB defines which resources a principal set is *eligible* to access, whatever allow policies say. A deny on one project (A) doesn't cover other resources outside the folder. VPC-SC (C) is about data-perimeter context, not identity eligibility. Gateway policies (D) govern only gateway-mediated traffic.

**Q2.** An agent must read support tickets through an internal MCP server but must never call `deleteTicket` or `updateTicket`. The MCP server is registered in Agent Registry and traffic flows through Agent Gateway. What is the most precise control?

- **A.** Remove the delete/update tools from the agent's instruction
- **B.** An IAM Unified Access Policy with an ALLOW rule on the MCP server and a DENY rule where `destination.agent_registry.mcp_server.tool.name in ['deleteTicket','updateTicket']`
- **C.** An IAM deny policy on the project denying `aiplatform.*`
- **D.** A Model Armor template with the dangerous-content filter at HIGH

**Answer: B.** Access policies evaluated by IAP understand MCP tool names and annotations, and deny rules override allow. Instructions (A) aren't enforcement. C is far too broad. Model Armor (D) classifies content, not authorization.

**Q3.** After binding an existing Agent Runtime agent to a new Agent Gateway in ENFORCE mode, every invocation fails with HTTP 498. What is the most likely cause?

- **A.** The Model Armor template is in a different region
- **B.** Essential platform endpoints (aiplatform, logging, telemetry, and so on, including mTLS and regional variants) aren't registered and allowed for the agent
- **C.** The agent lacks `roles/run.invoker`
- **D.** CAA blocked the agent's token

**Answer: B.** An enforcing gateway routes all egress, including Sessions/platform calls, and is default-deny with exact hostname matching. Allowlist the essential APIs first, or keep IAP in DRY_RUN until that's done.

**Q4.** A new egress governance policy must be validated in staging without blocking any agent traffic, and you need evidence of what would have been blocked. What do you do?

- **A.** Set `iamEnforcementMode: "DRY_RUN"` in the IAP authz extension metadata and review IAP entries in Cloud Audit Logs
- **B.** Deploy the gateway without an authz policy
- **C.** Set Model Armor to INSPECT_ONLY
- **D.** Disable CAA with `GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES=False`

**Answer: A.** In dry-run, IAP logs disallowed communications to Cloud Audit Logs (filter `metadata.iamEnforcementMode="DRY_RUN"`) without blocking. C covers content findings only, not access decisions.

**Q5.** An agent needs to create Jira issues *as the requesting employee*, so that Jira's own permissions apply and the audit trail shows the user. What is the recommended approach?

- **A.** Store a Jira admin API key in Secret Manager and read it in the tool
- **B.** Configure a 3-legged OAuth auth provider in Agent Identity auth manager, grant the agent `roles/agentidentity.user` on it, and attach it through `GcpAuthProviderScheme` (with `continue_uri`)
- **C.** A 2-legged OAuth auth provider
- **D.** Grant the agent identity a Jira role via IAM

**Answer: B.** 3LO gives user-delegated authority with consent, a vault, and refresh. ADK surfaces `adk_request_credential`, and audit attributes access to both agent and user. A and C act as the agent, not the user. Jira isn't governed by Google IAM (D).

**Q6.** A CI/CD pipeline deletes and re-creates an Agent Runtime agent on every release. After each release the agent gets 403s on BigQuery even though the Terraform grants haven't changed. What fixes this most robustly?

- **A.** Switch the agent back to a service account
- **B.** Grant baseline roles to the project `principalSet://...attribute.platformContainer/aiplatform/projects/PROJ_NUM`, and bind sensitive roles post-deploy to the new principal from `spec.effectiveIdentity`
- **C.** Disable CAA
- **D.** Add the agent to Agent Registry

**Answer: B.** Every re-created `reasoningEngines` resource gets a new principal, and old bindings don't carry over. PrincipalSet bindings survive re-creation, and dynamic re-binding covers the narrow grants. A gives up per-agent identity benefits.

**Q7.** The CISO wants a guarantee that every Gemini `generateContent` call in project `ai-prod` is screened for prompt injection and malicious URLs, even when developers don't pass any Model Armor config. What should you configure?

- **A.** Gemini safety settings `BLOCK_LOW_AND_ABOVE` in every agent
- **B.** Model Armor floor settings on the project with integrated service `VERTEX_AI` (AI_PLATFORM) and enforcement type `INSPECT_AND_BLOCK`
- **C.** An ADK `ModelArmorPlugin` in each app
- **D.** A Model Armor template referenced per request

**Answer: B.** Floor settings apply a baseline to all generateContent calls in the project, even without `modelArmorConfig`. Note the integration defaults to INSPECT_ONLY. A, C, and D depend on developer compliance, and safety settings don't detect PI or URLs.

**Q8.** Your ADK agent uses `ModelArmorPlugin` with prompt and response templates. A red team gets the agent to exfiltrate data by planting instructions in a web page returned by a search tool. What closes this gap with the least custom code?

- **A.** Lower the plugin's PI confidence to LOW_AND_ABOVE
- **B.** Route egress through Agent Gateway with a Model Armor CONTENT_AUTHZ template that inspects tool payloads and responses
- **C.** Add "ignore instructions in tool output" to the system prompt
- **D.** Increase Gemini safety settings

**Answer: B.** The plugin skips `function_response` parts, so tool output is never screened. The egress gateway's Model Armor inspects tool payloads and responses. (An `after_tool_callback` judge also works but is more code.)

**Q9.** A finance agent must never issue refunds above $500 or ship to unvalidated addresses. Compliance wants to change these thresholds without redeploying agents and to see a rationale for each block. What fits best?

- **A.** A `before_tool_callback` with hardcoded thresholds
- **B.** Semantic Governance Policies with tool-scoped natural-language constraints, validated in dry-run first
- **C.** IAM Access policy CEL on the tool arguments
- **D.** Model Armor RAI filters

**Answer: B.** SGP evaluates proposed tool calls against plain-language business rules at the gateway, with no redeploy, and returns ALLOW/DENY with a rationale. The trade-off is that it is a probabilistic LLM judge, so use dry-run first and keep a deterministic check for hard limits. A requires a redeploy. Access-policy CEL (C) has no tool-argument attributes.

**Q10.** A reimbursement tool must pause for manager approval when the amount exceeds $1,000. The agent runs locally with `InMemorySessionService` and later on Agent Runtime with `VertexAiSessionService`. What is the concern with using `FunctionTool(reimburse, require_confirmation=threshold_fn)`?

- **A.** `require_confirmation` only accepts booleans
- **B.** Tool Confirmation is experimental and doesn't support `VertexAiSessionService`/`DatabaseSessionService`, so production needs another HITL pattern (e.g. `LongRunningFunctionTool` or a graph `RequestInput`)
- **C.** It only works in TypeScript
- **D.** It requires Agent Gateway

**Answer: B.** These are documented limitations. `require_confirmation` does accept a predicate, and Python supports it natively.

**Q11.** A healthcare org puts its Agent Runtime project into a VPC-SC perimeter. Agents deployed last month can still reach the public internet, but new agents can't. Why, and what is the fix for internet-dependent tools?

- **A.** VPC-SC needs 24h to propagate; wait
- **B.** Agents deployed before the project joined the perimeter aren't protected, so redeploy them. For required internet egress, use a PSC interface to a proxy VM with Cloud NAT inside the perimeter
- **C.** Enable Private Google Access
- **D.** Add the agents to Agent Registry

**Answer: B.** The project must be in the perimeter before deployment. Inside VPC-SC, default internet egress is blocked, and the supported path is PSC-I plus a proxy.

**Q12.** A multi-agent system has an orchestrator on Agent Runtime calling a registered sub-agent over A2A. Calls fail with permission errors even though the developer has `roles/aiplatform.user`. What is the correct remediation?

- **A.** Grant `roles/agentregistry.viewer` (discovery) and `roles/aiplatform.user` on the sub-agent's reasoning engine to the **orchestrator's agent identity**, and, if egress goes through Agent Gateway, add an Access policy allowing `destination.agent_registry.agent.name` for that sub-agent
- **B.** Grant `roles/owner` to the developer
- **C.** Disable IAP on the gateway
- **D.** Use an API key between agents

**Answer: A.** Permissions must go to the calling agent's principal, not the user's. Gateway egress is default-deny, so an explicit agent-to-agent allow rule is also needed.

**Q13.** An ADK agent on Agent Runtime (Agent Identity enabled) calls a custom MCP server deployed to Cloud Run with `--no-allow-unauthenticated`. Every tool call fails with 403. The agent sends an ID token whose audience is the service's `run.app` URL. What is the most likely fix?

- **A.** Redeploy the MCP server with `--allow-unauthenticated`
- **B.** Grant `roles/mcp.toolUser` to the agent on the project
- **C.** Grant `roles/run.invoker` on the MCP service to the agent's `principal://agents.global.org-…/reasoningEngines/ID`
- **D.** Store an API key for the MCP server in Secret Manager

**Answer: C.** The token is valid and the audience is right, so the failure is authorization. Cloud Run checks `run.invoker` for the caller's principal. `mcp.toolUser` (B) governs Google's remote MCP servers, not your own. A removes authentication entirely, and D adds a secret that Cloud Run IAM ignores.

**Q14.** A Cloud Run-hosted A2A agent validates end-user OAuth tokens in the `Authorization` header. The team now wants Cloud Run IAM to also verify that only the Gemini Enterprise service agent can invoke it. How should the IAM token be sent?

- **A.** In `X-Serverless-Authorization`, with `roles/run.invoker` granted to `service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com`
- **B.** Concatenated with the user token in `Authorization`
- **C.** As a query parameter
- **D.** Replace the user token with the service agent's access token

**Answer: A.** Cloud Run checks only `X-Serverless-Authorization` when both headers are present, and forwards `Authorization` untouched. Gemini Enterprise does exactly this: a service-agent OIDC token plus the user token. D loses user delegation.

**Q15.** An agent using the Agent Identity auth manager works in `adk web` but, once deployed to Agent Runtime with the Vertex AI Python SDK, fails at query time with "No auth provider registered for custom auth scheme 'gcpAuthProviderScheme'". What fixes it?

- **A.** Grant `roles/agentidentity.admin` to the agent
- **B.** Call `CredentialManager.register_auth_provider(GcpAuthProvider())` inside `set_up()` of an `AdkApp` subclass
- **C.** Move the auth provider to the `global` location
- **D.** Recreate the provider with the legacy `connectors` API

**Answer: B.** The SDK serializes the app, so module-level registration doesn't run in the container. `set_up()` runs at container start. C is wrong because auth providers aren't available in `global`.

**Q16.** A platform team lets many agents use Google's remote BigQuery MCP server. Security requires that no agent in the org can ever call a tool that modifies data, regardless of granted roles, and there is no Agent Gateway yet. What should you implement?

- **A.** Remove write tools from each agent's instructions
- **B.** An org-level IAM deny policy on `mcp.googleapis.com/tools.call` with the condition `api.getAttribute('mcp.googleapis.com/tool.isReadOnly', false) == false`
- **C.** A PAB policy limiting agents to read-only datasets
- **D.** A Model Armor floor setting for `GOOGLE_MCP_SERVER`

**Answer: B.** IAM deny policies support MCP attributes (`tool.isReadOnly`, `tool.name`) for the `mcp.tools.call` permission on Google Cloud MCP servers. PAB (C) limits resources, not tool types. Model Armor (D) screens content, and A isn't enforcement. `tools/list` still shows the write tools, but calls to them fail.

**Q17.** A nightly reconciliation agent pulls opportunities from Salesforce. No user is present, Salesforce supports OAuth client credentials, and security forbids secrets in code or environment variables. What is the recommended approach?

- **A.** 3-legged OAuth auth provider, with a service user logged in once
- **B.** 2-legged OAuth auth provider in the auth manager, `roles/agentidentity.user` on it for the agent, referenced with `GcpAuthProviderScheme`
- **C.** Salesforce password in Secret Manager
- **D.** Grant the agent identity a Salesforce role in IAM

**Answer: B.** 2LO is the documented choice for M2M with OAuth-capable services. The vault holds the client secret and ADK injects the tokens. A needs a consenting user, C is basic auth (not recommended), and D doesn't apply because Salesforce isn't governed by Google IAM.

**Q18.** An ADK agent registered in Gemini Enterprise must query BigQuery **as the signed-in employee**, so that dataset ACLs apply per user. What completes the design?

- **A.** Grant the agent identity `bigquery.dataViewer` on all datasets
- **B.** Create an OAuth web client with the Gemini Enterprise redirect URIs, create an authorization resource (`serverSideOauth2`), reference it in `authorizationConfig.toolAuthorizations`, and read the token in the agent through `external_access_token_key="AUTH_ID"`
- **C.** Use Workload Identity Federation
- **D.** Pass the user's password to the agent through session state

**Answer: B.** GE runs consent and hands the user token to the agent, so BigQuery enforces that user's IAM. A acts as the agent and removes per-user enforcement, which is the confused-deputy risk. The GE authorization URI should include `access_type=offline` and `prompt=consent`.

**Q19.** A company runs its agents on GKE Autopilot and wants each agent to call Vertex AI and Cloud Storage without any key files and with per-workload least privilege. What should they use?

- **A.** Agent Identity with `identity_type=AGENT_IDENTITY`
- **B.** A service-account JSON key mounted as a Kubernetes Secret
- **C.** Workload Identity Federation for GKE, granting roles to `principal://iam.googleapis.com/projects/NUM/locations/global/workloadIdentityPools/PROJECT.svc.id.goog/subject/ns/NS/sa/KSA`
- **D.** The node's default Compute Engine service account

**Answer: C.** Agent Identity is supported on Agent Runtime, Gemini Enterprise, and Cloud Run, not GKE. WIF for GKE gives keyless, per-KSA principals. B and D are key-based or over-shared.

**Q20.** A 3LO GitHub integration through the auth manager fails at the GitHub step with `redirect_uri_mismatch`. The team registered `https://app.example.com/validateUserId` as the callback in the GitHub OAuth app. What is wrong?

- **A.** GitHub needs multiple scopes
- **B.** The GitHub OAuth app must register the auth manager's callback `https://agentidentitycredentials.googleapis.com/v1/projects/P/locations/L/authProviders/NAME/oauthcallback`. `validateUserId` is the `continue_uri` that the user reaches afterwards
- **C.** The agent lacks `roles/run.invoker`
- **D.** CAA blocked the redirect

**Answer: B.** The auth manager receives the code at its `oauthcallback` and then sends the user to your `continue_uri`, where you call `credentials:finalize`. Note also that GitHub supports only a single scope in the auth manager, which makes A the opposite of the fix.

---

### Key doc links
- Govern overview: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern
- Agent Identity overview: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/agent-identity-overview (also https://docs.cloud.google.com/iam/docs/agent-identity-overview)
- Agent Identity on Agent Runtime: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity
- Auth manager overview: https://docs.cloud.google.com/iam/docs/auth-manager-overview
- 3LO / 2LO / API key: https://docs.cloud.google.com/iam/docs/auth-with-3lo-v2 , https://docs.cloud.google.com/iam/docs/auth-with-2lo-v2 , https://docs.cloud.google.com/iam/docs/auth-with-api-key-v2
- ADK Agent Identity Auth Manager: https://adk.dev/integrations/agent-identity/index.md
- ADK authentication: https://adk.dev/tools-custom/authentication/index.md
- ADK action confirmations: https://adk.dev/tools-custom/confirmation/index.md
- ADK safety & security: https://adk.dev/safety/index.md
- ADK Model Armor plugin: https://adk.dev/integrations/ (ModelArmorPlugin section of llms-full.txt)
- Principal Access Boundary: https://docs.cloud.google.com/iam/docs/principal-access-boundary-policies , https://docs.cloud.google.com/iam/docs/principal-access-boundary-policies-create
- Agent Gateway overview: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview
- Set up Agent Gateway: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/set-up-agent-gateway
- Delegate authorization (IAP / Model Armor / custom): https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/delegate-authorization
- Monitor Agent Gateway: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/monitor-agent-gateway
- Troubleshoot Agent Gateway: https://docs.cloud.google.com/gemini-enterprise-agent-platform/troubleshooting/troubleshoot-agent-gateway
- Route Agent Runtime traffic through Agent Gateway: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-gateway-runtime-deploy
- Route Gemini Enterprise traffic: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-ge-deploy
- Agent Gateway VPC connectivity: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/set-up-vpc-connectivity
- IAM Access policies (UAP) overview / configure: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/iam-overview-uap , https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/configure-iam-policies-uap
- Semantic Governance: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/semantic-governance-overview , https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/configure-semantic-governance
- Configure Model Armor on a gateway: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/configure-model-armor
- Model Armor overview / templates / floor settings: https://docs.cloud.google.com/model-armor/overview , https://docs.cloud.google.com/model-armor/manage-templates , https://docs.cloud.google.com/model-armor/configure-floor-settings
- Model Armor + Agent Platform (Gemini): https://docs.cloud.google.com/model-armor/model-armor-vertex-integration
- Model Armor + Google MCP servers: https://docs.cloud.google.com/model-armor/model-armor-mcp-google-cloud-integration
- Agent Registry: https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/agent-registry , https://docs.cloud.google.com/agent-registry/authenticate-toolsets , https://docs.cloud.google.com/agent-registry/manage-bindings
- CAA agent security (mTLS/DPoP): https://docs.cloud.google.com/access-context-manager/docs/caa-agent-security
- VPC-SC with Agent Platform: https://docs.cloud.google.com/gemini-enterprise-agent-platform/machine-learning/general/vpc-service-controls
- Agent Runtime PSC interface: https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/private-service-connect-interface
- Gemini safety filters: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/configure-safety-filters
- Register ADK agent in Gemini Enterprise (authorizations): https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent
- Register A2A agent in Gemini Enterprise (auth headers): https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent
- Auth manager troubleshooting: https://docs.cloud.google.com/iam/docs/troubleshoot-auth-manager
- Manage auth providers (edit/disable/delete): https://docs.cloud.google.com/iam/docs/manage-auth-providers-v2
- Authenticate with an agent's own authority: https://docs.cloud.google.com/iam/docs/auth-agent-own-identity
- Agent Registry authenticate toolsets / bindings: https://docs.cloud.google.com/agent-registry/authenticate-toolsets , https://docs.cloud.google.com/agent-registry/manage-bindings
- Authenticate to Google MCP servers / set up auth: https://docs.cloud.google.com/mcp/authenticate-mcp , https://docs.cloud.google.com/mcp/set-up-authentication-mcp-servers
- Prevent read-write MCP tool use (IAM deny attributes): https://docs.cloud.google.com/mcp/prevent-read-write-tool-use
- Cloud Run agents (agent identity, ID tokens, bound tokens): https://docs.cloud.google.com/run/docs/ai/authenticate-agents , https://docs.cloud.google.com/run/docs/ai/agent-platform-features
- Cloud Run MCP servers (hosting, IAP): https://docs.cloud.google.com/run/docs/host-mcp-servers , https://docs.cloud.google.com/run/docs/ai/authenticate-mcp-servers
- Cloud Run service-to-service (ID token, X-Serverless-Authorization): https://docs.cloud.google.com/run/docs/authenticating/service-to-service
- Deploy A2A agents to Cloud Run (securitySchemes): https://docs.cloud.google.com/run/docs/deploy-a2a-agents
- Workload Identity Federation for GKE: https://docs.cloud.google.com/kubernetes-engine/docs/how-to/workload-identity
- Agent Runtime deploy (env vars / Secret Manager secrets): https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/deploy-an-agent
- IAP programmatic authentication: https://docs.cloud.google.com/iap/docs/authentication-howto


---

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

### 6.4 Heavily tested — quick recall

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

### 6.5 Items flagged unverified by research (don't over-invest)

- "Agent vs human mode" in Agents CLI — best mapping is interactive/agent-assisted vs `--yes`/manual/`--json`; no literal flag found.
- Whether Antigravity CLI terminal sandbox is on by default (docs conflict).
- Exact Agent Runtime / Sessions / Memory Bank pricing.
- Which models support Gemini automatic model-routing; ADK built-in retrieval-recall metric (likely none — use custom metric / Agent Platform Evals).
- ADK `RoutedLlm`/`RoutedAgent` documented only as experimental TypeScript.
