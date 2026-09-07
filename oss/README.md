# Upstream OSS bug-fixes — an audited record

Independent correctness auditing of open-source software libraries, mostly
(but no longer exclusively) deep-learning/neuromorphic-adjacent. Each entry
is a real bug **found by reading the code**, reproduced against a clean
install, root-caused to a named line, and — where the project's own policy
allows it — sent upstream with a failing-then-passing regression test.

The method (`METHODOLOGY.md`) and every finding write-up (`findings/`) are
here so the work can be checked, not just cited.

**Honesty rule, applied to this page:** only a **merged** pull request
counts as "contributed". An open or draft PR is "filed". A filed issue is
"reported". A finding fixed by someone else after being reported here is
credited to them, not claimed. A finding closed by a maintainer (technical
disagreement, "won't fix," or a blanket policy against AI-assisted
contributions) is recorded as **closed/declined**, not silently dropped.
Status is re-checked live via `gh api`/`gh pr view` before every update —
last full re-check: 2026-09-07.

---

## Status at a glance

| # | Repo | Change | Status |
|--:|------|--------|--------|
| 1 | **fangwei123456/spikingjelly** | [#743](https://github.com/fangwei123456/spikingjelly/pull/743) — `MemoryModule._apply`/reset-values don't move with the module on `.to()` | **MERGED** 2026-09-03 |
| 2 | fangwei123456/spikingjelly | [#744](https://github.com/fangwei123456/spikingjelly/pull/744) — `MSTDPLearner` builds its eligibility trace with no `dtype` | **MERGED** 2026-09-06 (maintainer's own follow-up commit also fixed the `reward` tensor's dtype/device, beyond the original submission) |
| 3 | fangwei123456/spikingjelly | [#745](https://github.com/fangwei123456/spikingjelly/pull/745) — neuron dtype-invariance regression tests | **closed**, not merged (2026-09-07) |
| 4 | fangwei123456/spikingjelly | [#750](https://github.com/fangwei123456/spikingjelly/pull/750) — `RAFNode` (resonate-and-fire neuron), maintainer-spec'd feature, closes [#746](https://github.com/fangwei123456/spikingjelly/issues/746) | **MERGED** 2026-09-06 — *feature, not a bug fix* |
| 5 | jeshraghian/snntorch | [#441](https://github.com/jeshraghian/snntorch/pull/441) + issues [#442](https://github.com/jeshraghian/snntorch/issues/442)/[#443](https://github.com/jeshraghian/snntorch/issues/443) — `LeakyParallel` silently drops a per-neuron `beta`; `SpikingNeuron.zeros()` is a no-op | filed (open) |
| 6 | jeshraghian/snntorch | [#430](https://github.com/jeshraghian/snntorch/issues/430) — `surrogate.LSO()` wraps the wrong autograd `Function` | **reported; fixed upstream by others** (PRs #374/#418, merged; closed COMPLETED) |
| 7 | SynSense/sinabs | [#336](https://github.com/synsense/sinabs/pull/336) — membrane potential read before `min_v_mem` clip (closes #236) | filed (open) |
| 8 | Brainchip-Inc/tenns-core | [#1](https://github.com/Brainchip-Inc/tenns-core/pull/1) + issue [#2](https://github.com/Brainchip-Inc/tenns-core/issues/2) — `SSMLayerInference.reset_state()` keeps a stale batch dim | filed (draft) |
| 9 | reservoirpy/reservoirpy | [#245](https://github.com/reservoirpy/reservoirpy/pull/245) — `dtype` node param never reaches state/output/`reset()` | filed (open) |
| 10 | stefanonardo/pytorch-esn | [#27](https://github.com/stefanonardo/pytorch-esn/pull/27) — `reset_parameters()` doesn't clear ridge stats as documented | filed (open) |
| 11 | huggingface/transformers | [#48509](https://github.com/huggingface/transformers/pull/48509) — `DynamicCache.reset()` corrupts instead of empties | filed (open) |
| 12 | pytorch/audio | [#4228](https://github.com/pytorch/audio/pull/4228) — `transforms.Fade` envelope has no `dtype=`, upcasts fp16/bf16 | filed (open; repo template says "no longer actively monitored") |
| 13 | kornia/kornia | [#4210](https://github.com/kornia/kornia/pull/4210) — auto-augment gate mask upcasts fp16/bf16 | **MERGED** 2026-09-05 (by maintainer ducha-aiki, after two informal "approve" comments finally became a real formal review) |
| 14 | lucidrains/rotary-embedding-torch | [#50](https://github.com/lucidrains/rotary-embedding-torch/pull/50) — `get_seq_pos` position index built in low-precision dtype | filed (open) |
| 15 | lucidrains/perceiver-pytorch | [#70](https://github.com/lucidrains/perceiver-pytorch/pull/70) — Fourier position grid has no `dtype=`, `gated`/`mixed_latents` variants crash in fp16 | filed (open) |
| 16 | lucidrains/vit-pytorch | [#373](https://github.com/lucidrains/vit-pytorch/pull/373) — `RvT`'s `AxialRotaryEmbedding` unpacks `dtype`, never uses it | filed (open) |
| 17 | librosa/librosa | [#2099](https://github.com/librosa/librosa/issues/2099) — `feature.spectral_*` upcasts fp32 spectrogram to fp64 | reported (issue) |
| 18 | Textualize/textual | [#6712](https://github.com/Textualize/textual/issues/6712) — `Tree.clear()`/`reset()` never clear `_tree_nodes` | filed (issue) — awaiting maintainer green-light on the underlying [#6383](https://github.com/Textualize/textual/pull/6383) before any PR, per the org's AI-disclosure policy |
| 19 | Textualize/rich | [#4216](https://github.com/Textualize/rich/issues/4216) — `Console.print(text)` drops `Text`'s own `justify` | filed (issue) — 3rd-party PR fix in flight (Vansh-Sharmaa); independently re-verified (fail-on-main/pass-with-fix, 96 tests green), corroborating comment posted |
| 20 | pallets/click | [#3838](https://github.com/pallets/click/issues/3838) — mutable default leaks across `CliRunner.invoke()` | **closed/declined** — Pallets' blanket no-AI-contributions policy, not a technical rebuttal |
| 21 | pallets/jinja | [#2263](https://github.com/pallets/jinja/issues/2263) — `overlay()` shares filters/globals/tests instead of copying | **closed/declined** — same Pallets policy |
| 22 | pallets/werkzeug | [#3264](https://github.com/pallets/werkzeug/issues/3264) — `Response`/`EnvironBuilder(headers=...)` alias the same `Headers` instance | **closed/declined** — same Pallets policy |
| 23 | fastapi/sqlmodel | [#2087](https://github.com/fastapi/sqlmodel/issues/2087) — `model_copy(deep=False)` shares SQLAlchemy state on a persisted instance | **closed** — converted to [discussion #2089](https://github.com/fastapi/sqlmodel/discussions/2089) by maintainer YuriiMotov (2026-09-07); no code fix. Analysis narrowed the root cause (pydantic `model_copy` bypasses `SQLModel.__setattr__`, so SQLAlchemy instrumentation never fires; `deep=True` also affected). |
| 24 | boto/botocore | [#3792](https://github.com/boto/botocore/issues/3792) — `Config.merge()` aliases nested option dicts | filed (open) |
| 25 | celery/celery | [#10560](https://github.com/celery/celery/issues/10560) — `Signature.clone()` aliases `.kwargs` (no-override branch + `_merge` + immutable short-circuit all pass `self.kwargs` by reference) → PR [#10571](https://github.com/celery/celery/pull/10571) (shallow-copy fix + tests; competing with a stalled 3rd-party `deepcopy` PR) | **MERGED** 2026-09-07 — the fix also resolved a 5-year-old open bug, [#6734](https://github.com/celery/celery/issues/6734) (nested group→chord→chain-body `GroupResult` never resolves) |
| 26 | python-pillow/Pillow | [#9963](https://github.com/python-pillow/Pillow/issues/9963) — `Image.copy()`/`transform()` alias list-valued `.info` entries | **reported; fixed upstream by others** (maintainer Andrew Murray, PR [#9964](https://github.com/python-pillow/Pillow/pull/9964); merged 2026-09-05) |
| 27 | aio-libs/aiohttp | [#13634](https://github.com/aio-libs/aiohttp/issues/13634) — `CookieJar.update_cookies()` aliases a `Morsel` | **reported; fixed upstream by others** (maintainer Sam Bull, PR [#13637 "Fix mutable morsels"](https://github.com/aio-libs/aiohttp/pull/13637); merged 2026-09-05) |
| 28 | scipy/scipy | [#26095](https://github.com/scipy/scipy/issues/26095) — `milp()` mutates the caller's options dict | **reported; fixed upstream by others** (maintainer j-bowhay, PR #26097; closed) |
| 29 | encode/httpx | [discussion #3786](https://github.com/encode/httpx/discussions/3786) — `Cookies.__init__` aliases a raw `http.cookiejar.CookieJar` argument | filed (discussion, per repo's discussion-before-PR norm) |
| 30 | pytorch/pytorch | [#196083](https://github.com/pytorch/pytorch/issues/196083) — `Optimizer.state_dict()`/`load_state_dict()` silently alias tensors across optimizers | filed (open) — filed personally by the author per PyTorch's `AI_POLICY.md`, after independently re-verifying on a second torch build |
| 31 | sympy/sympy | [#30420](https://github.com/sympy/sympy/issues/30420) — `_constructor_postprocessor_mapping` registrations silently ignored due to a stale `@cacheit` cache | filed (open) |
| 32 | apache/airflow | [#72544](https://github.com/apache/airflow/issues/72544) — `executor_config` aliased from a shared `default_args` dict | filed (open) |
| 33 | apache/arrow | [#51162](https://github.com/apache/arrow/issues/51162) — `Table.from_pandas()` zero-copy path isn't visible to pandas 3.0's Copy-on-Write tracker | filed (open) |
| 34 | fastapi/fastapi | [#16301](https://github.com/fastapi/fastapi/issues/16301) — direct `APIRouter.routes` mutation bypasses the route cache's version counter | **closed** (COMPLETED) 2026-09-07 — no merged fix yet; three community PRs open ([#16305](https://github.com/fastapi/fastapi/pull/16305)/[#16306](https://github.com/fastapi/fastapi/pull/16306)/[#16326](https://github.com/fastapi/fastapi/pull/16326)), each restating the reported root cause |
| 35 | jpadilla/pyjwt | `options` dict mutation, a regression of a previously-fixed issue | **private security advisory submitted** (`GHSA-gvp8-978c-rx2q`, state: triage) |
| 36 | python-jsonschema/jsonschema | [#1573](https://github.com/python-jsonschema/jsonschema/issues/1573) — deprecated `RefResolver`'s subschema cache never invalidates | **closed/declined** — maintainer Julian Berman: deprecated API, not worth fixing (a legitimate call, not disputed) |
| 37 | ansible/ansible | [#87492](https://github.com/ansible/ansible/issues/87492) — `VariableManager.set_host_facts()` aliases the caller's dict across hosts with no prior cache entry | filed (open) |
| 38 | saltstack/salt | [#70242](https://github.com/saltstack/salt/issues/70242) — `OptsDict.mutate_key()`'s `clear()` is a no-op | filed (open) |
| 39 | zulip/zulip | [#40088](https://github.com/zulip/zulip/issues/40088) — `do_scrub_realm()` doesn't invalidate the pre-scrub identity's cache entry | filed (open) |
| 40 | apache/superset | [#43918](https://github.com/apache/superset/issues/43918) — "Refresh columns" doesn't bump `changed_on`, so the chart cache goes stale | filed (open) |
| 41 | getsentry/sentry | Cross-tenant slug→pk pointer-cache staleness after a `post_init`-bypassing rename | **private disclosure** — no public tracker; disclosure email sent to `security@sentry.io` 2026-09-05, verified via a standalone Django reproduction (the `sentry` package itself couldn't be imported standalone) |
| 42 | kornia/kornia | [#4299](https://github.com/kornia/kornia/pull/4299) — `solve_cubic`'s three-real-roots branch differentiates `acos` at its own domain boundary; value is fine, gradient diverges | **MERGED** 2026-09-06 |
| 43 | kornia/kornia | [#4303](https://github.com/kornia/kornia/pull/4303) — `bbox_to_mask3d` unions three axis slabs and tries to recover the intersection with a three-way reduction, which fills the whole volume when any slab covers a full axis | **MERGED** 2026-09-06 |
| 44 | kornia/kornia | [#4319](https://github.com/kornia/kornia/pull/4319) — `RenderingDeFMO.times` is a plain attribute, not a registered buffer; `.half()` leaves it float32 and the first forward crashes | **MERGED** 2026-09-07 (rebased onto `main`, CI 40/40, maintainer ducha-aiki cleared the stale changes-requested with a fresh approval) |
| 45 | kornia/kornia | [#4336](https://github.com/kornia/kornia/pull/4336) — `bbox_to_mask`'s pixel-index grid is built at the caller's own dtype; float16 can't hold consecutive integers past 2048, so a float16 `boxes` on an image taller/wider than that silently mismasks rows or columns near the collapse | filed (open) — maintainer review "APPROVE after 2 edits" (CHANGELOG entry + pin bfloat16); both pushed 2026-09-07 |
| 46 | kornia/kornia | [#4337](https://github.com/kornia/kornia/pull/4337) — `RandomChannelDropout.fill_value`/`RandomGaussianBlurGenerator.sigma` are plain attributes, not registered buffers (both device and dtype already re-derived inline, so a hygiene/checkpoint-visibility gap, not a crash) | filed (open) — maintainer review "APPROVE after 1 CHANGELOG sentence"; pushed 2026-09-07 |
| 47 | ultralytics/ultralytics | [#26075](https://github.com/ultralytics/ultralytics/pull/26075) — SAM3 `TransformerDecoder`'s box-relative-position-bias coordinate cache keys on `(H, W)` alone; a same-size call in a different dtype (a float32 warm-up forward, then a half-precision inference forward) silently reuses stale-dtype coordinates and crashes the half-precision embedding layer that consumes them | **MERGED** 2026-09-07 (by maintainer glenn-jocher) |
| 48 | lucidrains/denoising-diffusion-pytorch | [#370](https://github.com/lucidrains/denoising-diffusion-pytorch/pull/370) — `SinusoidalPosEmb` has no parameters, so `.half()` never touches it; it keeps emitting float32 regardless of the standard `long` timestep input and crashes the next half-precision layer | filed (open) |
| 49 | lucidrains/video-diffusion-pytorch | [#40](https://github.com/lucidrains/video-diffusion-pytorch/pull/40) — same `SinusoidalPosEmb` bug, byte-identical class body, found by checking sibling repos for the same duplicated utility | filed (open) |
| 50 | lucidrains/imagen-pytorch | [#392](https://github.com/lucidrains/imagen-pytorch/pull/392) — same bug, present twice in one repo (`imagen_pytorch.py` image variant and `imagen_video.py` video variant); this repo documents `accelerate` mixed precision as an officially supported path, so the crash is reachable via the documented API, not just a manual `.half()` call | filed (open) |
| 51 | ultralytics/ultralytics | [#26083](https://github.com/ultralytics/ultralytics/pull/26083) — INT8 quantization-aware training via `quantize=8` in `train` mode (author: `Bovey0809`) | **MERGED** 2026-09-07 — *contribution, not sole-authored; feature, not a bug fix.* Credited alongside the author and others in the merge notice for independent exported-TensorRT-engine and two-GPU DDP-resume validation, and for holding the feature justification for a second model size before landing it (QAT gives a predictable INT8 floor where TensorRT's own calibrator is unreliable, +2.99 mAP50-95 on `yolo26s`, ~0 on `yolo26n`). |

**Confirmed fixes — bugs reported here, fixed upstream by the maintainer**
(none of these are our merged code; each is a real defect reported, then
independently fixed by the project itself, usually within a day):
- **jeshraghian/snntorch** [#430](https://github.com/jeshraghian/snntorch/issues/430) — `surrogate.LSO()` wrapped the wrong autograd `Function` → fixed via PRs [#374](https://github.com/jeshraghian/snntorch/pull/374)/[#418](https://github.com/jeshraghian/snntorch/pull/418), merged 2026-08-23.
- **scipy/scipy** [#26095](https://github.com/scipy/scipy/issues/26095) — `milp()` mutated the caller's options dict → fixed via PR [#26097](https://github.com/scipy/scipy/pull/26097) (j-bowhay), merged 2026-09-04.
- **aio-libs/aiohttp** [#13634](https://github.com/aio-libs/aiohttp/issues/13634) — `CookieJar.update_cookies()` aliased a `Morsel` → fixed via PR [#13637](https://github.com/aio-libs/aiohttp/pull/13637) (Sam Bull), merged 2026-09-05.
- **python-pillow/Pillow** [#9963](https://github.com/python-pillow/Pillow/issues/9963) — `Image.copy()` aliased list-valued `.info` entries → fixed via PR [#9964](https://github.com/python-pillow/Pillow/pull/9964) (Andrew Murray), merged 2026-09-05.

**Held back, drafted but NOT yet filed** (each project's own AI-contribution
policy requires the human author to personally review and submit, not an
autonomous filing):
- **scikit-learn** — `FrozenEstimator.__sklearn_clone__` returns `self`, not
  a copy, with a cross-clone-sharing consequence when composed into another
  estimator.
- **mypy** — `find_gitignores()`'s unbounded `lru_cache` isn't covered by
  `reset_global_state()`'s documented exception list.
- **Sphinx** — `viewcode`'s permanent "analysis failed" sentinel is skipped
  by `env_purge_doc()`, so a fixed source file never regains its link.
- **Home Assistant** — a state-cache staleness finding, held back given the
  project's blanket AI-policy block (same category as scikit-learn).

**Tally, 2026-09-07:** 9 merged (own) · 1 merged (a contribution to another
author's feature PR — ultralytics #26083) · 4 reported and fixed upstream by
others · 28 filed and open (PR, issue, or discussion) · 3 closed/declined on
a blanket AI-contribution policy (not technical) · 1 closed/declined on a
maintainer's technical judgment call · 3 closed without a merged fix
(spikingjelly #745 superseded; sqlmodel #2087 converted to a discussion;
fastapi #16301 closed with community PRs still open) · 2 private security
disclosures (1 formal advisory in triage, 1 email sent directly) · 4 drafted,
held back pending personal review under the target project's own AI policy.

Also **audited and deliberately not filed** (see `METHODOLOGY.md` §"honest
negatives"): `pytorch/_refs`, `torchmetrics`, `diffusers.schedulers`,
`scikit-learn IncrementalPCA`, `torchvision.models`, `redis-py`, `SQLAlchemy`,
`FastAPI`/`Starlette`/`Pydantic`, `Django`, `Polars`, `pyca/cryptography`,
`Ray`, `yt-dlp`, `Streamlit`, `websockets`, `LangChain`, and more — the
well-maintained library cores handle this pattern correctly, and a clean
audit that finds nothing is written up the same way as one that finds a bug.
Two additional real findings (`marshmallow`, `joblib`) were confirmed but
correctly **not** filed — both reproduce a bug shape a maintainer had already
explicitly declined to fix, so filing again would just be a duplicate of
settled prior art, not a new contribution.

---

## The one bug class

Most of these entries are the same shape:

> A stateful component's `reset()`, state-restore, `clone()`/`copy()`,
> parameter broadcast, or cache-invalidation path is written for **one**
> storage model — a scalar reset value, one batch size, float32, CPU, one
> subclass, "device threaded but not dtype," "instance-tracked but not
> deserialized" — and is silently wrong for another.

Two dominant sub-patterns account for most findings:
- **dtype/precision leak.** A helper or position primitive builds a tensor
  with `device=` but no `dtype=`, so it defaults to float32. `float64` inputs
  hide it (type promotion goes up); the tell is **float16/bfloat16 silently
  becoming float32**. Every ML-library audit now probes below float32 first.
  (kornia, snnTorch, rotary-embedding-torch, perceiver-pytorch, vit-pytorch,
  torchaudio, reservoirpy, pytorch-esn, ultralytics, and three lucidrains
  diffusion repos sharing one duplicated utility class.)
- **Shallow copy / aliasing.** A `copy()`/`clone()`/`merge()` does a
  **shallow** copy of a container whose *values* are themselves mutable (a
  nested dict, a list, a `Headers` object) — the container is new, but its
  contents are the same objects, so mutating the "copy" mutates the
  original. The dominant shape across the general-Python-backend-library
  audits (botocore, Celery, Pillow, aiohttp, SQLModel, kombu, ansible, salt).

The long tail beyond those two:
- **Reset/state-restore for one storage model.** Correct for the subclass it
  targeted, silently wrong for another (tenns-core, transformers
  `DynamicCache`, pytorch-esn).
- **Cache keyed on insufficient information.** The key doesn't capture
  everything that determines the value, so two different things collide on
  one cache entry (dask `tokenize`, setuptools `sys.modules`, SymPy
  `@cacheit`).
- **Refcount/bookkeeping asymmetry.** An increment/decrement pair balanced
  for "insert new" but not "replace existing" (zope.interface `register()`).
- **Falsy-check-as-memoization-guard.** `if not self._cache` instead of a
  real sentinel, so a legitimately falsy cached value reads as "not yet
  cached" (kombu `Message.decode()`).
- **Partial fix / sibling miss.** The fix landed on one code path or
  subclass, and a structurally identical sibling was never updated to match
  (Pillow's GIF fix never reaching IM/IPTC; zope.interface's
  `subscribe`/`unsubscribe` fix never reaching `register()`).

**On "Heisenbugs":** every finding above is the opposite of one — a
Bohrbug, deterministic and reproduced identically on every run, which is
exactly what "repro run twice, consistent" throughout this record is
checking for before anything gets filed. The one non-deterministic result
in the set, kornia's CI hang (issue #4214, hangs 6/6 on two Intel Xeon
runners tested and 0/8 on AMD EPYC), is environment-dependent rather than
observer-dependent — closer to a Mandelbug than a Heisenbug.

---

## How to cite

See `CITATIONS.md`. Short form:

> G. Branaa (`tritsystem`), *Upstream OSS bug-fixes — an audited record*,
> 2026. https://github.com/tritsystem/research-portfolio/tree/main/oss

Per-fix, cite the pull request, issue, or discussion directly (the URLs
above).

---

## AI assistance

Every fix here is done with AI assistance (Claude, via Claude Code) as a
pair-programmer, disclosed per each upstream project's policy — several
projects (Pallets, scikit-learn, Home Assistant, Textualize) have an explicit
policy against fully autonomous AI contributions, which is exactly why some
findings above are held back for personal filing or were closed outright.
The audit questions, the pre-registration, and the decision that a finding is
real are the author's; every number is from a script that was actually run.
Full statement: [`../AI_DISCLOSURE.md`](../AI_DISCLOSURE.md).
