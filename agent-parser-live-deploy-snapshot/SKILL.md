---
name: agent-parser-live-deploy-snapshot
description: Use when resuming the in-flight 2026-03-25 AWS rollout and you need the exact current deployment snapshot without loading the full migration history.
---

# Agent Parser Live Deploy Snapshot

Use this skill only for the current in-flight rollout continuation.

## Snapshot

- `origin/main` includes:
  - `eeb6f28` unified scan refactor
  - `f0e1667` model-api vLLM sidecar layout
  - `9b97fad` deploy/render defaults cleanup
- GPU host root volume had to be increased to `200 GiB` because `vllm` and `model-api` image layers together could exhaust the old host disk during pull/register
- one ASG recycle still launched hosts with old ECS bootstrap user-data (`systemctl enable --now ecs`), which left `ecs.service` inactive until the launch template was corrected and hosts were recycled again
- model-api caller endpoint is the internal ALB, not legacy internal hostnames
- sidecar rollout should require `model-api` redeploy only
- app needs fresh tasks whenever `DATABASE_URL` secret content changes
- one healthy model-api task proved the EFS-backed HF cache is populated, and the healthy container exposed a real snapshot path under `/root/.cache/huggingface/hub/models--Qwen--Qwen3.5-9B/snapshots/`

## What To Treat As Current

- `model-api` is the service most likely still in active rollout
- one private-compute subnet can be healthy while the other still times out to `huggingface.co`; do not assume both AZ paths are equivalent just because one task is green
- `scheduler` should already be pointing at the ALB if latest rendered task def is active
- `app` DB issues after secret edits are separate from model-api serving issues

## Resume Checklist

1. confirm live `PRIMARY` task definition per service
2. confirm ECS container instances are registered after any ASG/LT recycle
3. confirm model-api TG target health
4. confirm `vllm` sidecar logs
5. if one task is healthy, use that container to derive the real EFS HF cache path before changing `--model`
6. only then decide whether caller redeploy is still needed

## Do Not Re-load

- full migration history
- old `model-api`/`model-api-gpu1` endpoint debugging unless logs still show them
- generic Fargate migration guidance unless the failure bucket is unknown
