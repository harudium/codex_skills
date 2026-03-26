---
name: agent-parser-virtual-canary-deploy
description: Use when continuing the 2026-03-26 model-api vLLM sidecar rollout where compute-2b is the only known-good outbound placement, compute-2a still fails Hugging Face outbound, EFS already contains model artifacts, and the next step is a direct virtual-subnet canary from devdi without helper wrappers.
---

# Agent Parser Virtual Canary Deploy

Use this skill only for the current model-api canary decision.

## Current Facts

- only `model-api` rollout matters right now; `app` and `scheduler` are intentionally down
- keep one known-good `model-api` task alive on `private_compute_2b` until `virtual` is proven
- `aws ecs execute-command` from `devdi` requires the local Session Manager plugin; a missing plugin is a local workstation blocker, not a task/runtime signal
- the long-lived known-good control task was launched without ECS Exec enabled, so do not plan on shelling into that task directly
- there is no spare GPU capacity for an additional debug task; preserve the control task and inspect from the backing EC2 host instead
- known compute subnets:
  - `subnet-0af4a9e336878ee2a` (`private_compute_2a`)
  - `subnet-0169a334caa29e008` (`private_compute_2b`)
- target virtual subnets:
  - `subnet-0f705226bc734fc46` (`virtual-2a`)
  - `subnet-07d3762965d56bf9c` (`virtual-2b`)
- service security group: `sg-0211049d27ab2d3d5`
- capacity provider: `agent-parser-gpu-ec2`
- EFS file system: `fs-0f194741f7f2c89ce`
- EFS already has roughly `18 GiB` of model artifacts
- one task on `private_compute_2b` can reach Hugging Face and start
- a sibling placement on `private_compute_2a` still times out to Hugging Face
- live proof from the healthy `private_compute_2b` task:
  - `vllm` uses `HF_HOME=/root/.cache/huggingface`
  - `refs/main` currently points to snapshot `c20223635762e1c871ad0ccb60c8ee5ba337b9a`
  - `config.json` exists under that exact snapshot path
  - older shell history values such as `68a6be06-c973-4e79-9e36-ccc3a6569bf1` should be treated as stale until proven live again
- later proof from command invocation:
  - `/root/.cache/huggingface` is mounted as `nfs4 127.0.0.1:/huggingface`, which is consistent with an EFS-backed mount exposed inside the container
- final proof:
  - `agent-parser-model-api:23` can start successfully on `virtual-2a`
  - both `vllm` and `model-api` containers reached `RUNNING`
  - a clean service redeploy on `:23` later converged to one running service task in `virtual-2a`
  - the working service task kept `vllm` `HEALTHY`; `model-api` may still report ECS health as `UNKNOWN` even while the task is serving

## Preferred Canary Strategy

- do not hardcode a snapshot hash into `--model`
- keep `--model` as `Qwen/Qwen3.5-9B`
- keep `HF_HOME=/root/.cache/huggingface`
- add `HF_HUB_OFFLINE=1`
- add `TRANSFORMERS_OFFLINE=1`
- once `:23` is proven, prefer a clean service-only rollout over keeping a parallel standalone debug task alive

## Decision Frame

- do not switch the main service to `virtual` first
- do not discard the working `private_compute_2b` task before `virtual` proof exists
- without spare GPU, accept that the first `virtual` proof is a service replacement rather than a parallel canary
- if no spare GPU exists, use SSM `send-command` to the backing EC2 instance and inspect the live `vllm` container with `docker exec`
- run one virtual subnet at a time; start with `virtual-2b`, then repeat on `virtual-2a`
- sidecar Docker networking is a lower-priority hypothesis than subnet/AZ path drift because `awsvpc` tasks share the task ENI path
- the canary goal is to prove whether the same task definition can start on `virtual`, not to finish the full cutover
- standalone `family:agent-parser-model-api` tasks do not get adopted by the ECS service later; they must be removed before a clean service rollout
- standalone tasks launched with the same capacity provider can still influence managed scaling and lead to extra GPU instances during debugging

## Direct Workflow

1. on `devdi`, capture the current live task definition from `agent-parser-model-api`
2. register a new task definition revision only if the `vllm` environment must change
3. if proof is still needed, run a single standalone task in `virtual-2a` or `virtual-2b` and verify `vllm` plus `model-api` startup
4. once proof exists, stop the standalone task before touching the service again
5. scale the service to `0`, stop leftover service tasks, and wait until the service is empty and stable
6. update the service to the proven task definition and restore `desired-count=1`
7. only after clean service convergence should you scale beyond one task

## Direct Command Pattern

Use direct `aws ecs` CLI from `devdi`.

- avoid `deployment/deploy.sh`
- avoid render/deploy helpers unless the task definition itself must change
- prefer querying the live service/task definition instead of re-deriving values from memory

## Success Criteria

- the replacement `virtual-2b` canary reaches `RUNNING`
- `vllm` and `model-api` containers both stay healthy long enough to inspect
- the failure mode is clearly classified as one of:
  - pure outbound drift
  - EFS/path/bootstrap drift
  - sidecar runtime failure unrelated to subnet egress

## Output Contract

Return:

- the canary task ARN
- the actual subnet/AZ placement
- whether startup failure happened before or after `vllm` model initialization
- whether the next step should be `service cutover`, `offline/env fix`, or `network diff`
