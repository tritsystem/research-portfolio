# vit-pytorch #373 — RvT does not run in half precision

**Status:** filed (open) · **PR:** https://github.com/lucidrains/vit-pytorch/pull/373
· **Class:** position primitive built without threading the input dtype

`lucidrains/vit-pytorch` is ~25.5k stars and merges daily; the commit before
this one was "Stabilize DINO distillation loss in low precision".

## Summary

`RvT` (the rotary Vision Transformer) raises
`RuntimeError: expected m1 and m2 to have the same dtype` the moment you run it
in `float16` or `bfloat16`. `AxialRotaryEmbedding.forward` unpacks the input's
dtype and then never uses it:

```python
def forward(self, x):
    device, dtype, n = x.device, x.dtype, int(sqrt(x.shape[-2]))
    seq = torch.linspace(-1., 1., steps = n, device = device)   # no dtype= → float32
```

`seq` is float32, `scales.to(x)` cannot rescue it (`float32 * half → float32`),
so the returned `sin` / `cos` are float32. In `RvT` they multiply the
half-precision `q` / `k` inside `apply_rotary_emb`, and the attention matmul
gets mismatched dtypes.

## How it was found

A static pass flagged `AxialRotaryEmbedding.forward` — `torch.linspace(...,
device=device)` with no `dtype=`, and an **unused `dtype` local one line
above**. A dynamic check then constructed the module, ran a float16 forward,
and got float32 back. The whole-model reproduction produced the hard
`RuntimeError`.

## Root cause

The `dtype` local is dead. Every other Fourier/position grid in the library
threads the input dtype into its `torch.linspace`; this one does not.

## The fix

```python
seq = torch.linspace(-1., 1., steps = n, device = device, dtype = dtype)
```

One line — the value the method already unpacked. `float32` output is
unchanged. +1 / −1 in `rvt.py`, plus a test.

## Verification

- `AxialRotaryEmbedding`: `float16` / `bfloat16` in → same dtype out (was
  float32).
- `RvT(...).half()` on a `(1, 3, 32, 32)` float16 image: runs, output dtype
  `float16` (was `RuntimeError`).
- `tests/test_rvt.py` builds `RvT` in float16 and bfloat16 and asserts output
  shape + dtype. Fails on `main` twice; passes with the fix. Full test file
  green.

## Relevance — who this affects

Anyone who trains or runs `RvT` in mixed precision — which for a Vision
Transformer is the common case. Currently it does not run at all below float32;
the failure is a bare matmul dtype error with no hint that the rotary embedding
is the cause.

## Links

- PR: https://github.com/lucidrains/vit-pytorch/pull/373
- Sibling finding (same shape): [`perceiver-pytorch-70-variant-dtype.md`](perceiver-pytorch-70-variant-dtype.md)
