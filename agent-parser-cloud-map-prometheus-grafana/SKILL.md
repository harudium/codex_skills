---
name: agent-parser-cloud-map-prometheus-grafana
description: Use when wiring Prometheus and Grafana to agent_parser services on AWS with Cloud Map private DNS, especially for ECS service discovery, Prometheus scrape targets, Grafana datasource wiring, and SG/ALB decisions for internal vs user-facing access.
---

# Agent Parser Cloud Map Prometheus Grafana

Use this skill when building or troubleshooting the monitoring path for `agent_parser` on AWS with Cloud Map.

Typical situations:

- Prometheus needs to scrape `app` and `model-api` without adding another internal ALB
- a private compute subnet has no spare CIDR for a new ALB
- Grafana should read Prometheus inside the VPC but may still be exposed to users through an ALB
- the team needs a clear boundary between service discovery, scraping, visualization, and public access

## Core Rule

Treat monitoring as separate services, not as hidden features inside `app`.

- `app` exposes metrics
- `model-api` exposes metrics if needed
- `prometheus` is its own ECS service that scrapes targets
- `grafana` is its own ECS service that reads Prometheus
- Cloud Map provides stable private DNS names so Prometheus can scrape without per-task IP hardcoding

Do not treat Prometheus or Grafana as if they are already “inside app” just because the app exports metrics.

## Recommended Topology

Preferred path when internal ALB capacity is constrained:

1. create a Cloud Map private DNS namespace
2. register ECS services such as `agent-parser-app` and `agent-parser-model-api`
3. run Prometheus as a separate ECS service in the same VPC/subnets
4. configure scrape targets with Cloud Map DNS names:
   - `agent-parser-app.<namespace>:8000`
   - `agent-parser-model-api.<namespace>:8001`
5. run Grafana as a separate ECS service
6. configure Grafana datasource to point to Prometheus by private service DNS or internal ALB

This avoids per-task IP drift and avoids creating an extra internal ALB just for scraping.

## When To Use Cloud Map vs Internal ALB

Use Cloud Map when:

- Prometheus is inside the same VPC
- you mainly need machine-to-machine discovery
- you want to avoid new ALB/TG resources
- you are scraping ECS services, not serving browser users

Use an internal ALB when:

- multiple callers outside the ECS service mesh need a stable HTTP endpoint
- you need richer listener/routing behavior
- you need an endpoint for many internal clients, not just Prometheus

For Prometheus scraping alone, Cloud Map is usually the cleaner default.

## ECS Service Discovery Pattern

For each scraped ECS service:

- enable Service Discovery on the ECS service
- attach it to the private DNS namespace
- register the actual container port
- use the service DNS name in Prometheus config

Important:

- the registered container/port must match the task definition container that actually serves metrics
- do not assume Service Discovery magically picks the right container if multiple containers exist
- unhealthy tasks must be removed from DNS by ECS/Cloud Map task health propagation

## Security Group Rules

Model the traffic explicitly.

Minimum required flows:

- `prometheus-sg -> app-sg` on `8000`
- `prometheus-sg -> model-api-sg` on `8001` if scraping model-api directly
- `grafana-sg -> prometheus-sg` on Prometheus port, often `9090`

If Grafana is exposed through an ALB:

- `alb-sg -> grafana-sg` on Grafana container port
- user/on-prem/client CIDR -> `alb-sg` on `443` or `80/443`

Do not open app/model-api SGs broadly just because Prometheus needs access.

## Prometheus Contract

Prometheus should scrape private DNS names, not task IPs copied by hand.

Common target patterns:

- `PROMETHEUS_APP_TARGETS=agent-parser-app.<namespace>:8000`
- `PROMETHEUS_MODEL_API_TARGETS=agent-parser-model-api.<namespace>:8001`

If the stack uses environment-driven rendered config, keep the target contract stable and inject only the DNS names.

Prometheus should be considered healthy only when:

- the service itself is up
- target status pages show app/model-api as `UP`
- scrape errors are not continuously accumulating

## Grafana Contract

Grafana should read Prometheus as its datasource, not scrape app/model-api directly.

Preferred pattern:

- Grafana datasource URL points to Prometheus private service endpoint
- dashboards only depend on Prometheus metric names
- any user-facing Grafana access happens through a dedicated ALB, not through app ALBs

If Grafana is only for internal operators, keep it private and skip a public ALB.

## Common Failure Buckets

- Cloud Map service exists but ECS service is not actually registered to it
- wrong container port registered in Service Discovery
- Prometheus SG cannot reach app/model-api SG
- Grafana cannot reach Prometheus even though Prometheus itself is healthy
- users expect Grafana to appear because app is deployed, but Grafana service was never created
- the team assumes Prometheus is “running with app” because metrics exist, but no Prometheus ECS service is deployed

## Validation Checklist

After setup:

1. confirm Prometheus and Grafana are separate ECS services
2. confirm the private DNS namespace exists
3. confirm each ECS service is registered in Cloud Map
4. resolve private DNS names from inside the Prometheus task/container
5. verify Prometheus target page shows `UP`
6. verify Grafana datasource health against Prometheus
7. if Grafana is user-facing, confirm ALB listener, TG, and SG chain separately from Prometheus scraping

## Output Contract

Return:

- whether Cloud Map is the right discovery layer
- which ECS services must exist separately
- exact traffic/SG relationships
- what Prometheus should scrape
- what Grafana should connect to
- the first missing proof if the monitoring path still fails
