# Recommendation: A Browser-Based Coding Agent We Can Sell

**Date:** 2026-11-08
**Builds on:** `01-open-source-coding-agent.md`, `02-commercial-licensing.md`
**Use case:** a paid product that people use in a web browser. The agent reads and edits code, runs commands and shows results, like a hosted Claude Code.

---

## 1. The recommendation in one table

| Layer | Choice | Licence | Why this one |
|---|---|---|---|
| Browser UI | **Our own web app** (React or Svelte) with **Monaco** editor and **xterm.js** terminal | MIT | We own the brand and UX; both libraries are the standard for browser IDEs |
| Agent engine | **Fork of OpenCode**, run as `opencode serve` inside each user's sandbox, renamed | MIT | Already built as a server with an OpenAPI interface that a browser can talk to; works with any OpenAI-compatible model |
| Code sandbox | **E2B** (Firecracker microVMs). Use their managed cloud for the MVP, then self-host | Apache-2.0 (SDK and infra) | VM-level isolation for running untrusted code, and we can self-host it without licence fees |
| Model server | **vLLM** | Apache-2.0 | Standard high-throughput server with an OpenAI-compatible API |
| Default model | **Qwen3-Coder-30B-A3B-Instruct** | Apache-2.0 | Only 3B parameters are active per token, so one GPU can serve many users at once |
| Premium model | **Qwen3-Coder-Next** (80B total / 3B active) | Apache-2.0 | ~70.6% SWE-bench Verified. Same family and tool-call format, so one fine-tuning pipeline covers both |
| GPU hosting | **Serverless GPUs with scale-to-zero** (Modal or RunPod) at launch; **dedicated GPUs** once busy | — | Pay nothing while idle at launch; switch when utilisation passes ~40% |
| Weights and data | **Hugging Face Hub** (private repos for our fine-tunes) | — | Standard and cheap; keep a mirror |
| Fine-tuning | **Unsloth** LoRA on SWE-smith, SWE-Gym, SWE-rebench and NVIDIA SWE sets | Apache-2.0 / MIT / CC BY 4.0 | All allow commercial use (see report 02) |

This answers open question 1: **fork OpenCode (TypeScript)**.

---

## 2. Architecture

```
 Browser (our web app)
   chat panel · file tree · Monaco editor · xterm terminal · live preview
        │  HTTPS / WebSocket
        ▼
 Control plane (our backend)
   accounts · billing · sessions · usage limits · routes each user to a sandbox
        │
        ├──────────────► Sandbox per session (E2B Firecracker microVM)
        │                  ├─ our OpenCode fork running as an HTTP server
        │                  ├─ user's repo checkout, shell, test runner
        │                  └─ preview port exposed to the browser
        │                          │ OpenAI-compatible API calls
        ▼                          ▼
 Model gateway (rate limits, logging, routing per plan)
        │
        ▼
 vLLM on GPU: Qwen3-Coder-30B-A3B (default) / Qwen3-Coder-Next (premium)
```

How a request flows:

1. The user types a task in the browser.
2. The control plane starts or resumes their sandbox.
3. The OpenCode server in the sandbox runs the agent loop: it calls the model, reads and edits files, and runs commands. All of this happens inside the microVM.
4. Results stream back to the browser over a WebSocket.

The model never runs in the browser, and user code never runs on our main servers.

---

## 3. Why these choices, and what we ruled out

### Agent engine: OpenCode over OpenHands

- **OpenHands** is browser-first and its core is MIT. But the features a paid multi-user product needs (multi-tenancy, SSO, RBAC, budgets) sit in its `enterprise/` folder under a **Polyform Free Trial licence**, which needs a paid licence after about a month. We would end up rebuilding those parts anyway, on a heavier Python codebase.
- **OpenCode** splits the agent server from its interface. `opencode serve` exposes an OpenAPI HTTP API, so our own browser UI can drive it. Two things to change in our fork:
  - by default its web UI is served by proxying to `app.opencode.ai`, so we must ship our own UI;
  - "OpenCode" is their trademark, so we rename the product.
- **Writing our own engine** from scratch is still possible later. Forking gets us to a working product months sooner.

### Running code: server-side sandbox, not in-browser

| Option | Verdict |
|---|---|
| **WebContainers** (StackBlitz; used by bolt.new and bolt.diy) | ❌ A **paid commercial licence is required for production use in a for-profit setting**, priced on request. Also Node.js only. bolt.diy itself is MIT, but it depends on WebContainers. |
| **Pyodide** (Python in WebAssembly) | ⚠️ MPL-2.0, outside our approved licences. Python only, with no real shell or git. |
| **Daytona** | ❌ Was AGPL-3.0. 2026 reports say it went closed source and archived its open repo. |
| **E2B** | ✅ Apache-2.0 SDK and infrastructure, Firecracker VM isolation, full Linux (any language, git, package managers). Managed cloud to start, self-host later via Terraform. Self-hosting takes real ops work. |

### Model: server GPU, not in the browser

- **WebLLM** (Apache-2.0) can run small models in the browser over WebGPU. But the models that fit in a browser tab (≤ ~8B) are much weaker at multi-step tool use. Model files are several GB to download, and WebGPU support still varies across Safari and Firefox.
- Keep it as a possible **"offline lite" feature** later, not as the core product.

### Why the Qwen3-Coder family

- Apache-2.0 with no revenue caps (unlike Devstral 2 or Kimi K2).
- With a mixture-of-experts design, only ~3B parameters are active per token. That gives high throughput per GPU, which keeps hosting cost per user down.
- One family across tiers means one tool-call format and one fine-tuning pipeline.

---

## 4. "Free forever": what changes for a browser product

A browser product moves the compute from the user's machine to ours, so hosting can't be free. What *can* be free forever is the **software**: every component above is MIT or Apache-2.0, with no licence fees or revenue caps. Running costs scale with paying users.

Ways to keep costs and lock-in low:

- **Scale-to-zero GPUs at launch.** Serverless H100s cost roughly **$4/hour while running** (Modal ~$3.95, RunPod ~$4.18–4.55) and nothing while idle. L40S cards cost roughly $1.75–1.95/hour.
- **Move to dedicated GPUs** (e.g. RunPod on-demand H100 from ~$1.99/hour) once a GPU is busy more than ~40% of the time.
- **Bring-your-own-model option.** Users point the product at their own Ollama, vLLM or API key. This works for a free or cheap tier, and it costs us no GPU time.
- **No single-vendor dependency.** vLLM, E2B and Hugging Face can all run on any cloud, so we can move providers when prices change.

Rough planning numbers (check current prices before budgeting):

| Item | Launch (serverless) | Steady state (dedicated) |
|---|---|---|
| 1× H100 for the default model | Pay per second used | ~$1,450–2,900/month always on |
| Sandboxes | E2B cloud, usage-based | Self-hosted on VM/bare-metal hosts with nested virtualisation |
| Weights/datasets storage | Hugging Face (free public, low-cost private) | Same + mirror |

---

## 5. Build plan

| Phase | What we build | Outcome |
|---|---|---|
| **1. Prototype** | Fork OpenCode; run `opencode serve` in an E2B cloud sandbox; minimal web UI (chat + file tree + terminal); Qwen3-Coder-30B-A3B on Modal vLLM | One user can complete a coding task in the browser |
| **2. MVP** | Accounts, sessions, GitHub repo import, live preview, model gateway with per-user limits, billing | First paying users |
| **3. Quality** | Eval suite (SWE-bench Verified subset run through our engine); LoRA fine-tune of Qwen3-Coder on our tool format | Measurable gain over the stock model |
| **4. Scale and cost** | Self-host E2B; dedicated GPUs; premium tier on Qwen3-Coder-Next | Lower cost per user; higher-quality paid tier |
| **5. Optional** | Bring-your-own-model; WebLLM offline lite mode; agentic RL (SkyRL/verl) | Extra tiers and features |

---

## 6. Risks to manage

1. **Running untrusted code.** Use microVM isolation (E2B/Firecracker), never plain Docker on shared hosts. Restrict network egress per sandbox and add abuse monitoring (crypto mining, spam).
2. **Upstream drift.** Track OpenCode releases and keep our changes in a separate layer to make merging easier.
3. **GPU cost spikes.** The model gateway must enforce per-user token and time budgets from day one.
4. **Model quality gap.** Open models still trail top closed models on hard tasks. Fine-tuning and a strong eval loop are how we close it.
5. **Licence hygiene.** Follow report 02: notices file, pinned revisions, licence check in CI, rename the fork.

---

## Sources

- OpenHands licensing and GUI: [OpenHands docs](https://docs.openhands.dev/overview/introduction), [OpenHands self-hosted blog](https://www.openhands.dev/blog/openhands-cloud-self-hosted-secure-convenient-deployment-of-ai-software-development-agents), [rywalker.com research note](https://rywalker.com/research/openhands)
- OpenCode server and web mode: [opencode.ai server docs](https://opencode.ai/docs/server/), [opencode.ai web docs](https://opencode.ai/docs/cli/web), [OpenCode issue #11616](https://github.com/anomalyco/opencode/issues/11616)
- WebContainers licensing: [webcontainers.io/enterprise](https://webcontainers.io/enterprise), [EveryDev listing](https://www.everydev.ai/tools/webcontainer-api)
- Sandboxes: [Morph – E2B vs Daytona](https://www.morphllm.com/comparisons/e2b-vs-daytona), [Helix – self-hosted agent sandboxes](https://helix.ml/blog/self-hosted-agent-sandboxes), [Beam – 2026 sandbox guide](https://www.beam.cloud/blog/2026-sandbox-guide), [E2B licence](https://github.com/e2b-dev/E2B), [E2B infra licence](https://github.com/e2b-dev/infra)
- In-browser models: [WebLLM paper](https://arxiv.org/html/2412.15803v2), [Qwen3 in the browser with WebLLM](https://dev.to/creeta/qwen3-in-the-browser-zero-keys-webllm-0283-hands-on-3ai2), [Cross-browser WebGPU notes](https://jwatte.com/blog/in-browser-llm-byo-model-cross-browser/)
- GPU pricing: [Prem AI – RunPod vs Modal vs Lambda](https://blog.premai.io/serverless-llm-deployment-runpod-vs-modal-vs-lambda-2026), [RunPod pricing](https://www.runpod.io/gpu-cloud/pricing), [Koyeb – serverless GPU platforms](https://www.koyeb.com/blog/best-serverless-gpu-platforms-for-ai-apps-and-inference-in-2026), [Findstack – vLLM GPU hosting](https://findstack.com/resources/vllm-gpu-hosting)
- Licences verified from LICENSE files on 2026-11-08: E2B, e2b-dev/infra, web-llm, vLLM, transformers.js (Apache-2.0); bolt.diy (MIT); Pyodide (MPL-2.0).

> Prices and some licence reports (Daytona, OpenHands enterprise) come from third-party sources. Re-check before committing money or code.
