# perceiver-pytorch #70 — a base-class fix that never reached the variants

**Status:** filed (open) · **PR:** https://github.com/lucidrains/perceiver-pytorch/pull/70
· **Class:** a dtype fix landed in the base class and not its sibling files

## Summary

The `gated` and `mixed_latents` Perceiver variants raise
`RuntimeError: mixed dtype (CPU): expect parameter to have scalar type of Float`
in half precision. Each builds its Fourier position grid with

```python
axis_pos = list(map(lambda size: torch.linspace(-1., 1., steps = size, device = device), axis))
```

— no `dtype=`, so `enc_pos` is float32. `torch.cat((data, enc_pos), dim=-1)`
then promotes a `float16` / `bfloat16` `data` to float32, and that hits the
half-precision attention projections.

## How it was found

A static pass flagged the `torch.linspace(..., device=device)` in
`experimental.py`, `gated.py`, and `mixed_latents.py` — **but not** in the main
`perceiver_pytorch.py`. A dynamic check constructed the `gated` and
`mixed_latents` models, ran a float16 forward, and got the `RuntimeError`; the
main model ran clean. The asymmetry is the finding.

## Root cause

The main `Perceiver` was fixed for exactly this in the repo's PR #59 (2022):
`b, *axis, _, device, dtype = *data.shape, data.device, data.dtype` and
`torch.linspace(..., dtype=dtype)`. The three variant files are near-copies of
the main model that predate the fix and never received it.

## The fix

Apply PR #59's two-line change to `gated.py` and `mixed_latents.py`: unpack
`dtype` from `data`, pass it to `torch.linspace`. +6 / −4 across the two files,
plus a test.

`experimental.py` has the identical line but its
`Perceiver(fourier_encode_data=True)` is independently broken by a shape
mismatch on `main`, so a half-precision test for it cannot pass yet — noted in
the PR body as out of scope rather than shipped untested.

## Verification

- `gated` and `mixed_latents` Perceiver in float16 / bfloat16: run, output dtype
  preserved (was `RuntimeError`).
- `tests/test_perceiver.py::test_perceiver_variants_preserve_low_precision`
  (2 variants × 2 dtypes): 4 fail on `main`, all pass with the fix.

## Relevance — who this affects

Anyone using the gated or mixed-latents Perceiver variants in mixed precision —
they do not currently run below float32. The `experimental` variant carries the
same latent bug behind its separate shape issue.

## Links

- PR: https://github.com/lucidrains/perceiver-pytorch/pull/70
- Precedent: `lucidrains/perceiver-pytorch` PR #59
- Sibling finding (same shape): [`vit-pytorch-373-rvt-half-precision.md`](vit-pytorch-373-rvt-half-precision.md)
