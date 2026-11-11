<!-- markdownlint-disable -->

## Overview

Switch the lm-eval-harness GSM8K accuracy tests in `.buildkite/lm-eval-harness/` from lm-eval's `gsm8k` task to `gsm8k_platinum`. That task uses [GSM8K Platinum](https://huggingface.co/datasets/madrylab/gsm8k-platinum), a manually cleaned version of the GSM8K test set. vLLM already uses `gsm8k_platinum` in `tests/models/quantization/test_gpt_oss.py`; this PR applies it to the lm-eval CI as well.

## Claims

- The 29 lm-eval CI model configs that ran `gsm8k` now run `gsm8k_platinum`.
- `run-lm-eval-gsm-vllm-baseline.sh` and `run-lm-eval-gsm-hf-baseline.sh` now produce baselines on `gsm8k_platinum`.
- Accuracy checks no longer count ambiguous questions or wrong answer keys against the model, so a measured accuracy change reflects the model rather than label noise.

## Validation

**Not validated on GPUs yet.** The expected `exact_match,strict-match` and `exact_match,flexible-extract` values in each config were measured on the original `gsm8k`, and I haven't re-measured them on `gsm8k_platinum`. They need to be regenerated with the updated baseline scripts before this can merge, for example:

```bash
bash .buildkite/lm-eval-harness/run-lm-eval-gsm-vllm-baseline.sh -m meta-llama/Meta-Llama-3-8B-Instruct -l 250 -f 5 -t 1
```

`test_lm_eval_correctness.py` looks up results by the task name in each config, so it needs no change.

## Details

### Drawbacks of the original GSM8K test set

- **Wrong answer keys.** Some test questions have incorrect reference answers. A model that solves them correctly is scored as wrong, and a model that reproduces the faulty answer is scored as right.
- **Ambiguous or poorly written questions.** Some problems can reasonably be read more than one way, or are missing information needed to get a unique answer. Whether a model "passes" them depends on which reading it picks, not on its reasoning.
- **Noise hides real regressions.** Strong models already score in the 90s on GSM8K, so the remaining errors are a mix of real mistakes and bad questions. For CI that compares a kernel, quantization or parallelism change against a baseline, this noise floor makes small accuracy regressions harder to tell apart from benchmark errors.
- **Subsets inherit the noise.** Configs that use `limit: 250` or `limit: 1000` score only the first N questions, so any flawed questions in that range carry more weight.

### How GSM8K Platinum finds and fixes these flaws

GSM8K Platinum comes from *Do Large Language Model Benchmarks Test Reliability?* (Vendrow, Vendrow, Beery, Madry, 2025, [arXiv:2502.03461](https://arxiv.org/abs/2502.03461)). The authors:

1. **Ran a variety of frontier models** on every question in the GSM8K test set.
2. **Manually re-annotated every question that at least one model got wrong**, so each disagreement was reviewed by a person.
3. **Fixed the labels** of questions whose reference answer was wrong.
4. **Removed questions they judged poorly written**, most often because the problem statement was ambiguous.

The result is a smaller test set where every remaining question has a single, verified answer. The question and answer format is unchanged (answers still end in `#### <number>`), so lm-eval's existing `strict-match` and `flexible-extract` filters work as-is.

### Trade-offs and limitations

- **Scores are not comparable** with past `gsm8k` numbers, including published model-card results. This is why the thresholds need re-baselining.
- **Few-shot examples come from the scored test set.** Platinum has no train split, so lm-eval's `gsm8k_platinum` task sets `fewshot_split: test`.
- **`limit: 1319` is safe.** Platinum has fewer questions, and lm-eval caps the limit at the dataset size.
- **Out of scope:** vLLM's own GSM8K script (`tests/evals/gsm8k/gsm8k_eval.py`) loads GSM8K from vLLM's S3 bucket and is shared with spec-decode, fault-tolerance and EPLB tests. This PR leaves it unchanged; it could be switched in a follow-up.

---

<details>
<summary> Pull Request Checklist </summary>

- [ ] I used vLLM's `/pr-checklist` skill. (Mandatory for agents, optional for humans).
- [x] AI assistance was used during the creation of this PR.

- [x] **Design Fit:** Minimizes impact on core components, reuses existing functionality, and justifies added complexity.
- [ ] **Testing and Validation:** Validates the change and ensures any added tests are meaningful and reliable, with CI coverage or documented CI resource constraints and validation performed outside CI.
- [x] **Code Quality and Style:** Keeps code and comments clear and concise, and updates relevant documentation and examples.
- [x] **Pull Request Contents:** Includes a brief summary and relevant links, supports claims with evidence, explains root causes and implementation trade-offs, and follows the contributing guide.
</details>

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01TY3YAS9vsvLhTJjbWJkgXA
