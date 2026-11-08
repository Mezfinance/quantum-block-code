# CLAUDE.md — quantum-block-code

Context for any new Claude Code session working in this repo. **Read this first, and update it whenever you push changes.**

## Project goal

Build and sell a browser-based AI coding agent (a hosted Claude Code–style product) using open-weight LLMs and open tooling. Every component must be licensed for commercial use with no fees or revenue caps, and hosting must not depend on any single vendor or free tier.

## Current status (2026-11-08)

- Research only so far; no code yet. Next step is Phase 1 of report 03 (prototype).
- Research report: `docs/research/01-open-source-coding-agent.md`
- Commercial licensing check: `docs/research/02-commercial-licensing.md`. **The product will be sold**, so every component must allow commercial use.
- Browser product recommendation: `docs/research/03-browser-product-recommendation.md`. **The product is browser-based** (hosted SaaS), and this is the current plan of record.

## Key decisions so far (plan of record, see report 03)

- **Product shape:** browser-based paid SaaS. Thin browser client; agent and code run server-side.
- **Browser UI:** our own web app with Monaco (editor) and xterm.js (terminal). Do not reuse OpenCode's web UI (it proxies to app.opencode.ai, and the name is their trademark).
- **Agent engine:** fork of **OpenCode (MIT, TypeScript)**, renamed, run as `opencode serve` inside each user's sandbox. OpenHands rejected: its multi-tenant/enterprise features are Polyform-licensed.
- **Sandbox:** **E2B** (Apache-2.0, Firecracker microVMs). Managed cloud for MVP, self-host later. Rejected: WebContainers (paid commercial licence), Daytona (closed/AGPL), Pyodide (MPL-2.0, Python-only).
- **Models:** **Qwen3-Coder-30B-A3B** default, **Qwen3-Coder-Next** premium (both Apache-2.0), served with **vLLM**. In-browser WebLLM only as a possible later "offline lite" mode.
- **GPU hosting:** serverless scale-to-zero (Modal/RunPod) at launch, dedicated GPUs once utilisation passes ~40%. Offer bring-your-own-model for a cheap tier.
- **Training:** LoRA SFT with Unsloth on SWE-smith, SWE-Gym, SWE-rebench and NVIDIA SWE sets (MIT / CC BY 4.0). RL (SkyRL / verl) is a later, paid-compute phase.
- **Storage:** code on GitHub; weights/datasets on Hugging Face Hub (private for our fine-tunes) with a mirror.

## Hard rules

- Only MIT / Apache-2.0 / BSD / CC BY dependencies, models and datasets unless the team explicitly approves otherwise.
- Never copy Claude Code's proprietary source, prompts or branding; never use the "Claude" name for the product.
- Never train on outputs from Claude or other closed models (their terms forbid building competing models from outputs).
- Commercial blocklist: **Devstral 2 (123B)** (licence ends above $20M monthly revenue; Devstral *Small* 2 is fine), **CoderForge-Preview** (no licence published), anything non-commercial, Llama-style or GPL/AGPL. Kimi K2 needs approval (UI attribution clause at scale).
- Pin model/dataset revision hashes and keep `THIRD_PARTY_NOTICES` and `DATA_LICENSES.md` up to date once code exists.

## Open questions

1. Product name (the fork must be rebranded).
2. Launch GPU provider (Modal vs RunPod) and monthly hosting budget.
3. Budget / credits for the RL phase?
4. Frontend framework (React vs Svelte) for the web app.

## Working conventions

- Develop on the branch assigned to the session; commit with clear messages; do not open PRs unless asked.
- Work in small steps; do not run agents in parallel.
- Add new research as `docs/research/NN-topic.md` and list it under "Current status" above.
