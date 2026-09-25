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
> [![Continuous evaluation lifecycle](../visual-memory/10-eval-lifecycle.png)](../visual-memory/10-eval-lifecycle.png)

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
> [![Agent Runtime vs Cloud Run vs GKE](../visual-memory/11-runtime-selection.png)](../visual-memory/11-runtime-selection.png)

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
> [![Troubleshooting agents in production](../visual-memory/12-troubleshooting.png)](../visual-memory/12-troubleshooting.png)

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
A. Add `final_response_match_v2` with threshold 0.9 to `test_config.json`.
B. Add expected `tool_uses` to the `.test.json` cases and gate on `tool_trajectory_avg_score` with `match_type: EXACT` at 1.0.
C. Enable an online monitor with the Tool Use Quality metric.
D. Add `response_match_score` at 0.95.
**Answer: B.** Duplicate calls are a trajectory defect; `EXACT` fails on extra calls and is deterministic and cheap for CI. Response metrics (A, D) pass because answers are correct; online monitors (C) find it only after production.

**Q2.** A research agent issues between three and seven web searches in varying order before calling `write_report`. You need a trajectory metric that verifies `search` and `write_report` were both called, without failing on extra searches or order. Which ADK configuration fits?
A. `tool_trajectory_avg_score` with `EXACT`.
B. `tool_trajectory_avg_score` with `IN_ORDER`.
C. `tool_trajectory_avg_score` with `ANY_ORDER`.
D. `response_match_score` at 0.8.
**Answer: C** (B is acceptable only if order `search`→`write_report` must be enforced; the stem says order doesn't matter). `ANY_ORDER` requires all expected calls present, tolerates extras and ordering. `EXACT` fails on variable search counts.

**Q3.** A multi-turn travel-booking agent asks for missing details in different orders depending on how the user phrases requests, so fixed scripted turns keep failing even when the booking succeeds. You want automated evaluation of goal completion. What should you do?
A. Record more golden conversations and use `tool_trajectory_avg_score`.
B. Use ADK user simulation with a `ConversationScenario` (starting prompt, conversation plan, persona) and evaluate with `multi_turn_task_success_v1` and `hallucinations_v1`.
C. Use `final_response_match_v2` on the last turn only.
D. Use `adk conformance test` in replay mode.
**Answer: B.** User simulation generates dynamic user turns; reference-free multi-turn criteria are supported with it. Reference-based criteria (A, C) aren't supported with user simulation, and replay (D) enforces the brittle fixed path.

**Q4.** Your LLM judge metric for "regulatory compliance of responses" frequently disagrees with your compliance SMEs. You must prove the judge is trustworthy before using it as a release gate. What should you do first?
A. Raise `sampling_count` to 32.
B. Switch to BLEU against SME-written references.
C. Build a human-rated dataset (`compliance/human_rating`), run the custom pointwise metric, and compute agreement with `evaluate_autorater`; iterate the rubric or tune a judge model until agreement is acceptable.
D. Replace the judge with `safety_v1`.
**Answer: C.** Autorater calibration against human ratings is the documented way to validate a judge; tuning the judge is the next step. More samples (A) reduce variance but not bias; BLEU (B) and safety (D) measure the wrong thing.

**Q5.** You're comparing a new system prompt against the production prompt on 2,000 historical queries stored in BigQuery, and you want the judge to pick the better response per query while controlling for position bias. Which approach is best?
A. Pointwise `GENERAL_QUALITY` on each variant, compare averages.
B. Pairwise model-based metric in the Gen AI evaluation service with `flip_enabled=True`, dataset loaded from BigQuery.
C. ADK `response_match_score` between the two variants.
D. Online monitors on both revisions.
**Answer: B.** Pairwise metrics are designed for candidate-vs-baseline comparison, and response flipping mitigates position bias; BigQuery is a supported dataset source. A is possible but less sensitive; C isn't a quality judgment; D requires production exposure.

**Q6.** An ADK agent deployed on Agent Runtime with default settings shows median latency of 4 s but max latency of 60 s during traffic bursts of ~300 concurrent requests. CPU usage per container is low. What should you change first?
A. Increase `resource_limits` to 8 CPU / 32 Gi.
B. Increase `container_concurrency` to a multiple of 9 (e.g., 36) and raise `min_instances`.
C. Switch the model to Flash-Lite.
D. Increase `max_instances` to 1000.
**Answer: B.** Default concurrency (9) assumes sync code; async ADK agents are underused, so requests queue while it scales out. Google's guidance is multiples of 9 plus enough `min_instances` for baseline load. Low CPU rules out A; D doesn't fix scale-out lag.

**Q7.** A regulated bank needs an ADK agent that calls an internal pricing API reachable only on a private RFC 1918 address in a Shared VPC. The agent must run with minimal ops overhead inside a VPC Service Controls perimeter. Which design fits?
A. Agent Runtime with a Serverless VPC Access connector.
B. Agent Runtime with a Private Service Connect interface (network attachment in the service project, ≥ /28 subnet), DNS peering to the private zone, and the Agent Platform Service Agent granted network permissions on the host project.
C. Cloud Run with public ingress and an API key.
D. Agent Runtime with `max_instances=1000`.
**Answer: B.** Agent Runtime reaches private networks through PSC-I with DNS peering; Shared VPC needs the service agent's network roles. Connectors (A) are for Cloud Run; C violates the requirement; with VPC-SC/PSC-I `max_instances` is capped at 100 (D is also invalid).

**Q8.** After moving an ADK agent from local testing to Cloud Run with `adk deploy cloud_run --project P --region R ./agent`, users report the agent forgets earlier turns intermittently, especially after quiet periods. What's the most likely cause and fix?
A. Context window overflow; enable context caching.
B. Sessions are held in the in-memory session service; redeploy with `--session_service_uri` pointing at Agent Platform Sessions (`agentengine://...`) or a Cloud SQL database URL.
C. Session affinity is disabled; enable it.
D. `max_llm_calls` is too low.
**Answer: B.** Without `--session_service_uri`, the container uses in-memory sessions that are lost on scale-in and not shared across instances. Session affinity (C) is best-effort and doesn't survive instance termination.

**Q9.** A LoopAgent with a critic and a reviser runs until `max_iterations=10` every time, doubling cost, even for documents the critic considers finished. What's the correct fix?
A. Lower `max_llm_calls` to 50.
B. Give the critic (or reviser) an `exit_loop` tool that sets `tool_context.actions.escalate = True`, and instruct it to call the tool when no further changes are needed.
C. Convert the LoopAgent to a ParallelAgent.
D. Increase the temperature of the critic.
**Answer: B.** LoopAgent stops at `max_iterations` or when a sub-agent emits an event with `escalate=True`; the loop has no exit signal. A only caps the damage; C changes semantics.

**Q10.** Your SRE team needs to find which of 12 tools causes a p95 latency regression in a production agent on Agent Runtime and wants per-tool trends alertable in Cloud Monitoring. What should you use?
A. Search Cloud Logging for "tool" and eyeball timestamps.
B. Enable `GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true`, use the Observability Tools view / `gen_ai.execute_tool.duration` by `gen_ai.tool.name`, and inspect `execute_tool` spans in Cloud Trace for slow traces.
C. Run `adk eval` with `tool_trajectory_avg_score`.
D. Enable DEBUG logging in production.
**Answer: B.** The ADK/OTel tool-duration metric and trace spans attribute latency per tool and can be alerted on. Eval (C) checks correctness, not latency; DEBUG (D) is noisy and risks PII.

**Q11.** Product leadership wants weekly SQL reports on cost per user, tool error rates by MCP server, and human-approval (HITL) turnaround time across all agents, joinable with Cloud Trace. What is the most direct solution?
A. Export Cloud Monitoring metrics to CSV.
B. Add `BigQueryAgentAnalyticsPlugin` to the ADK `App`, writing to an `agent_events` table, and query its event views (`v_llm_response`, `v_tool_error`, HITL events) joined on `trace_id`.
C. Build a custom Pub/Sub logger in each tool.
D. Use online monitors.
**Answer: B.** The plugin captures token usage, tool provenance (including MCP), HITL events, and trace IDs into partitioned BigQuery tables with ready views. C reinvents it; D measures quality, not usage analytics.

**Q12.** Four weeks after launch, a RAG support agent's answers are rated worse by customers, but error rates and latency are flat. Offline evalsets still pass. What's the best next step?
A. Roll back to the previous revision.
B. Configure online monitors on production traces with hallucination and response-quality metrics, alert on the resulting Cloud Monitoring metrics, and add low-scoring production conversations to the golden evalset for regression.
C. Raise `min_instances`.
D. Switch to `trajectory_exact_match` in CI.
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
