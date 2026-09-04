# reservoirpy #245 — the `dtype` node parameter is half-ignored

**Status:** filed (open) · **PR:** https://github.com/reservoirpy/reservoirpy/pull/245
· **Class:** dtype not threaded (numpy)

## Summary

`reservoirpy` nodes take a `dtype` parameter documented as "numerical type for
node parameters". It reaches the weight matrices but **not** the node's running
state: `Node.reset()` / `initialize()` seed the state with `np.zeros(shape)` —
no `dtype` argument — so the state is always float64 regardless of what the user
asked for. `run()` output then follows the state, not the configured dtype.

## How it was found

The method generalises past torch: same reset / state-init failure mode, numpy
instead of tensors. Hand-audit of `Node` — reading every `reset`, `initialize`,
and state-allocation site — plus the docstring-vs-code check: the `dtype`
docstring makes an explicit promise the code only half-keeps.

## Root cause

`np.zeros(shape)` defaults to `float64`. Every state-seeding call omits
`dtype=self.dtype`.

## The fix

Thread `self.dtype` into the `np.zeros(...)` calls in the `Node` machinery
(`Reservoir`, `LocalPlasticityReservoir`, the base `Node`). +53 / −5.

**Scope narrowed during implementation, and the PR says so:** `ES2N`,
`IPReservoir`, and the `LIF` node have *deeper* float64 leaks
(`mat_gen.orthogonal` ignores its `dtype` arg; `IP.a` / `IP.b`; `LIF._step`'s
`np.where(cond, 1.0, 0.0)`). Those are named as an explicit follow-up rather
than shipped as a half-fix or expanded into a whack-a-mole.

## Verification

- `Reservoir(..., dtype=np.float32)` → state and `run()` output are float32
  (were float64).
- Regression test in reservoirpy's test style; fails without the change.

## Relevance — who this affects

Anyone running reservoirpy in float32 for memory or speed — the state and
outputs silently double in width, and the `dtype` parameter reads as if it
worked.

## Links

- PR: https://github.com/reservoirpy/reservoirpy/pull/245
