# Upstream OSS bug-fixes — an audited record

Independent correctness auditing of open-source deep-learning libraries.
Each entry is a real bug **found by reading the code**, reproduced against a
clean install, root-caused to a named line, fixed with the smallest possible
change, and sent upstream with a failing-then-passing regression test.

The method (`METHODOLOGY.md`) and every finding write-up (`findings/`) are
here so the work can be checked, not just cited.

**Honesty rule, applied to this page:** only a **merged** pull request counts
as "contributed". An open or draft PR is "filed". A filed issue is "reported".
Status below is re-checked with `gh pr view` before each update.

---

## Status at a glance

| # | Repo | Change | Class | Status |
|--:|------|--------|-------|--------|
| 1 | **fangwei123456/spikingjelly** | [#743](https://github.com/fangwei123456/spikingjelly/pull/743) — `MemoryModule._apply` / reset-values don't move with the module on `.to()` | dtype/device not threaded through a base method | **MERGED** 2026-09-03 |
| 2 | fangwei123456/spikingjelly | [#744](https://github.com/fangwei123456/spikingjelly/pull/744) — `MSTDPLearner` builds its eligibility trace with no `dtype` | dtype not threaded | filed (open) |
| 3 | fangwei123456/spikingjelly | [#745](https://github.com/fangwei123456/spikingjelly/pull/745) — neuron dtype-invariance regression tests | test hardening | filed (open) |
| 4 | jeshraghian/snntorch | [#441](https://github.com/jeshraghian/snntorch/pull/441) + issues [#442](https://github.com/jeshraghian/snntorch/issues/442) / [#443](https://github.com/jeshraghian/snntorch/issues/443) — `LeakyParallel` silently drops a per-neuron `beta`; `SpikingNeuron.zeros()` is a no-op | per-unit param collapse; silent no-op | filed (open) |
| 5 | jeshraghian/snntorch | [#430](https://github.com/jeshraghian/snntorch/issues/430) — `surrogate.LSO()` wraps the wrong autograd `Function` and raises `TypeError` | wrong callable wired into a factory | **reported; fixed upstream** (PRs #374 / #418 by other contributors, merged; issue closed COMPLETED 2026-08-29) |
| 6 | SynSense/sinabs | [#336](https://github.com/synsense/sinabs/pull/336) — membrane potential read for the layer output *before* being clipped to `min_v_mem` (closes issue #236) | read-before-clip ordering | filed (open) |
| 7 | Brainchip-Inc/tenns-core | [#1](https://github.com/Brainchip-Inc/tenns-core/pull/1) + issue [#2](https://github.com/Brainchip-Inc/tenns-core/issues/2) — `SSMLayerInference.reset_state()` keeps a stale batch dim | reset written for one storage model | filed (draft) |
| 8 | reservoirpy/reservoirpy | [#245](https://github.com/reservoirpy/reservoirpy/pull/245) — `dtype` node param never reaches node state / run output / `reset()` | dtype not threaded (numpy) | filed (open) |
| 9 | stefanonardo/pytorch-esn | [#27](https://github.com/stefanonardo/pytorch-esn/pull/27) — `ESN.reset_parameters()` docstring promises it clears the ridge stats; the body never does | reset incomplete vs the documented contract | filed (open) |
| 10 | huggingface/transformers | [#48509](https://github.com/huggingface/transformers/pull/48509) — `DynamicCache.reset()` corrupts the cache instead of emptying it | base `reset()` written for one subclass's storage model | filed (open) |
| 11 | pytorch/audio | [#4228](https://github.com/pytorch/audio/pull/4228) — `transforms.Fade` builds its envelope with no `dtype=`, upcasting a float16/bfloat16 waveform | helper tensor pinned to float32 | filed (open; repo template says "no longer actively monitored") |
| 12 | kornia/kornia | [#4210](https://github.com/kornia/kornia/pull/4210) — `OperationBase.forward` gates auto-augment with a float32 mask, upcasting half-precision batches | gate tensor pinned to float32 | filed (open, maintainer-approved pending changelog; also clears 19 of the repo's tracked half-precision xfails) |
| 13 | lucidrains/rotary-embedding-torch | [#50](https://github.com/lucidrains/rotary-embedding-torch/pull/50) — `get_seq_pos` builds the position index in the query dtype; fp16 can't represent integers past 2048 | position primitive built in the input's low precision | filed (open) |
| 14 | lucidrains/perceiver-pytorch | [#70](https://github.com/lucidrains/perceiver-pytorch/pull/70) — `gated` / `mixed_latents` Perceiver variants build the Fourier position grid with no `dtype=`; the model raises `RuntimeError: mixed dtype` in half precision | a base-class fix that never reached the variant files | filed (open) |
| 15 | lucidrains/vit-pytorch | [#373](https://github.com/lucidrains/vit-pytorch/pull/373) — `RvT`'s `AxialRotaryEmbedding` unpacks `dtype` from the input and never uses it; `RvT` does not run in half precision at all | position primitive built without threading the input dtype | filed (open) |
| 16 | librosa/librosa | [#2099](https://github.com/librosa/librosa/issues/2099) — `feature.spectral_*` and `poly_features` upcast a float32 spectrogram to float64 via the frequency grid | float64 helper grid promotes the input | reported (issue) |

**Tally, 2026-09-04:** 1 own PR merged · 1 reported bug fixed upstream (by
others) · 12 PRs filed and open · 2 issues open. "Merged" means a merged pull
request authored by `tritsystem` — that is exactly one, spikingjelly #743.

Also **audited and deliberately not filed** (see `METHODOLOGY.md` §"honest
negatives"): `pytorch/_refs`, `torchmetrics`, `diffusers.schedulers`,
`scikit-learn IncrementalPCA`, `torchvision.models`, and ~10 more — the
well-maintained library cores handle this pattern correctly, and a clean
audit that finds nothing is written up the same way as one that finds a bug.

---

## The one bug class

Twelve of the fourteen entries are the same shape:

> A stateful component's `reset()`, state-restore, parameter broadcast, or
> internal tensor init is written for **one** storage model — a scalar reset
> value, one batch size, float32, CPU, one subclass, "device threaded but not
> dtype" — and is silently wrong for another.

The most productive sub-pattern: **a helper or position primitive builds a
tensor with `device=` but no `dtype=`**, so it defaults to float32. `float64`
inputs hide it (type promotion goes up); the tell is **float16 / bfloat16
silently becoming float32**, or — for an integer position grid — rounding once
the low-precision format runs out of mantissa. Every audit now probes below
float32 first.

---

## How to cite

See `CITATIONS.md`. Short form:

> G. Branaa (`tritsystem`), *Upstream OSS bug-fixes — an audited record*,
> 2026. https://github.com/tritsystem/research-portfolio/tree/main/oss

Per-fix, cite the pull request or issue directly (the URLs above).

---

## AI assistance

Every fix here is done with AI assistance (Claude, via Claude Code) as a
pair-programmer, disclosed per each upstream project's policy. The audit
questions, the pre-registration, and the decision that a finding is real are
the author's; every number is from a script that was actually run. Full
statement: [`../AI_DISCLOSURE.md`](../AI_DISCLOSURE.md).
