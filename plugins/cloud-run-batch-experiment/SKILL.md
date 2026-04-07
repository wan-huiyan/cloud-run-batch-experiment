---
name: cloud-run-batch-experiment
version: "1.0.0"
description: |
  Deploy and run batch Python experiments on GCP Cloud Run Jobs with parallel execution and
  result collection via GCS. Use this skill whenever the user wants to run experiments on Cloud
  Run, deploy a batch job to GCP, run N parallel Python tasks, execute a parameter sweep or
  sensitivity analysis on cloud infrastructure, or scale local experiments to the cloud. Trigger
  on phrases like "run on Cloud Run", "batch experiment", "parallel jobs on GCP", "deploy to
  Cloud Run", "run N experiments in parallel", "scale this to the cloud", "parameter sweep on
  GCP", "sensitivity analysis on cloud", "run permutation tests on Cloud Run", "Cloud Run Jobs",
  or "gcloud run jobs". Also trigger when the user has a Python script that takes too long locally
  and wants to parallelize it across cloud workers. This skill covers the full pattern: Dockerfile,
  Cloud Build, gcloud run jobs execute with --tasks, env var routing, GCS result collection, and
  cost estimation.
  NOT for: Cloud Run services (long-running HTTP servers), Cloud Functions, Kubernetes/GKE
  deployments, or non-Python workloads.
input: |
  - A Python script that runs one experiment given parameters (via env vars or CLI args)
  - A GCP project with Cloud Run, Cloud Build, and GCS enabled
  - Experiment parameters: what varies across runs (e.g., random seeds, config IDs, dates)
output: |
  - Deployed Cloud Run Job with the experiment container
  - Parallel execution of N tasks
  - Results collected in GCS bucket, one file per task
  - Summary of results aggregated locally
---

# Cloud Run Batch Experiment

This skill deploys Python experiments to GCP Cloud Run Jobs for parallel batch execution. The
pattern is simple: package your script in a Docker container, deploy it as a Cloud Run Job, and
run N parallel tasks — each picking up its own parameters via environment variables.

This is ideal for parameter sweeps, sensitivity analyses, permutation tests, and any workload
where you need to run the same script with different inputs many times.

## When to Use

- Running 10+ independent experiments that each take 1-60 minutes
- Local execution would take hours but individual runs are independent
- You need reproducible, containerized execution environments
- The experiments are CPU-bound (not GPU — use Vertex AI or GKE for GPU workloads)

## The Pattern

```
1. Write runner script → 2. Create Dockerfile → 3. Deploy with Cloud Build
→ 4. Execute with --tasks=N → 5. Collect results from GCS
```

## Step 1: Write the Runner Script

The runner script reads `CLOUD_RUN_TASK_INDEX` (0-indexed) to determine which experiment to run,
and writes results to GCS.

```python
#!/usr/bin/env python3
"""cloud_run_task.py — runs one experiment per Cloud Run task."""
import os
import json
from google.cloud import storage

# Cloud Run provides CLOUD_RUN_TASK_INDEX (0, 1, 2, ...)
TASK_INDEX = int(os.environ.get('CLOUD_RUN_TASK_INDEX', '0'))
TASK_COUNT = int(os.environ.get('CLOUD_RUN_TASK_COUNT', '1'))

# Your experiment parameters — passed via env vars
SPEC_ID = os.environ.get('SPEC_ID', 'default')
GCS_BUCKET = os.environ.get('GCS_BUCKET', 'my-experiment-bucket')
GCS_PREFIX = os.environ.get('GCS_PREFIX', 'experiments/run-001')

# Map TASK_INDEX to experiment config
EXPERIMENT_CONFIGS = [
    {"name": "config_a", "param1": 7, "param2": "mask_xmas"},
    {"name": "config_b", "param1": 14, "param2": "mask_nov_jan"},
    # ... one per task
]

def run_experiment(config):
    """Your experiment logic here. Returns a results dict."""
    # ... do the work ...
    return {"config": config["name"], "metric": 42.0, "status": "success"}

def upload_to_gcs(data, bucket_name, blob_path):
    """Upload JSON results to GCS."""
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_path)
    blob.upload_from_string(json.dumps(data, indent=2), content_type='application/json')
    print(f"Uploaded results to gs://{bucket_name}/{blob_path}")

def main():
    if TASK_INDEX >= len(EXPERIMENT_CONFIGS):
        print(f"Task {TASK_INDEX} has no config (only {len(EXPERIMENT_CONFIGS)} configs). Exiting.")
        return

    config = EXPERIMENT_CONFIGS[TASK_INDEX]
    print(f"Task {TASK_INDEX}/{TASK_COUNT}: running {config['name']}")

    results = run_experiment(config)

    # Write results to GCS
    blob_path = f"{GCS_PREFIX}/task_{TASK_INDEX:03d}_{config['name']}.json"
    upload_to_gcs(results, GCS_BUCKET, blob_path)

if __name__ == '__main__':
    main()
```

### Alternative: Environment Variable Routing

For simpler cases, pass the experiment ID via a single env var and let the script look up its config:

```python
EXPERIMENT_ID = os.environ.get('EXPERIMENT_ID', 'A0')
# Script internally maps EXPERIMENT_ID to full config
```

This is useful when you want to submit jobs one at a time with different env vars rather than
using the --tasks parallelism.

## Step 2: Create the Dockerfile

Keep it minimal. The container only needs to run your Python script:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy experiment code
COPY . .

# Entry point — Cloud Run will invoke this
CMD ["python", "cloud_run_task.py"]
```

### Important Dockerfile Considerations

**For PyMC/CausalPy workloads**, add g++ for PyTensor C compilation — without it, MCMC sampling
is ~100x slower:

```dockerfile
RUN apt-get update && apt-get install -y g++ && rm -rf /var/lib/apt/lists/*
```

**For tfcausalimpact + CausalPy**, you may need separate images due to numpy version conflicts:
- Image A: `numpy<2.0` + `tfcausalimpact` + `tensorflow`
- Image B: `numpy>=2.0` + `causalpy` + `pymc`

## Step 3: Create .gcloudignore

Cloud Build uses `.gcloudignore` to decide what to upload. Critical: data files excluded by
`.gitignore` may need to be INCLUDED for Cloud Build:

```
# .gcloudignore
.git
.github
__pycache__
*.pyc
.venv
node_modules

# DO NOT ignore data files that the experiment needs:
# !data/features_enriched.csv  ← uncomment if needed
```

If your experiment reads data from GCS or BigQuery at runtime, you don't need to bundle data
files. But if it reads from local CSV, make sure `.gcloudignore` doesn't exclude it.

## Step 4: Deploy with Cloud Build

```bash
# Build and push the container image
gcloud builds submit \
  --tag europe-west2-docker.pkg.dev/PROJECT_ID/REPO/IMAGE_NAME:latest \
  --region europe-west2

# Create the Cloud Run Job (first time only)
gcloud run jobs create EXPERIMENT_JOB \
  --image europe-west2-docker.pkg.dev/PROJECT_ID/REPO/IMAGE_NAME:latest \
  --region europe-west2 \
  --memory 4Gi \
  --cpu 2 \
  --task-timeout 3600 \
  --max-retries 1 \
  --set-env-vars "GCS_BUCKET=my-bucket,GCS_PREFIX=experiments/run-001"

# Or update an existing job's image
gcloud run jobs update EXPERIMENT_JOB \
  --image europe-west2-docker.pkg.dev/PROJECT_ID/REPO/IMAGE_NAME:latest \
  --region europe-west2
```

## Step 5: Execute with Parallel Tasks

```bash
# Run 50 parallel tasks
gcloud run jobs execute EXPERIMENT_JOB \
  --region europe-west2 \
  --tasks 50 \
  --set-env-vars "SPEC_ID=A2.4,GCS_PREFIX=experiments/run-001"

# Monitor execution
gcloud run jobs executions list --job EXPERIMENT_JOB --region europe-west2
gcloud run jobs executions describe EXECUTION_NAME --region europe-west2
```

Each task gets `CLOUD_RUN_TASK_INDEX` (0 to N-1) and `CLOUD_RUN_TASK_COUNT` (N) automatically.

### Submitting Multiple Independent Jobs

For experiments with different env vars (not just different task indices):

```bash
for spec_id in A0 A1 A2 A2.4 A3; do
  gcloud run jobs execute EXPERIMENT_JOB \
    --region europe-west2 \
    --tasks 1 \
    --set-env-vars "EXPERIMENT_ID=${spec_id},GCS_PREFIX=experiments/phase-a/${spec_id}" \
    --async
done
```

The `--async` flag returns immediately so jobs run in parallel.

## Step 6: Collect Results from GCS

```python
from google.cloud import storage
import json

def collect_results(bucket_name, prefix):
    """Download all result JSON files from a GCS prefix."""
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blobs = list(bucket.list_blobs(prefix=prefix))

    results = []
    for blob in blobs:
        if blob.name.endswith('.json'):
            data = json.loads(blob.download_as_text())
            results.append(data)
            print(f"  {blob.name}: {data.get('status', 'unknown')}")

    print(f"\nCollected {len(results)} results from gs://{bucket_name}/{prefix}")
    return results
```

Or use gsutil for quick inspection:

```bash
gsutil ls gs://my-bucket/experiments/run-001/
gsutil cat gs://my-bucket/experiments/run-001/task_000_config_a.json
```

## Cost Estimation

Cloud Run Jobs pricing (europe-west2):
- vCPU: ~$0.000024/vCPU-second
- Memory: ~$0.0000025/GiB-second
- No charge when idle (only pay for actual execution time)

**Example costs:**
| Workload | Config | Per-task time | N tasks | Total cost |
|---|---|---|---|---|
| BSTS permutation | 2 vCPU, 4GB | ~2 min | 50 | ~$7.50 |
| CausalPy MCMC | 2 vCPU, 4GB | ~5 min | 50 | ~$18 |
| Parameter sweep | 1 vCPU, 2GB | ~30 sec | 100 | ~$2 |
| Full exploration | 2 vCPU, 4GB | ~3 min | 200 | ~$36 |

## Timeout Guidance

- Default timeout: 600s (10 min) — too short for MCMC workloads
- BSTS (tfcausalimpact): 1800s (30 min) is usually sufficient
- CausalPy (PyMC NUTS): 3600s (60 min) recommended — sampling can be slow
- Set via `--task-timeout` on job creation or update

## Common Pitfalls

### TensorFlow Retracing Warnings
If running tfcausalimpact, you may see "Function was called with different argument types"
warnings. These are harmless but noisy. Suppress with:
```python
import tensorflow as tf
tf.get_logger().setLevel('ERROR')
```

### Memory Limits
BSTS with many covariates can exceed 4GB. If tasks are OOM-killed, increase to 8GB:
```bash
gcloud run jobs update JOB --memory 8Gi --region europe-west2
```

### Task Index Off-by-One
`CLOUD_RUN_TASK_INDEX` is 0-indexed. If you have 50 experiment configs, use `--tasks 50`
and handle indices 0-49. Always guard against index-out-of-bounds.

## Composability

- Pairs with `causal-impact-campaign` for running multi-spec analysis at scale
- Pairs with `permutation-validation` for parallel permutation tests
- Use `gcp-pipeline-cost-analysis` to estimate costs before large runs
