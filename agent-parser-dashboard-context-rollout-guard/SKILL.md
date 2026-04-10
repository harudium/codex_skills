---
name: agent-parser-dashboard-context-rollout-guard
description: Use when Threat Stream/Conversation Context optimization is needed and rollout safety matters, especially when app tasks fail health checks without explicit errors during dashboard-heavy traffic.
---

# Agent Parser Dashboard Context Rollout Guard

Use this skill when dashboard context loading was optimized (or is about to be optimized) and a bad rollout could destabilize `agent-parser-app`.

Typical symptoms:

- message click keeps spinning on conversation context
- filter changes then quick message reselect causes long stalls
- ECS keeps replacing app tasks with `failed container health checks`
- startup logs look normal, but tasks still flap
- new task definition fails while old one remains healthy

## History Baseline

Use commit `735946f` as the known-safe reference pattern.

What it teaches:

- avoid event-loop blocking in heavy sanitize/format paths
- avoid duplicate enrichment triggers caused by stale dependencies
- keep one clear fast path for first paint, then background enrichment

## Non-Negotiable Rules

1. Keep hot API routes event-loop safe.
2. Move CPU-heavy or large-list formatting/sanitize steps to `asyncio.to_thread(...)`.
3. In frontend context loading, do not let enrichment run from a separate stale-dependent effect that can retrigger on old state.
4. Tie enrichment scheduling to successful fast-context completion for the same selection.
5. Ensure timer/controller cleanup cancels both pending and in-flight enrichment when selection changes.

## Backend Pattern (from 735946f)

Target file:

- `app/routers/events.py`

Required shape:

- `threat_stream_query` and `threat_stream_page` sanitize path should run off-loop:
  - `await asyncio.to_thread(_sanitize_snapshot_page, ...)`
- `events_recent_delta` expensive sanitize/compact loop should run off-loop:
  - wrap list sanitize in helper and call `await asyncio.to_thread(helper, recent_logs)`

Why:

- these paths often include Redis/format-heavy work under load
- keeping them on-loop can starve health checks and produce false "unhealthy" flaps

## Frontend Pattern (from 735946f)

Target file:

- `dashboard/src/pages/ThreatStreamV3Page.tsx`

Required shape:

- fast fetch (`includeEnrichment: false`) first
- schedule enrichment only inside the fast fetch success path
- use dedicated refs:
  - `conversationEnrichmentControllerRef`
  - `conversationEnrichmentTimerRef`
- on selection/context effect rerun:
  - clear pending timer
  - abort in-flight enrichment controller
- remove stale-trigger dependency (for example broad `conversationContext` dependency) from enrichment scheduling effect

Why:

- prevents duplicate enrichment storms
- prevents connection pool saturation
- avoids "infinite loading" from repeated race-triggered fetches

## Rollout Guardrails

Before deploying dashboard optimization to app:

1. verify task definition diff contains only intended files
2. deploy with at least 2 healthy tasks retained during rollout
3. avoid one-task service while validating a new context-loading change
4. verify subnet IP headroom in both AZs is sufficient for rolling replacement
5. check NLB target health transitions per AZ during rollout

Do not conclude "AWS issue" until you compare old vs new revision health behavior.

If old revision is healthy and only new revision flaps, treat as app/runtime/code-path risk first.

## Validation Checklist

After patch and before merge:

1. `python3 -m py_compile app/routers/events.py`
2. dashboard build succeeds (`npm run build`)
3. manual flow:
   - load stream
   - toggle `blocked` / `attachments`
   - immediately click another message
   - verify context fast paint appears without long spinner
4. monitor ECS events for replacement loops
5. confirm no event-loop-blocking warnings and no health-check flap burst

## Output Contract

Return:

- whether the issue was event-loop blocking, enrichment trigger storm, or both
- exact backend and frontend guardrails applied
- rollout proof points (task health, ECS events, user flow)
- remaining operational risk (for example subnet IP headroom)
