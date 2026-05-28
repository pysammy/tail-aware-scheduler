# Serving Benchmark by Actual Output Length

Short requests are classified as `output_len <= 64`. Long requests are classified from the actual generated token count, not from prompt labels or requested output length.

## Class Summary

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

## Delta vs Baseline

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
