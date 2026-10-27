# Voice generation and word alignment

The pipeline needs, per script block: **one audio clip + word-level timings** (`[{text, norm, start, end}]` in seconds). The provider is a detail behind that contract. Everything else (cache, manifest, consent, verification) is provider-independent and is described first.

## Provider-independent rules (apply to every TTS provider)

- **Consent before spending.** Always run a dry-run (pending blocks + characters, and bytes for Fish Audio), show it, and ask before generating. Don't invent a price: quote the provider's billing unit and let the user's plan decide. A clear instruction like "generate video 2" after you've stated the cost counts as consent for that video only.
- **Send only the spoken text** of one block (`tts_text`), never the whole script, visual notes, answers or production fields.
- **Numbers as words in `tts_text`** ("siete", not "7"). The alignment then contains the number word, subtitles can split cues at it, and the "nothing shown before it's said" check can find it. Turn the provider's text normalization off or leave it harmless.
- **Cache key** = sha256 of every input that changes the audio: provider, exact text, voice id, model, language, output format, effective voice settings, prep version. A re-render must never re-bill.
- **Mark a block "uncertain" before sending.** Write audio + raw response atomically on success. A timeout or network error leaves the block uncertain (it may have been billed). Never resend it blindly: recover it, or ask.
- **Never overwrite a good clip**; move the previous one to `superseded/`.
- **Manifest per video**: provider, origin (generated / imported / recovered), hashes, request or history ids, alignment source, verification, an events log.
- **Errors**: auth/payment/validation errors (401, 402, 403, 400, 404, 422) → stop the run, no retry. Rate limit or overload (429, 503, other 5xx) → bounded retries (≤ 3) with backoff, honoring `Retry-After`.
- **Keys** come from env / `.env` (project dir, then parent). Never print them, never put them in a frontend bundle, never commit `.env`.
- **"Listen" by proxy** after generating: transcribe each clip (Whisper) and compare to the script (normalize case/accents, map digits ↔ number words, ≥ 90 % sequence similarity), and check every number word is present. Report that the audio was not actually listened to.
- **One provider and one voice per series.** Mixing providers or voices inside a video sounds like two narrators. A provider switch means regenerating the whole video.

## ElevenLabs (default)

### Setup
1. Account + API key → `ELEVENLABS_API_KEY=...` in `.env` (git-ignored; add `.env.example` without the value).
2. Pick a voice in the web app → its `voice_id`. **Free plans cannot use Voice Library voices via API** (402 `paid_plan_required`). Offer: upgrade, the user generates clips in the web app (then import + local alignment), or a different voice.
3. Model `eleven_v3` (expressive, audio tags). Language `language_code: "es"` (or the series' language).
4. Store voice, model and language in the series config and copy them into every script, so a script is self-contained. Validate them with the schema (e.g. `const` on `voice_id`).

### Request
```
POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/with-timestamps?output_format=mp3_44100_128
headers: xi-api-key: $ELEVENLABS_API_KEY, Content-Type: application/json
body: {"text": tts_text, "model_id": "eleven_v3", "language_code": "es", ["voice_settings": {...}]}
→ {audio_base64, alignment{characters[], character_start_times_seconds[], character_end_times_seconds[]}, normalized_alignment}
```
- Send `voice_settings` only when explicitly configured **and tested by ear**; otherwise the voice's defaults are used.
- **Audio tags (v3)**: `[happy]`, `[excited]`, `[curious]`, `[warm]`, `[short pause]`, `[playfully]`, `[laughs softly]`… Put at least one tag at the start of each block. Keep tags in `tts_text`; subtitles use the text with tags stripped. Don't use SSML `<break>` with v3. Keep an **allowed-tags list** per series and reject unknown tags when building scripts, so a typo isn't read aloud.
- **Recovering an uncertain block**: `GET /v1/history?voice_id=…` → match by text + time → `GET /v1/history/{history_item_id}/audio`. Recovered audio has no timestamps: align it with forced alignment (below).
- **Forced alignment of any clip** (precise alternative to Whisper): `POST /v1/forced-alignment` (multipart `file` + `text`).

### Char alignment → words
Map text index → alignment index: identity if the joined characters equal the text, otherwise a tolerant greedy match. A word's time = min start / max end over its alphanumeric characters (punctuation doesn't stretch it). Tags (`[...]`) are excluded from words but kept with the time of the next word. Never delete array entries (indices would shift).

### Commands in an engine repo
Expose them per video: a dry-run (`npm run tts -- --video V --dry-run`) → confirm → generate (`npm run tts -- --video V`) → the timeline step. Add a way to regenerate one block on purpose (`--block 03 --force`) and to recover an uncertain one (`--block 03 --recover`). If scripts are built from a compact source (one file per series listing every episode's blocks), that builder is the right place to reject unknown tags, digits in `tts_text` and questions without a wait.

## Fish Audio (alternative)

Use it when the user prefers it, wants voice cloning from a reference recording, or wants a cheaper per-byte price. Same contract: clip + word timings per block. Verified against docs.fish.audio in October 2026. Check the API reference again before implementing, because models and fields change often.

### Setup
1. API key from the Fish Audio developer dashboard → `FISH_AUDIO_API_KEY=...` in `.env`. API billing is pay-as-you-go and separate from web-app subscription credits.
2. Voice = **`reference_id`**: the model ID shown in the voice's fish.audio URL. Or clone your own voice: create a model from a clean reference recording (`POST /model`). Keep the `reference_id` fixed for the whole series.
3. Model (sent as the **`model` HTTP header**, not in the body): `s2.1-pro` (default, production), `s2-pro`, `s2.1-pro-free` (free developer tier, for tests), `drama-3`, or `s1` (older, deprecated). An unknown value silently falls back to `s2.1-pro`, so validate the name yourself.

### Request (with word timestamps)
The plain `POST /v1/tts` returns only audio. For the pipeline, use the **streaming endpoint with timestamps**:
```
POST https://api.fish.audio/v1/tts/stream/with-timestamp
headers: Authorization: Bearer $FISH_AUDIO_API_KEY, Content-Type: application/json, model: s2.1-pro
body: {"text": tts_text, "reference_id": "<voice id>", "format": "mp3", "mp3_bitrate": 128,
       "normalize": false, "language": "es", "latency": "normal",
       ["temperature": 0.7, "top_p": 0.7, "prosody": {"speed": 1, "volume": 0}]}
→ text/event-stream; each event: {audio_base64, content, alignment: null | {audio_duration, segments: [{text, start, end}]},
                                   chunk_seq, chunk_audio_offset_sec}
```
How to read the stream:
- **Audio**: concatenate every `audio_base64` chunk in arrival order.
- **Timings**: alignment is cumulative **per `chunk_seq`**. Keep a map `chunk_seq → latest alignment` and **replace** the stored snapshot instead of appending. Events may repeat a snapshot, and `alignment` can be `null`.
- **Absolute time** of a segment = `chunk_audio_offset_sec + start` (seconds). Segments are word-like but not guaranteed to be single words: map them onto the script tokens with the same sequence alignment used for Whisper, and flag unmatched words as `interpolated`.
- Store the raw events (`NN.alignment.raw.jsonl`) and the word conversion (`NN.alignment.json`), as with ElevenLabs.
- If timestamps come back empty or partial, fall back to forced alignment of the finished clip (ElevenLabs forced-alignment or stable-ts). Never invent timings.

### Expression tags: the syntax depends on the model
- `s2-pro` / `s2.1-pro`: **square brackets with free-form descriptions**, e.g. `[excited]`, `[whispering]`, `[pause]`, `[laugh]`, `[speaking slowly and warmly]`. They can go anywhere in the sentence.
- `s1`: **parentheses** from a fixed list (64+), e.g. `(happy)`, `(excited)`, at the start of the sentence.
- `drama-3`: directed in plain language, with paired tags for specific words.
- ElevenLabs v3 tags are not guaranteed to mean the same thing here. Keep a **per-provider allowed-tags list** and rewrite tags when switching provider (e.g. v3 `[short pause]` → S2 `[pause]`), then test one block by ear before a whole series.
- Strip tags of either syntax for subtitles and for word matching.

### Text normalization and language
- `normalize: true` (the default) spells out numbers, dates and units. With numbers already written as words, use **`normalize: false`**, so what is said is exactly what's written.
- `language` (BCP-47) only drives normalization and timestamp alignment, not the voice. Use `es` (or `es-ES`). `es-MX` reads currency as pesos, and `es-419` / `es-US` skip normalization.

### Cost, limits and errors
- **Billing is per million UTF-8 bytes** of input text (about $15 / M bytes for `s2.1-pro`; `s2.1-pro-free` is $0 for development). Accented letters (á, é, ñ) are **2 bytes**, so Spanish costs more than its character count suggests: the dry-run must show bytes (`Buffer.byteLength(text, 'utf8')`), not just characters. Tags count too.
- **Concurrency** is limited by spending tier (about 5 parallel requests at the starter tier). Generate blocks sequentially or with a small pool.
- Errors: 401 (no permission) and 402 (no balance) → stop. 503 (high load) → bounded retries with backoff.
- **No generation-history endpoint is documented**, so a timed-out request can't be recovered like ElevenLabs' history. Keep the block uncertain, tell the user it may have been billed, and regenerate only with their consent.

### Minimal Node sketch (no SDK)
```ts
const res = await fetch('https://api.fish.audio/v1/tts/stream/with-timestamp', {
	method: 'POST',
	headers: {Authorization: `Bearer ${key}`, 'Content-Type': 'application/json', model: 's2.1-pro'},
	body: JSON.stringify({text, reference_id, format: 'mp3', normalize: false, language: 'es'}),
});
if (!res.ok) throw new Error(`${res.status} ${await res.text()}`); // 401/402 stop, 503 retry
const audio: Buffer[] = [];
const align = new Map<number, {offset: number; segments: {text: string; start: number; end: number}[]}>();
let buf = '';
for await (const chunk of res.body as any) {
	buf += Buffer.from(chunk).toString('utf8');
	let i: number;
	while ((i = buf.indexOf('\n\n')) >= 0) {
		const ev = buf.slice(0, i);
		buf = buf.slice(i + 2);
		const data = ev.split('\n').filter((l) => l.startsWith('data:')).map((l) => l.slice(5).trim()).join('');
		if (!data) continue;
		const e = JSON.parse(data);
		if (e.audio_base64) audio.push(Buffer.from(e.audio_base64, 'base64'));
		// cumulative per chunk: replace the snapshot, don't append
		if (e.alignment) align.set(e.chunk_seq, {offset: e.chunk_audio_offset_sec, segments: e.alignment.segments});
	}
}
const words = [...align.entries()].sort(([a], [b]) => a - b).flatMap(([, a]) => a.segments.map((s) => ({text: s.text, start: a.offset + s.start, end: a.offset + s.end})));
```

## Choosing a provider

| | ElevenLabs | Fish Audio |
|---|---|---|
| Timestamps | Per character, in the normal HTTP response | Word-like segments, only on the SSE endpoint |
| Expression | v3 `[tags]` | S2 `[free text]` · S1 `(tags)` |
| Billing | Characters (plan credits) | UTF-8 bytes (pay-as-you-go) |
| Recover a timeout | History API | Not available: ask before regenerating |
| Voices | Library (paid plan for API) or cloned | Public models by `reference_id` or cloned |

Default to the provider the series already uses. For a new series, run one block through each with the same text, transcribe both, look at the timestamps, and let the user choose by ear.

## Imported / manual clips

1. Transcribe each clip (Whisper small via stable-ts) and match it to script blocks by **content**, not filename: clips can be missing, merged or misnumbered. Support an explicit mapping (`--map 04=faltante.mp3`).
2. Verify: normalize words (lowercase, strip accents, **map digits ↔ number words**) and compute sequence-alignment similarity; ≥ 90 % = OK.
3. Force-align against the script text (tags stripped): `model.align(audio, text, language=…)` → words with times. Map them to script tokens with Needleman–Wunsch, interpolate unmatched words and flag them `interpolated` (QA must report the count).
4. Precision: Whisper-based ≈ ±0.1 s; ElevenLabs character timestamps are tighter. Optional: ElevenLabs `POST /v1/forced-alignment`.

## Tokenization rules

- Word = run of non-space characters with its punctuation; norm = lowercase, no diacritics, `[a-z0-9ñ]` only. Drop tokens with an empty norm ("...").
- Number words (including compounds like "treinta y cinco", "veintitrés" and ordinals like "primer/segunda") must be recognized for the "don't reveal before it's said" checks. Write numbers as words in `tts_text`; subtitles convert them back to digits for reading (see `timeline-and-audio.md` → Subtitles).
