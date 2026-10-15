# QA: automatic validation + visual review

## Automatic (script `validate`, writes `qa-automatico.json`) — must be 100% before render

Script integrity: JSON schema, block order, text hashes, question references.

Per question wait window (sample start, middle, end):
- measured wait = target (±1 frame);
- visible count (or `answerGroup` count) = expected answer; anticipated answers (`'!group'`) → group empty;
- no labels/numerals/rings visible, objects still (positions identical), no SFX, no new objects;
- position answers (number line): character not on the answer tick and not moving.

Global:
- every displayed number/digit was **already spoken** in the current block (support compounds and ordinals);
- math facts: sums/comparisons match group counts in some wait (allow regrouping when the total matches);
- objects not overlapping when static: distance ≥ max(size × scale) per kind; invisible helpers (holes) size ≈ 0;
- regroupings preserve count (moves that end in removal are "take away", not regrouping);
- nothing enters the subtitle band / platform safe areas;
- no interpolated word timings.

Keep the validator strict; when a rule is truly inapplicable, add an explicit, documented scene flag instead of weakening the rule. Re-validate **all** videos after changing engine or validator.

## Visual (always)

- Before render: `stills --t <12–15 key instants>` → contact sheet; look for: overlaps/crowding, elements hidden behind the character, labels on wrong objects, off-by-one anchors (repeated phrases), text overflow, unreadable sizes, things that "run together" (e.g. pairs that don't read as pairs).
- After render: contact sheet from the MP4 at 12/30/50/70/85/95%, thumbnail check.
- Pipeline should refuse to render when validation fails.

## Output verification (`check-output`)

ffprobe: h264 High, yuv420p (bt709, tv range), expected WxH, r=avg=30/1, AAC 48 kHz, moov before mdat (faststart), duration, size/bitrate. Loudness `ebur128`: I ≈ −16 LUFS, true peak < −1 dBFS. Files present with naming conventions (SRT, thumbnail).

Always tell the user which checks were automatic, which were visual, and what was not checked (listening).
