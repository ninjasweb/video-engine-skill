# Changelog

## 1.1.0 — 2026-10-27

- Voice: step-by-step ElevenLabs setup (key, voice, `eleven_v3` tags, history recovery, forced alignment) and **Fish Audio as an
  alternative**: the streaming endpoint with timestamps (`/v1/tts/stream/with-timestamp`, cumulative alignment per chunk),
  the `model` header, `reference_id` voices, tag syntax per model (S2 `[free text]`, S1 `(tags)`), `normalize: false`,
  billing per UTF-8 byte, and no history to recover timeouts.
- Provider-independent rules (consent, cache key with the provider, uncertain blocks, error handling, listen by proxy,
  one provider and one voice per series) and a table for choosing a provider.
- New lessons: tags are provider-specific (keep an `allowed_tags` list), and numbers go in `tts_text` as words.

## 1.0.0 — 2026-10-15

First public version.
- Generic version of the skill: no links to local projects or absolute paths.
- Multi-video layout from the first video (`references/project-layout.md`): shared engine, reusable styles,
  per-video folders, 3 levels of look reuse.
- Lessons for supplied audio that differs from the script, technical fidelity (computed vs illustrative), formulas
  via MathJax, automatic recentering of diagrams, seamless loops, and photos inside SVG.
