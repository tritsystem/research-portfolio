# spikingjelly #743 — reset values don't move with the module on `.to()`

**Status:** **MERGED** (2026-09-03) · **PR:** https://github.com/fangwei123456/spikingjelly/pull/743
· **Class:** dtype / device not threaded through a base method

The first of this record's fixes to land on a library's default branch.

## Summary

`spikingjelly.activation_based.base.MemoryModule` lets a neuron register a
*reset value* for each piece of state (`v` → `v_reset`, etc.). When the module
is moved with `.to(device)` / `.half()` / `.double()`, PyTorch's `nn.Module._apply`
walks parameters and buffers — but the registered reset values are plain Python
attributes, not buffers, so `_apply` never touches them. After `net.half()`,
calling `reset()` writes a **float32** reset value back into what is now a
**float16** state tensor, and the next forward runs on mismatched dtypes.

## How it was found

An automated invariance sweep over the neuron family, checking that state
dtype and device survive a `reset()`. It came back float32 after `.half()`
across the family. Traced by hand to `MemoryModule._apply` overriding
`nn.Module._apply` without carrying the reset-value dict through the same
transformation.

## Root cause

`MemoryModule._apply(fn)` calls `super()._apply(fn)` (which handles params and
buffers) but does not apply `fn` to the entries of `self._memories_rv` (the
reset-value store). `fn` is exactly the closure that `.to()` / `.half()` builds
to move a tensor, so the reset values are left in their birth dtype/device.

## The fix

In `MemoryModule._apply`, after `super()._apply(fn)`, apply `fn` to every
tensor-valued reset value in `self._memories_rv` (guarding non-tensor reset
values, which are legal — a scalar `0.0` is common). +75 / −0.

## Verification

- New behaviour: `net.half(); net.reset()` → every state tensor and its reset
  value is float16. `net.to('cuda')` → both on CUDA.
- Regression test added in spikingjelly's own test style; fails on `master`,
  passes with the change.
- Full neuron test module: no regressions.

## Relevance — who this affects

Anyone training a spikingjelly network in mixed precision or on GPU who calls
`functional.reset_net(net)` between sequences — the standard training loop. The
symptom is a dtype-mismatch error or a silent upcast on the first step after a
reset, which is easy to misattribute to your own code.

## Links

- PR: https://github.com/fangwei123456/spikingjelly/pull/743 (merged)
- Method: [`../METHODOLOGY.md`](../METHODOLOGY.md)
