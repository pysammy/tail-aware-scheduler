# Tail-Aware Scheduler Implementation TODO

## Current Status

- Offline EOS smoke experiment is complete for a small Qwen workload.
- vLLM submodule has an env-gated token-count-only scheduler patch on
  `tail-aware-scheduler`.
- Implemented behavior:
  - demote requests after `VLLM_TAIL_AWARE_SHORT_THRESHOLD` output tokens;
  - keep demoted request ids in a long set;
  - reorder the running list to approximate `VLLM_TAIL_AWARE_LONG_QUOTA`;
  - clean up long-set state when requests are freed.

## Immediate TODO

1. Run the new scheduler unit tests in Kubernetes:

   ```bash
   kubectl apply -f k8s/tail-aware-vllm-scheduler-pytest-job.yaml
   kubectl logs -n ucsc-hsc job/tail-aware-vllm-scheduler-pytest -f
   ```

2. Add EOS-aware demotion:

   ```text
   if generated_tokens >= short_threshold
      and generated_tokens >= eos_window
      and max(recent_p_eos) <= eos_thresh:
        demote to long
   ```

3. Add scheduler metrics/logging:

   - demotion count;
   - output token count at demotion;
   - queue class per request;
   - per-step number of scheduled probation/short vs long requests.

4. Add an end-to-end serving experiment job:

   - baseline FCFS;
   - token-count-only tail-aware;
   - EOS-aware tail-aware;
   - quota ablations such as `0.1`, `0.2`, `0.3`.

## Known Design Gaps

- The current patch is not yet the full two-queue design. It only reorders
  `running` requests; it does not split the waiting queue.
- `long_quota` is currently approximate because vLLM schedules by token budget,
  while the patch reorders requests.
- The implementation has no preemption or KV swap-out. Long requests already in
  `running` remain resident, so short requests may still wait under full batch
  occupancy.
- The current demotion signal is token count only; it does not yet use `P(EOS)`.

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

token-count:
  VLLM_TAIL_AWARE_SCHEDULING=1
  VLLM_TAIL_AWARE_SHORT_THRESHOLD=64
  VLLM_TAIL_AWARE_LONG_QUOTA=0.2

quota sweep:
  VLLM_TAIL_AWARE_LONG_QUOTA in 0.1,0.2,0.3,0.5

future EOS sweep:
  VLLM_TAIL_AWARE_EOS_WINDOW in 8,16,32
  VLLM_TAIL_AWARE_EOS_THRESHOLD in 0.001,0.005,0.01,0.05
```

## Stretch Goals

- Implement real waiting-queue separation for new/probation vs demoted long
  requests.
- Add aging or a minimum long-service guarantee if starvation appears in
  benchmark traces.
- Add optional request-arrival length prediction as an ablation, not as the
  default design.
