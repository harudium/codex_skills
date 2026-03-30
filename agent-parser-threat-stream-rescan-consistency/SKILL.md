---
name: agent-parser-threat-stream-rescan-consistency
description: Use when Threat Stream rows, sidebars, summaries, or live feeds disagree after re-scan, especially for prescan-blocked rows that should transition to a new final state.
---

# Agent Parser Threat Stream Rescan Consistency

Use this skill when a re-scan result appears to update only part of Threat Stream behavior.

Typical symptoms:

- the sidebar card changes to `SAFE` but the row stays `BLOCK`
- `LLM Analysis` does not update after re-scan
- `PRE-SCAN ANALYSIS` still shows after a successful non-prescan re-scan
- a row briefly jumps to a different list/work snapshot and then returns
- blocked-only counts, prescan counts, or audit summaries disagree with the detail panel

## Core Rule

There must be exactly one current-state interpretation for a row.

- `metadata.context_analysis` is the current truth when present
- `metadata.prescan_context_analysis` is fallback only for old rows that have never been replaced by a newer current context
- legacy top-level `prescan_*` flags are compatibility fields, not the source of truth for post-rescan behavior

Do not solve this with row-specific exceptions, UI-only overrides, or extra cleanup branches.

## Required Invariants

- A re-scan is a row mutation, not a separate display-only overlay.
- The re-scan API should return the canonical updated row when possible.
- The frontend should replace the row from canonical server data, not partially merge risk/action fields into stale metadata.
- A non-prescan re-scan must stop emitting effective prescan state everywhere:
  - list filters
  - detail sidebar
  - live stream/SSE payloads
  - work snapshot rows
  - read-model/preagg summaries
  - audit metrics
  - DB-derived fields like `prescan_blocked` and `prescan_module`
- If a mutation occurs while viewing a work snapshot, rebuild the snapshot; do not keep reusing an immutable snapshot id as if it were live state.

## High-Risk Failure Pattern

The most common root cause is split truth across layers:

- backend mutates `context_analysis`
- frontend still reads `prescan_context_analysis`
- live/SSE compaction still copies legacy `prescan_*`
- preagg/audit logic still classifies by raw legacy flags
- DB sync recomputes `prescan_blocked` from stale metadata

If those are not unified, the same row can appear `SAFE` in one panel and `BLOCK/PRESCAN` in another.

## Canonical Fix Pattern

Start by adding or using a single helper layer for current-state interpretation.

Recommended responsibilities:

- `resolve_effective_context_analysis(event)`
- `is_effective_prescan_blocked(event)`
- `resolve_effective_prescan_module(event)`

Then route every consumer through that helper layer:

- backend filter logic
- Threat Stream read/sanitize logic
- workset/snapshot classification
- event/SSE compactors
- stats/preagg compaction
- audit metrics
- DB persistence-derived fields
- frontend row/detail rendering helpers

## Frontend Rules

- Re-scan should not force a live view into work context unless the user explicitly requested a snapshot-style action.
- If the current mode is work context and a mutation happens, rebuild the workset after the mutation.
- The detail panel should derive both `PRE-SCAN ANALYSIS` and `LLM Analysis` from the same canonical current context rule as the row.
- Badge/status/risk/detail rendering must use the same helper path.

Good targets to inspect:

- `dashboard/src/lib/threatStream.ts`
- `dashboard/src/pages/ThreatStreamV2Page.tsx`

## Backend Rules

- Re-scan endpoints should persist the final current context and return the canonical updated row.
- Event compactors must not emit stale `prescan_context_analysis` for rows whose current context is no longer prescan.
- Preagg and audit summaries must classify current rows, not historical prescan residues.
- DB import/update paths must derive `prescan_blocked` and `prescan_module` from the canonical current state, not by blindly OR-ing legacy metadata.

Good targets to inspect:

- `app/core/effective_state.py`
- `app/routers/analyze.py`
- `app/routers/events.py`
- `app/core/threat_stream_access.py`
- `app/core/threat_stream_workset.py`
- `app/core/recent_log_filters.py`
- `app/core/stats_preagg.py`
- `app/core/audit_metrics.py`
- `app/core/database.py`
- `app/repositories/log_repository.py`
- `app/services/conversation_context.py`

## Validation Checklist

After the patch:

1. rebuild/redeploy `app` if dashboard assets changed
2. confirm app and model-api are healthy
3. test a real prescan sample and force a synthetic non-prescan current context
4. prove all of these flip together:
   - blocked filter match becomes false
   - prescan filter match becomes false
   - effective block source becomes `none`
   - sanitized payload stops carrying `prescan_context_analysis`
   - preagg/latest compact payload stops carrying `prescan_context_analysis`
5. verify the row no longer jumps into a stale work snapshot during single-row re-scan

## Output Contract

Return:

- the single root-cause category
- which layers had split truth
- the canonical state rule adopted
- what was changed in frontend vs backend
- what was proven in runtime validation
