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

1. **LM Studio** — [lmstudio.ai](https://lmstudio.ai) (local, Apple Silicon optimized, no API key)
2. **ollama** — `brew install ollama` (local, no API key needed)
3. **llm** CLI — `pipx install llm` (Simon Willison's multi-provider CLI)
4. **Anthropic API** — set `ANTHROPIC_API_KEY` env var
5. **OpenAI API** — set `OPENAI_API_KEY` env var

**Optional (for `--method whisper`):**

- **mlx_whisper** — `pipx install mlx-whisper` (Apple Silicon optimized)

## Options Reference

| Flag | Description | Default |
|------|-------------|---------|
| `-u, --url URL` | YouTube video URL. Prompted interactively if omitted. | — |
| `-m, --method METHOD` | Transcript source: `auto`, `youtube`, `whisper` | `auto` |
| `-w, --whisper-model MODEL` | Whisper model identifier | `mlx-community/whisper-large-v3-turbo` |
| `-o, --output-dir DIR` | Base directory for all output | `.` (current dir) |
| `-p, --llm-provider PROVIDER` | LLM backend: `lmstudio`, `ollama`, `llm`, `anthropic`, `openai` | auto-detected |
| `-l, --llm-model MODEL` | Model name for the chosen provider | provider-specific |
| `-a, --audio-only` | Download audio only (m4a) — no video | off |
| `-t, --transcript-only` | Get transcript only — no media download | off |
| `-d, --diarize` | Add speaker labels via LLM post-processing | off |
| `-n, --no-summary` | Skip the LLM summarization step | off |
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

The script makes exactly **one** LLM request per video — sending the full transcript with a structured summarization prompt. With `--diarize`, a second request is made for speaker attribution.

### `lmstudio` (default when running)
Connects to [LM Studio](https://lmstudio.ai)'s local server at `localhost:1234` via its OpenAI-compatible API. Auto-selects the currently loaded model. No API key required.

LM Studio runs Apple Silicon-optimized models (MLX, GGUF) and is the recommended local provider for macOS. Install from [lmstudio.ai](https://lmstudio.ai), load a model, and enable the local server.

```sh
# Auto-detected when LM Studio server is running
./get-yt "$URL"

# Explicitly select LM Studio
./get-yt -p lmstudio "$URL"

# Use a specific loaded model
./get-yt -p lmstudio -l "qwen/qwen3-vl-8b" "$URL"
```

### `ollama`
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
Direct API call via `python3`. Requires `ANTHROPIC_API_KEY` in the environment.

```sh
export ANTHROPIC_API_KEY="sk-ant-..."
./get-yt -p anthropic "$URL"           # default: claude-sonnet-4-20250514
./get-yt -p anthropic -l claude-opus-4-20250514 "$URL"
```

### `openai`
Direct API call via `python3`. Requires `OPENAI_API_KEY` in the environment.

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

## Selective Output Modes

### `--diarize` (`-d`)
Adds speaker labels to the transcript using an LLM post-processing pass. The raw transcript is kept as `transcript.txt` and the labeled version is saved as `transcript.diarized.txt`. The summary (if generated) uses the diarized version for better context.

The LLM infers speaker names from the video title and dialogue context (e.g. "DICK CAVETT:", "BETTE DAVIS:"). Works with any LLM provider. Can be combined with `--no-summary` to get just the labeled transcript.

**Note:** Diarization quality depends heavily on model capability and context window size. Small local models may struggle with long transcripts. For best results, use a large-context model (Claude, GPT-4o, or a larger ollama model like `llama3:70b`).

### `--audio-only` (`-a`)
Downloads only the audio track as `audio.m4a` instead of the full video. Useful for podcasts, interviews, or saving disk space. The transcript and summary steps work the same — Whisper can transcribe directly from the audio file.

### `--transcript-only` (`-t`)
Skips all media downloads and only produces `transcript.txt`. Uses YouTube subtitles (or subtitle-only fetch). Implies `--no-summary`. Ideal for quickly grabbing what was said without storing any media.

### `--no-summary` (`-n`)
Skips the LLM summarization step. Produces the media file and transcript but no `summary.md`. Useful when you don't have an LLM provider configured or just want the raw transcript.

## Examples

```sh
# Basic — auto-detect everything
./get-yt "https://www.youtube.com/watch?v=VckmK-ZCpAU"

# Add speaker labels to transcript
./get-yt --diarize "$URL"

# Audio only — no video file
./get-yt --audio-only "$URL"

# Just the transcript — no media, no summary
./get-yt --transcript-only "$URL"

# Download + transcript, skip the LLM summary
./get-yt --no-summary -o ~/archive "$URL"

# Use Whisper for higher-quality transcript
./get-yt --method whisper "$URL"

# Use Anthropic API with a specific model
./get-yt -p anthropic -l claude-sonnet-4-20250514 "$URL"

# Batch process a list of URLs
while IFS= read -r url; do
  ./get-yt -o ~/archive "$url"
done < urls.txt
```

## Notes

- **Context window limits:** Very long videos (3+ hours) may produce transcripts that exceed the context window of smaller models. Use a model with a large context window (e.g., Gemini, Claude, GPT-4o) for long content.
- **YouTube subtitle quality:** Auto-generated subtitles have no punctuation or speaker labels. Whisper transcripts are generally more readable.
- **Video format:** Videos are downloaded with h264 video + m4a audio in an mp4 container for broad compatibility.
- **LLM calls:** The script sends one LLM request per video for summarization. With `--diarize`, a second request is made for speaker attribution.

## Troubleshooting

**"No LLM provider found"**
Install LM Studio ([lmstudio.ai](https://lmstudio.ai)), ollama (`brew install ollama && ollama serve`), the llm CLI (`pipx install llm`), or set an API key (`export ANTHROPIC_API_KEY=...`).

**"mlx_whisper not installed"**
Run `pipx install mlx-whisper`. Only needed when using `--method whisper`.

**"No YouTube subtitles available"**
The video has no English subs. Use `--method auto` (to fall back to Whisper) or `--method whisper`.

**LLM timeout or empty output**
For local models (ollama), ensure the server is running (`ollama serve`). For cloud APIs, check your API key and network connection. The script allows up to 10 minutes for the LLM response.

**Video download fails**
Update yt-dlp: `brew upgrade yt-dlp` or `pipx upgrade yt-dlp`. YouTube frequently changes its format negotiation.
