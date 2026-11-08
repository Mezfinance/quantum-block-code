# CLAUDE.md — quantum-block-code

Context for any new Claude Code session working in this repo. **Read this first, and update it whenever you push changes.**

## Project goal

Build our own open-source terminal coding agent (a Claude Code–style CLI) using open-weight LLMs and open tooling. Every dependency must stay free to use permanently, and hosting must not depend on any single free tier.

## Current status (2026-11-08)

- Repo was empty; the first commit adds research only. No code yet.
- Research report: `docs/research/01-open-source-coding-agent.md`
- Commercial licensing check: `docs/research/02-commercial-licensing.md`. **The product will be sold**, so every component must allow commercial use.

## Key decisions so far (from the research)

- **Harness:** fork OpenCode (MIT) or build our own harness modelled on mini-swe-agent (MIT). Decision still open.
- **Models:** Apache-2.0 open weights only by default: Qwen3-Coder-30B-A3B, Devstral Small 2 (24B), Qwen3-Coder-Next (80B/3B active).
- **Training:** LoRA SFT with Unsloth on permissively licensed trajectory datasets (SWE-smith, CoderForge-Preview, NVIDIA SWE-ZERO / Open-SWE-Traces). RL (SkyRL / verl) is a later, paid-compute phase.
- **Hosting:** code on GitHub, weights/datasets on Hugging Face Hub, inference local-first (Ollama / llama.cpp / vLLM). Harness must speak any OpenAI-compatible API so free cloud endpoints (OpenRouter, Groq) are optional fallbacks.

## Hard rules

- Only MIT / Apache-2.0 / BSD / CC BY dependencies, models and datasets unless the team explicitly approves otherwise.
- Never copy Claude Code's proprietary source, prompts or branding; never use the "Claude" name for the product.
- Never train on outputs from Claude or other closed models (their terms forbid building competing models from outputs).
- Commercial blocklist: **Devstral 2 (123B)** (licence ends above $20M monthly revenue; Devstral *Small* 2 is fine), **CoderForge-Preview** (no licence published), anything non-commercial, Llama-style or GPL/AGPL. Kimi K2 needs approval (UI attribution clause at scale).
- Pin model/dataset revision hashes and keep `THIRD_PARTY_NOTICES` and `DATA_LICENSES.md` up to date once code exists.

## Open questions

1. Fork OpenCode vs. build our own harness (and in which language)?
2. Target user hardware (decides default model size)?
3. Budget / credits for the RL phase?

## Working conventions

- Develop on the branch assigned to the session; commit with clear messages; do not open PRs unless asked.
- Work in small steps; do not run agents in parallel.
- Add new research as `docs/research/NN-topic.md` and list it under "Current status" above.
