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
- source of truth repo is `/home/kevin/projects/agent_parser` on `pgpu`; `/Users/1110847/Documents/AWS MIGRATION/agent_parser_patch` is only a local reference snapshot and should not be treated as the production worktree
- GPU host root volume had to be increased to `200 GiB` because `vllm` and `model-api` image layers together could exhaust the old host disk during pull/register
- one ASG recycle still launched hosts with old ECS bootstrap user-data (`systemctl enable --now ecs`), which left `ecs.service` inactive until the launch template was corrected and hosts were recycled again
- model-api caller endpoint is the internal ALB, not legacy internal hostnames
- sidecar rollout should require `model-api` redeploy only
- app needs fresh tasks whenever `DATABASE_URL` secret content changes
- one healthy model-api task proved the EFS-backed HF cache is populated, and the healthy container exposed a real snapshot path under `/root/.cache/huggingface/hub/models--Qwen--Qwen3.5-9B/snapshots/`
- `agent-parser-model-api:23` remains the known-good AWS model-api baseline for EFS-backed HF cache behavior; `:24` drifted back toward Hugging Face lookups because the render path was still able to omit offline envs
- `model-api` renderer/deployer hardening landed in `origin/main` via `7449ae3`, `74383fe`, and `0d07dd8`: EFS mode now defaults `HF_HUB_OFFLINE=1` / `TRANSFORMERS_OFFLINE=1`, normalizes `None/null` secret selectors to empty, and `deploy.sh model-api` re-renders plus validates offline envs before registering a task definition
- the safest immediate rollback/redeploy pattern is still: treat live `agent-parser-model-api:23` as source of truth and register a new revision by swapping only the `model-api` image tag; the renderer does not literally inherit ECS `:23` unless you clone it yourself
- multi-AZ is now active for model-api and scheduler, but `scan_reasoning` `504` warnings pre-dated multi-AZ and should be treated as a pre-existing quality/performance issue, not a new HA regression
- one intermittent scheduler-to-model-api `504 Gateway Timeout` can be persisted as `rescan_error` while the original fast-stage decision is still stored; historical rows can stay if the error is not still recurring
- queue saturation is now directly evidenced in `/metrics`: `model_api_queue_timeouts_total{task_type="scan_reasoning"}`, `pending_reasoning`, `oldest_job_age_seconds`, and `deadletter` can rise even when the ALB target is healthy
- the important topology correction is that AWS model-api tasks still use a shared `REDIS_URL` queue; the dominant AWS/pgpu difference is not “local per-instance queue”, but the front door and caller topology: pgpu uses NGINX `least_conn` with `300s` proxy timeouts plus a direct two-endpoint pool, while AWS callers currently collapse to a single internal ALB URL
- `origin/main` now also includes `7c4a601`, which reduces scheduler ledger polling overhead and changes the scheduler render default to `COMPLIANCE_ANALYSIS_JOB_CLAIM_LIMIT=128`
- rendered scheduler task definitions can still lag that source default; if `deployment/rendered/ecs-task-definition-scheduler.prod.json` still says `32`, rerender or explicitly pass `COMPLIANCE_ANALYSIS_JOB_CLAIM_LIMIT=128` before deploy
- direct-vs-ledger synthetic comparison on `pgpu` showed the current Aurora-backed analysis ledger adds roughly `+1.9s` to `+2.6s` fixed overhead per `256` items, so current AWS slowdown should be treated as scheduler ledger/poll overhead before blaming raw `/v1/scan` latency
- the current distributed run lock does not do a short retry loop; an instance that loses the lock normally retries only on its next `30` minute schedule tick unless the process restarts or a manual run is triggered
- app healthcheck failures were traced to ECS/Docker liveness hitting `/health`; app now has a `/readyz` split in commit `ed39608`, but rendered task definitions must be regenerated before deploy because `deployment/rendered/ecs-task-definition-app.prod.json` can lag the template
- app dashboard is served by the same FastAPI process on port `8000`; current public exposure work uses `agent-parser-public-alb` plus `agent-parser-public-tg`
- Redis `hash value is not an integer` warnings now most likely come from stale `ta:*:summary` or `ta:overall:summary` preagg keys because startup only rebuilds stats-only preagg, not TA preagg
- GitLab Runner deployment scaffolding is drafted in the pgpu worktree via `.gitlab-ci.yml` and `deployment/GITLAB_RUNNER_DEPLOYMENT.md`: `main` is intended to run `validate` plus dev image builds automatically, while dev deploy / prod promote / prod deploy stay manual
- do not assume CI can infer the final deploy target from changed files alone; service builds can be narrowed automatically, but the actual deploy decision should stay explicit because shared modules, contracts, and env-only changes can affect multiple services
- one EC2 can host multiple GitLab runners, but that is still logical separation only; prod safety should come from tags, protected refs, and distinct job routing rather than assuming host-level isolation

## What To Treat As Current

- `model-api` is the service most likely still in active rollout
- one private-compute subnet can be healthy while the other still times out to `huggingface.co`; do not assume both AZ paths are equivalent just because one task is green
- `scheduler` has already been redeployed with richer model-api timeout diagnostics; do not redeploy it again just to refresh model-api image changes unless scheduler-specific code changed
- `model-api` image-only redeploys should leave scheduler/app untouched and preferably clone the known-good live task definition when you need zero-drift recovery
- if local `docker exec ... curl http://127.0.0.1:8001/v1/scan` timing on the model-api host is close to `pgpu`, treat scheduler ledger/poll/post-processing as the first bottleneck instead of model-api serving
- scheduler throughput changes tied to `COMPLIANCE_ANALYSIS_JOB_CLAIM_LIMIT` require rerender + redeploy; changing the render script default alone does not update an already rendered task definition
- `app` DB issues after secret edits are separate from model-api serving issues
- public app access now depends more on listener/TG correctness than on certificate attachment alone; if `HTTP:80` works but `HTTPS:443` returns `503`, first confirm the `443` listener points to the same healthy TG ARN as the app tasks
- ALB target groups with the same visible name can still be different resources; always verify the exact ARN before assuming `80` and `443` share the same backend
- the desired public ALB pattern is `HTTP:80` redirecting to `HTTPS:443`, with `HTTPS:443` forwarding to the single healthy app TG
- a private certificate still gives real TLS termination at the ALB, but browsers can still warn if users hit the raw `*.elb.amazonaws.com` hostname or if the private CA is not trusted on the client
- dashboard frontend uses same-origin `/v1/...` API calls, so a public dashboard task also exposes the same host's app/API surface

## Resume Checklist

1. confirm you are working from `/home/kevin/projects/agent_parser` on `pgpu`
2. confirm live `PRIMARY` task definition per service
3. confirm ECS container instances are registered after any ASG/LT recycle
4. confirm model-api TG target health and scheduler `MODEL_API_URL(S)` envs
5. if app template or healthcheck changed, rerender `deployment/rendered/ecs-task-definition-app.prod.json` and verify it contains `/readyz` before deploy
6. if one model-api task is healthy, use that container to derive the real EFS HF cache path before changing `--model`
7. only then decide whether caller redeploy is still needed

## Do Not Re-load

- full migration history
- old `model-api`/`model-api-gpu1` endpoint debugging unless logs still show them
- generic Fargate migration guidance unless the failure bucket is unknown
- local snapshot patches under `/Users/1110847/Documents/AWS MIGRATION/agent_parser_patch` as if they were the live repo
