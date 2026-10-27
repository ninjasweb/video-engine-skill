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

General rule for every video, burned-in or SRT.

**Cues**
- Build cues from the real word timings mapped to `subtitle_text` lines (tags stripped), split into sentences, max 2 balanced lines (~40 chars/line landscape, ~24 on 9:16).
- **Cumulative cues at number words**, so a number never appears before it is spoken. Articles like "una canasta" are not numbers.
- Extend each cue up to 0.8 s for reading, never overlapping the next one.
- File name convention: `<video-basename>.<lang>_<REGION>.srt` (e.g. `ep-05.es_CO.srt`), always next to the MP4. The SRT uses the same cues as the burned-in captions.

**Numbers: words for the voice, digits for the reader**
- `tts_text` keeps numbers as words ("dos mil veinte") so the voice reads them right and the alignment contains them.
- Subtitles and SRT show digits ("2020"). After mapping words to timings, run a `numeralize` step:
  - merge each run of number words into one token that starts at the first word and ends at the last;
  - a single "un/una/uno" followed by a word is an article and stays a word ("una ciudad"); "Una." as a count is a number;
  - join "y" only between tens and units ("noventa y dos" → 92);
  - times after "las/la": "las nueve y ocho" → "9:08", "las once y media" → "11:30", "y cuarto" → ":15";
  - years (1000–2099 after "en/de/del/desde/hasta/año") without a separator ("en 2020"); other values ≥ 1000 grouped with the locale's separator ("23.000", "1.500", "229.000"), matching the on-screen cards;
  - keep the punctuation of the first and last word ("¿", ",", ".").
- The merged token keeps the number keyword kind, so it is still highlighted and still splits cues.
- Other languages: same idea with that language's number words and separators.

**Burned-in captions (default on in shorts)**
- White bold font with a dark outline or a dark rounded box, in a reserved bottom band clear of the platform UI.
- **Karaoke (CapCut style)**:
  - the word being spoken changes to one accent colour (e.g. `#FFD23F`); the active word is the last one with `start ≤ t + 0.02`;
  - upcoming words of the cue are drawn dimmer (~0.78 opacity), spoken ones in white;
  - a short pop on the active word: scale ≤ 1.07, decaying in ~0.13 s;
  - never a persistent scale: it eats the space and words run together ("advertenciasno"). Keep a fixed gap between words (`margin: 0 0.06em` plus the normal space).
- **Keywords** (numbers, comparisons, concepts; per-project list) bold or coloured.
- Shorts: bigger and centred, 1–2 lines at a time.

**Check** on the final MP4, not only on stills: crop the subtitle band at lines that contain numbers, times and long words. Look for digits, one highlighted word, and visible spaces.
