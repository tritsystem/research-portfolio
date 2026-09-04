# spikingjelly #744 / #745 — MSTDP eligibility dtype, and dtype-invariance tests

**Status:** filed (open) · **PRs:**
[#744](https://github.com/fangwei123456/spikingjelly/pull/744),
[#745](https://github.com/fangwei123456/spikingjelly/pull/745) ·
**Class:** dtype not threaded; test hardening

The two follow-ups to [spikingjelly #743](spikingjelly-743-apply-reset-dtype.md)
(merged).

## #744 — `MSTDPLearner` builds its eligibility trace without a dtype

`MSTDPLearner` (modulated STDP) initialises its eligibility trace with a
`torch.zeros(...)` that omits `dtype=`, so it is float32 regardless of the
network's precision. In a float16 network the trace and the weights it updates
are at different widths, and the update silently runs in float32.

**Fix:** thread the connection's dtype into the eligibility-trace init.
+30 / −0.

**Verification:** `MSTDPLearner` on a float16 connection → eligibility trace is
float16 (was float32); update runs at the network precision.

## #745 — regression tests for neuron dtype invariance

A test module that asserts, across the neuron family, that state and output
follow the module's dtype after `.half()` / `.double()` and after a `reset()`.
These lock in the #743 fix and would have caught #744. +59 / −0, no production
change.

## Relevance — who this affects

#744: anyone training with modulated STDP in mixed precision. #745: the library
maintainers — it turns "dtype is threaded" from a property that has to be
re-checked by hand into one the CI enforces.

## Links

- #744: https://github.com/fangwei123456/spikingjelly/pull/744
- #745: https://github.com/fangwei123456/spikingjelly/pull/745
- Parent (merged): [spikingjelly-743-apply-reset-dtype.md](spikingjelly-743-apply-reset-dtype.md)
