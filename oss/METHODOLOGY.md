# Method — how these bugs get found

This is an overview of the discipline behind the record in `README.md`, not a
recipe. The specific audit pipeline and the private tooling that mechanises it
stay in-house — what is public is the standard it is held to and the track
record it has produced.

## What the method is for

Correctness bugs in deep-learning libraries that **don't announce themselves**:
no crash, no warning, just a number that is quietly wrong or a piece of state
that is quietly stale. They survive because the test suites that would catch
them test the common case — float32, one batch size, no reset between calls,
short sequences — and the bug lives one step outside that.

## The three rules it runs on

- **Measured only.** Nothing is claimed without a script that was run and can
  be re-run. No estimated numbers. A finding is "real" only after a minimal
  reproduction prints a value that is obviously wrong.
- **Pre-registration.** Every hypothesis is written down — a specific
  `file:line` and a falsifiable prediction — *before* the check runs. Each then
  gets an explicit verdict with its evidence line. The ones that turn out wrong
  stay in the ledger; they are not quietly dropped.
- **Honest negatives.** An audit that finds nothing is written up in as much
  detail as one that finds a bug. `pytorch/_refs`, `torchmetrics`,
  `diffusers.schedulers`, `scikit-learn IncrementalPCA`, `torchvision.models`
  and a dozen more are on that list — the well-maintained library cores handle
  the pattern under audit correctly, and saying so plainly is part of the
  work.

## The bug class

Most of the record is one shape, and naming it is fair game — it is an
observation anyone can verify against the linked PRs, not a technique:

> A stateful component's `reset()`, state-restore, parameter broadcast, or
> internal tensor init is written for **one** storage model — a scalar reset
> value, one batch size, float32, CPU, one subclass, "device is threaded but
> dtype is not" — and is silently wrong for another.

The most productive corner of it: a helper or a position primitive builds a
tensor with `device=` but no `dtype=`, so it defaults to float32. `float64`
inputs hide it — type promotion goes up. The tell is **float16 / bfloat16
silently widening to float32**, or an integer position grid that rounds once
the low-precision format runs out of mantissa. Every audit checks below
float32 first.

## The shape of the pipeline

Standard practice, deliberately unglamorous:

1. Pick a target with one checkable invariant; scope to **one bounded module**,
   never "the library".
2. Read the whole module. Note the invariant, the places the same operation is
   implemented twice, every reset / state path, and every spot where the
   docstring's promise and the code disagree.
3. Pre-register hypotheses.
4. Run one check per hypothesis. Fill the ledger honestly.
5. Minimal reproduction, captured against the default branch and the fix.
6. Surgical fix — the smallest change, in the repo's own style, no formatter
   sweep over the file.
7. A regression test that fails without the fix and passes with it; whole
   suite green.
8. House-style PR: symptom → repro → root cause → fix → test, matched to what
   that repo actually merges, with AI assistance disclosed per its policy.

Steps 1–4 are where the private method lives. The rest is just how a good
patch is put together.

## Tooling

Two in-house tools turn the hand-audit into a sweep. Both emit **candidates**
only — the full pipeline above still runs before anything is filed.

- A **behavioural sweep** that runs every module in a package through a set of
  invariance checks derived from a real fix track record (not from a
  textbook), and always probes below float32.
- A **targeted static + dynamic finder** aimed at fan-out points — a base
  class with many subclasses, a shared helper, a position/frequency primitive
  — that constructs the smallest instance routing through a suspicious line and
  confirms or clears it dynamically, with a pre-registered claim recorded
  before each check.

Their internals are not published. What they have produced is:
rotary-embedding-torch #50, kornia #4210, torchaudio #4228 re-derived from cold
scans, and perceiver-pytorch #70 and vit-pytorch #373 found fresh.

## Track record

Only a merged pull request authored here counts as "contributed". As of
2026-09-04 that is one: **spikingjelly #743**. One further reported bug
(snntorch #430, the `LSO` surrogate) was fixed and merged upstream by other
contributors. The rest — twelve pull requests and two issues — are filed and
open: reproduced bugs with fixes and tests, awaiting review. The full table,
with live status, is in `README.md`.
