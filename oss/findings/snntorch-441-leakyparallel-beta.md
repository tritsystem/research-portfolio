# snntorch #441 — `LeakyParallel` silently drops a per-neuron `beta`

**Status:** filed (open) · **PR:** https://github.com/jeshraghian/snntorch/pull/441
· **Issues:** [#442](https://github.com/jeshraghian/snntorch/issues/442),
[#443](https://github.com/jeshraghian/snntorch/issues/443) ·
**Class:** per-unit parameter collapsed to a scalar; a silent no-op

## Summary

Two findings in `snntorch`, filed together:

1. **`LeakyParallel` ignores a per-neuron `beta`.** You can pass `beta` as a
   length-`hidden_size` tensor to give each neuron its own decay. The
   `_beta_to_weight_hh` helper has an `if/elif` chain where the per-neuron
   branch is a **sibling of a branch that always matches first**, so it is
   unreachable — the per-neuron `beta` is collapsed to a scalar (its first
   element), and a wrong-length `beta` is silently accepted rather than
   raising.
2. **`SpikingNeuron.zeros()` is a no-op.** It was meant to zero the neuron's
   state; the implementation returns a fresh zero tensor without writing it
   back, so the state is untouched.

## How it was found

Reading the `if/elif` chain in `_beta_to_weight_hh` line by line — an earlier
branch shadowing a later one is a listed bug shape. The probe printed the
diagonal of `weight_hh` and compared it to the `beta` vector that was passed:
they did not match. The `zeros()` no-op was caught the same way — reading the
method against what its name promises.

## Root cause

(1) branch ordering: the always-true branch precedes the per-neuron branch.
(2) `zeros()` builds a new tensor instead of mutating the state in place.

## The fix

(1) Reorder so the per-neuron branch is tested first, and add a length check
that raises on a wrong-length `beta`. (2) Make `zeros()` actually zero the
state, leaf-tensor-safe for autograd. +84 / −9, following snntorch's `black`
(line-length 79) + flake8 style.

## Verification

- `LeakyParallel(beta=<length-hidden vector>)`: `weight_hh` diagonal now equals
  the passed vector; a wrong-length `beta` raises.
- `zeros()` now zeroes the state.
- Regression tests in snntorch's test style; fail on `master`, pass with the
  change; suite pass-count up by exactly the number of new tests.

## Relevance — who this affects

Anyone using `LeakyParallel` with heterogeneous decay — a common way to give a
population a spread of time constants. Currently every neuron silently gets the
same `beta`, and the model trains anyway, so the loss of heterogeneity is
invisible.

## Note on attribution

An earlier, separate `snntorch` dtype issue in this area (issue #422 by another
user, fixed by PR #432 by a third contributor) is **not** part of this record —
see the honesty note in [`../CITATIONS.md`](../CITATIONS.md). The work here is
PR #441 and issues #442 / #443, all authored by `tritsystem` and all open.

## Links

- PR: https://github.com/jeshraghian/snntorch/pull/441
