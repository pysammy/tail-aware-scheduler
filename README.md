# Tail-Aware Scheduler

Course project workspace for tail-aware LLM request scheduling. The implementation plan is intentionally staged:

1. Validate whether early `P(EOS)` traces predict final output length.
2. Sweep scheduler parameters on offline traces.
3. Integrate the selected policy into vLLM's scheduler.
4. Run serving benchmarks against FCFS and ablations.

## vLLM Submodule

This repo tracks our vLLM fork as a submodule:

```bash
git clone --recurse-submodules https://github.com/labyrinth-ssr/Tail-Aware-Scheduer.git
```

For an existing clone:

```bash
git submodule update --init --recursive
```

Submodule target:

```text
vllm -> https://github.com/labyrinth-ssr/vllm.git
branch -> tail-aware-scheduler
```

Current scheduler skeleton is env-gated and uses both token count and P(EOS):

```bash
VLLM_TAIL_AWARE_SCHEDULING=1
VLLM_TAIL_AWARE_SHORT_THRESHOLD=64
VLLM_TAIL_AWARE_LONG_QUOTA=0.2
VLLM_TAIL_AWARE_EOS_WINDOW=16
VLLM_TAIL_AWARE_EOS_THRESH=0.05
```

Default vLLM behavior is unchanged unless `VLLM_TAIL_AWARE_SCHEDULING=1` is set.

Run the scheduler unit test job in Kubernetes:

```bash
kubectl apply -f k8s/tail-aware-vllm-scheduler-pytest-job.yaml
kubectl logs -n ucsc-hsc job/tail-aware-vllm-scheduler-pytest -f
```

## Current Recommendation

Do not start by editing vLLM directly. First collect offline traces with logit scores enabled and verify that `P(EOS)` separates short and long generations. This reduces the risk of spending time on scheduler integration before the core signal is validated.

One design correction matters before implementation: if `N=8-16` and `short_threshold=64`, a rule applied only once at step `N` will never demote anything because `tokens < short_threshold` is always true. The safer operational rule is:

```text
Before N generated tokens:
  keep request in probation.

At each decode step after N:
  if generated_tokens >= short_threshold
     and max(P_EOS over recent N tokens) <= eos_thresh:
       demote to long queue.
  else:
       keep request in probation.
```

This keeps the spirit of the proposal while making demotion reachable.

## Offline P(EOS) Analysis

Prepare a JSONL prompt file:

```jsonl
{"id":"wmt_0001","prompt":"Translate to French: The meeting starts at noon.","source":"wmt"}
{"id":"gsm8k_0001","prompt":"Solve step by step: ...","source":"gsm8k"}
```

Collect traces:

```bash
python3 experiments/eos_signal/collect_eos_traces.py \
  --model Qwen/Qwen2.5-7B-Instruct \
  --input data/prompts.jsonl \
  --output results/eos_signal/traces.jsonl \
  --max-new-tokens 512 \
  --temperature 0.0
```

Analyze signal quality and parameter sensitivity:

```bash
python3 experiments/eos_signal/analyze_eos_traces.py \
  --input results/eos_signal/traces.jsonl \
  --out-dir results/eos_signal \
  --length-threshold 100 \
  --n-values 8,16,32 \
  --short-threshold-values 64,100,128 \
  --eos-thresh-values 0.005,0.01,0.02,0.05,0.1
```

Outputs include summary metrics, optional plots, and a CSV sweep table.

## vLLM Integration

The scheduler changes live on the `tail-aware-scheduler` branch of our vLLM fork. Two phases have been implemented:

### Phase 1: Token-Count Demotion

Requests are demoted from probation (high-priority) to the long queue after generating `short_threshold` output tokens. The running list is reordered each step to approximate `long_quota`, interleaving long requests among short/probation ones.

Modified files:
- `vllm/v1/core/sched/scheduler.py` — demotion logic, running-list reorder, long-set cleanup.

### Phase 2: P(EOS)-Aware Demotion

P(EOS) — the probability the model assigns to the EOS token — is extracted at each decode step as a complementary demotion signal. If `max(recent P(EOS))` within a sliding window exceeds `eos_thresh`, the request stays in the high-priority queue even if its token count exceeds `short_threshold`. This prevents premature demotion of requests that are close to finishing.

The P(EOS) computation happens on the raw logits (before logits processors, temperature, top-k/top-p) to ensure the probability reflects the model's genuine belief.

Modified files:
- `vllm/v1/sample/metadata.py` — added `eos_token_ids` field to `SamplingMetadata`.
- `vllm/v1/sample/sampler.py` — added `_compute_eos_probs()` using `gather + logsumexp`, called in `forward()` before logits processors.
- `vllm/v1/outputs.py` — added `eos_probs` to `SamplerOutput` (GPU tensor) and `ModelRunnerOutput` (CPU list).
- `vllm/v1/worker/gpu_input_batch.py` — builds the `eos_token_ids` tensor from `sampling_params._eos_token_id`, gated behind `VLLM_TAIL_AWARE_SCHEDULING`.
- `vllm/v1/worker/gpu_model_runner.py` — propagates `eos_probs` through both sync and async (overlapped) GPU→CPU copy paths.
- `vllm/v1/request.py` — added `recent_eos_probs: list[float]` to store per-step P(EOS) history.
- `vllm/v1/core/sched/scheduler.py` — extracts P(EOS) in `update_from_output()`, checks the sliding window in `_maybe_demote_tail_aware_request()`.

### Unit Tests

Seven scheduler tests cover tail-aware behavior (`tests/v1/core/test_scheduler.py`):

| Test | Description |
|------|-------------|
| `test_tail_aware_token_demotion_threshold` | Requests are demoted after exceeding `short_threshold` output tokens. |
| `test_tail_aware_reorder_running_with_long_quota` | Running list is reordered to interleave long requests at `long_quota` ratio. |
| `test_tail_aware_cleanup_on_free_blocks` | Long-set entry is removed when a request finishes. |
| `test_tail_aware_eos_prevents_demotion` | High P(EOS) in the window prevents demotion despite exceeding token threshold. |
| `test_tail_aware_eos_low_allows_demotion` | Low P(EOS) allows normal token-count demotion. |
| `test_tail_aware_eos_window_sliding` | Only the most recent `eos_window` values are considered; old high values are ignored. |
| `test_tail_aware_eos_prob_stored_on_request` | P(EOS) values are correctly stored on the request object. |

Run tests on the Kubernetes cluster:

```bash
kubectl delete job tail-aware-vllm-scheduler-pytest -n ucsc-hsc --ignore-not-found
kubectl apply -f k8s/tail-aware-vllm-scheduler-pytest-job.yaml
kubectl logs -n ucsc-hsc -l job-name=tail-aware-vllm-scheduler-pytest -c pytest -f
```

See `proposal/implementation_todo.md` for the remaining follow-up plan.
