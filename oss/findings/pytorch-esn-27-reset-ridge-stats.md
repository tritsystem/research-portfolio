# pytorch-esn #27 — `reset_parameters()` doesn't clear what it says it does

**Status:** filed (open) · **PR:** https://github.com/stefanonardo/pytorch-esn/pull/27
· **Class:** reset incomplete versus the documented contract

## Summary

`ESN.reset_parameters()` has a docstring that says it "clears the accumulated
statistics". The body re-initialises the reservoir weights but never clears the
ridge-regression accumulators `XTX`, `XTy`, `X`. So a `reset_parameters()`
followed by a re-fit solves the readout on the **union** of the pre-reset and
post-reset data — not on the fresh data, as the documentation promises.

## How it was found

Docstring-vs-code, promoted to a first-class target: the doc makes an
*explicit* promise ("clears the accumulated statistics") that the code breaks,
with a real silent consequence. Reading `reset_parameters()` against its own
docstring was the whole audit.

## Root cause

`reset_parameters()` was written to reset the *weights*; the ridge accumulators
were added later and the reset was never extended to cover them.

## The fix

Zero `XTX`, `XTy`, `X` in `reset_parameters()` so the method matches its
docstring. +65 / −0 (the fix plus its regression test).

## Verification

- Fit on data A, `reset_parameters()`, fit on data B → the readout matches a
  fresh ESN fit on B alone (it previously matched a fit on A ∪ B).
- Regression test fails on `master`, passes with the change.

## Relevance — who this affects

Anyone using `pytorch-esn` for online / streaming echo-state learning who
relies on `reset_parameters()` to start a clean fit — a standard part of the
API surface. The contamination is invisible unless you compare against a
from-scratch model.

## Links

- PR: https://github.com/stefanonardo/pytorch-esn/pull/27
