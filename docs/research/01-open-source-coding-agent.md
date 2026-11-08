# Building an Open-Source Terminal Coding Agent

**Research date:** 2026-11-08
**Goal:** Build our own version of a Claude Code–style terminal coding agent using open-source models and tooling, where every dependency stays free to use permanently and the hosting is sustainable.

---

## 1. Summary of recommendations

| Layer | Recommendation | License | Why |
|---|---|---|---|
| Agent harness (the CLI) | Fork **OpenCode**, or build a small harness modelled on **mini-swe-agent** | MIT | OpenCode is the most complete open terminal agent and supports any OpenAI-compatible backend |
| Default model (laptop / single GPU) | **Devstral Small 2 (24B)** or **Qwen3-Coder-30B-A3B** | Apache-2.0 | Fits a 24–32 GB machine when quantized; good tool-calling |
| Stronger model (workstation / server) | **Qwen3-Coder-Next (80B total, 3B active MoE)** | Apache-2.0 | ~70.6% SWE-bench Verified (model card), cheap to run because only 3B params are active |
| Fine-tuning data | **SWE-smith**, **CoderForge-Preview**, **NVIDIA SWE-ZERO / Open-SWE-Traces** | MIT / CC BY 4.0, permissively licensed repos only | Test-verified agent trajectories, legally clean to train on |
| Fine-tuning tools | **Unsloth** (LoRA/SFT, small-scale GRPO) → **SkyRL / verl** for full agentic RL later | Apache-2.0 | Unsloth fits free 16 GB GPUs; SkyRL targets SWE-bench-style RL but needs 8×H100-class hardware |
| Code hosting | **GitHub** (this repo) | — | Free for public and private repos |
| Model + dataset hosting | **Hugging Face Hub** | — | Free public storage of weights and datasets, standard for the ecosystem |
| Inference | **Local first** (Ollama / llama.cpp / vLLM on our own hardware); free cloud APIs as optional fallbacks | MIT / Apache-2.0 | The only option that is free forever; free API tiers change without notice |
| Training compute | **Kaggle** (~30 GPU-hrs/week T4/P100) for small LoRA runs; rented GPUs for anything 24B+ | — | Free tiers are fine for experiments but cannot train frontier-scale models |

**Licensing rule for the whole project:** only take dependencies under MIT, Apache-2.0, BSD or CC BY. Avoid "modified MIT" and custom model licenses (e.g. Kimi's) unless we read the terms first.

---

## 2. What "replicating Claude Code" means in practice

A terminal coding agent has three separable parts:

1. **The harness** – the CLI: chat loop, tool definitions (read/write/edit files, run shell, search), permission prompts, context management, MCP support.
2. **The model** – an LLM that is good at multi-turn tool calling on code.
3. **The serving layer** – where the model runs and how the harness talks to it (almost always an OpenAI-compatible HTTP API).

Training a base model from scratch is out of reach on free compute (frontier models take thousands of GPUs for months). The sustainable approach is:

- take an **existing permissively licensed open-weight model**,
- optionally **fine-tune it** (LoRA) on open agent trajectories to specialise it for our harness and tool format,
- and put our effort into the **harness**, which is where product differentiation lives.

### Legal guardrails

- **Claude Code itself is proprietary.** We must not copy its source code, prompts or assets. We can implement similar *features* from scratch or on top of MIT/Apache projects.
- **Do not train on Claude (or other closed-model) outputs.** Anthropic's and OpenAI's terms prohibit using outputs to build competing models. Use only the open datasets listed in §5.
- **Do not use the "Claude" name or branding** for our product.

---

## 3. Agent harness options

Licenses below were checked against each project's LICENSE file on 2026-11-08.

| Project | Language | License | Notes |
|---|---|---|---|
| **OpenCode** (`sst/opencode`) | TypeScript/Go | MIT | Closest feature match to Claude Code; ~160K+ stars; any provider via OpenAI-compatible API. **Best fork candidate.** |
| **OpenHands** | Python | MIT (core) | Full autonomous agent platform with sandboxing; heavier. Check its `enterprise/` folder, which may be licensed separately. |
| **Goose** (`block/goose`) | Rust | Apache-2.0 | CLI + desktop app, strong MCP support. |
| **Codex CLI** (`openai/codex`) | Rust | Apache-2.0 | Clean, fast terminal agent; can point at local models. |
| **Qwen Code** (`QwenLM/qwen-code`) | TypeScript | Apache-2.0 | Fork of Gemini CLI tuned for Qwen3-Coder's tool format. |
| **Aider** | Python | Apache-2.0 | Mature pair-programmer; less agentic, diff-based editing. |
| **mini-swe-agent** | Python | MIT | ~100 lines of agent loop; ideal reference for a from-scratch harness and the scaffold used in many RL papers. |

**Recommendation:** start by forking **OpenCode** (fastest route to a usable product). If we want a codebase we fully understand and control, write our own harness using **mini-swe-agent** as the reference design and borrow ideas from Codex CLI / Goose.

---

## 4. Model options (open weights)

| Model | Size | License (verified on HF) | SWE-bench Verified* | Runs on |
|---|---|---|---|---|
| **Devstral Small 2** (`mistralai/Devstral-Small-2-24B-Instruct-2512`) | 24B dense | Apache-2.0 | 68.0% | Single RTX 4090 or 32 GB Mac (quantized) |
| **Qwen3-Coder-30B-A3B-Instruct** | 30B total / 3B active | Apache-2.0 | — | 24–32 GB RAM/VRAM quantized; fast because of MoE |
| **Qwen3-Coder-Next** | 80B total / 3B active | Apache-2.0 | 70.6% (Pro: 44.3%) | 64 GB+ workstation quantized |
| **gpt-oss-20b** | 21B MoE | Apache-2.0 | — | 16 GB GPU; good general tool-use baseline |
| **DeepSeek V3.2 / V4** | 685B+ | MIT | 70–80% | Multi-GPU server only; use via API |
| **Kimi K2.x** | ~1T MoE | Modified MIT | ~80% | Multi-GPU server; license has extra conditions |

\* Scores are vendor-reported and use different scaffolds, so they are only roughly comparable.

**Recommendation:** default to **Qwen3-Coder-30B-A3B** or **Devstral Small 2** for local use, and **Qwen3-Coder-Next** when we have a bigger box. All are Apache-2.0, so they can be used, modified, fine-tuned and redistributed commercially forever.

---

## 5. Training / fine-tuning plan

### 5.1 Datasets (permissive, test-verified)

| Dataset | Size | License | Notes |
|---|---|---|---|
| **SWE-smith** | ~52K task instances from 128 repos | MIT (toolkit) | Also a generator: can synthesise new tasks from any repo |
| **CoderForge-Preview** (Together AI, Feb 2026) | 258K trajectories, 51K tasks, 1,655 repos | Permissive-repo-only filter | Built from R2E-Gym, SWE-smith, SWE-rebench |
| **SWE-ZERO trajectories** (NVIDIA) | 318K OpenHands trajectories | CC BY 4.0 | Code drawn only from MIT/Apache/BSD repos |
| **Open-SWE-Traces** (NVIDIA, Jun 2026) | 207K trajectories, 9 languages | CC BY 4.0 | Built on SWE-rebench v2 (32K containerised tasks, 20 languages) |
| **SWE-Gym / SWE-rebench** | 2.4K / 21K+ instances | Check dataset cards | Useful as RL environments (they ship Docker images and tests) |

Before training, confirm each dataset card's license and keep a `DATA_LICENSES.md` record.

### 5.2 Stages

1. **Stage 0 – No training.** Point the harness at a stock Qwen3-Coder / Devstral model. Measure on a small SWE-bench Verified subset with our harness. This sets the baseline.
2. **Stage 1 – SFT with LoRA.** Convert trajectories into *our* harness's tool-call format and fine-tune with **Unsloth** (Apache-2.0). Small models (≤14B) fit a free 16 GB T4 on Kaggle/Colab; a 24–30B model needs a rented 48–80 GB GPU for a few hours.
3. **Stage 2 – RL (optional, later).** Use **SkyRL** or **verl / VerlTool** for multi-turn agentic RL against Dockerised SWE tasks with test-based rewards. Expect 8×H100-class hardware for days; budget for rented compute or research credits.
4. **Publish** LoRA adapters and merged weights to the Hugging Face Hub under Apache-2.0, with a model card listing data sources.

---

## 6. Hosting: what is actually free forever

Free cloud tiers are not a safe foundation. In 2026 Oracle reportedly halved its "Always Free" Ampere A1 allowance (4 OCPU/24 GB → 2 OCPU/12 GB), and free LLM API limits change frequently. Plan so that **no free tier is a single point of failure**.

| Need | Primary (free forever) | Free-tier fallbacks (may change) |
|---|---|---|
| Source code | GitHub | GitLab, Codeberg mirror |
| Weights & datasets | Hugging Face Hub (public repos) | Mirror key artefacts to a second host (e.g. ModelScope, or a torrent/IPFS pin) |
| Inference | **User's own hardware** via Ollama / llama.cpp / vLLM (all MIT/Apache) | OpenRouter `:free` models (~20 RPM, ~50 req/day without credits), Groq (~30 RPM, short context on free tier), Cerebras (credits) |
| Always-on demo server | A small owned box (mini PC / used GPU) | Oracle Always Free ARM (CPU only, ~4–12 tok/s on a 7B Q4 model), HF Spaces with ZeroGPU (minutes per day) |
| Training compute | — (nothing serious is free forever) | Kaggle ~30 GPU-hrs/week, Colab free T4 (unpublished limits) |

**Recommendation:**
- The harness talks to **any OpenAI-compatible endpoint** so users can switch between local Ollama, vLLM, OpenRouter, Groq, etc. by changing one config value.
- **Local inference is the default.** It is the only path with zero ongoing cost and no dependency on a vendor's free tier.
- Keep **GitHub + Hugging Face** as the canonical homes, with mirrors for anything we cannot afford to lose.
- Treat free GPU tiers as **experiment budgets**, and plan for a modest paid budget (or academic/startup credits) when we reach Stage 2 RL.

---

## 7. Proposed roadmap

| Phase | Deliverable | Est. cost |
|---|---|---|
| 1 | Fork OpenCode (or scaffold our own harness), wire to local Ollama + Qwen3-Coder-30B-A3B | $0 |
| 2 | Evaluation harness: run a 50–100 task SWE-bench Verified subset in Docker | $0 (local) |
| 3 | Data pipeline: convert CoderForge / SWE-ZERO trajectories into our tool format | $0 |
| 4 | LoRA SFT on a 4–14B model with Unsloth on Kaggle; then a 24–30B run on a rented GPU | $0 → ~$20–100 |
| 5 | Publish adapters to Hugging Face; compare against baseline | $0 |
| 6 | (Optional) Agentic RL with SkyRL/verl | Credits or paid GPUs |

---

## 8. Open questions for the team

1. **Fork vs. build:** fork OpenCode (TypeScript/Go) for speed, or write our own harness (Python or Rust) for full control?
2. **Target hardware:** what machines will users run this on? That decides the default model size.
3. **Commercial intent:** if we plan to sell this, we stay strictly on Apache-2.0/MIT models and permissive datasets (already the plan above).
4. **Budget for Stage 2:** are we willing to pay for or apply for GPU credits, or do we stop at LoRA SFT?

---

## Sources

- Harness comparisons: [OpenHands – Claude Code alternatives](https://www.openhands.dev/blog/claude-code-alternatives), [OpenHands – Is Claude Code open source?](https://www.openhands.dev/blog/is-claude-code-open-source), [Pinggy – Best open source CLI coding agents](https://pinggy.io/amp/blog/best_open_source_cli_coding_agents), [Morph – Claude Code alternatives](https://morphllm.com/comparisons/claude-code-alternatives)
- License files: [OpenCode](https://github.com/sst/opencode), [OpenHands](https://github.com/OpenHands/OpenHands), [Goose](https://github.com/block/goose), [Codex CLI](https://github.com/openai/codex), [Qwen Code](https://github.com/QwenLM/qwen-code)
- Models: [Modal – Best open source models for SWE-bench agents (Jul 2026)](https://modal.com/resources/best-open-source-models-swe-bench-coding-agents), [Morph – Best open-source coding model 2026](https://www.morphllm.com/best-open-source-coding-model-2026), [Faros – Open-weight models](https://www.faros.ai/blog/open-weight-models), Hugging Face model cards: [Qwen3-Coder-Next](https://huggingface.co/Qwen/Qwen3-Coder-Next), [Devstral Small 2](https://huggingface.co/mistralai/Devstral-Small-2-24B-Instruct-2512), [Qwen3-Coder-30B-A3B](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B-Instruct), [gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b)
- Datasets: [SWE-smith](https://github.com/SWE-bench/SWE-smith), [SWE-smith paper](https://arxiv.org/abs/2504.21798v2), [Together – CoderForge-Preview](https://together.ai/blog/coderforge-preview), [SWE-ZERO trajectories mirror](https://huggingface.co/datasets/alucent/mirror-SWE-Zero-openhands-trajectories), [Open-SWE-Traces paper](https://arxiv.org/html/2606.16038), [SWE task dataset comparison](https://arxiv.org/html/2603.20691v1)
- Training frameworks: [SkyRL v0](https://novasky-ai.notion.site/skyrl-v0), [VerlTool](https://arxiv.org/abs/2509.01055), [MCP-Universe RL](https://arxiv.org/pdf/2608.22167), [Unsloth notebooks](https://docs.unsloth.ai/get-started/unsloth-notebooks), [Unsloth GRPO tutorial](https://unsloth.ai/docs/get-started/reinforcement-learning-rl-guide/tutorial-train-your-own-reasoning-model-with-grpo)
- Compute & hosting: [Free cloud GPUs in 2026](https://gpuperhour.com/blog/free-cloud-gpus-and-credits), [AIMultiple – Free cloud GPU](https://aimultiple.com/free-cloud-gpu), [Oracle free tier changes 2026](https://terminalbytes.com/oracle-cloud-free-tier-changes-2026), [Oracle Cloud free tier for LLMs](https://blog.easecloud.io/ai-cloud/launch-oracle-cloud-llms-in/), [OpenRouter – Free LLM APIs compared](https://openrouter.ai/blog/tutorials/free-llm-apis-compared/), [Free LLM API tiers 2026](https://wetheflywheel.com/en/ai-model-access/free-llm-api-tiers-2026/)

> Figures for free-tier limits and benchmark scores come largely from third-party blogs and vendor claims. Re-verify on the official pages before making commitments.
