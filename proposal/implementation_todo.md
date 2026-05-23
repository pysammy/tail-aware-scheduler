# Tail-Aware Scheduler Implementation TODO

## Completed

### Phase 1: Token-Count Demotion

- Env-gated tail-aware scheduling (`VLLM_TAIL_AWARE_SCHEDULING=1`).
- Demote requests after `VLLM_TAIL_AWARE_SHORT_THRESHOLD` output tokens.
- Keep demoted request IDs in a long set.
- Reorder the running list to approximate `VLLM_TAIL_AWARE_LONG_QUOTA`.
- Clean up long-set state when requests are freed.
- Three unit tests: threshold, reorder, cleanup.

### Phase 2: P(EOS)-Aware Demotion

- Extract P(EOS) from raw logits in `Sampler.forward()` using
  `gather + logsumexp`, before logits processors are applied.
- Propagate P(EOS) through both sync and async GPU-to-CPU copy paths
  in `gpu_model_runner.py`.
- Build `eos_token_ids` tensor in `gpu_input_batch.py`, gated behind
  `VLLM_TAIL_AWARE_SCHEDULING`.
- Store per-step P(EOS) history on `Request.recent_eos_probs`.
- Check sliding window `max(recent_eos_probs[-eos_window:])` in
  `_maybe_demote_tail_aware_request()`: high P(EOS) prevents demotion.
- Four new unit tests: eos_prevents_demotion, eos_low_allows_demotion,
  eos_window_sliding, eos_prob_stored_on_request.
- All 7 tail-aware tests pass on the Kubernetes cluster.

### Offline EOS Signal Validation

- Smoke run with Qwen2.5-7B-Instruct on 10 prompts.
- Strong early P(EOS) separation: AUC = 1.0 for short/long classification.
- Stable parameter region: `N=8-32, short_threshold>=64, eos_thresh=0.001-0.1`.

## Immediate TODO

1. Add scheduler metrics and logging:

   - demotion count;
   - output token count at demotion;
   - queue class per request;
   - per-step number of scheduled probation/short vs long requests.

2. Add an end-to-end serving experiment job:

   - baseline FCFS;
   - token-count-only tail-aware;
   - EOS-aware tail-aware;
   - quota ablations such as `0.1`, `0.2`, `0.3`.

3. Larger offline validation with real datasets (WMT, TriviaQA, GSM8K,
   HumanEval) to confirm P(EOS) signal generalizes beyond smoke test.

## Known Design Gaps

- The current patch is not yet the full two-queue design. It only reorders
  `running` requests; it does not split the waiting queue.
- `long_quota` is currently approximate because vLLM schedules by token budget,
  while the patch reorders requests.
- The implementation has no preemption or KV swap-out. Long requests already in
  `running` remain resident, so short requests may still wait under full batch
  occupancy.

## Experiment Matrix

Use a mixed workload with short translation/QA/summarization and long
reasoning/code prompts.

Primary metrics:

- short-request TTFT and end-to-end latency, especially p50/p90/p99;
- long-request throughput and starvation rate;
- aggregate tokens/sec;
- demotion precision and false demotion rate;
- GPU utilization if available from cluster telemetry.

Suggested runs:

```text
baseline:
  VLLM_TAIL_AWARE_SCHEDULING=0

token-count only:
  VLLM_TAIL_AWARE_SCHEDULING=1
  VLLM_TAIL_AWARE_SHORT_THRESHOLD=64
  VLLM_TAIL_AWARE_LONG_QUOTA=0.2

eos-aware:
  VLLM_TAIL_AWARE_SCHEDULING=1
  VLLM_TAIL_AWARE_SHORT_THRESHOLD=64
  VLLM_TAIL_AWARE_LONG_QUOTA=0.2
  VLLM_TAIL_AWARE_EOS_WINDOW=16
  VLLM_TAIL_AWARE_EOS_THRESH=0.05

quota sweep:
  VLLM_TAIL_AWARE_LONG_QUOTA in 0.1,0.2,0.3,0.5

eos parameter sweep:
  VLLM_TAIL_AWARE_EOS_WINDOW in 8,16,32
  VLLM_TAIL_AWARE_EOS_THRESH in 0.001,0.005,0.01,0.05
```

## Stretch Goals

- Implement real waiting-queue separation for new/probation vs demoted long
  requests.
- Add aging or a minimum long-service guarantee if starvation appears in
  benchmark traces.
- Add optional request-arrival length prediction as an ablation, not as the
  default design.
