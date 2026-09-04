# tenns-core #1 — `reset_state()` keeps a stale batch dimension

**Status:** filed (draft) · **PR:** https://github.com/Brainchip-Inc/tenns-core/pull/1
· **Issue:** [#2](https://github.com/Brainchip-Inc/tenns-core/issues/2) ·
**Class:** a reset written for one storage model

## Summary

`SSMLayerInference.reset_state()` zeros the recurrent state **in place**,
keeping its current shape. The shape includes a batch dimension. If you process
one sequence at batch size 4, call `reset_state()`, then feed a sequence at
batch size 1, the state is still `(4, ...)` — and the gate-mode path crashes on
the shape mismatch. `reset_state()` is supposed to return the layer to its
birth state, which has no committed batch size.

## How it was found

An automated `reset()`-invariance check, run with a batch-size change between
sequences — an edge the library's own suite never varied. The suite's fixtures
were named `trained_*` but were never actually trained, and that gap is part of
why the bug survived: it only shows once the state tensor has a real shape from
a real forward pass.

## Root cause

`reset_state()` calls `state.zero_()` (in place, shape-preserving) instead of
dropping the state so the next forward re-allocates it at the incoming batch
size — correct for a fixed pre-allocated buffer, wrong for a state whose batch
dimension is set by the data.

## The fix

Reset by clearing the state reference (or re-allocating at the birth shape)
rather than zeroing in place, so the next forward sizes it from its input.
+89 / −36 (the fix plus test-fixture repairs and a regression test).

## Verification

- Batch size 4 → `reset_state()` → batch size 1: runs (was a shape-mismatch
  crash in gate mode).
- The FFT-conv training path and the recurrent streaming path agree to ~1e-6
  after real training at T=512 — the core invariant held under stress, so the
  finding is scoped tightly to the reset.

## Relevance — who this affects

Anyone doing variable-batch inference with a `tenns-core` SSM layer — e.g. a
server batching requests dynamically — who resets state between requests.

## Links

- PR (draft): https://github.com/Brainchip-Inc/tenns-core/pull/1
- Issue: https://github.com/Brainchip-Inc/tenns-core/issues/2
