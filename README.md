# get-yt

One-command tool to download a YouTube video, generate a transcript, and produce an LLM-powered summary — all organized into a clean per-video directory.

## Quick Start

```sh
# Simplest usage — prompts for URL, auto-detects everything
./get-yt

# Pass URL directly
./get-yt "https://www.youtube.com/watch?v=VckmK-ZCpAU"
```

## Prerequisites

**Required:**

- **yt-dlp** — `brew install yt-dlp`
- **ffmpeg** — `brew install ffmpeg`
- **python3** — included with macOS / Homebrew

**At least one LLM backend** (auto-detected in this order):

1. **ollama** — `brew install ollama` (local, no API key needed)
2. **llm** CLI — `pipx install llm` (Simon Willison's multi-provider CLI)
3. **Anthropic API** — set `ANTHROPIC_API_KEY` env var
4. **OpenAI API** — set `OPENAI_API_KEY` env var

**Optional (for `--method whisper`):**

- **mlx_whisper** — `pipx install mlx-whisper` (Apple Silicon optimized)

## Options Reference

| Flag | Description | Default |
|------|-------------|---------|
| `-u, --url URL` | YouTube video URL. Prompted interactively if omitted. | — |
| `-m, --method METHOD` | Transcript source: `auto`, `youtube`, `whisper` | `auto` |
| `-w, --whisper-model MODEL` | Whisper model identifier | `mlx-community/whisper-large-v3-turbo` |
| `-o, --output-dir DIR` | Base directory for all output | `.` (current dir) |
| `-p, --llm-provider PROVIDER` | LLM backend: `ollama`, `llm`, `anthropic`, `openai` | auto-detected |
| `-l, --llm-model MODEL` | Model name for the chosen provider | provider-specific |
| `-k, --keep-scratch` | Preserve the `.scratch/` working directory | off |
| `-h, --help` | Print usage and exit | — |
| `-v, --version` | Print version and exit | — |

## Transcript Methods (`--method`)

### `auto` (default)
Tries to download YouTube subtitles first (manual preferred, then auto-generated). If none are available, falls back to local Whisper transcription.

### `youtube`
Uses only YouTube-provided subtitles. Fails with a clear error if the video has no English subtitles. This is the fastest option — no audio extraction or model download needed.

### `whisper`
Extracts audio from the downloaded video and transcribes locally using `mlx_whisper`. Produces higher-quality transcripts than YouTube auto-subs but takes longer (roughly 1 minute per 10 minutes of video on Apple Silicon). Requires `mlx_whisper` to be installed.

## LLM Providers (`--llm-provider`)

The script makes exactly **one** LLM request per video — sending the full transcript with a structured summarization prompt.

### `ollama` (default when running)
Connects to the local Ollama server at `localhost:11434`. Auto-selects the first installed model if `--llm-model` is not set.

```sh
# Start ollama if not running
ollama serve &

# Use a specific model
./get-yt -p ollama -l gemma3:4b "$URL"
```

### `llm`
Uses Simon Willison's [llm](https://llm.datasette.io) CLI, which supports dozens of backends via plugins.

```sh
pipx install llm
llm keys set openai          # or install a plugin: llm install llm-claude-3

./get-yt -p llm -l gpt-4o "$URL"
```

### `anthropic`
Direct API call via `curl`. Requires `ANTHROPIC_API_KEY` in the environment.

```sh
export ANTHROPIC_API_KEY="sk-ant-..."
./get-yt -p anthropic "$URL"           # default: claude-sonnet-4-20250514
./get-yt -p anthropic -l claude-opus-4-20250514 "$URL"
```

### `openai`
Direct API call via `curl`. Requires `OPENAI_API_KEY` in the environment.

```sh
export OPENAI_API_KEY="sk-..."
./get-yt -p openai "$URL"              # default: gpt-4o
./get-yt -p openai -l gpt-4o-mini "$URL"
```

## Output Structure

```
output-dir/
├── .scratch/                    # temporary working space (auto-cleaned)
│   └── VIDEO_ID/
│       ├── video.mp4            # downloaded video (moved to final dir)
│       ├── video.en.vtt         # raw YouTube subtitles
│       ├── audio.m4a            # extracted audio (whisper method)
│       └── llm_prompt.txt       # the prompt sent to the LLM
│
└── Video_Title_VIDEO_ID/        # one directory per video
    ├── video.mp4                # the video file (h264/mp4)
    ├── transcript.txt           # plain-text transcript
    └── summary.md               # LLM-generated markdown summary
```

The `.scratch/` directory is automatically deleted after a successful run unless `--keep-scratch` is used.

## Resumability

The script checks for existing output files at each step and skips work that's already done. If a run fails partway through (e.g., network error during LLM call), just re-run the same command — it picks up where it left off.

## Examples

```sh
# Basic — auto-detect everything
./get-yt "https://www.youtube.com/watch?v=VckmK-ZCpAU"

# Use Whisper for higher-quality transcript
./get-yt --method whisper "$URL"

# Organize into a dedicated archive folder
./get-yt -o ~/video-archive "$URL"

# Use Anthropic API with a specific model
./get-yt -p anthropic -l claude-sonnet-4-20250514 "$URL"

# Keep scratch files for debugging
./get-yt --keep-scratch "$URL"

# Batch process a list of URLs
while IFS= read -r url; do
  ./get-yt -o ~/archive "$url"
done < urls.txt
```

## Notes

- **Context window limits:** Very long videos (3+ hours) may produce transcripts that exceed the context window of smaller models. Use a model with a large context window (e.g., Gemini, Claude, GPT-4o) for long content.
- **YouTube subtitle quality:** Auto-generated subtitles have no punctuation or speaker labels. Whisper transcripts are generally more readable.
- **Video format:** Videos are downloaded with h264 video + m4a audio in an mp4 container for broad compatibility.
- **Single LLM call:** The script sends exactly one LLM request per video to minimize cost and latency.

## Troubleshooting

**"No LLM provider found"**
Install ollama (`brew install ollama && ollama serve`), the llm CLI (`pipx install llm`), or set an API key (`export ANTHROPIC_API_KEY=...`).

**"mlx_whisper not installed"**
Run `pipx install mlx-whisper`. Only needed when using `--method whisper`.

**"No YouTube subtitles available"**
The video has no English subs. Use `--method auto` (to fall back to Whisper) or `--method whisper`.

**LLM timeout or empty output**
For local models (ollama), ensure the server is running (`ollama serve`). For cloud APIs, check your API key and network connection. The script allows up to 10 minutes for the LLM response.

**Video download fails**
Update yt-dlp: `brew upgrade yt-dlp` or `pipx upgrade yt-dlp`. YouTube frequently changes its format negotiation.
