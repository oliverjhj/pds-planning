# Verification Spike — Summary

Two empirical questions from the third adversarial review of the v3.8 plan
were tested in this scratch folder. Both have answers; full detail is in
`findings-pg_boss.md` and `findings-tts-frequency.md`.

## Verdicts

- **pg-boss in a Deno Edge Function: B — works, with caveats.**
- **Google TTS `en-GB-Neural2-B` vs Reg 13(4): A — passes.**

## What the v3.8 plan should do as a result

Keep both choices but tighten the pg-boss section of the plan. **pg-boss
10.1.6 runs cleanly in a short-lived Supabase Edge Function** (50 ms cold
start, 8–14 ms warm), and the retry state machine survives across
isolates because it is purely database-driven — but only if the worker is
constructed with `{ supervise: false, schedule: false }` to disarm the
in-process timers that the library normally relies on; the plan must
make that a hard requirement, and must explicitly schedule a periodic
`boss.maintain()` call (e.g. once an hour) to replace the supervision
loop's archival/expiry duties. The TTS voice needs no change: the
intelligibility-bearing formants sit firmly in the 300–3000 Hz band
(41.79 % of total energy in 500–2500 Hz, 66.67 % surviving a 300 Hz
hi-pass, smoothed upper cutoff at 3 451 Hz, out-of-band sibilants at
−17.8 dB). One adjacent recommendation: keep the audio pipeline in
LINEAR16 end-to-end and avoid MP3 conversion, which would distort the
sub-300 Hz fundamental that this analysis intentionally tolerated.

## Deliverables in this folder

```
spikes/
  .gitignore
  .env                                          (secrets — never committed)
  package.json                                  (pg-boss@10.1.6 pinned)
  supabase/
    config.toml
    functions/audio-render-worker-spike/index.ts
  enqueue.mjs                                   (Node, pg-boss client)
  render-sample.mjs                             (Google TTS REST → WAV)
  analyse.py                                    (numpy + scipy FFT analysis)
  sample.wav                                    (6.11 s LINEAR16 24 kHz)
  logs/                                         (numbered 01–11, raw outputs)
  findings-pg_boss.md
  findings-tts-frequency.md
  SPIKE-SUMMARY.md                              (this file)
```

To re-run after `supabase stop`: `supabase start` (config is preserved),
then `node enqueue.mjs 5`, hit the worker, `python312 analyse.py`.
