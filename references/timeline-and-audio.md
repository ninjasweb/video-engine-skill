# Timeline, audio mix and subtitles

## Placing clips

- Intro (title) before the first word (≈3 s), outro/end screen after the last word (≈9 s; YouTube end screens need 5–20 s).
- Gap between blocks is measured **word to word**: last audible word of block i → first audible word of block i+1.
  - Question blocks: gap = `response_wait_seconds` (the child's thinking time).
  - Others: a natural gap (≈1.1 s).
  - Never overlap clips; if clip silences exceed the target, keep the larger gap and report it.
- Keep float seconds everywhere; quantize to frames only when drawing.

## Master voice

ffmpeg: `adelay` each clip, `amix normalize=0`, 48 kHz stereo, measure loudness (`ebur128`), apply a single linear gain to ≈ −16 LUFS + a gentle limiter. Keep originals untouched; the master is a derived file.

Also compute a per-frame RMS **voice envelope** (normalize by the 98th percentile) → drives mouth animation and music ducking.

## Mix (in the renderer)

- Voice at 1.0.
- Music: loop at ~0.2 in intro/outro, ~0.045 under speech (ramp 0.6 s around each block's words), ~0.06 during waits, fade out at the end. Per-frame `volume={(f) => …}` callback — deterministic.
- SFX as `<Sequence from={frame}><Audio/></Sequence>`, ≈0.55 × gain. Never during waits/answers-to-come.
- Target final: ≈ −16 LUFS integrated, true peak < −1 dBFS (check with `ffmpeg -af ebur128=peak=true`).

## Procedural assets (license-free)

Python/numpy (one `tools/gen_assets.py` script): paper texture (value noise + grain + vignette), SFX (pop = pitch-drop sine; crayon/pencil = band-passed noise with stroke AM; check = two bell tones; sparkle = glockenspiel arpeggio; whoosh = filtered noise swell), music (marimba chord loop with seamless tail wrap). Always record them in a licenses table.

## Subtitles

- Build cues from the real word timings mapped to `subtitle_text` lines, split into sentences, max 2 balanced lines (~40 chars/line).
- **Cumulative cues at number words** in the SRT so a number never appears before it's spoken (articles like "una canasta" are not numbers).
- File name convention: `<video-basename>.<lang>_<REGION>.srt` (e.g. `ep-05.es_CO.srt`), always next to the MP4.
- Burned-in captions (default on): white rounded bold font with dark outline in a reserved bottom band; each word appears when spoken with a small pop; the current word slightly scaled; **keywords** (numbers, comparisons, concepts — per-project list) colored + animated underline. Shorts: bigger, centered, 1–3 words at a time.
