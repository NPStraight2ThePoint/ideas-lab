# Adaptive Batch Fallback for API Extraction

**Status:** Implemented

## Problem

Documented API limits (e.g. max batch size) often differ from *operationally stable* limits. An extraction pipeline can succeed under normal conditions but fail in specific regions of the data due to payload size sensitivity, endpoint-specific quirks, or transient instability — even while the batch size stays within the documented limit.

Given a paginated/batched API, a configured base batch size, and successful extraction up to a certain point, we hit a failure at a specific batch/window. How do we recover without sacrificing overall throughput?

## Naive approaches

- Reduce batch size globally — kills throughput everywhere for a problem that's local
- Restart extraction from the beginning — wasteful, doesn't scale
- Hardcode conservative limits — doesn't reflect real-world, region-specific behavior

## Approach

**1. Base execution** — run extraction at a default batch size (e.g. 100), tracking progress via a checkpoint (last successful position).

**2. Localized failure handling** — when a batch fails, don't restart or reduce the global batch size. Instead, operate only on the unresolved range and apply a fallback strategy to it.

**3. Adaptive batch fallback** — use a predefined fallback ladder (e.g. `100 → 50 → 25 → 10 → 5 → 1`). Retry the failing range at progressively smaller batch sizes until a stable size is found, and traverse the problematic region using that reduced step.

**4. Recovery to base throughput** — once the failing region is cleared, revert to the base batch size and resume normal throughput. This assumes failures are localized, not global.

**Optional — payload/field isolation (advanced):** if failures persist even at minimal batch size, progressively reduce payload scope (e.g. drop columns), isolate and log the problematic field, and continue extraction around it.

## Key concepts

- **Operational safe limits vs. documented limits** — the two are often different, and can vary by endpoint or data region
- **Local vs. global adaptation** — failures are assumed local in data space, so adaptation stays local and global throughput is preserved
- **Checkpointed progress** — extraction resumes from the last successful batch, never reprocessing already-retrieved data

## Simulation design (optional module)

A controlled simulation can compare strategies given: total data size, base batch size, fallback ladder, a failure window (start position + width), and a failure threshold (e.g. "fails when batch ≥ 50). Useful metrics: total API calls, retry count, minimum batch size reached, completion success, recovery efficiency.

## What this demonstrates

- Handling the mismatch between documented and actual API limits
- Localized recovery without global throughput degradation
- Adaptive batch sizing under failure conditions
- Checkpoint-based extraction as a foundation for practical ingestion resilience

## What this doesn't claim

- Universal applicability across all APIs
- Optimality of any specific fallback ladder or recovery policy
- Full modeling of API behavior

## Next steps / extensions

- Dynamic adjustment of fallback ladders based on observed failure patterns
- Cost-aware extraction strategies (minimizing API call cost, not just call count)
- Adaptive behavior profiling per endpoint
- Deeper payload-level fault isolation

---
*Added: 2026-04*
