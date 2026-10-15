# Voice generation and word alignment

## ElevenLabs TTS with timestamps

```
POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/with-timestamps?output_format=mp3_44100_128
headers: xi-api-key: $ELEVENLABS_API_KEY, Content-Type: application/json
body: {"text": tts_text, "model_id": "eleven_v3", "language_code": "es", ["voice_settings": {...}]}
→ {audio_base64, alignment{characters[], character_start_times_seconds[], character_end_times_seconds[]}, normalized_alignment}
```

- Send **only the spoken text** of one block (never the whole script, notes or answers). v3 supports audio tags like `[excited]`, `[curious]`, `[short pause]` — keep them in `tts_text`; don't use SSML `<break>` with v3.
- Omit `voice_settings` unless explicitly configured and tested.
- Key from env / `.env` (search project dir then parent). Never print it, never ship it to the frontend/bundle.
- **Free plans cannot use Voice Library voices via API** (402 `paid_plan_required`) → offer: upgrade, user generates clips in the web app (then import + local alignment), or a different voice.
- Errors: 401/403/402/400/404/422 → stop, no retry. 429/5xx → bounded retries honoring `Retry-After`. Timeout/network error → mark block **uncertain** (may have been charged), never blind-resend; recover via `GET /v1/history?voice_id=…` matching text + time, then `GET /v1/history/{id}/audio`.
- Mark a block "uncertain" **before** sending; write audio + raw response atomically on success. Never overwrite a previous good clip — move it to `superseded/`.
- Cache key = sha256(text, voice_id, model_id, language, output_format, effective voice_settings, prep_version). A new render must never re-bill.
- Always show a dry-run (blocks + characters) and ask before spending. Don't invent a price.
- Keep a per-episode manifest: origin (generated/imported/recovered), hashes, request/history ids, alignment source, verification, events log.

## Char alignment → words

Map text index → alignment index (identity if the joined characters equal the text; otherwise a tolerant greedy match). A word's time = min start / max end over its alphanumeric characters (punctuation doesn't stretch it). Tags (`[...]`) are excluded from words but kept with the time of the next word. Never delete array entries (indices would shift).

## Imported/manual clips

1. Transcribe each clip (Whisper small via stable-ts) and match to script blocks by content, not filename — clips can be missing, merged or misnumbered. Support explicit mapping (`--map 04=faltante.mp3`).
2. Verify: normalize words (lowercase, strip accents, **map digits ↔ number words**) and compute sequence-alignment similarity; ≥90% = OK.
3. Force-align against the script text (tags stripped): `model.align(audio, text, language=…)` → words with times; map to script tokens with Needleman–Wunsch; interpolate unmatched words and flag them `interpolated` (QA must report the count).
4. Precision: Whisper-based ≈ ±0.1 s; ElevenLabs char timestamps are tighter. Optional: ElevenLabs `POST /v1/forced-alignment` (multipart file + text).

## Tokenization rules

- Word = run of non-space chars with its punctuation; norm = lowercase, no diacritics, `[a-z0-9ñ]` only; drop tokens with empty norm ("...").
- Number words (incl. compounds like "treinta y cinco", "veintitrés", ordinals "primer/segunda") must be recognized for "don't reveal before it's said" checks.
