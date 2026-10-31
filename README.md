# voice-host

The app's own speech service. It replaces the two voice capabilities that used to be rented:
a voice catalogue with text-to-speech (Piper, the same engine `avatar-host` already renders with)
and a voice clone (XTTS-v2, when its weights are on the volume).

No vendor is called. Reference audio never leaves the container's volume.

## Contract

Same shape as `avatar-host`, so one client pattern serves both.

| route | what it does |
|---|---|
| `GET /health` | engine state, clone count, limits. First thing a deploy is verified against. |
| `GET /voices` | presets plus every clone on the volume |
| `POST /clone` | multipart WAV + name → stores reference audio privately, returns a voice id |
| `POST /speak` | `{text, voice_id}` → writes a WAV, returns its id, byte count and url |
| `GET /renders` | every WAV on the volume |
| `GET /renders/{id}.wav` | the file itself |
| `GET /renders/{id}` | its size and url |

Everything except `/health` requires `Authorization: Bearer $HOST_TOKEN`.

## Limits (env)

| var | default | meaning |
|---|---|---|
| `MAX_CHARS` | 20000 | longest single utterance |
| `MAX_SECONDS` | 600 | synthesis timeout |
| `MAX_UPLOAD_BYTES` | 26214400 | largest reference audio accepted |
| `PIPER_VOICE` | `en_US-lessac-medium` | baked-in voice |
| `DATA_DIR` | `/data` | volume root for `renders/`, `voices/`, `piper/`, `xtts/` |

## Deploy (Railway, same pattern as avatar-host)

1. New service from this repo, **root directory `voice-host`**.
2. Variables: `HOST_TOKEN` (a long random string). Leave `PORT` alone, Railway injects it.
3. Volume mounted at `/data`.
4. Deploy, then check `GET /health` answers `{ok: true}`.

For the clone engine, build with `--build-arg WITH_CLONE=true` and allow ~2.5 GB for weights on the
volume at `/data/xtts`. Without it, `/clone` still stores reference audio and `/speak` on a clone
answers `409` with the reason instead of producing a broken file.

## Honest ceiling

Piper speaks; it does not clone. XTTS clones closely but not identically, and its per-language
quality varies. Neither is HeyGen's voice model, and neither is claimed to be.