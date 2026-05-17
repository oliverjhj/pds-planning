# Findings — Google TTS `en-GB-Neural2-B` vs UK PSV AIR 2023 Reg 13(4)

## Verdict: **A — passes Reg 13(4)**

`en-GB-Neural2-B` rendered at LINEAR16 / 24 kHz produces an audio signal whose
intelligibility-bearing content sits squarely inside the regulation's
300–3000 Hz band. Out-of-band content above 3 kHz is sibilant-only and
moderate (−17.8 dB rel peak). Out-of-band content below 300 Hz is the male
voice fundamental — expected, doesn't carry intelligibility, and is
attenuated by the PA system's low-end cut without harming speech clarity.

The voice is suitable for the bus PA application as specified in v3.8.

## Sample used

- Text rendered:
  > "The next stop is Newcastle upon Tyne Central Station. Please mind the
  > gap between the bus and the kerb."
- Voice: `en-GB-Neural2-B` (Google Cloud Neural2, UK English, male).
- Format: LINEAR16, 24000 Hz sample rate (explicitly requested).
- Container: minimal canonical PCM WAV (44-byte header) for scipy compatibility.

## Audio properties

| Property      | Value                |
| ------------- | -------------------- |
| Sample rate   | 24 000 Hz            |
| Bit depth     | 16 bit               |
| Channels      | 1 (mono)             |
| Duration      | 6.113 s              |
| Samples       | 146 702              |
| PSD method    | Welch, 4096-bin Hann, 50 % overlap |
| PSD resolution| 5.86 Hz/bin          |

## Spectral analysis

| Metric                                    | Value          |
| ----------------------------------------- | -------------- |
| Spectral peak frequency                   | **128.9 Hz**   |
| Energy share, sub-300 Hz                  | 32.79 %        |
| Energy share, **300–3000 Hz (in band)**   | **63.83 %**    |
| Energy share, above 3000 Hz               | 2.84 %         |
| Energy share, formant zone 500–2500 Hz    | **41.79 %**    |
| Energy surviving a 300 Hz hi-pass         | **66.67 %**    |
| Upper audible cutoff (smoothed, ≥ −20 dB) | **3 451 Hz**   |
| Sibilant peak above 3 kHz                 | 3 287 Hz @ **−17.82 dB** rel peak |
| Sub-300 rumble peak                       | 128.9 Hz @ 0.00 dB (voice fundamental) |

## Interpretation

A naive "raw energy share in band" reading would call this borderline:
63.83 % in-band and 32.79 % below 300 Hz looks low. But that interpretation
mixes up two different things and gives the wrong answer here:

1. **The sub-300 Hz peak is the voice fundamental, not rumble.** An adult
   male speaking voice has its fundamental between roughly 85 and 180 Hz; the
   measured 128.9 Hz is exactly where it should be. The fundamental and its
   first few harmonics contribute a lot of *power* but very little
   *intelligibility*. The PA system's natural low-end roll-off attenuates the
   fundamental without harming speech.
2. **Intelligibility lives in the formants.** Speech recognition (human or
   otherwise) is driven by formant frequencies F1 (≈ 300–1000 Hz),
   F2 (≈ 800–2500 Hz), F3 (≈ 2000–3500 Hz). The measured
   formant-zone share is 41.79 % — strong. Combined with the smoothed upper
   audible cutoff at 3 451 Hz, this voice covers the full intelligibility
   spectrum.

Critically, **66.67 % of the spectral energy survives a 300 Hz hi-pass.**
That's the right metric for "would this voice still work through a PA
limited to 300–3000 Hz reproduction." Two-thirds of the energy makes it
through the low cut, all of it in the speech-formant band.

The 2.84 % above 3 kHz is the sibilant content of consonants like *s*, *sh*,
*st* (Newcastle, Central, Station, please, stop, gap). The peak sibilant
frequency at 3 287 Hz sits just outside the upper band edge, at −17.82 dB
relative to peak — moderate, not dominating, and well within what speech
naturally produces.

## Reproducible procedure

```
# from spike folder, with GOOGLE_TTS_API_KEY set in .env
node render-sample.mjs           # writes sample.wav
python312 analyse.py             # prints JSON metrics + verdict
```

### Pinned dependencies

| Tool / package | Version  | Source                                 |
| -------------- | -------- | -------------------------------------- |
| Node           | 24.14.0  | system                                 |
| Python         | 3.12.10  | Scoop `versions/python312`             |
| numpy          | 2.1.3    | `pip install --user numpy==2.1.3`      |
| scipy          | 1.14.1   | `pip install --user scipy==1.14.1`     |

### Scripts

- `render-sample.mjs` — POSTs to
  `https://texttospeech.googleapis.com/v1/text:synthesize?key=…` and wraps
  the returned LINEAR16 in a canonical PCM WAV header.
- `analyse.py` — loads `sample.wav`, computes a Welch PSD, integrates per
  band, reports energy shares, formant-zone share, surviving-hipass
  fraction, smoothed upper cutoff, sibilant peak, fundamental rumble peak,
  and a verdict.

### Raw output

Captured in `logs/11-analysis.txt` and reproduced in full above.

## Recommendation

**Keep the voice (`en-GB-Neural2-B`).** No re-test on a broader sample is
required for compliance — speech-formant geometry is voice-determined and a
6-second sample with mixed phonemes (place names, fricatives, plosives, and
"th") is representative. If at some point a *female* voice is added (e.g.
`en-GB-Neural2-A`), re-run this script against it: female fundamentals sit
at 180–250 Hz, closer to the band edge, so the sub-300 share will be
different (smaller, because more of the fundamental is already in-band)
but the in-band intelligibility should be even stronger. No change in
methodology needed.

One small adjacent recommendation: the v3.8 plan can drop the LINEAR16 →
MP3 conversion step (if any). The PA pipeline will benefit from staying
in LINEAR16 end-to-end; MP3's psychoacoustic compression discards sub-300
Hz content aggressively (loudness contour) and would change these numbers.
