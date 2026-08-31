# TimbreFrame

TimbreFrame is a multichannel additive synthesis project. A source recording is analyzed into harmonic partials, those partials are turned into a JSON preset, and the browser frontend plays them as up to 24 independent sine channels (one speaker per partial in the installation).

Project Confluence link: **[https://momath1.atlassian.net/wiki/x/CwCEJw](https://momath1.atlassian.net/wiki/x/CwCEJw)**

```
source audio  →  analyze.py  →  presets.json  →  front_end/presets.json  →  browser
```

Use `sound_decomposition_backend.ipynb` to explore and tune analysis before committing settings to `analyze.py`.

---

## Project layout

```
timbreframe/
├── analyze.py                        # CLI: analyze audio and write presets
├── presets.json                      # Instrument library (written by analyze.py)
├── sound_decomposition_backend.ipynb # Interactive R&D for analysis and synthesis
├── assets/                           # Source recordings (WAV/MP3)
└── front_end/
    ├── index.html                    # UI: preset picker, ADSR, visualizer
    ├── main.js                       # Web Audio synthesis (24-channel)
    ├── styles.css
    └── presets.json                  # Copy of presets.json — loaded by the browser
```

---

## Adding a new instrument (`analyze.py`)

`analyze.py` is the production script. It:

1. Loads a mono audio file and runs an STFT.
2. Finds spectral peaks (target: **24 harmonics** above 200 Hz).
3. Simulates the same multichannel synthesis the frontend uses (ADSR, vibrato, amplitude modulation, per-partial attack).
4. Normalizes magnitudes so the loudest partial peaks at `0.5`.
5. Appends or overwrites an entry in `presets.json` at the repo root.

### Prerequisites

```bash
pip install numpy librosa scipy
```

Place your source recording in `assets/` (for example `assets/my_instrument.wav`). A short, steady sustain segment works best.

### Basic usage

```bash
python analyze.py \
  --id my_instrument \
  --name "My Instrument" \
  --audio ./assets/my_instrument.wav
```

`--id` is the key in `presets.json`. `--name` is the label shown in the frontend dropdown.

### Tuning synthesis parameters

All parameters have CLI flags. Defaults live in `DEFAULT_CONFIG` at the top of `analyze.py`.

| Flag | Purpose |
|------|---------|
| `--sr` | Sample rate (default `44100`) |
| `--duration` | Synthesis length in seconds (default `8.0`) |
| `--vib-freq`, `--vib-mag`, `--vib-attack` | Vibrato rate, depth, and fade-in time |
| `--am-freq`, `--am-depth` | Amplitude modulation (tremolo / breath flutter) |
| `--attack-time`, `--decay-time`, `--sustain-level`, `--release-time` | Global ADSR envelope |
| `--attack-per-partial` | Stagger attack by harmonic rank (useful for brass) |

Example for a brass-like attack:

```bash
python analyze.py \
  --id trumpet \
  --name "Trumpet" \
  --audio ./assets/chopped_trumpet.wav \
  --attack-time 0.01 \
  --decay-time 0.4 \
  --sustain-level 0.8 \
  --release-time 0.15 \
  --attack-per-partial 0.002 \
  --vib-mag 0.5
```

### What gets written to `presets.json`

Each instrument entry contains:

- `frequencies` — detected partial frequencies (Hz)
- `magnitudes` — normalized peak amplitudes (0–0.5)
- `adsr`, `vibrato`, `amplitude_modulation` — envelope and modulation settings
- `audio_settings` — `sr` and `duration`

### Peak detection notes

During analysis, `analyze.py` prints how many peaks were found:

- **Exactly 24** — ideal for the 24-channel installation.
- **Fewer than 24** — try a cleaner sustain segment, or adjust peak-finding in the notebook first.
- **More than 24** — the list is truncated to 24.

### Sync presets to the frontend

`analyze.py` writes to `presets.json` at the repo root. The frontend loads `front_end/presets.json`, so copy after each run:

```bash
cp presets.json front_end/presets.json
```

---

## Jupyter notebook (`sound_decomposition_backend.ipynb`)

Use this notebook **before** running `analyze.py` when you are working on a new instrument or tuning parameters interactively.

It covers the same pipeline as `analyze.py`, but step by step with plots and audio playback:

1. **Load and crop** — load from `assets/`, trim to a useful sustain region, listen back.
2. **Spectrogram** — inspect harmonic structure over time.
3. **Peak picking** — experiment with `find_peaks` / `librosa.util.peak_pick` until you get ~24 stable partials.
4. **Synthesis prototype** — build ADSR, vibrato, and AM envelopes; hear the reconstructed timbre.
5. **Export partials** — optionally write per-harmonic WAV files for listening on individual channels.
6. **Config workbench** — dial in `test_config` (vibrato, ADSR, AM) and compare results before copying values into `analyze.py` CLI flags.

**Typical workflow:**

```
notebook (explore + tune)  →  analyze.py  →  cp presets.json  →  frontend
```

Once parameters sound right in the notebook, pass the same values to `analyze.py`, copy the preset file, and verify in the browser.

---

## Running the frontend

The page must be served over HTTP (not opened as a `file://` URL) so the browser can fetch `presets.json`.

```bash
cd front_end
python -m http.server 8000
```

Open `http://localhost:8000`.

The UI includes:

- **Preset picker** — switch between instruments from `presets.json`
- **Per-partial sliders** — tune frequency and magnitude per channel
- **Magnitude multislider** — adjust all partial levels at once (NexusUI)
- **ADSR envelope** — visual envelope editor; enable "Interactive ADSR" to reshape attack/decay/sustain/release live
- **Spectrum visualizer** — real-time frequency display while playing
- **Stereo preview** — pan partials left/right when you do not have a 24-channel audio interface; leave unchecked for the real multichannel output path

---

## Existing instruments

`presets.json` currently includes:

| ID | Name |
|----|------|
| `acoustic_cello` | Acoustic Cello |
| `acoustic_cello1` | Acoustic Cello1 |
| `bass_clarinet` | Bass Clarinet |
| `trumpet` | Trumpet |

Source recordings for these live in `assets/`.
