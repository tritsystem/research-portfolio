# librosa #2099 — spectral features upcast a float32 spectrogram to float64

**Status:** reported (issue) · **Issue:**
https://github.com/librosa/librosa/issues/2099 · **Class:** a float64 helper
grid promotes the input

## Summary

`feature.spectral_centroid`, `spectral_bandwidth`, `spectral_rolloff`,
`spectral_contrast`, and `poly_features` all do

```python
freq = fft_frequencies(sr=sr, n_fft=n_fft)   # float64
... freq * S ...                             # S is the caller's spectrogram
```

`fft_frequencies` returns float64. `freq * S` promotes a float32 spectrogram
`S` to float64 — inconsistent with `mfcc`, `melspectrogram`, and `stft`, which
all preserve the input dtype.

## How it was found

The "dtype silently forced to a wider type" bug shape, applied to librosa's
`feature` module: compute a float32 spectrogram, pass it to each feature
function, check the output dtype. Five of them widen it; the mel/MFCC/STFT
family does not.

## Suggested fix

`spectral_centroid` and `spectral_bandwidth` are a one-line `.astype` on the
frequency grid. `spectral_rolloff`, `spectral_contrast`, and `poly_features`
have deeper float64 internals (comparisons, `np.polyfit`) that need more than a
cast — the issue says so rather than proposing a half-fix.

## Verification

- Float32 spectrogram in → float64 out for all five functions; float32 out for
  `mfcc` / `melspectrogram` / `stft` on the same input.

## Relevance — who this affects

Anyone extracting spectral features from a float32 pipeline (the default for
most audio ML) — the feature vector silently doubles in width, and a
downstream model expecting float32 either errors or down-casts.

## Status

Filed as an issue, not a PR: two of the five have a clean one-line fix, the
other three need a design call from the maintainers about how far to push
float32 through the polynomial code.

## Links

- Issue: https://github.com/librosa/librosa/issues/2099
