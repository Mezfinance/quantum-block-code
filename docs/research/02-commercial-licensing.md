# Commercial Licensing Check

**Date:** 2026-11-08
**Question:** Can we sell a product built on the components from `01-open-source-coding-agent.md`?

**Short answer:** Yes. MIT, Apache-2.0, BSD and CC BY 4.0 all allow commercial use, including selling, closed-source distribution and SaaS. Each one has attribution duties. Two of the larger models carry extra conditions and should be avoided or used with care.

Licences were checked on 2026-11-08 against Hugging Face model/dataset metadata and the projects' LICENSE files. This is engineering due diligence, not legal advice. Get a lawyer to review before launch.

---

## 1. What each licence allows

| Licence | Sell it? | Keep our changes closed? | What we must do |
|---|---|---|---|
| **MIT** | Yes | Yes | Include the original copyright and licence text in our distribution. |
| **Apache-2.0** | Yes | Yes | Include the licence and any `NOTICE` file; mark files we changed. Includes a patent grant, which ends if we sue anyone over patents in that work. Gives **no trademark rights**. |
| **BSD (2/3-clause)** | Yes | Yes | Keep the copyright notice; 3-clause also bars using the authors' names to endorse our product. |
| **CC BY 4.0** (datasets) | Yes | Yes | Credit the creators, link the licence, say if we changed the data. |

None of these are copyleft. Selling, offering as SaaS or shipping a closed binary are all allowed.

---

## 2. Component-by-component verdict

### Harness (the CLI)

| Component | Licence | Commercial? |
|---|---|---|
| OpenCode | MIT | ✅ Yes |
| mini-swe-agent | MIT | ✅ Yes |
| Goose, Codex CLI, Qwen Code, Aider | Apache-2.0 | ✅ Yes |
| OpenHands | MIT (core) | ✅ Core yes. ⚠️ The repo has an `enterprise/` area that may be licensed differently, so don't copy from it until we've checked. |

### Models

| Model | Licence | Commercial? |
|---|---|---|
| Qwen3-Coder-30B-A3B, Qwen3-Coder-Next, Qwen3-Coder-480B | Apache-2.0 | ✅ Yes, no revenue limits |
| Devstral **Small 2** (24B) | Apache-2.0 | ✅ Yes, no revenue limits |
| gpt-oss-20b | Apache-2.0 | ✅ Yes |
| DeepSeek-V3.2 | MIT | ✅ Yes |
| Devstral **2** (123B) | Modified MIT | ❌ **Avoid.** Licence ends if our company's monthly revenue exceeds **$20M**, and this covers fine-tunes and derivatives too. |
| Kimi K2.x | Modified MIT | ⚠️ Allowed, but above 100M monthly users or $20M monthly revenue we must show "Kimi K2" prominently in our UI. |

**Watch the naming:** Devstral Small 2 (Apache-2.0) and Devstral 2 (restricted) look alike but have different licences. Check the exact model ID every time.

### Training data

| Dataset | Licence | Commercial? |
|---|---|---|
| SWE-smith | MIT | ✅ Yes |
| SWE-Gym | MIT | ✅ Yes |
| SWE-rebench (Nebius) | CC BY 4.0 | ✅ Yes, with attribution |
| NVIDIA Nemotron-SWE / SWE-ZERO / Open-SWE-Traces | CC BY 4.0 | ✅ Yes, with attribution |
| CoderForge-Preview (Together) | **No licence on the HF card** | ⚠️ Don't use it commercially until Together publishes a licence or confirms in writing. |

---

## 3. Remaining risks

1. **Trademarks.** Apache and MIT don't let us use other projects' names as our brand. Our product can't be called "OpenCode", "Qwen" or "Claude", but we can say it is "powered by Qwen3-Coder".
2. **Code inside the datasets.** Trajectories contain snippets from third-party repos. The datasets we picked filter to MIT/Apache/BSD repos, but each of those also expects attribution. Keep the dataset's source lists in our records.
3. **No closed-model outputs.** Training on outputs from Claude, GPT, Gemini and similar models breaks their terms. Before using any third-party dataset, check how it was generated. Some "open" trajectory sets were produced with closed models.
4. **Licences can change for new versions.** Releases we've already downloaded keep their licence. Pin model and dataset revisions (commit hashes) so we can always show which licence applied.
5. **Patents.** Apache-2.0 gives a patent grant, but MIT and BSD don't say anything about patents. The risk is low, but it's one reason to prefer Apache-2.0 components where we have a choice.

---

## 4. What to do

- **Use:** the Qwen3-Coder family, Devstral Small 2, gpt-oss and DeepSeek for models; SWE-smith, SWE-Gym, SWE-rebench and the NVIDIA sets for data.
- **Avoid:** Devstral 2 (123B), CoderForge-Preview (until it has a licence), and anything under Llama-style or custom non-commercial licences.
- **Set up compliance now:**
  - a `THIRD_PARTY_NOTICES` file bundled with every release (MIT/Apache texts plus `NOTICE` files),
  - a `DATA_LICENSES.md` listing each dataset with its revision hash and licence,
  - model cards on Hugging Face that credit data sources (needed for CC BY),
  - a licence-check step in CI (e.g. `licensee`, `pip-licenses` or `license-checker`) that fails the build on GPL/AGPL/non-commercial dependencies unless approved.

---

## Sources

- Model licences: Hugging Face cards for [Qwen3-Coder-Next](https://huggingface.co/Qwen/Qwen3-Coder-Next), [Qwen3-Coder-30B-A3B](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B-Instruct), [Qwen3-Coder-480B](https://huggingface.co/Qwen/Qwen3-Coder-480B-A35B-Instruct), [Devstral Small 2](https://huggingface.co/mistralai/Devstral-Small-2-24B-Instruct-2512), [Devstral 2 (123B) LICENSE](https://huggingface.co/mistralai/Devstral-2-123B-Instruct-2512/blob/main/LICENSE), [Kimi K2 LICENSE](https://huggingface.co/moonshotai/Kimi-K2-Instruct/blob/main/LICENSE), [DeepSeek-V3.2](https://huggingface.co/deepseek-ai/DeepSeek-V3.2), [gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b)
- Dataset licences: [SWE-smith](https://huggingface.co/datasets/SWE-bench/SWE-smith), [SWE-Gym](https://huggingface.co/datasets/SWE-Gym/SWE-Gym), [SWE-rebench](https://huggingface.co/datasets/nebius/SWE-rebench), [Nemotron-SWE-v1](https://huggingface.co/datasets/nvidia/Nemotron-SWE-v1), [CoderForge-Preview](https://huggingface.co/datasets/togethercomputer/CoderForge-Preview)
- Harness licences: [OpenCode](https://github.com/sst/opencode), [OpenHands](https://github.com/OpenHands/OpenHands), [Goose](https://github.com/block/goose), [Codex CLI](https://github.com/openai/codex), [Qwen Code](https://github.com/QwenLM/qwen-code)
- Licence texts: [MIT](https://opensource.org/license/mit), [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0), [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
