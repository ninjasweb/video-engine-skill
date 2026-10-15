# video-engine — a Claude Code skill

A skill that teaches an agent to produce narrated, animated videos with code. The pipeline is
script → voice (ElevenLabs TTS or a supplied recording) → word-level alignment → deterministic animation (Remotion 4 + SVG)
→ burned-in subtitles + SRT, SFX/music mix → validated MP4 (16:9 / 9:16) + thumbnail + upload metadata.

It is style-agnostic and has been used for kids' math series and for technical explainers. Its core rules:
- the audio is the clock;
- every visual event is anchored to a spoken word;
- the scene is a pure function of time;
- validate before rendering and look at the frames;
- spend money only with consent;
- report honestly what was and wasn't checked.

## Install

```bash
git clone https://github.com/ninjasweb/video-engine-skill.git ~/.claude/skills/video-engine
```

Or copy the folder into a project's `.claude/skills/video-engine/`. Claude Code loads it automatically when a task
matches its description. You can also invoke it with `/video-engine`.

## Contents

| File | What it covers |
|---|---|
| `SKILL.md` | Principles, workflow, stack, lessons learned, how to start a project |
| `references/project-layout.md` | Many videos per repo: shared engine, reusable styles, one folder per video |
| `references/style-design.md` | Inferring or proposing a visual identity (style bible) |
| `references/voice-and-alignment.md` | ElevenLabs with timestamps, caching, imported clips, forced alignment |
| `references/timeline-and-audio.md` | Clip placement, loudness, music ducking, SFX, subtitles |
| `references/animation-remotion.md` | Scene data model, authoring kit, character rig, motion language |
| `references/textures-and-rendering-looks.md` | Seeded SVG filter recipes (crayon, watercolor, neon…) |
| `references/qa-and-validation.md` | Automatic rules, visual review, output checks |
| `references/render-and-performance.md` | Render settings, memory, disk, deliverables, upload metadata |
| `references/formats-and-niches.md` | Aspect ratios, safe areas, niche playbooks |

## Requirements (for the videos, not the skill)

Node 18+, Remotion 4, ffmpeg. Optionally Python with stable-ts (local alignment) and an ElevenLabs API key (TTS).
Check Remotion's license for your company size.

## License

MIT — see [LICENSE](LICENSE). See [CHANGELOG.md](CHANGELOG.md) for versions.
