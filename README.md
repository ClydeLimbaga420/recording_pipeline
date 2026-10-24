# CLI Audio Capture & Asset Pipeline

Modular, object-oriented for recording, filtering, reviewing, and
archiving audio assets with a structured text index.

## Features

- **AudioRecorder** — `sounddevice` device stream allocation, raw input capture
  (NumPy float32), and live playback.
- **AudioFilter** — `noisereduce` spectral-gate noise reduction for removing
  structural background hums and hiss.
- **AudioArchiver** — converts processing buffers to highly compressed `.mp3`
  via `pydub`, auto-creates output directories, and writes the structured
  `audio_index.txt` mapping each whole sentence to its audio file.
- **AudioPipeline** — orchestrates *Record → Filter → Review → Archive* with an
  interactive confirmation loop (play / re-record / re-filter / save / discard).
- **PipelineApp (GUI)** — Tkinter interface with tap-to-record / tap-again-to-stop,
  adjustable length, playback, denoise, and save/discard (\`python -m audio_pipeline --gui\`).

## Requirements

- Python 3.9+
- `ffmpeg` (or `libav`) on your PATH for MP3 encoding.

## Installation

```bash
pip install -r requirements.txt
```

## Configuration

Copy `.env.example` to `.env` and edit:

```bash
cp .env.example .env
```

Key settings (all optional — sensible defaults are used if omitted):

| Variable | Purpose |
|----------|---------|
| `OUTPUT_DIR` | Base directory for archived assets (`./recordings`) |
| `ARTIFACT_ROOT` | Absolute path override; takes precedence over `OUTPUT_DIR` |
| `AUDIO_SUBDIR` | Subfolder for `.mp3` files (e.g. `audio`); index stays at root |
| `INDEX_FILENAME` | Index file name inside the output dir |
| `DURATION_SECONDS` | Default recording length |
| `INPUT_DEVICE` | Device id (see below); empty = system default |
| `SAMPLE_RATE` / `CHANNELS` | Capture parameters |
| `FILTER_PROP_DECREASE` | Noise-reduction strength first pass |
| `FILTER_AGGRESSIVE_PROP` | Strength for re-filter passes |
| `MAX_FILTER_PASSES` | Max passes before auto-commit |
| `MP3_BITRATE` | Target MP3 bitrate |
| `MULTI_SPEAKER` | Keep every take of a sentence instead of overwriting |
| `LOG_LEVEL` | Logging verbosity |

## Usage

```bash
python -m audio_pipeline
```

You'll be prompted for the **sentence being recorded**, then for a short
audio capture. On first run, list your input devices, then set `INPUT_DEVICE`
in `.env` if the default isn't correct. Devices are always printed at startup.

### GUI mode

Launch a Tkinter UI with tap-to-record and adjustable length:

```bash
python -m audio_pipeline --gui
```

In the GUI:
- Enter the **sentence** to record. The whole recording is saved as one audio
  file indexed under that full sentence; the hint next to the field shows the
  index key.
- Set **Start numbering at** to control where auto-increment begins.
- Tap **Record** to start and tap **Record (Stop)** again to finish (freeform
  duration with a live timer), or check **Fixed duration** to record a set
  number of seconds.
- **Play** to hear the take, **Denoise** to apply noise reduction, then
  **Save** (writes `<n>.mp3` + index) or **Discard**.
- **Clean recordings** deletes all captured audio files and resets the index
  (with a confirmation prompt).

More commands: `python -m audio_pipeline --help`, `--list-devices`.

### Recording sentences

Every entry is one sentence, even if it is a single word. On **Save** the
whole recording is written as one audio file and indexed under the full
sentence. Whitespace is collapsed and text is lowercased for the index key,
so `Ako ay   mabait` and `ako ay mabait` share one entry.

Type `ako ay mabait`, say it, and you get one file and this index entry:

```ini
[ak]
ako ay mabait = 1
```

Re-recording the same sentence follows the dupe controls: *Keep every take
(multi-speaker)* adds a new number, while *Overwrite take* replaces the
selected take's audio.

### Interactive review keys

| Key | Action |
|-----|--------|
| `p` | Play current buffer via system speakers |
| `r` | Re-record and restart processing |
| `f` | Re-process with more aggressive filter weights |
| `s` | Commit buffer to archive (numeric-named MP3 + index) |
| `d` | Discard cache and exit |

## Output layout

```
# destination root (ARTIFACT_ROOT / OUTPUT_DIR)
├── audio/               # .mp3 files go here (if AUDIO_SUBDIR=audio)
│   └── 5.mp3            # numeric-named asset, e.g. 5.mp3 for one sentence
└── index.txt            # sentence→number index (grouped by 2-letter prefix)
```

## Index format

`index.txt` is an INI-style text file grouping each recorded sentence by a
2-letter lowercase prefix and mapping it to the numeric stem of its audio file:

```ini
[ak]
ako ay mabait = 4

[ku]
kumain ako = 5

[si]
siya ay mabait = 9
```

- **`siya ay mabait = 9`** means that sentence is stored as `9.mp3`.
- Sections use the first `INDEX_PREFIX_LENGTH` letters (default 2).
- Numbers auto-increment globally; `INDEX_START_NUMBER` sets the starting
  value (useful when continuing an existing library). Re-recording the same
  sentence reuses its number unless *Keep every take* is on.
- Sentences are sorted alphabetically within each section.

## Project structure

```
audio_pipeline/         # package
├── __init__.py         # public exports + version
├── __main__.py         # entry point (python -m audio_pipeline [--gui])
├── config.py           # Settings + .env loading (python-dotenv)
├── models.py           # RecordingConfig, PipelineResult
├── index.py            # AudioIndex (sentence→number parse/render)
├── recorder.py         # AudioRecorder (sounddevice, fixed + streaming)
├── filter.py           # AudioFilter (noisereduce)
├── archiver.py         # AudioArchiver (pydub + index)
├── gui.py              # PipelineApp (Tkinter tap-to-record UI)
└── pipeline.py         # AudioPipeline orchestrator

.env.example            # sample configuration
requirements.txt        # Python dependencies
.gitignore              # ignores artifacts, .env, recordings, index
README.md               # this file
```

## Notes

- The index is grouped by the first two letters of each sentence and sorts
  entries alphabetically; each entry maps a full sentence to its numeric
  audio file.
- Numbers auto-increment across the whole index; re-recording a sentence
  reuses its existing number instead of creating a duplicate unless
  *Keep every take* is on.
- Tune filtering intensity via `prop_decrease`; the pipeline escalates to a
  configurable aggressive weight on subsequent passes.
