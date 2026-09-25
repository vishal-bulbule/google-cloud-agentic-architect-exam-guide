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
> [![Sandbox isolation tiers](../visual-memory/04-sandbox-tiers.png)](../visual-memory/04-sandbox-tiers.png)

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
> [![Coding-agent customization primitives](../visual-memory/03-coding-agent-primitives.png)](../visual-memory/03-coding-agent-primitives.png)

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
> [![Antigravity customization map](../visual-memory/16-antigravity-customization.png)](../visual-memory/16-antigravity-customization.png)

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
A. `headers: {"Authorization": "Bearer <SA key token>"}` in `~/.gemini/config/mcp_config.json`
B. `serverUrl` with `authProviderType: "google_credentials"` in a committed `.agents/mcp_config.json`, with developers using `gcloud auth application-default login`
C. `url` with an OAuth client ID and secret per server
D. A shared service account key referenced through `env` in a stdio MCP wrapper
**Answer: B.** ADC reuses each developer's identity and IAM with no stored secret, and the workspace file distributes the config. A and D put long-lived credentials on disk and collapse everyone onto one identity. C uses the unsupported `url` field, and OAuth client secrets are unnecessary for Google Cloud servers that accept ADC.

**Q2.** A fintech runs an ADK agent that includes a "code interpreter" tool executing Python the LLM writes. The tool must start in under a second per session, must not reach the Kubernetes API, and must isolate the host kernel. What is the best design?
A. Cloud Run jobs with the default compute service account
B. GKE Agent Sandbox: a gVisor `SandboxTemplate` with `automountServiceAccountToken: false`, a `SandboxWarmPool`, and a `SandboxClaim` per session
C. GKE Standard pods with `privileged: true` inside a dedicated namespace
D. Cloud Workstations with Turbo preset
**Answer: B.** gVisor provides kernel-level isolation, warm pools deliver sub-second claims, and the admission policy *requires* no SA token. A lacks a warm, sandboxed claim model and uses an over-privileged default SA. C violates the prohibited-config list. D is a developer environment, not a multi-tenant execution runtime.

**Q3.** A team added `.agents/rules/frontend/react.md` with `trigger: alwaysOn`. The agent ignores it. What are the two root causes?
A. The rule exceeds 12,000 characters, and rules need a `name` field
B. The file is nested in a subdirectory (flat scan) and `alwaysOn` is an invalid trigger (it must be `always_on`)
C. Rules only load from `~/.gemini/config/rules/`, and require `/rules reload`
D. React rules must be skills
**Answer: B.** `.agents/rules/` is scanned flat unless the file is registered in `.agents/rules.json`, and an invalid camelCase trigger is silently discarded. The 12,000-character limit belongs to legacy workflows, and rules don't need `name`. Workspace rules are fully supported.

**Q4.** Compliance requires that no agent in the repo can run `terraform apply` or `gcloud … delete`, regardless of prompts or model behavior, and that every blocked attempt is logged to an internal endpoint. What should you use?
A. An `AGENTS.md` rule stating "never run terraform apply"
B. A skill named `safe-infra` describing the approved process
C. A `PreToolUse` hook on `run_command` in `.agents/hooks.json` that logs and returns `{"decision":"deny"}` for those patterns (optionally with matching `deny` permission rules)
D. A subagent with `model: pro`
**Answer: C.** Hooks and permission denies are deterministic and can run logging code, while rules and skills are advisory prompt content. A model tier doesn't enforce anything.

**Q5.** 300 engineers will use Claude Code through Agent Platform. Finance wants predictable per-token cost and platform wants controlled upgrades. Some models aren't served on the `global` endpoint. Which configuration fits best?
A. `CLAUDE_CODE_USE_VERTEX=1`, `CLOUD_ML_REGION=global`, `ANTHROPIC_VERTEX_PROJECT_ID`, pinned `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL`, and `VERTEX_REGION_CLAUDE_<MODEL>` for regional-only models
B. Use the `opus` alias so users always get the newest model
C. An Anthropic API key per developer with a spending cap
D. `CLOUD_ML_REGION=us-east5` and grant `roles/aiplatform.admin`
**Answer: A.** Pinning controls both cost and upgrade timing, `global` improves availability, and per-model region overrides cover the gaps. B drifts cost and model. C leaves GCP governance and billing. D is over-privileged (`aiplatform.user` suffices) and forgoes global availability.

**Q6.** A regulated bank wants developers to use AI coding agents, but source code must never leave its perimeter, direct internet egress is forbidden except for an approved package mirror, and SSH to VMs must be auditable through IAM. What should you choose?
A. Antigravity 2.0 on laptops with the Default preset
B. Cloud Workstations: private cluster with PSC, a VPC-SC perimeter (restricting both the Workstations and Compute Engine APIs), public IPs disabled, Secure Web Proxy allowlisting the mirror, and `--disable-ssh-to-vm`
C. GKE Agent Sandbox for each developer
D. Cloud Shell with Gemini CLI
**Answer: B.** Each listed control maps to a documented Workstations security practice. A keeps code on endpoints outside the perimeter. C is a code-execution runtime, not a developer IDE platform with gateway SSH controls. D offers no perimeter or egress control.

**Q7.** Your team's coding agent must build a new ADK agent, create an evalset, deploy it to Cloud Run with CI/CD and Terraform, and make it available in Gemini Enterprise. What should you do with the least custom tooling?
A. Write a custom MCP server that wraps `gcloud run deploy`
B. Install Agents CLI (`uvx google-agents-cli setup`) and let the agent drive `create`, `eval run`, `scaffold enhance --deployment-target cloud_run`, `infra setup-cicd`, `deploy`, and `publish gemini-enterprise`
C. Use `adk create` and deploy manually with a Dockerfile
D. Use `agents-cli create --prototype` and deploy to production
**Answer: B.** Agents CLI skills and commands cover the full lifecycle, including publishing. A reinvents existing tooling. C lacks evals, CI/CD and publishing. D's `--prototype` omits CI/CD and Terraform.

**Q8.** An org has 400 internal skills. Loading them all into every agent's context is too expensive, and security wants immutable, versioned snapshots with central management. ADK agents run on Agent Runtime. What should you use?
A. Put all skills in `AGENTS.md`
B. Skill Registry with ADK `SkillToolset(registry=GCPSkillRegistry(...))`, so agents call `search_skills` and `load_skill` on demand, backed by immutable skill revisions
C. Bake all skills into the container image under `skills/`
D. One MCP server per skill
**Answer: B.** The registry provides on-demand discovery with progressive loading, and revisions are immutable snapshots. A loads everything on every turn. C has no central governance and requires a redeploy per change. D misuses MCP for procedural knowledge and multiplies operational overhead.

**Q9.** A nightly CI job runs `agy -p "fix lint errors and run tests"`. It exits 0, but no tests ran and stderr mentions a tool being denied. What is the best fix?
A. Add `--dangerously-skip-permissions`
B. Add scoped rules such as `command(regex:npm run (lint|test))` under `permissions.allow` in `~/.gemini/antigravity-cli/settings.json` for the CI runner
C. Switch to Turbo preset
D. Increase `--print-timeout`
**Answer: B.** In headless mode, unapproved tools are soft-denied and the run still exits 0, so pre-granting the exact commands is the least-privilege fix. A and C remove all guardrails. D doesn't address the denial.

**Q10.** Developers on Cloud Workstations run Claude Code and Antigravity CLI for multi-hour refactors, often pausing for review. Workstations keep timing out and losing agent state, and finance objects to raising idle timeouts. What should you do?
A. Set the idle timeout to 24 hours
B. Configure `IdleAction.SUSPEND` and install the sample keep-alive hooks (for example Claude Code `UserPromptSubmit` to start and `Notification` to stop `/google/scripts/keep_alive.sh`)
C. Move developers to GKE Agent Sandbox
D. Run agents with `nohup` on the workstation
**Answer: B.** Suspend preserves RAM and agent context while stopping compute billing, and the hooks keep the VM alive only during active work. A burns compute while idle. C is the wrong tool for interactive dev environments. D doesn't prevent the idle shutdown.

**Q11.** A team is moving from Gemini CLI to Antigravity CLI. Their repo has ten custom skills in `.gemini/skills/`, and their `GEMINI.md` files are at the repo root and in several service folders. After switching, the rules still apply but none of the skills appear as slash commands. What should they do?
A. Run `agy plugin import gemini` to convert the skills
B. Move `.gemini/skills/` to `.agents/skills/` in the repo. The `GEMINI.md` files need no change
C. Copy the skills into `~/.gemini/config/rules/` so they load globally
D. Rename every `SKILL.md` to `AGENTS.md`
**Answer: B.** The migration guide says workspace skills must be moved by hand from `.gemini/skills/` to `.agents/skills/`, while `GEMINI.md`/`AGENTS.md` context files work unchanged. A converts *extensions* into plugins, not a repo's skill folder. C turns procedures into rules (and rule files need `trigger` frontmatter). D turns on-demand skills into always-on context.

**Q12.** A platform team wants protobuf conventions (never reuse a field number, always mark deleted fields `reserved`) applied whenever the agent edits `.proto` or generated `.pb.go` files. The conventions must not consume context in other tasks. Which rule file is correct?
A. `.agents/rules/proto.md` with `trigger: glob` and `globs: "*.proto, **/*.pb.go"`
B. `.agents/rules/proto.md` with `trigger: always_on`
C. `.agents/rules/proto/conventions.md` with `trigger: glob` and `globs: *.proto`
D. `AGENTS.md` at the repo root with a `globs:` frontmatter block
**Answer: A.** A `glob` rule activates only when the agent touches matching files, and the quoted, comma-separated `globs` string is the documented format. B costs tokens on every turn. C is nested (ignored without `rules.json`) and has an unquoted `*` that YAML parses as an alias. D is wrong because `AGENTS.md` takes no frontmatter and is always on.

**Q13.** Internal audit has a 15-page security review rubric. Auditors want the agent to use it only when they explicitly ask for an audit, and it must never load automatically, even if a task looks security-related. What should you configure?
A. A rule with `trigger: model_decision` and a description mentioning security audits
B. A rule with `trigger: manual`, which auditors pull in by `@`-mentioning it in chat
C. A `PreInvocation` hook that injects the rubric as an `ephemeralMessage`
D. Add the rubric to the global `~/.gemini/GEMINI.md`
**Answer: B.** `manual` rules are never loaded automatically, only on an explicit `@` mention, and the docs cite audit rubrics as the use case. A lets the model decide to load it. C injects it every turn. D makes it always on across every project and eats into the 20k-token budget.

**Q14.** Agents in a repo often stop and report "done" while unit tests are failing. There is already an `AGENTS.md` rule saying "always run tests before finishing". The team wants a deterministic gate that sends the agent back to work with the failure output, with minimal extra machinery. What should you add?
A. A `PostToolUse` hook on `write_to_file` that returns `{"decision":"deny"}` when tests fail
B. A `Stop` hook in `.agents/hooks.json` that runs the tests and, on failure, returns `{"decision":"continue","reason":"<failures>"}`, with a retry cap
C. Change the rule to `trigger: always_on` with stronger wording
D. A `PreToolUse` hook on `run_command` that returns `force_ask`
**Answer: B.** A `Stop` hook with `decision: "continue"` re-enters the loop and injects the reason as a system message, which makes it a deterministic definition-of-done gate. A fails because `PostToolUse` only returns `{}` and cannot block. C is still advisory. D only adds prompts and never checks the test result.

**Q15.** You are porting a Gemini CLI `BeforeTool` hook (defined in `.gemini/settings.json`, `"timeout": 5000`, blocks by exiting with code 2) to Antigravity. Which set of changes is correct?
A. Keep the file and event name. Antigravity reads Gemini CLI `settings.json` hooks
B. Move it to `.agents/hooks.json` as `PreToolUse` under a named hook, set `"timeout": 5` (seconds), and block by printing `{"decision":"deny","reason":"…"}` to stdout
C. Move it to `.agents/rules/hooks.md` with `trigger: always_on`
D. Move it to `.agents/hooks.json` as `PreInvocation` with a `matcher` and keep `"timeout": 5000`
**Answer: B.** Antigravity uses `hooks.json`, the `PreToolUse` event with a regex tool matcher, a timeout in seconds (default 30), and a JSON `decision` for blocking. A is wrong because Gemini CLI hook config is not a documented Antigravity location. C turns enforcement into advisory text. D ignores the matcher on lifecycle events, and 5000 would mean 5000 seconds.

**Q16.** A central platform team must roll out the same four skills, two `glob` rules, a secrets-blocking hook, and a read-only BigQuery MCP server to 300 developers using Antigravity 2.0 and the CLI across 150 repos. Updates must ship as one versioned unit. What is the best approach?
A. Publish the skills to Skill Registry and ask developers to copy the rest by hand
B. Package everything as a plugin (`plugin.json` plus `skills/`, `rules/`, `hooks.json`, `mcp_config.json`) and install it globally (`~/.gemini/config/plugins/`, or `agy plugin install <git-url>` for the CLI)
C. Add a `.agents/rules.json` with `inherits` pointing at a shared repo
D. Put all the content into one large `AGENTS.md` in every repo
**Answer: B.** Plugins are the distribution unit for skills, rules, hooks, MCP servers and agents. A misuses Skill Registry, which serves ADK and runtime agents rather than IDE customization, and it leaves most of the bundle manual. C shares only rules. D can't carry hooks or MCP config and bloats the always-on budget.

**Q17.** You need a security reviewer that can only read code and search, runs in its own context so it doesn't pollute the main conversation, uses the stronger model tier, and runs any shell commands only in the sandbox. The main agent should delegate to it automatically. What should you create?
A. A skill `.agents/skills/security-review/SKILL.md` describing the review steps
B. A subagent `.agents/agents/security-reviewer.md` with `description`, `tools: [view_file, grep_search]`, `model: pro`, `commandExecutionPolicy: sandbox`, and `subagent: true`
C. A rule with `trigger: model_decision` about security reviews
D. A plugin containing only `plugin.json`
**Answer: B.** Only a subagent provides context isolation plus a tool allowlist, a model tier and an execution policy, and the planner delegates to it based on `description`. A and C run inside the main agent's context with its full toolset. D carries no behavior. Spell tool names exactly, because a misspelled tool can hang the subagent.

**Q18.** A team has 25 legacy workflows in `.agents/workflows/` and `~/.gemini/config/workflows/`, several of them near the 12,000-character limit, and they want them to keep working after the retirement date. Some workflow names match skills that already exist. What should they do?
A. Nothing. Workflows remain supported indefinitely
B. Run `/migrate-workflows` in Antigravity 2.0 to scaffold `.agents/skills/<name>/SKILL.md` for each one (originals are renamed `.bak`), then move embedded scripts into `scripts/`. Where names collide, the existing skill already takes precedence
C. Convert each workflow to an `always_on` rule
D. Split each workflow into two files under 6,000 characters
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
