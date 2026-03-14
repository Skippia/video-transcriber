# MediaScribe

> *Point at a video. Get a markdown transcript.*

Local video/audio transcription to clean markdown files. Pass a file or an entire folder — get accurate transcriptions with automatic language detection, paragraph merging, and a live progress bar. Powered by [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (CTranslate2 backend) running entirely on your machine — or use `--cloud` to transcribe via OpenRouter API (Gemini models) with concurrent processing and cost estimation.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/Python-3.11+-green.svg)](https://www.python.org/)
[![faster-whisper](https://img.shields.io/badge/Whisper-faster--whisper-orange.svg)](https://github.com/SYSTRAN/faster-whisper)

## Why MediaScribe?

Uploading videos to Gemini/ChatGPT for transcription is slow, manual, and uncomfortable. This tool does it in one command — point at a file or folder, get `.md` files next to each video. Batch processing, smart paragraph merging, and it remembers what's already been transcribed. Need to process hundreds of files fast? Switch to `--cloud` mode for parallel API-based transcription.

## Features

- **Single file or entire folder** — pass a file path or a directory, it handles both
- **Smart paragraph merging** — Whisper segments are joined into coherent sentences based on punctuation and pauses, not dumped line-by-line
- **Skip already transcribed** — re-run on a folder safely; files with existing `.md` are skipped
- **Live progress bar** — percentage, elapsed time, and ETA for each file
- **Auto language detection** — or force a specific language with `-l`
- **Zero setup** — `uv` auto-creates the Python environment and installs dependencies on first run
- **100% local** — nothing leaves your machine (default mode)
- **Cloud mode** — transcribe via OpenRouter API (Gemini models) with parallel processing
- **Dry run cost estimation** — preview total duration and estimated API cost before committing
- **Audio extraction** — extract `.mp3` from video files or entire folders

## How It Works

```
┌──────────┐    ┌──────────┐    ┌─────────────────┐    ┌──────────┐
│ Video    │───>│ ffmpeg   │───>│ faster-whisper   │───>│ Markdown │
│ (.mp4)   │    │ (audio)  │    │ (CTranslate2)   │    │ (.md)    │
└──────────┘    └──────────┘    └─────────────────┘    └──────────┘
                  16kHz mono        int8 quantized        paragraphs
                  PCM wav           CPU inference          merged
```

**Folder mode:**

```
┌────────────────┐    ┌─────────────┐    ┌──────────────────────┐
│ Scan folder    │───>│ Load model  │───>│ Transcribe each file │
│ for media      │    │ (once)      │    │ [1/N] ... [N/N]      │
│ files          │    │             │    │ skip if .md exists   │
└────────────────┘    └─────────────┘    └──────────────────────┘
```

## Supported Formats

| Video | Audio |
|-------|-------|
| `.mp4` `.mkv` `.webm` `.avi` `.mov` `.flv` `.wmv` | `.mp3` `.wav` `.flac` `.ogg` `.m4a` |

## Requirements

- **Linux** (tested on Arch/CachyOS)
- **Python 3.11+**
- **ffmpeg** — for audio extraction from video files
- **[uv](https://github.com/astral-sh/uv)** — Python package manager (auto-manages venv and dependencies)

## Installation

```bash
# Clone the repo
git clone https://github.com/Skippia/mediascribe.git
cd video-transcriber

# Make globally available
ln -s "$(pwd)/transcribe.sh" ~/.local/bin/transcribe

# That's it. First run will auto-install Python dependencies via uv.
```

### Install system dependencies (Arch Linux)

```bash
sudo pacman -S python ffmpeg
# Install uv if you don't have it
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## Usage

### Single file

```bash
transcribe video.mp4
# => creates video.md next to the video
```

### Entire folder

```bash
transcribe ./lectures/
# => creates .md for every media file in the folder
# => skips files that already have a .md
```

### With options

```bash
# Force English, use small model (faster)
transcribe video.mp4 -m small -l en

# Custom output path
transcribe video.mp4 -o notes.md

# Include timestamps
transcribe video.mp4 --timestamps
```

### Cloud transcription (OpenRouter)

```bash
# Set your API key
export OPENROUTER_API_KEY="sk-or-v1-..."

# Single file via cloud
transcribe video.mp4 --cloud

# Entire folder (recursive), 5 parallel jobs
transcribe ./lectures/ --cloud

# Use a different model, 3 parallel jobs
transcribe ./lectures/ --cloud --cloud-model google/gemini-2.5-flash --concurrency 3
```

### Dry run (cost estimation)

```bash
# Preview cost before transcribing — no API key needed
transcribe ./lectures/ --cloud --dry
```

Output:
```
☁️  ./lectures/ (dry run)
   42 media file(s), 38 to process, 4 already done

   File                          Duration
   lecture-01.mp4                01:02:15
   lecture-02.mp4                00:45:30
   ...
   ──────────────────────────────────────
   Total duration:               31:15:42 (1875.7 min)
   Estimated tokens:             ~3,601,344
   Model:                        google/gemini-3-flash-preview
   Estimated cost:               ~$3.60
```

### Audio extraction

```bash
# Extract .mp3 from a single video
transcribe --audio video.mp4

# Extract .mp3 from all videos in a folder (recursive)
transcribe --audio ./lectures/
```

> **Note:** paths with spaces must be quoted: `transcribe "/path/to/my videos/"`

## CLI Options

| Option | Description | Default |
|--------|-------------|---------|
| `input` | Path to video/audio file or folder | *(required)* |
| `-o, --output` | Output markdown file (ignored for folders) | `<input>.md` |
| `-m, --model` | Whisper model size: `tiny`, `base`, `small`, `medium`, `large-v3` | `medium` |
| `-l, --language` | Language code: `en`, `uk`, `ru`, `de`, etc. | auto-detect |
| `--timestamps` | Include `[HH:MM:SS]` timestamps in output | off |
| `-b, --batch-size` | Batch size for inference — lower = less RAM | `4` |
| `--audio PATH` | Extract `.mp3` from video file or folder (recursive) | — |
| `--cloud` | Use cloud API (OpenRouter) instead of local Whisper | off |
| `--cloud-model` | Cloud model to use | `google/gemini-3-flash-preview` |
| `--concurrency` | Number of parallel cloud transcription jobs | `5` |
| `--dry` | Estimate cost without transcribing (requires `--cloud`) | off |

## Model Size Guide

All models run locally on CPU with int8 quantization.

| Model | Speed (per 10 min audio) | Accuracy | RAM |
|-------|--------------------------|----------|-----|
| `tiny` | ~1 min | Low | ~0.5 GB |
| `base` | ~3 min | Good | ~0.5 GB |
| `small` | ~8 min | Better | ~1 GB |
| `medium` | ~15 min | Best practical | ~1.5 GB |
| `large-v3` | ~40 min | Maximum | ~3 GB |

`medium` is the default — best accuracy-to-speed ratio for CPU. Use `small` or `base` if you need faster processing and can tolerate lower accuracy.

## Output Example

```markdown
# Transcription: lecture-01.mp4

- **Language:** en (100% confidence)
- **Duration:** 07:25
- **Model:** whisper-medium

## Content

So foreplay and dirty talk, this is where it's at.

This is where the rubber hits the road because what happens, she starts to get extremely turned on.

She wants to get to a point where she wants you, where she can't wait.
```

## Progress Output

Single file:
```
⏳ Loading model 'medium'... done (3.1s)

📄 lecture-01.mp4
  📦 Extracting audio... done (0.2s)
  🔊 Transcribing...
  ┃████████████████░░░░░░░░░░░░░░┃ 54.2%  elapsed 00:26  eta 00:22
  ✅ Done in 00:35 | en (100%) | 01:04 audio
  💾 Saved: lecture-01.md

──────────────────────────────────────────────────
✅ All done in 00:35
```

Folder:
```
⏳ Loading model 'medium'... done (3.1s)

📂 ./lectures
   5 media file(s) found

──────────────────────────────────────────────────

🎬 [1/5] lecture-01.mp4
  📦 Extracting audio... done (0.2s)
  🔊 Transcribing...
  ┃██████████████████████████████┃ 100.0%  elapsed 00:35  eta --:--
  ✅ Done in 00:35 | en (100%) | 01:04 audio
  💾 Saved: lecture-01.md

⏭️  [2/5] lecture-02.mp4 — skipped (already transcribed)

🎬 [3/5] lecture-03.mp4
  ...

──────────────────────────────────────────────────
✅ All done in 04:26
   📊 4 transcribed, 1 skipped, 5 total
```

## File Structure

```
mediascribe/
├── transcribe.sh          # Shell entry point (symlink this to ~/.local/bin/)
├── pyproject.toml         # Python project config & dependencies
├── transcriber/
│   ├── __init__.py
│   ├── cli.py             # Argument parsing, file/folder dispatch
│   ├── core.py            # Audio extraction, Whisper inference, markdown output
│   └── cloud.py           # Cloud transcription via OpenRouter API
└── README.md
```

## How It Works Under the Hood

### Local mode (default)

1. **Audio extraction** — `ffmpeg` converts video to 16kHz mono WAV (optimal for Whisper)
2. **Inference** — `faster-whisper` runs the Whisper model with int8 quantization via CTranslate2
3. **Paragraph merging** — raw Whisper segments (2-5 seconds each) are merged into coherent paragraphs based on sentence-ending punctuation and pauses >1.5 seconds
4. **Markdown output** — clean markdown with metadata header and content body

In folder mode, the model is loaded **once** and reused for all files (model loading takes ~3s, so this saves significant time on batch jobs).

### Cloud mode (`--cloud`)

1. **Audio compression** — `ffmpeg` compresses to 32kbps mono MP3 (minimal upload size, sufficient for speech)
2. **API call** — base64-encoded audio is sent to OpenRouter (Gemini models) with a structured prompt
3. **JSON parsing** — the model returns timestamped paragraphs as JSON, parsed into markdown
4. **Parallel processing** — multiple files are transcribed concurrently (default: 5 workers)

Cloud mode scans directories **recursively**, skips already-transcribed files, and requires `OPENROUTER_API_KEY` environment variable.

### Dry run (`--cloud --dry`)

Reads file durations via `ffprobe` (no decoding, no API calls) and calculates cost based on Gemini's audio tokenization rate (32 tokens/sec) and model pricing.

## License

MIT
