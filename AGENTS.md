# AGENTS.md

Python 3.9+ package for recording, noise-filtering, reviewing, and archiving
audio sentences into numeric `.mp3` files with an INI-style index.  Every
entry is one whole sentence: the recording is saved as a single file and
indexed under the full sentence text (no word splitting).

## Commands

- Install deps: `pip install -r requirements.txt`
- Run: `python -m audio_pipeline` (it is a **package**, not a script — `python audio_pipeline.py` will not work)
- GUI mode: `python -m audio_pipeline --gui` (Tkinter tap-to-record); `--list-devices` lists devices, `--help` for flags
- No tests, lint, typecheck, or CI configured. Only sanity check: `python -m py_compile audio_pipeline\*.py` fails on Windows globbing — compile each file explicitly instead.

## Non-obvious setup / gotchas

- **ffmpeg is required for *every* save, including `wav`** — pydub routes all `AudioSegment.export(...)` calls through the ffmpeg binary, so there is no codec-free fallback. Not listed in requirements. `ensure_ffmpeg_on_path()` in `archiver.py` checks `PATH` first, then scans the WinGet `Gyan.FFmpeg` install (matching any `ffmpeg-*` build variant, not just `full_build`) and prepends its `bin` dir to the process `PATH`, so it works from a fresh shell. If it finds nothing it logs a warning naming the real failure (`[WinError 2]`) instead of letting pydub fail opaquely. On a non-WinGet install, ffmpeg must already be on `PATH`.
- **Windows console defaults to cp1252** and crashes (`UnicodeEncodeError`) on the box-drawing glyphs in the CLI. `_ensure_utf8()` in `audio_pipeline/__main__.py` must run before any print.
- **`sounddevice` and `noisereduce` are imported lazily**, inside `recorder._sd()` and `AudioFilter.reduce_noise` respectively. Both are slow to import (PortAudio init, scipy) and delayed the GUI window appearing. `import audio_pipeline` therefore no longer pulls them in — keep it that way, and do not add a module-level `import sounddevice`/`import noisereduce` back. `numpy` and `pydub` are still eager.
- `.env` is loaded via `python-dotenv` in `config.py`; `.env` is gitignored. It is **auto-created from `.env.example` on first run** (`config.ensure_env_file`, called by `load_env_file`), so a fresh checkout is runnable without manual setup — an existing `.env` is never overwritten. `python-dotenv` is treated as optional, but without it `load_settings` ignores `.env` entirely (the copy still happens).

## Architecture

Each concern is one module under `audio_pipeline/`:
- `recorder.py` — `AudioRecorder` (sounddevice) record + playback
- `filter.py` — `AudioFilter` (noisereduce spectral gating)
- `archiver.py` — `AudioArchiver`: mp3 encode + the sentence→number index
- `index.py` — `AudioIndex`: parse/render of the index text + `normalize_sentence`
- `pipeline.py` — `AudioPipeline` orchestrator: Record → Filter → Review → Archive
- `config.py` — `Settings`/`load_settings` (.env)
- `models.py` — `RecordingConfig`, `PipelineResult`
- `gui.py` — `PipelineApp` (Tkinter tap-to-record UI); optional, guarded in `__init__.py`
- `__main__.py` — entry point

## Storage layout (`.env`)

- `ARTIFACT_ROOT` (absolute, takes precedence) or `OUTPUT_DIR` sets the index destination.
- `AUDIO_SUBDIR` puts `.mp3` in a subfolder; the index stays at the root.
- `INDEX_FILENAME` names the index file (e.g. `index.txt`).
- `INDEX_START_NUMBER` sets where auto-increment begins; `INDEX_PREFIX_LENGTH` (default 2) sets section grouping width.
- `MULTI_SPEAKER` (default 0) picks the dupe policy; `REDUCE_NOISE` (default 1) toggles background noise reduction. CLI overrides: `--multi-speaker`, `--no-filter`.
- `REDUCE_NOISE` is the one setting the GUI **writes back**: the checkbox calls `config.set_env_value(...)`, which edits `.env` in place (comments and other keys preserved) so the choice survives a restart. `MULTI_SPEAKER` is deliberately session-only — the checkbox always starts from `.env`, defaulting to off.

## Index format (critical — do not regress)

`index.txt` is INI-style, grouped by a lowercase prefix of the sentence
(default 2 letters), mapping each sentence to **one or more** numeric stems
of its audio files:

```ini
[ak]
ako ay mabait = 10
[ku]
kumain ako = 1, 2
```

Meaning `ako ay mabait` is stored as `10.mp3`, while `kumain ako` has two
takes: `1.mp3` and `2.mp3`. Rules:
- Each entry is `sentence = number[, number…]` under a `[prefix]` section (prefix length = `INDEX_PREFIX_LENGTH`, default 2). A sentence maps to `Dict[str, List[int]]` internally — a single number parses into a one-element list, so pre-multi-take index files load unchanged and still render identically.
- Sentences sorted alphabetically within each section; sections sorted; numbers ascending.
- Numbers auto-increment globally (`max+1` across every take of every sentence), starting at `INDEX_START_NUMBER`.
- `normalize_sentence()` collapses whitespace and lowercases before keying, so spacing/case variants share one entry; punctuation is kept.
- NOT JSONL — an earlier version used JSONL; keep the INI format.

## Dupe sentences (two takes per sentence)

A sentence may hold several takes, typically one per speaker. `AudioArchiver.multi_speaker`
selects the policy, and it is decided in `resolve_target`, never in `register_word`:

- **Multi-speaker on** — an already-indexed sentence gets a fresh number, so nothing is overwritten: `amo = 1` + `amo = 1, 2` with `1.mp3` and `2.mp3`.
- **Multi-speaker off (default)** — an already-indexed sentence reuses a number and the audio file at that stem is replaced. `resolve_target(sentence, overwrite=n)` picks which take; `overwrite=None` means the latest. `SaveTarget.overwrites` is `True` when a save will destroy existing audio, so callers can confirm first (the GUI dropdown + `askyesno`, the CLI prompt in `_resolve_cli_target`).
- `register_word(sentence, number)` is **idempotent** — it only ensures `number` is in the sentence's list. That single call serves both policies: it appends after a multi-speaker save and is a no-op after an overwrite.
- `clean()` deletes every number of every sentence, so a multi-take sentence loses all of its files.

## Sentence archiving (one file per sentence)

There is no cutting/splitting and no speech-to-text. The user types the
whole sentence in the entry field (or at the CLI prompt) and speaks it; the
complete capture is saved as **one** audio file under that sentence, using
the same resolve → save → register sequence as any other entry. Even a
single word is treated as a one-word sentence. `normalize_sentence()` is the
only text processing: whitespace collapsed, lowercased, punctuation kept.

## Archiver sequencing (order matters)

`AudioArchiver` reads the index file from disk on **every** call, so:
1. `resolve_target(sentence, overwrite=None)` → a `SaveTarget` (number + what already exists), or `numbers_for_word(sentence)` to just read the take list
2. `save_as_mp3(audio, str(target.number))` → writes `<number>.mp3`
3. `register_word(sentence, number)` → persists the index

Number must be resolved and the audio saved before `register_word`. Do not batch
resolution calls ahead of `register_word` — each reloads the file, so unpersisted
increments cancel out. `number_for_word(word)` still exists as a thin shim over
`resolve_target` for callers that don't need the context.

## GUI (gui.py)

`PipelineApp` (Tkinter, launched via `--gui`) exposes a **Clean recordings**
button that calls `AudioArchiver.clean()` (deletes indexed `.mp3` files and the
index). The **Start numbering at** field updates `archiver.start_number` — read
it from the UI variable and push it to the archiver before any
`resolve_target`/`clean` call (see `_apply_start_number`).

Dupe + filter controls:
- **Keep every take (multi-speaker)** checkbutton → `_apply_multi_speaker()` pushes it onto `archiver.multi_speaker` and enables/disables the take picker.
- **Overwrite take** dropdown lists the takes already indexed for the sentence in the **Sentence to record** field, newest last and preselected. Populated by `_refresh_overwrite_choices()` on field `FocusOut`, after every save, after `clean()`, and manually via the **Refresh** button (`_on_refresh_takes_click`, which also reports the count in the status line). `_selected_overwrite()` re-validates the choice against the index and returns `None` (fall back to the latest) if it went stale, so a stale dropdown can never clobber the wrong file.
- **Background noise reduction** checkbutton → when on, `_on_capture_done` denoises each capture automatically; when off the raw signal is archived. The manual **Denoise** button works either way. `_on_reduce_noise_toggle` persists the choice to `REDUCE_NOISE` via `config.set_env_value` (and warns if the write fails, since the setting would otherwise silently reset).
- Every entry follows one path: `_on_save_click` normalises the field with `normalize_sentence()`, resolves the take, saves the whole buffer as one file, and registers that sentence. `_current_sentence()` centralises the normalisation used by the hint, the take picker, and saving. The plan hint updates from a `word_var` trace (no disk access); the index is re-read on `FocusOut` and around saves.
