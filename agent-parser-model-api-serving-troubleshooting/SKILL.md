---
name: agent-parser-model-api-serving-troubleshooting
description: Use when model-api is deployed but scan traffic still fails, especially ALB/TG health, EFS path issues, vLLM sidecar startup, `/v1/scan` 500s, or legacy model-selection 404s.
---

# Agent Parser Model-API Serving Troubleshooting

Use this skill when the model-api task exists but serving is still bad.

## Failure Buckets

- ALB/TG registration or health
- internal ALB security-group ingress mismatch
- EFS mount/subdirectory/bootstrap
- vLLM sidecar missing or unhealthy
- subnet/AZ-specific outbound path drift
- legacy model selection mismatch
- caller endpoint drift
- vLLM context overflow without successful auto-adjust

## Current Contract

- callers should use the internal ALB on port `8001`
- common path is `POST /v1/scan`
- `model-api` should talk to local sidecar `vllm`
- prefer checking `/readyz` from inside the `model-api` container namespace, not from the EC2 host namespace
- `readyz` should expose `queue_enabled`, `queue_connected`, and `queue_worker_running`; record those before blaming caller code

## High-Signal Symptoms

- TG has zero targets:
  - service not attached to TG
- app/scheduler logs show `Inference API unreachable (all endpoints): All connection attempts failed` against the internal model-api ALB even though listener/TG are healthy:
  - internal model-api ALB SG likely does not allow the real app/scheduler task SG on port `8001`
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
- 500 on `/v1/scan` with vLLM text like `You passed ... input tokens ... maximum input length ...`:
  - context overflow, not generic app failure
- local `docker exec <model-api> curl http://127.0.0.1:8001/v1/scan` is fast but end-to-end scheduler throughput is still much worse than `pgpu`:
  - suspect scheduler ledger/poll/post-processing overhead before changing model-api serving
- `MODEL_API_QUEUE_ENABLED=false` in the AWS container while local `/v1/scan` timing is still fine:
  - note the runtime drift, but do not treat it as the root cause without caller-side timing proof

## Current Gotchas

- healthy listener + healthy TG is not enough; the internal ALB SG must explicitly allow the current app/scheduler task SGs on `8001`
- if callers hit the ALB but `ConnectError` persists, compare:
  - ALB SG allowed source groups
  - actual ECS service task SGs for `agent-parser-app` and `agent-parser-compliance-scheduler`
- current `origin/main` includes a model-api fix that:
  - detects vLLM sidecar by backend flavor rather than brittle base URL substring matching
  - parses both old and new vLLM context-overflow messages
  - retries with a smaller prompt instead of immediately surfacing a 500
- the paired manual re-scan HITL tagging change lives in `app`; if the goal is full behavior parity for truncated manual re-scan, deploy `app` as well as `model-api`

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
- internal ALB listener exists on `8001`
- internal ALB SG ingress allows the live app/scheduler task SGs
- model-api logs show runtime/queue start
- vllm logs show actual model serving startup
- if you are on the GPU EC2 host via SSM, use `docker exec` for localhost checks because `awsvpc` tasks do not share the host loopback
- prove whether local `/v1/scan` latency is bad or whether only caller end-to-end latency is bad; those point to different owners
- if `/v1/scan` returns 500, capture the first upstream error body before assuming task replacement is needed
- if only one task is bad, compare subnet and ENI placement before concluding the task definition is wrong

## Output Contract

Return:

- which serving layer is broken
- the first missing proof in the chain
- whether caller redeploy is even relevant yet
