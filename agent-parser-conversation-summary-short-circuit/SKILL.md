---
name: agent-parser-conversation-summary-short-circuit
description: Use when conversation summary generation is slow or keeps polling for short threads, especially when the system already knows there are fewer than the minimum required user turns before summary generation should start.
---

# Agent Parser Conversation Summary Short Circuit

Use this skill when conversation summary behavior feels slow or misleading for short conversations.

Typical symptoms:

- the UI spins on `대화요약생성중...` for a long time and then says summary is unavailable
- the backend says fewer than 3 user messages only after a slow request
- auto-triggered conversation summary generation still runs heavy context collection for short threads
- the frontend treats an empty summary as pending instead of a terminal "not eligible" state

## Core Rule

Eligibility must be decided before expensive summary preparation starts.

- first compute a lightweight turn count
- only collect full summary inputs when the minimum user-turn threshold is met
- return an explicit terminal status such as `insufficient_user_messages` for non-eligible conversations

Do not solve this with longer timeouts, extra retries, or ad hoc UI exceptions.

## Root Cause Pattern

The common failure mode is split responsibility across backend and frontend:

- backend performs heavy context loading before checking minimum user turns
- backend returns `""` for both "not generated yet" and "cannot generate"
- frontend interprets `""` as pending and keeps retrying

That creates slow requests, wasted model/context work, and a misleading loading state.

## Canonical Fix Pattern

Introduce a two-step summary pipeline:

1. lightweight eligibility check
2. expensive summary preparation only for eligible conversations

Recommended backend structure:

- `get_conversation_turn_counts(conversation_id)`
- `_prepare_conversation_summary_generation(conversation_id, prompt_hint=None)`

Expected behavior:

- if `user_message_count < minimum`, immediately return:
  - `status: insufficient_user_messages`
  - `summary: ""`
  - `user_message_count`
  - `assistant_message_count`
- if generation is in progress, return:
  - `status: pending`
  - `summary: 요약중...`
- only schedule background generation after eligibility succeeds

## Backend Targets

Inspect and align these paths:

- `app/services/conversation_context.py`
- `app/routers/analyze.py`
- `app/routers/deps.py`

Requirements:

- GET summary endpoint must distinguish `pending` vs `insufficient_user_messages`
- manual refresh endpoint must short-circuit the same way
- auto-trigger path must use the same eligibility helper, not a separate heavy path

## Frontend Targets

Inspect and align these paths:

- `dashboard/src/lib/api.ts`
- `dashboard/src/pages/ThreatStreamV2Page.tsx`

Requirements:

- summary fetch type must include `status`
- retry loop must continue only for `pending`
- `insufficient_user_messages` must stop loading immediately
- manual refresh should stop instantly when the backend reports non-eligible status
- empty string alone must not be treated as proof of pending generation

## Validation Checklist

After the patch:

1. run Python compile checks for summary-related backend files
2. build the dashboard
3. verify a short conversation returns `insufficient_user_messages` without long delay
4. verify the UI stops retrying for that status
5. verify an eligible conversation still returns `pending` first and later resolves to a real summary

## Output Contract

Return:

- the single root-cause category
- where eligibility was being checked too late
- how backend and frontend status semantics were unified
- what was validated in runtime or build checks
