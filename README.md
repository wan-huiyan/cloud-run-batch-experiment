# cloud-run-batch-experiment
[![GitHub release](https://img.shields.io/github/v/release/wan-huiyan/cloud-run-batch-experiment)](https://github.com/wan-huiyan/cloud-run-batch-experiment/releases) [![Claude Code](https://img.shields.io/badge/Claude_Code-skill-orange)](https://claude.com/claude-code) [![license](https://img.shields.io/github/license/wan-huiyan/cloud-run-batch-experiment)](LICENSE) [![last commit](https://img.shields.io/github/last-commit/wan-huiyan/cloud-run-batch-experiment)](https://github.com/wan-huiyan/cloud-run-batch-experiment/commits)

Running 50+ experiment variants locally ties up your machine for hours. This skill deploys them to GCP Cloud Run Jobs in parallel — for pennies — and collects results to GCS automatically.

## What it does

This Claude Code skill guides you through the full pattern for running batch experiments on GCP Cloud Run Jobs:

- Writes a runner script that reads `CLOUD_RUN_TASK_INDEX` to pick up per-task parameters
- Creates a minimal Dockerfile and `.gcloudignore`
- Deploys via Cloud Build to Artifact Registry
- Executes N parallel tasks with `gcloud run jobs execute --tasks N`
- Collects JSON results from GCS

## Why a Claude Code Skill?

Running Cloud Run Jobs requires juggling Dockerfiles, job YAML, polling loops, GCS paths, and IAM permissions. The value isn't in any single step — it's in Claude handling the full sequence: writing the runner script, building and pushing the image, deploying the job, monitoring task status, and surfacing failures. A shell script would need to be re-written for each experiment; this skill adapts to whatever Python code you hand it.

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

**Claude Code (plugin install — recommended):**
```bash
# Add the marketplace, then install the plugin
claude plugin marketplace add wan-huiyan/cloud-run-batch-experiment
claude plugin install cloud-run-batch-experiment@wan-huiyan-cloud-run-batch-experiment
```

**Claude Code (git clone):**
```bash
git clone https://github.com/wan-huiyan/cloud-run-batch-experiment.git ~/.claude/skills/cloud-run-batch-experiment
```

**Cursor** (2.4+):
```bash
# Per-project rule (most reliable)
mkdir -p .cursor/rules
# Copy plugins/cloud-run-batch-experiment/SKILL.md content into .cursor/rules/cloud-run-batch-experiment.mdc with alwaysApply: true

# Or via npx skills CLI
npx skills add wan-huiyan/cloud-run-batch-experiment --global
```

## Cost estimation

| Workload | Config | Per-task time | N tasks | Total cost |
|---|---|---|---|---|
| BSTS permutation | 2 vCPU, 4GB | ~2 min | 50 | ~$0.35 |
| CausalPy MCMC | 2 vCPU, 4GB | ~5 min | 50 | ~$0.87 |
| Parameter sweep | 1 vCPU, 2GB | ~30 sec | 100 | ~$0.09 |
| Full exploration | 2 vCPU, 4GB | ~3 min | 200 | ~$2.10 |

## Requirements

- Claude Code v1.0+
- GCP project with Cloud Run Jobs API enabled
- gcloud CLI authenticated
- Docker installed locally

## Limitations

- CPU-only. For GPU workloads, use Vertex AI Training or GKE with GPU node pools.
- Not for long-running HTTP services. Use Cloud Run services instead.
- Not for Cloud Functions, Kubernetes, or non-Python workloads.
- Requires a GCP project with Cloud Run, Cloud Build, and GCS APIs enabled.
- Maximum 10,000 tasks per job execution.

## Related skills

- [causal-impact-campaign](https://github.com/wan-huiyan/causal-impact-campaign) -- Measure causal impact of marketing campaigns using Bayesian structural time series. Pairs with this skill for running multi-spec analysis at scale.
- [permutation-validation](https://github.com/wan-huiyan/permutation-validation) -- Validate causal inference results with empirical permutation tests. Pairs with this skill for parallel permutation execution on Cloud Run.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-03 | Initial release |

## License

[MIT](LICENSE)
