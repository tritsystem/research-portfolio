# kornia #4210 — auto-augment upcasts half-precision batches

**Status:** filed (open); maintainer-approved pending a changelog entry, which
has been added and the merge conflict resolved · **PR:**
https://github.com/kornia/kornia/pull/4210 · **Class:** a gate tensor pinned to
float32

## Summary

`RandAugment`, `AutoAugment`, `TrivialAugment` and `AugMix` all route through
`OperationBase.forward`, the shared gate:

```python
batch_prob = params["batch_prob"][...].to(device = input.device)   # no dtype=
return batch_prob * self.op(input, params = params) + (1 - batch_prob) * input
```

`batch_prob` is float32. In `batch_prob * op(x) + (1 - batch_prob) * x`, a
`float16` / `bfloat16` `input` is promoted to float32 — the op output and the
input are both low precision, only the gate is not. `float32` / `float64`
inputs are unaffected (promotion goes the other way), so the bug is invisible
unless you test below float32.

## How it was found

An automated forward-invariance sweep over `kornia.augmentation`, checking that
a float input's dtype survives the forward. `TrivialAugment` reported
`float16 → float32`; reading the four auto-augment policies back to a common
line put it on the shared `OperationBase.forward` mask multiply.

## The fix

```python
batch_prob = params["batch_prob"][...].to(device = input.device, dtype = input.dtype)
```

+2 / −1 in `operations/base.py`. `float32` output is byte-identical before and
after.

## Verification

- `TrivialAugment` / `RandAugment` / `AutoAugment` in float16 and bfloat16:
  output dtype preserved (was float32).
- New `test_operation_preserves_input_dtype` iterates every op from
  `_find_all_ops()` and asserts `op(x).dtype == x.dtype`; fails on `main`,
  passes with the fix.
- Clears **19** of the Linux-CPU half-precision xfail lines the repo tracks in
  its issue #4153 (all ten `cpu_bfloat16`, nine of ten `cpu_float16`; the tenth
  changes from a `RuntimeError` to an `AssertionError` — a smaller residual,
  stated as such in the PR).
- The repo's own strict half-precision CI legs (float16 + bfloat16) are green
  on the PR.

## Relevance — who this affects

Anyone doing mixed-precision training with a kornia auto-augment policy in the
data pipeline — the batch silently leaves the augmentation at float32 and the
model's first op has to down-cast it, wasting the memory and bandwidth the
half-precision run was for.

## Links

- PR: https://github.com/kornia/kornia/pull/4210
- Related: kornia issue #4153 (the half-precision xfail list)
