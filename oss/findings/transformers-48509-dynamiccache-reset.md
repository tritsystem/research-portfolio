# transformers #48509 — `DynamicCache.reset()` corrupts the cache

**Status:** filed (open) · **PR:** https://github.com/huggingface/transformers/pull/48509
· **Class:** a base-class `reset()` written for one subclass's storage model

## Summary

`transformers`'s KV cache has two layer types. `StaticLayer` holds fixed
pre-allocated K/V buffers; `DynamicLayer` grows its K/V by `torch.cat` and its
length *is* `keys.shape[-2]`. `CacheLayerMixin.reset()` zeros K/V **in place**
— correct for `StaticLayer`, wrong for `DynamicLayer`, which has no `reset()`
override.

After `DynamicCache.reset()`:

- `get_seq_length()` still returns the pre-reset length (the tensor was zeroed,
  not shortened);
- `is_initialized` stays `True`;
- the next `update()` concatenates new K/V onto a block of stale zeros.

The cache is not emptied — it is corrupted into a non-obvious state.

## How it was found

An automated `reset()`-invariance check, scoped to the cache module — the
discipline of narrowing a 160k-star repo to *one bounded class* rather than
"the library". The check: `update(...)`, then `reset()`, then
`get_seq_length() == 0`. It returned the stale length.

## Root cause

`reset()` lives on the mixin and assumes the `StaticLayer` storage model
(fixed buffer, zero-in-place). `DynamicLayer` violates that assumption and does
not override.

## The fix

Give `DynamicLayer` a `reset()` that actually empties it — drop the K/V tensors
and clear `is_initialized` — so `reset()` restores the birth state for both
layer types. +47 / −2.

## Verification

- `DynamicCache`: `update` → `reset` → `get_seq_length() == 0`,
  `is_initialized is False`, next `update` starts from empty.
- `StaticCache` behaviour unchanged.
- Regression test in the repo's cache test module; fails on `main`, passes with
  the fix.

## Relevance — who this affects

Any generation loop that reuses a `DynamicCache` across prompts by calling
`reset()` instead of allocating a fresh cache — a common memory optimisation.
The stale-zero prefix silently degrades the next generation.

## Links

- PR: https://github.com/huggingface/transformers/pull/48509
