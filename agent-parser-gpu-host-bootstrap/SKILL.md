---
name: agent-parser-gpu-host-bootstrap
description: Use when the ECS EC2 GPU host itself is suspect, especially container instance registration, GPU discovery, stale ecs-agent state, or permissions-boundary issues before model-api task placement.
---

# Agent Parser GPU Host Bootstrap

Use this skill only when the issue is below the ECS service layer.

## Trigger Signs

- capacity provider exists but usable container instances are missing
- ECS container instance has no GPU resource
- `nvidia-smi` works but ECS placement still fails
- stale/inactive container instance re-registration
- permissions boundary blocks `ecs:RegisterContainerInstance`

## Core Rules

- Debug host bootstrap before changing task definitions again.
- Treat `nvidia-smi` success as necessary but not sufficient.
- Inspect ECS GPU as `STRINGSET` UUIDs, not just integer counters.
- If local ECS state is stale, clear `/var/lib/ecs/data/*` before re-testing service placement.

## Known Incident Patterns

- one host had no `/var/lib/ecs/gpu/nvidia-gpu-info.json`
- another host had GPU but re-registration failed until stale ECS state was cleared
- permissions boundary can silently override attached IAM policy fixes
- old root volumes were too small for `vllm` + `model-api` image pulls; `CannotPullContainerError ... no space left on device` required raising the GPU host root EBS volume to `200 GiB`
- an ASG recycle can still fail to register if the live launch template keeps old user-data with `systemctl enable --now ecs`; this can leave `ecs.service` `inactive (dead)` while cloud-init is still running
- preferred bootstrap pattern is `systemctl enable ecs` plus a delayed async `start/restart`, not `enable --now`
- ASGs created with `new-instances-protected-from-scale-in` may ignore `desired=0` until scale-in protection is removed or instances are explicitly terminated

## Output Contract

Return:

- whether the host failure is discovery, registration, or IAM boundary
- whether launch template user-data or root volume size must be corrected before another recycle
- what host-level fix is required before service-level deploy changes
