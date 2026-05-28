# Serving Benchmark by Actual Output Length

## Experiment Setup

- Cluster job: `tail-aware-serving-experiment` in namespace `ucsc-hsc`.
- Model: `Qwen/Qwen2.5-7B-Instruct`.
- Server: `vllm serve` with `--max-num-seqs 16` and
  `--max-num-batched-tokens 2048`.
- Benchmark: `vllm bench serve`, custom mixed prompt set, `200` requests,
  `8` RPS, `temperature=0`, `--disable-shuffle`.
- Runtime fix: `FLASHINFER_DISABLE_VERSION_CHECK=1` was set to bypass a
  `flashinfer` / `flashinfer-jit-cache` version mismatch in the cluster image.
- Tail-aware logging was disabled for this run, so old `*.tail_aware.jsonl`
  files remaining in the PVC should not be used for this experiment.

## Cases

- `baseline`: vLLM default FCFS scheduling.
- `token_count_only`: tail-aware scheduling enabled, token-count demotion only
  (`eos_thresh=1.0`, `long_quota=0.2`).
- `eos_aware`: EOS-aware demotion (`eos_thresh=0.05`, `long_quota=0.2`).
- `quota_0_1`: EOS-aware demotion with `long_quota=0.1`.
- `quota_0_3`: EOS-aware demotion with `long_quota=0.3`.

Short requests are classified as `output_len <= 64`. Long requests are classified from the actual generated token count, not from prompt labels or requested output length.

The classification is post-hoc: each completed request is labeled from its
actual generated token count in the benchmark result JSON. This is the right
metric split for this scheduler because the scheduler does not know the true
completion length at admission time.

## Class Summary

| case | class | count | failed | mean output | TTFT p95 ms | TTFT p99 ms | E2E p95 ms | E2E p99 ms | req/s | out tok/s |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| baseline | all | 200 | 0 | 107.92 | 5130.08 | 5413.16 | 10120.01 | 10476.06 | 5.76 | 621.90 |
| baseline | short | 124 | 0 | 17.16 | 5211.38 | 5401.79 | 5503.48 | 5834.60 |  |  |
| baseline | long | 76 | 0 | 256 | 5000.70 | 5219.19 | 10352.92 | 10612.11 |  |  |
| eos_aware | all | 200 | 0 | 107.91 | 280.00 | 343.18 | 30161.92 | 32773.94 | 5.78 | 623.84 |
| eos_aware | short | 124 | 0 | 17.15 | 280.01 | 348.61 | 1012.50 | 1048.64 |  |  |
| eos_aware | long | 76 | 0 | 256 | 261.86 | 322.97 | 31851.87 | 33630.41 |  |  |
| quota_0_1 | all | 200 | 0 | 107.88 | 1390.36 | 1619.93 | 29938.46 | 32855.34 | 5.75 | 620.67 |
| quota_0_1 | short | 124 | 0 | 17.10 | 1447.42 | 1701.76 | 1736.16 | 2210.86 |  |  |
| quota_0_1 | long | 76 | 0 | 256 | 863.55 | 1229.76 | 32018.06 | 33797.23 |  |  |
| quota_0_3 | all | 200 | 0 | 107.89 | 431.52 | 526.47 | 29987.50 | 32711.10 | 5.79 | 624.93 |
| quota_0_3 | short | 124 | 0 | 17.12 | 419.21 | 522.22 | 1040.83 | 1185.07 |  |  |
| quota_0_3 | long | 76 | 0 | 256 | 456.38 | 535.39 | 31784.04 | 33564.13 |  |  |
| token_count_only | all | 200 | 0 | 107.83 | 231.84 | 313.75 | 30023.55 | 32890.70 | 5.75 | 620.21 |
| token_count_only | short | 124 | 0 | 17.01 | 229.28 | 272.85 | 947.57 | 1037.91 |  |  |
| token_count_only | long | 76 | 0 | 256 | 261.91 | 321.08 | 31927.18 | 33629.35 |  |  |

## Delta vs Baseline

| case | class | TTFT p95 | TTFT p99 | E2E p95 | E2E p99 |
|---|---:|---:|---:|---:|---:|
| eos_aware | all | -94.5% | -93.7% | +198.0% | +212.8% |
| eos_aware | short | -94.6% | -93.5% | -81.6% | -82.0% |
| eos_aware | long | -94.8% | -93.8% | +207.7% | +216.9% |
| quota_0_1 | all | -72.9% | -70.1% | +195.8% | +213.6% |
| quota_0_1 | short | -72.2% | -68.5% | -68.5% | -62.1% |
| quota_0_1 | long | -82.7% | -76.4% | +209.3% | +218.5% |
| quota_0_3 | all | -91.6% | -90.3% | +196.3% | +212.2% |
| quota_0_3 | short | -92.0% | -90.3% | -81.1% | -79.7% |
| quota_0_3 | long | -90.9% | -89.7% | +207.0% | +216.3% |
| token_count_only | all | -95.5% | -94.2% | +196.7% | +214.0% |
| token_count_only | short | -95.6% | -94.9% | -82.8% | -82.2% |
| token_count_only | long | -94.8% | -93.8% | +208.4% | +216.9% |

## Analysis

Waiting priority plus recompute preemption achieves the intended short-request
latency effect. Compared with baseline, short-request p99 TTFT drops from
`5401.79 ms` to `272.85 ms` in `token_count_only`, `348.61 ms` in `eos_aware`,
and `522.22 ms` in `quota_0_3`. Short-request p99 end-to-end latency also drops
from `5834.60 ms` to roughly `1.0-1.2 s` for the strongest tail-aware cases.

Throughput is roughly unchanged. Baseline output throughput is `621.90 tok/s`;
the tail-aware cases are between `620.21 tok/s` and `624.93 tok/s`. This means
the short-request improvement is not coming from a large throughput loss.

The cost is severe long-request degradation. Baseline long p99 E2E is
`10612.11 ms`, while the tail-aware cases are around `33.6 s`. Aggregate p99
E2E also gets much worse because it is dominated by long requests: baseline is
`10476.06 ms`, while tail-aware cases are around `32.7-32.9 s`.

The current preemption policy is therefore too aggressive. It proves that the
mechanism can protect short requests, but it does so by repeatedly pushing
demoted long requests out of the running set. Since the current implementation
uses vLLM's recompute preemption path rather than KV swap-out, preempted long
requests pay extra recomputation cost.

## Takeaways

- The design goal should be evaluated on short-request TTFT/E2E tail latency,
  not only aggregate p95/p99.
- Waiting priority and preemption are effective for short requests under mixed
  load.
- The current recompute-preemption policy is not production-ready because it
  over-penalizes long requests.
- The next implementation should keep waiting priority, but make preemption
  bounded and optional.

## Next Steps

- Add a separate preemption gate such as `VLLM_TAIL_AWARE_PREEMPTION`.
- Add a per-request preemption cap, for example
  `VLLM_TAIL_AWARE_MAX_PREEMPTS_PER_REQ=1`.
- Only preempt a long request when the running long ratio is above
  `VLLM_TAIL_AWARE_LONG_QUOTA`.
- Add long aging or a starvation guard so demoted long requests can regain
  service after waiting too long.
- Re-run three cases: baseline, waiting-priority-only, and bounded preemption.

## Bounded Preemption Follow-Up

After the aggressive preemption run above, the scheduler was changed to make
preemption optional and bounded:

- `VLLM_TAIL_AWARE_PREEMPTION=0/1` controls whether tail-aware preemption is
  enabled.
- `VLLM_TAIL_AWARE_MAX_PREEMPTS_PER_REQ=1` caps each request to at most one
  recompute preemption.
- A long request is preempted only when the current running long-request ratio
  is above `VLLM_TAIL_AWARE_LONG_QUOTA`.
- Waiting priority remains enabled whenever `VLLM_TAIL_AWARE_SCHEDULING=1`.

The follow-up job wrote results to
`/workspace/tail-aware-results/serving/qwen-mixed-bounded`, copied locally to
`results/cluster/serving/qwen-mixed-bounded`.

### Bounded Run Cases

- `baseline`: vLLM default FCFS scheduling.
- `token_count_only_waiting`: tail-aware scheduling with token-count demotion,
  waiting priority enabled, preemption disabled.
- `eos_aware_waiting`: EOS-aware demotion, waiting priority enabled,
  preemption disabled.
- `bounded_preemption`: EOS-aware demotion, waiting priority enabled,
  bounded preemption enabled, `long_quota=0.2`.
- `bounded_preemption_quota_0_3`: same as `bounded_preemption`, but
  `long_quota=0.3`.

### Bounded Run Class Summary

| case | class | count | failed | mean output | TTFT p95 ms | TTFT p99 ms | E2E p95 ms | E2E p99 ms | req/s | out tok/s |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| baseline | all | 200 | 0 | 107.97 | 8946.52 | 9575.28 | 11478.63 | 12383.58 | 5.39 | 581.75 |
| baseline | short | 124 | 0 | 17.23 | 8930.04 | 9558.84 | 9321.59 | 9600.84 |  |  |
| baseline | long | 76 | 0 | 256 | 8868.06 | 9482.71 | 12173.33 | 13336.71 |  |  |
| bounded_preemption | all | 200 | 0 | 107.94 | 6623.73 | 7111.41 | 10309.23 | 10973.34 | 5.63 | 607.57 |
| bounded_preemption | short | 124 | 0 | 17.19 | 6589.69 | 7045.18 | 6900.23 | 7208.38 |  |  |
| bounded_preemption | long | 76 | 0 | 256 | 6626.65 | 7227.00 | 10829.26 | 11060.45 |  |  |
| bounded_preemption_quota_0_3 | all | 200 | 0 | 108.01 | 5260.34 | 5887.65 | 9441.69 | 9955.55 | 5.80 | 626.93 |
| bounded_preemption_quota_0_3 | short | 124 | 0 | 17.31 | 5281.20 | 5822.95 | 5702.52 | 6350.34 |  |  |
| bounded_preemption_quota_0_3 | long | 76 | 0 | 256 | 5194.23 | 5502.14 | 9926.35 | 10159.82 |  |  |
| eos_aware_waiting | all | 200 | 0 | 107.97 | 11485.24 | 12398.08 | 14051.74 | 14818.84 | 5.07 | 547.92 |
| eos_aware_waiting | short | 124 | 0 | 17.25 | 11365.24 | 12602.96 | 11566.92 | 12724.28 |  |  |
| eos_aware_waiting | long | 76 | 0 | 256 | 11595.78 | 12257.92 | 14558.51 | 15438.62 |  |  |
| token_count_only_waiting | all | 200 | 0 | 108.03 | 3331.15 | 3558.17 | 7370.14 | 7851.72 | 6.55 | 707.80 |
| token_count_only_waiting | short | 124 | 0 | 17.33 | 3410.55 | 3727.60 | 3720.05 | 4088.70 |  |  |
| token_count_only_waiting | long | 76 | 0 | 256 | 3189.66 | 3370.45 | 7803.10 | 8135.48 |  |  |

### Bounded Run Delta vs Baseline

| case | class | TTFT p95 | TTFT p99 | E2E p95 | E2E p99 |
|---|---:|---:|---:|---:|---:|
| bounded_preemption | all | -26.0% | -25.7% | -10.2% | -11.4% |
| bounded_preemption | short | -26.2% | -26.3% | -26.0% | -24.9% |
| bounded_preemption | long | -25.3% | -23.8% | -11.0% | -17.1% |
| bounded_preemption_quota_0_3 | all | -41.2% | -38.5% | -17.7% | -19.6% |
| bounded_preemption_quota_0_3 | short | -40.9% | -39.1% | -38.8% | -33.9% |
| bounded_preemption_quota_0_3 | long | -41.4% | -42.0% | -18.5% | -23.8% |
| eos_aware_waiting | all | +28.4% | +29.5% | +22.4% | +19.7% |
| eos_aware_waiting | short | +27.3% | +31.8% | +24.1% | +32.5% |
| eos_aware_waiting | long | +30.8% | +29.3% | +19.6% | +15.8% |
| token_count_only_waiting | all | -62.8% | -62.8% | -35.8% | -36.6% |
| token_count_only_waiting | short | -61.8% | -61.0% | -60.1% | -57.4% |
| token_count_only_waiting | long | -64.0% | -64.5% | -35.9% | -39.0% |

### Bounded Run Analysis

The bounded-preemption run no longer shows the catastrophic long-request
regression from the aggressive run. In the same bounded-run experiment,
`bounded_preemption_quota_0_3` improves short p99 E2E from `9600.84 ms` to
`6350.34 ms` and also improves long p99 E2E from `13336.71 ms` to
`10159.82 ms`.

The best result in this run is `token_count_only_waiting`: it improves short
p99 E2E from `9600.84 ms` to `4088.70 ms`, improves long p99 E2E from
`13336.71 ms` to `8135.48 ms`, and increases output throughput from
`581.75 tok/s` to `707.80 tok/s`. This suggests that waiting priority plus
token-count demotion is the strongest current policy in this workload.

`eos_aware_waiting` is worse than baseline in this run. The likely explanation
is that the EOS-aware demotion rule is too conservative or poorly calibrated
for this mixed workload, so it delays demotion and loses the benefit of waiting
priority.

The bounded preemption variants are now reasonable tradeoffs instead of
long-request starvation. However, they are not better than
`token_count_only_waiting` in this run, so the next benchmark should include a
cleaner ablation:

- baseline;
- token-count waiting-only;
- token-count bounded preemption;
- EOS-aware waiting-only with a threshold sweep;
- EOS-aware bounded preemption with a threshold sweep.
