# sayq

Collision-free queued TTS for macOS — built for AI assistants.

When multiple Claude (or other AI) sessions run simultaneously, they can try to speak at the same time. `sayq` serializes voice output using atomic filesystem locks, so sessions queue up instead of talking over each other.

## Features

- **Atomic mutex** — `mkdir`-based lock, safe on all filesystems
- **Dead-process detection** — stale locks from crashed sessions are cleared within seconds, not minutes
- **Cooldown** — configurable gap between utterances (default 5 s)
- **MacWhisper awareness** — waits for transcription to finish before speaking
- **High-quality TTS** — Chatterbox TTS API with macOS `say` fallback
- **Interruptible** — Escape / Ctrl+C stops playback cleanly

## Scripts

| Script | Purpose |
|--------|---------|
| `say-queued` | Main entry point. Acquires lock, enforces cooldown, speaks. |
| `say-tts` | TTS backend. Tries Chatterbox API, falls back to macOS `say`. |

## Usage

```bash
say-queued "Hello, world"
say-queued                    # reads from /tmp/claude-say.txt
echo "Hello" | say-queued -   # reads from stdin
```

## Configuration

Edit variables at the top of `say-queued`:

| Variable | Default | Description |
|----------|---------|-------------|
| `COOLDOWN` | `5` | Seconds to wait after previous speech |
| `STALE_LOCK_SECONDS` | `120` | Steal lock from dead process after N seconds |
| `MACWHISPER_CPU_THRESHOLD` | `15` | CPU% above which MacWhisper is considered active |
| `MACWHISPER_SETTLE_SECONDS` | `3` | Extra wait after MacWhisper goes idle |

Edit variables at the top of `say-tts`:

| Variable | Default | Description |
|----------|---------|-------------|
| `TTS_HOST` | `ollama.home` | Chatterbox API host |
| `TTS_PORT` | `8880` | Chatterbox API port |
| `TTS_VOICE` | `dutch` | Voice name passed to Chatterbox |
| `CHATTERBOX_MAX_CHARS` | `3000` | Texts longer than this go directly to `say` |

## Installation

```bash
git clone https://github.com/erikdebruijn/sayq ~/github.com/erikdebruijn/sayq
cd ~/github.com/erikdebruijn/sayq
chmod +x say-queued say-tts
ln -s "$PWD/say-queued" /usr/local/bin/say-queued
ln -s "$PWD/say-tts" /usr/local/bin/say-tts
```

## Requirements

- macOS (uses `say`, `afplay`, `pgrep`)
- [Chatterbox TTS](https://github.com/resemble-ai/chatterbox) server (optional — falls back to `say`)
- `jq` (`brew install jq`) for Chatterbox JSON payload

## How it works

```
say-queued "text"
  └── acquire_lock()           # atomic mkdir; steals lock if holder is dead
        └── wait_for_say()     # wait for any running say/afplay
        └── cooldown           # gap since last speech
        └── wait_for_macwhisper()  # don't speak while user is transcribing
        └── wait_for_say()     # final check
        └── say-tts "text"     # speak
              ├── Chatterbox API → afplay
              └── macOS say (fallback)
```

## License

MIT
