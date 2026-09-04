# torchaudio #4228 — `Fade` silently promotes half-precision audio

**Status:** filed (open); the repo's PR template states it is "no longer
actively monitored" · **PR:** https://github.com/pytorch/audio/pull/4228 ·
**Class:** a helper tensor pinned to float32

## Summary

`torchaudio.transforms.Fade` builds its fade-in / fade-out envelope with

```python
torch.linspace(0.0, 1.0, fade_len, device = waveform.device)   # no dtype=
```

The envelope is float32. `waveform * envelope` promotes a `float16` /
`bfloat16` waveform to float32. `float64` is unaffected (promotion up), which
is why a test suite that only checks float32 vs float64 misses it entirely.

## How it was found

An automated forward-invariance sweep over `torchaudio.transforms`, run below
float32 as a matter of course. `Fade` passed the float64 check and failed
float16 — which is exactly why the audit tests below float32.

## The fix

Thread `waveform.dtype` into the `torch.linspace` calls that build the two
envelopes. +26 / −7 (two envelope sites plus the surrounding `.to(...)` calls
in the same method).

## Verification

- `Fade()(waveform)` with a float16 / bfloat16 waveform: output dtype
  preserved (was float32).
- Verified in torchaudio's own test harness; `black` (22.3) + `flake8` clean.

## Relevance — who this affects

Anyone applying a fade to a half-precision audio tensor in a preprocessing
pipeline — common in on-device / streaming inference where the whole graph is
fp16 for latency. The result is a silent widening back to float32 at the fade.

## A methodology note

This is where the rule "check a repo's PR template and recent *code* merge
cadence **before** investing the full pipeline" was written down: `pytorch/audio`
declares itself unmonitored in its PR template. The fix was completed and filed
because the bug is real and the fix is small, but the expectation of a review
is set accordingly.

## Links

- PR: https://github.com/pytorch/audio/pull/4228
