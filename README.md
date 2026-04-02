# cloud-run-batch-experiment
[![GitHub release](https://img.shields.io/github/v/release/wan-huiyan/cloud-run-batch-experiment)](https://github.com/wan-huiyan/cloud-run-batch-experiment/releases) [![Claude Code](https://img.shields.io/badge/Claude_Code-skill-orange)](https://claude.com/claude-code) [![license](https://img.shields.io/github/license/wan-huiyan/cloud-run-batch-experiment)](LICENSE) [![last commit](https://img.shields.io/github/last-commit/wan-huiyan/cloud-run-batch-experiment)](https://github.com/wan-huiyan/cloud-run-batch-experiment/commits)

Deploy and run batch Python experiments on GCP Cloud Run Jobs with parallel execution and GCS result collection.

## What it does

This Claude Code skill guides you through the full pattern for running batch experiments on GCP Cloud Run Jobs:

- Writes a runner script that reads `CLOUD_RUN_TASK_INDEX` to pick up per-task parameters
- Creates a minimal Dockerfile and `.gcloudignore`
- Deploys via Cloud Build to Artifact Registry
- Executes N parallel tasks with `gcloud run jobs execute --tasks N`
- Collects JSON results from GCS

## When to use

- Running 10+ independent experiments that each take 1-60 minutes
- Parameter sweeps, sensitivity analyses, permutation tests
- Local execution would take hours but individual runs are independent
- CPU-bound workloads (not GPU -- use Vertex AI or GKE for GPU)

## Quick start

```
You: Run my 50 permutation tests on Cloud Run

Claude: I'll set up a Cloud Run Job for your permutation tests.

1. Creating cloud_run_task.py that maps CLOUD_RUN_TASK_INDEX to each
   permutation seed...
2. Building Dockerfile with your dependencies...
3. Deploying via Cloud Build...
4. Executing with --tasks 50...
5. Collecting results from GCS...

All 50 tasks completed. 3/50 permutations produced effects >= observed
(permutation p = 0.06). Results at gs://my-bucket/permutations/run-001/
```

## Installation

```bash
claude install-skill github:wan-huiyan/cloud-run-batch-experiment
```

## Cost estimation

| Workload | Config | Per-task time | N tasks | Total cost |
|---|---|---|---|---|
| BSTS permutation | 2 vCPU, 4GB | ~2 min | 50 | ~$7.50 |
| CausalPy MCMC | 2 vCPU, 4GB | ~5 min | 50 | ~$18 |
| Parameter sweep | 1 vCPU, 2GB | ~30 sec | 100 | ~$2 |
| Full exploration | 2 vCPU, 4GB | ~3 min | 200 | ~$36 |

## Limitations

- CPU-only. For GPU workloads, use Vertex AI Training or GKE with GPU node pools.
- Not for long-running HTTP services. Use Cloud Run services instead.
- Not for Cloud Functions, Kubernetes, or non-Python workloads.
- Requires a GCP project with Cloud Run, Cloud Build, and GCS APIs enabled.
- Maximum 10,000 tasks per job execution.

## Related skills

- [causal-impact-campaign](https://github.com/wan-huiyan/causal-impact-campaign) -- Measure causal impact of marketing campaigns using Bayesian structural time series. Pairs with this skill for running multi-spec analysis at scale.
- [permutation-validation](https://github.com/wan-huiyan/permutation-validation) -- Validate causal inference results with empirical permutation tests. Pairs with this skill for parallel permutation execution on Cloud Run.

## License

[MIT](LICENSE)
