# Citing this work

## The body of work

> G. Branaa (`tritsystem`). *Upstream OSS bug-fixes — an audited record.*
> 2026. https://github.com/tritsystem/research-portfolio/tree/main/oss

BibTeX:

```bibtex
@misc{branaa2026ossaudit,
  author       = {Branaa, Gavin},
  title        = {Upstream {OSS} bug-fixes --- an audited record},
  year         = {2026},
  howpublished = {\url{https://github.com/tritsystem/research-portfolio/tree/main/oss}},
  note         = {Correctness audits of deep-learning libraries; method and per-finding write-ups}
}
```

## The method

> G. Branaa. *Methodology — auditing open-source libraries for real bugs.*
> In *Upstream OSS bug-fixes — an audited record*, 2026.
> https://github.com/tritsystem/research-portfolio/blob/main/oss/METHODOLOGY.md

## An individual fix

Cite the pull request or issue directly. Each is a self-contained artifact
with a root cause, a reproduction, a fix, and a regression test.

| finding | cite this |
|---|---|
| spikingjelly reset-value dtype (**merged**) | `fangwei123456/spikingjelly` PR #743 |
| spikingjelly MSTDPLearner eligibility dtype | PR #744 |
| spikingjelly dtype-invariance tests | PR #745 |
| snntorch `LeakyParallel` per-neuron beta | `jeshraghian/snntorch` PR #441, issues #442 / #443 |
| tenns-core `reset_state` stale batch dim | `Brainchip-Inc/tenns-core` PR #1, issue #2 |
| reservoirpy `dtype` not threaded | `reservoirpy/reservoirpy` PR #245 |
| pytorch-esn `reset_parameters` ridge stats | `stefanonardo/pytorch-esn` PR #27 |
| transformers `DynamicCache.reset()` | `huggingface/transformers` PR #48509 |
| torchaudio `Fade` dtype | `pytorch/audio` PR #4228 |
| kornia auto-augment dtype | `kornia/kornia` PR #4210 |
| rotary-embedding-torch long-sequence positions | `lucidrains/rotary-embedding-torch` PR #50 |
| perceiver-pytorch variant dtype | `lucidrains/perceiver-pytorch` PR #70 |
| vit-pytorch RvT half precision | `lucidrains/vit-pytorch` PR #373 |
| librosa spectral-feature float64 upcast | `librosa/librosa` issue #2099 |

A finding's write-up in `findings/` links its PR, its root-cause line, its
reproduction, and its status. If you are checking a claim, start there.

## Status, and what "contributed" means

Only a **merged** pull request is a contribution. As of **2026-09-07** that is
**nine** sole-authored — spikingjelly #743, #744, #750; kornia #4210, #4299,
#4303, #4319; ultralytics #26075; celery #10571 — plus **one** merged as a
credited contribution to another author's feature PR (ultralytics #26083).
The rest are filed and open — genuine reproduced bugs with fixes and tests,
awaiting maintainer review — or closed without a merged fix. This page
re-checks every status with `gh pr view` before it is updated, and says
"filed" or "reported" everywhere it is not "merged". See `README.md` for the
full row-by-row status.
