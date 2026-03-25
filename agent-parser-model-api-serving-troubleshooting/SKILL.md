---
name: agent-parser-model-api-serving-troubleshooting
description: Use when model-api is deployed but scan traffic still fails, especially ALB/TG health, EFS path issues, vLLM sidecar startup, `/v1/scan` 500s, or legacy model-selection 404s.
---

# Agent Parser Model-API Serving Troubleshooting

Use this skill when the model-api task exists but serving is still bad.

## Failure Buckets

- ALB/TG registration or health
- EFS mount/subdirectory/bootstrap
- vLLM sidecar missing or unhealthy
- subnet/AZ-specific outbound path drift
- legacy model selection mismatch
- caller endpoint drift

## Current Contract

- callers should use the internal ALB on port `8001`
- common path is `POST /v1/scan`
- `model-api` should talk to local sidecar `vllm`

## High-Signal Symptoms

- TG has zero targets:
  - service not attached to TG
- `context deadline exceeded` on mount:
  - EFS SG/NFS path
- `No such file or directory` on mount:
  - missing `/huggingface` or `/adapters`
- one task is healthy but another task on a different subnet has `vllm=UNHEALTHY`, `model-api=PENDING`, and repeated `huggingface.co` timeouts:
  - suspect subnet/AZ-specific outbound drift before changing app code
- `HFValidationError` after setting `--model` to an absolute path:
  - the supplied path does not actually exist inside the running container, so vLLM fell back to HF repo-id validation
- 404 on `/v1/models/<legacy>/scan`:
  - legacy selection/routing path still active
- 500 on `/v1/scan`:
  - likely vLLM sidecar or upstream serving issue

## EFS Cache Guidance

- `HF_HOME` is EFS-backed, but `--model Qwen/Qwen3.5-9B` still allows metadata lookups unless offline mode is forced
- if one healthy task already populated the cache, prefer:
  - keep `--model` as the repo id
  - set `HF_HUB_OFFLINE=1`
  - set `TRANSFORMERS_OFFLINE=1`
  - keep `HF_HOME` on the EFS mount
- if you must use a local snapshot path, derive the exact path from a healthy running `vllm` container instead of transcribing an EFS identifier by hand

## What To Prove

- `loadBalancers` on ECS service is non-empty
- target group target is registered and healthy
- model-api logs show runtime/queue start
- vllm logs show actual model serving startup
- if only one task is bad, compare subnet and ENI placement before concluding the task definition is wrong

## Output Contract

Return:

- which serving layer is broken
- the first missing proof in the chain
- whether caller redeploy is even relevant yet
