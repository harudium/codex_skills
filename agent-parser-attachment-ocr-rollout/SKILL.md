---
name: agent-parser-attachment-ocr-rollout
description: Use when attachment OCR was added to agent_parser and you need to validate or finish the rollout end-to-end, especially message-turn duplication, conversation-context identity drift, Paddle vs model-api OCR behavior, and commit readiness.
---

# Agent Parser Attachment OCR Rollout

Use this skill when attachment OCR is partially implemented and you need to prove the whole path works instead of checking isolated pieces.

Typical symptoms:

- Threat Stream shows the same user message twice
- attachment OCR appears as a separate assistant turn
- one file produces two turns, or two files produce repeated copies of the same message
- OCR text exists in backend metadata but is rendered on the wrong turn
- Paddle OCR "works" in isolation but the scan pipeline still appends nothing
- OCR quality or latency is unclear because different engines were tested with different bytes

## First Rule

Work from the canonical user message outward.

- attachment files belong to one message
- decoded text is a child segment of that same message
- scan input may append attachment text to the message context, but must not create a new logical turn
- frontend should render attachment OCR as a bounded section inside the original bubble, not as a separate assistant message

Do not start from dashboard symptoms alone. Prove the backend message identity and the frontend turn model match.

## Remote-First Workflow

For this codebase, do the investigation on `pgpu` through a direct SSH PTY session.

- start with `ssh pgpu`
- work in `/home/kevin/projects/agent_parser`
- prefer code and live runtime over stale skill memory
- when checking recent rollout state, inspect the KST 2026-04-07 to 2026-04-08 change window first

## Canonical Validation Order

### 1. Confirm the backend data model

Inspect the attachment OCR path in this order:

- `app/services/compliance_scheduler.py`
- `app/services/attachment_ocr/context.py`
- `app/services/attachment_ocr/pipeline.py`
- `app/services/attachment_ocr/decoders/`
- `app/services/conversation_context.py`

You want to prove these invariants:

- message and attachments are already mapped before OCR runs
- OCR result is stored back onto the same message family
- analysis input appends attachment-derived text to the existing context instead of creating a second synthetic message
- request metadata keeps `attachment_context_inputs` and `attachment_decode_summaries`

### 2. Confirm the frontend turn model

Inspect:

- `dashboard/src/pages/ThreatStreamV2Page.tsx`
- `dashboard/src/pages/ThreatStreamV3Page.tsx`

Required frontend rule:

- never synthesize a separate assistant OCR turn
- render OCR only inside the original message bubble with a dotted or boxed boundary

If the UI duplicates content, do not stop at the page component. Trace whether the duplicate already exists in conversation reconstruction.

### 3. Check conversation identity before patching UI

Inspect `app/services/conversation_context.py` before making any dedupe fix.

High-risk failure pattern:

- raw conversation contains original message ids
- selected audit log uses another valid alias id such as `patch-*`
- audited turn matching fails on exact message id
- code falls back to event-built turns
- event turn reconstruction treats aliases as separate messages
- OCR metadata may then attach once or twice depending on fallback matching

Preferred fix direction:

- treat alias ids as the same message family, not as malformed ids
- keep raw conversation as the canonical turn source whenever possible
- only use event-based turn reconstruction as fallback, not as the default for alias mismatch
- merge alias-linked metadata onto the same logical turn

Do not globally dedupe by text alone. That can hide real user repeats.

## OCR Runtime Rules

### Always compare engines on decrypted bytes

Stored attachment files may be encrypted at rest. Do not benchmark OCR by reading the raw file directly from disk unless you have proven the bytes are already decrypted.

Use:

- `app/services/attachment_ocr/resolver.py`
- `read_attachment_bytes(storage_path)`

If `file` says a supposed PNG is `ASCII text` or the signature starts with `APENCv1`, you are testing encrypted storage bytes, not the image.

### Compare engines with the same input

When benchmarking Paddle vs model-api:

- resolve one attachment path through `read_attachment_bytes`
- feed the same `image_bytes` to both engines
- capture elapsed time, empty-or-not, char count, and a short preview

Useful targets:

- Paddle: `app/services/attachment_ocr/decoders/paddle_engine.py`
- model-api client: `app/services/model_api_client.py`
- model-api decode endpoint: `model_api/main.py`
- multimodal adapter: `model_api/adapters/llamacpp_a1.py`

### Runtime checks that matter

- `docker-compose.yml` GPU memory utilization for `vllm` containers
- whether app containers can see `/dev/nvidia*`
- whether Paddle was installed as GPU build, not CPU-only build
- whether document/image OCR deps are actually present in the running image
- whether scheduler and app images have the same OCR runtime, not just a hotfixed container

## Performance Evaluation Checklist

For each sample set, report:

- sample count
- average elapsed seconds per engine
- empty count per engine
- one short preview per sample
- whether the text is human-readable or mostly garbage

Treat these as bad measurements and rerun:

- encrypted file bytes were used instead of resolved attachment bytes
- first-run warmup and queued retries were mixed into a "comparison" without noting it
- one engine used direct function calls while the other used a different network path or payload format

## Release Gate

Do not recommend commit or repo reflection until all of these are true:

1. the same message no longer duplicates in Threat Stream
2. attachment OCR is rendered inside the original turn only
3. backend message identity is preserved even when audited logs use alias ids
4. OCR engine comparison used the same decrypted bytes
5. the chosen default OCR engine has acceptable empty rate and readable quality
6. runtime validation was done on the real `pgpu` stack, not just local edits

Once a major milestone is truly proven, explicitly propose reflecting it back to the repo.

## Output Contract

Return:

- the single broken layer first
- the canonical message/attachment rule you enforced
- whether duplication came from frontend rendering or conversation reconstruction
- OCR benchmark summary with the exact comparison basis
- whether the rollout is commit-ready yet
