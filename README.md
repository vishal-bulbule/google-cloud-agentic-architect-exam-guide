# Google Cloud Agentic Architect Exam Guide

A study guide for the **Google Cloud Certified — Professional Agentic Architect** exam, organized around the five sections of the official exam guide.

Every chapter covers its sub-objectives with core concepts, configuration snippets, decision tables, explicit trade-offs, **exam signals** (question keyword → likely answer), common distractors, and scenario-style practice questions with explanations. Architecture infographics sit alongside the text for the topics that are easier to see than to read.

> **Product names follow the 2026 Gemini Enterprise Agent Platform naming** (Agent Engine → Agent Runtime, Vertex AI Search → Agent Search, Vector Search 2.0 → Agent Retrieval, and so on). The full rename map is in [the overview](sections/00-overview.md#02-the-2026-rename-map-memorize--questions-will-use-new-names).

## Start here

| | |
|---|---|
| 📘 **Read it end to end** | [AGENTIC_ARCHITECT_EXAM_GUIDE.md](AGENTIC_ARCHITECT_EXAM_GUIDE.md) — the whole guide in one file |
| 🗺️ **Plan your prep** | [Overview](sections/00-overview.md) — blueprint, where the marks are, 4-week study plan |
| ⚡ **Last 72 hours** | [Cheat sheet](sections/06-cheat-sheet.md) — decision trees, numbers to memorize, traps |

## Chapters

| # | Exam section | Weight | Practice Qs |
|---|---|---|---|
| 0 | [Overview — blueprint, renames, study plan](sections/00-overview.md) | — | — |
| 1 | [Building agents using low-code tools](sections/01-low-code-agents.md) | ~13% | 10 |
| 2 | [Using coding agents for application development](sections/02-coding-agents.md) | ~17% | 10 |
| 3 | [Developing custom agents](sections/03-custom-agents.md) | ~33% | 15 |
| 4 | [Evaluating and deploying agentic workflows](sections/04-evaluate-deploy.md) | ~22% | 12 |
| 5 | [Securing and governing agentic workflows](sections/05-security-governance.md) | ~15% | 12 |
| 6 | [Last-72-hours cheat sheet](sections/06-cheat-sheet.md) | — | — |

## Infographics

Click a thumbnail for the full-size image.

<table>
<tr><td align="center" width="25%"><a href="infographics/00-exam-blueprint.png"><img src="infographics/00-exam-blueprint.png" width="200"></a><br><sub><b>00</b> · Exam blueprint & platform map</sub></td><td align="center" width="25%"><a href="infographics/01-low-code-surfaces.png"><img src="infographics/01-low-code-surfaces.png" width="200"></a><br><sub><b>01</b> · Low-code surfaces</sub></td><td align="center" width="25%"><a href="infographics/02-ge-enterprise-data.png"><img src="infographics/02-ge-enterprise-data.png" width="200"></a><br><sub><b>02</b> · Enterprise data</sub></td><td align="center" width="25%"><a href="infographics/03-coding-agent-primitives.png"><img src="infographics/03-coding-agent-primitives.png" width="200"></a><br><sub><b>03</b> · Customization primitives</sub></td></tr>
<tr><td align="center" width="25%"><a href="infographics/04-sandbox-tiers.png"><img src="infographics/04-sandbox-tiers.png" width="200"></a><br><sub><b>04</b> · Sandbox tiers</sub></td><td align="center" width="25%"><a href="infographics/05-model-selection.png"><img src="infographics/05-model-selection.png" width="200"></a><br><sub><b>05</b> · Model selection</sub></td><td align="center" width="25%"><a href="infographics/06-adk-orchestration-patterns.png"><img src="infographics/06-adk-orchestration-patterns.png" width="200"></a><br><sub><b>06</b> · Orchestration patterns</sub></td><td align="center" width="25%"><a href="infographics/07-state-memory.png"><img src="infographics/07-state-memory.png" width="200"></a><br><sub><b>07</b> · State & memory</sub></td></tr>
<tr><td align="center" width="25%"><a href="infographics/08-rag-pipeline.png"><img src="infographics/08-rag-pipeline.png" width="200"></a><br><sub><b>08</b> · RAG pipeline</sub></td><td align="center" width="25%"><a href="infographics/09-protocols-identity-registry.png"><img src="infographics/09-protocols-identity-registry.png" width="200"></a><br><sub><b>09</b> · MCP · A2A · Registry</sub></td><td align="center" width="25%"><a href="infographics/10-eval-lifecycle.png"><img src="infographics/10-eval-lifecycle.png" width="200"></a><br><sub><b>10</b> · Eval lifecycle</sub></td><td align="center" width="25%"><a href="infographics/11-runtime-selection.png"><img src="infographics/11-runtime-selection.png" width="200"></a><br><sub><b>11</b> · Runtime selection</sub></td></tr>
<tr><td align="center" width="25%"><a href="infographics/12-troubleshooting.png"><img src="infographics/12-troubleshooting.png" width="200"></a><br><sub><b>12</b> · Troubleshooting</sub></td><td align="center" width="25%"><a href="infographics/13-defense-in-depth.png"><img src="infographics/13-defense-in-depth.png" width="200"></a><br><sub><b>13</b> · Defense in depth</sub></td><td align="center" width="25%"><a href="infographics/14-identity-access-controls.png"><img src="infographics/14-identity-access-controls.png" width="200"></a><br><sub><b>14</b> · Identity & access</sub></td><td align="center" width="25%"><a href="infographics/15-guardrail-layers.png"><img src="infographics/15-guardrail-layers.png" width="200"></a><br><sub><b>15</b> · Guardrail layers</sub></td></tr>
</table>

## Watch — related videos

Hands-on walkthroughs from the [TechTrapture YouTube channel](https://youtube.com/@techtrapture), grouped by exam section.

**§1 · Low-code agents & enterprise data**

<table>
<tr><td align="center" width="33%"><a href="https://youtu.be/9Drx6KCNxHs"><img src="https://img.youtube.com/vi/9Drx6KCNxHs/mqdefault.jpg" width="260" alt="Gemini Enterprise Explained — Connect & Deploy ADK Agents"></a><br><sub><b>Gemini Enterprise Explained — Connect & Deploy ADK Agents</b> · 26 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/P9jdr1JztIQ"><img src="https://img.youtube.com/vi/P9jdr1JztIQ/mqdefault.jpg" width="260" alt="Build Enterprise RAG Apps with Agent Search"></a><br><sub><b>Build Enterprise RAG Apps with Agent Search</b> · 49 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/HSJIrCiAmOc"><img src="https://img.youtube.com/vi/HSJIrCiAmOc/mqdefault.jpg" width="260" alt="Custom LLM Chatbot on Your Data with Agent Builder & Dialogflow"></a><br><sub><b>Custom LLM Chatbot on Your Data with Agent Builder & Dialogflow</b> · 25 min</sub></td></tr>
</table>

**§2 · Coding agents & MCP**

<table>
<tr><td align="center" width="33%"><a href="https://youtu.be/oHwWdDhPnzE"><img src="https://img.youtube.com/vi/oHwWdDhPnzE/mqdefault.jpg" width="260" alt="I Explored Google Antigravity"></a><br><sub><b>I Explored Google Antigravity</b> · 19 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/EF6LLGR9k0I"><img src="https://img.youtube.com/vi/EF6LLGR9k0I/mqdefault.jpg" width="260" alt="Build Your First MCP Server — Google Maps & VS Code"></a><br><sub><b>Build Your First MCP Server — Google Maps & VS Code</b> · 14 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/9jRBRNRDRh4"><img src="https://img.youtube.com/vi/9jRBRNRDRh4/mqdefault.jpg" width="260" alt="Build Your First MCP Server — Google Maps & Claude Desktop"></a><br><sub><b>Build Your First MCP Server — Google Maps & Claude Desktop</b> · 10 min</sub></td></tr>
</table>

**§3 · Custom agents with ADK**

<table>
<tr><td align="center" width="33%"><a href="https://youtu.be/F1a9lLySxLI"><img src="https://img.youtube.com/vi/F1a9lLySxLI/mqdefault.jpg" width="260" alt="ADK Session, State & Memory Explained"></a><br><sub><b>ADK Session, State & Memory Explained</b> · 18 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/E9BcLtuuI7U"><img src="https://img.youtube.com/vi/E9BcLtuuI7U/mqdefault.jpg" width="260" alt="ADK Session Management Hands-On (InMemory, Cloud SQL, Vertex AI)"></a><br><sub><b>ADK Session Management Hands-On (InMemory, Cloud SQL, Vertex AI)</b> · 23 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/JNO9xb1p1mQ"><img src="https://img.youtube.com/vi/JNO9xb1p1mQ/mqdefault.jpg" width="260" alt="ADK State Management Explained with Demo"></a><br><sub><b>ADK State Management Explained with Demo</b> · 27 min</sub></td></tr>
<tr><td align="center" width="33%"><a href="https://youtu.be/zTwuzJWQIiI"><img src="https://img.youtube.com/vi/zTwuzJWQIiI/mqdefault.jpg" width="260" alt="Build a GitHub Agent with ADK — Function Tools"></a><br><sub><b>Build a GitHub Agent with ADK — Function Tools</b> · 8 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/Q9OC2e6yGgk"><img src="https://img.youtube.com/vi/Q9OC2e6yGgk/mqdefault.jpg" width="260" alt="Build a BigQuery Agent with ADK — Built-in Tools"></a><br><sub><b>Build a BigQuery Agent with ADK — Built-in Tools</b> · 16 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/jsXHoTAc2A8"><img src="https://img.youtube.com/vi/jsXHoTAc2A8/mqdefault.jpg" width="260" alt="ADK Third-Party Tools — LangChain & CrewAI"></a><br><sub><b>ADK Third-Party Tools — LangChain & CrewAI</b> · 10 min</sub></td></tr>
<tr><td align="center" width="33%"><a href="https://youtu.be/wr0SwuPDT0g"><img src="https://img.youtube.com/vi/wr0SwuPDT0g/mqdefault.jpg" width="260" alt="RAG Explained — Keyword vs Semantic vs Hybrid Search"></a><br><sub><b>RAG Explained — Keyword vs Semantic vs Hybrid Search</b> · 22 min</sub></td></tr>
</table>

**§4 · Deploy & operate agents**

<table>
<tr><td align="center" width="33%"><a href="https://youtu.be/zA0Y3smlavA"><img src="https://img.youtube.com/vi/zA0Y3smlavA/mqdefault.jpg" width="260" alt="Deploy an ADK Agent to Cloud Run"></a><br><sub><b>Deploy an ADK Agent to Cloud Run</b> · 14 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/ZlcJFnVaXD4"><img src="https://img.youtube.com/vi/ZlcJFnVaXD4/mqdefault.jpg" width="260" alt="AI SRE Agent with ADK + MCP — Auto RCA & Log Analysis"></a><br><sub><b>AI SRE Agent with ADK + MCP — Auto RCA & Log Analysis</b> · 36 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/N6uvIPZbM_U"><img src="https://img.youtube.com/vi/N6uvIPZbM_U/mqdefault.jpg" width="260" alt="An AI Agent That Runs My Google Cloud Operations"></a><br><sub><b>An AI Agent That Runs My Google Cloud Operations</b> · 19 min</sub></td></tr>
</table>

**§5 · Guardrails & secure execution**

<table>
<tr><td align="center" width="33%"><a href="https://youtu.be/DyeW_0mcqI4"><img src="https://img.youtube.com/vi/DyeW_0mcqI4/mqdefault.jpg" width="260" alt="ADK Model Callbacks — Before & After LLM Hooks"></a><br><sub><b>ADK Model Callbacks — Before & After LLM Hooks</b> · 9 min</sub></td><td align="center" width="33%"><a href="https://youtu.be/Ee1Y7gwvhy8"><img src="https://img.youtube.com/vi/Ee1Y7gwvhy8/mqdefault.jpg" width="260" alt="ADK Tool Callbacks — Before & After Tool Hooks"></a><br><sub><b>ADK Tool Callbacks — Before & After Tool Hooks</b> · 13 min</sub></td></tr>
</table>

## How this guide was built

- Scope comes straight from the official exam guide's sections and in-scope tool list.
- Content is researched against official Google documentation (docs.cloud.google.com, adk.dev, antigravity.google) and the ADK docs.
- Anything that could not be confirmed in the docs is marked **(unverified)** in the chapters and collected in [§6.4](sections/06-cheat-sheet.md#64-items-flagged-unverified-by-research-dont-over-invest).

> **Not affiliated with Google.** This is an independent study resource. Google Cloud products change quickly — always confirm details against the current [official documentation](https://docs.cloud.google.com/gemini-enterprise-agent-platform/overview) and the exam guide on [Google Cloud certification](https://cloud.google.com/learn/certification).

## Author

**Vishal Bulbule** — Founder @ [TechTrapture](https://www.techtrapture.com)

[YouTube](https://youtube.com/@techtrapture) · [LinkedIn](https://www.linkedin.com/in/vishal-bulbule/) · [Medium](https://vishalbulbule.medium.com/) · [X](https://x.com/vishal__bulbule)
