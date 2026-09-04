# rotary-embedding-torch #50 — long-sequence positions collapse in fp16

**Status:** filed (open) · **PR:** https://github.com/lucidrains/rotary-embedding-torch/pull/50
· **Class:** an integer position grid built in the input's low precision

## Summary

`RotaryEmbedding.get_seq_pos` builds the token-position index in the query
dtype:

```python
seq = torch.arange(seq_len, device = device, dtype = t.dtype)
```

For `float16`, integers are exact only up to 2048; for `bfloat16`, up to 256.
Past that, consecutive positions round to the **same** value — so tokens
`2048` and `2049` get the identical rotary frequency, and the positional
signal flattens for the rest of the sequence. Because the frequency table is
then cached, the collapse persists for every later call at that length.

## How it was found

A check of position primitives against the reference behaviour — `transformers`
and `torchtune` build their index grid in a fixed float32, not the caller's
dtype. A dynamic run then compared a long-sequence forward at fp16 against
fp32: the two agree below position 2048 and diverge by order-1 above it. That
step-change is the signature of integer rounding, not ordinary fp16 noise.

## Root cause

`arange(seq_len, dtype=t.dtype)` treats the position index as if it were data
that should follow the input's precision. It is a counter; it must be exact.

## The fix

Widen `float16` / `bfloat16` to `float32` for the position arithmetic, then let
the downstream ops cast back. +24 / −0.

## Verification

- Length-4096 forward, fp16 vs fp32: max\|Δ\| ≈ 3e-3 for positions < 2048,
  ≈ 4 for positions ≥ 2048 on `main`; uniformly ≈ 3e-3 with the fix.
- The cached-frequency path is exercised (a second call at the same length)
  and stays correct.

## Relevance — who this affects

Any model using this library for rotary embeddings with a context length over
2048 in float16 (or over 256 in bfloat16) — i.e. essentially every
long-context transformer trained in mixed precision. The failure is silent:
no error, just a model that cannot tell late tokens apart by position.

## Links

- PR: https://github.com/lucidrains/rotary-embedding-torch/pull/50
- Correct references: `transformers` RoPE, `torchtune` `RotaryPositionalEmbeddings`
