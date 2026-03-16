# AutoEIT Audio-to-Text Transcription Pipeline

A reproducible Automatic Speech Recognition pipeline for Spanish Elicited Imitation Task recordings, implemented as a single end-to-end notebook. The system processes participant MP3 recordings, segments continuous Whisper output into 30 sentence-aligned transcriptions per participant, applies minimal post-processing that preserves learner production, and exports analysis-ready artifacts including Excel outputs and evaluation plots.

## Scope and Research Context

This repository focuses on transcription for the AutoEIT workflow, not direct rubric scoring. The pipeline is designed for second language acquisition analysis where learner disfluencies and grammar deviations must be retained rather than normalized away.

In EIT data, each participant hears a stimulus sentence and repeats it from memory. The transcription target is the learner's produced utterance, not a corrected grammatical form.

## Repository Layout

```text
audio_to_text_transcription/
├── notebook.ipynb
├── dataset/
│   ├── Example_EIT Transcription and Scoring Sheet.xlsx
│   ├── Faretta-Stutenberg et al (2023) RMAL.pdf
│   ├── GSoC 2026 TEST Description - AutoEIT.docx
│   ├── README.docx
│   ├── Spanish EIT - Experimental Protocol and Scoring Rubric.docx
│   ├── Spanish EIT Scoring Rubric.docx
│   └── Sample Audio Files and Transcriptions/
│       ├── 038010_EIT-2A.mp3
│       ├── 038011_EIT-1A.mp3
│       ├── 038012_EIT-2A.mp3
│       ├── 038015_EIT-1A.mp3
│       ├── AutoEIT Sample Audio for Transcribing.xlsx
│       └── AutoEIT Sample Transcriptions for Scoring.xlsx
└── results/
    ├── clean_transcripts.csv
    ├── dataset_overview.png
    ├── dataset_summary.json
    ├── evaluation_metrics.csv
    ├── final_transcriptions.xlsx
    ├── model_comparison.png
    ├── raw_transcripts.csv
    ├── results_dashboard.png
    ├── sentence_analysis.png
    ├── wer_analysis.png
    └── whisper_segments.json
```

## Dataset and Experimental Units

- Audio inputs: 4 participant recordings (`038010`, `038011`, `038012`, `038015`)
- EIT versions: `1A` and `2A`
- Expected unit count: 30 sentence responses per participant per model
- Total transcribed records per model set: $4 \times 30 = 120$
- Total rows in the combined raw/clean outputs for two models: $2 \times 120 = 240$

### Audio Handling Assumptions

The notebook applies fixed offsets to skip instruction/practice content before EIT responses:

- `038010_EIT-2A.mp3`: 150 s
- `038011_EIT-1A.mp3`: 150 s
- `038012_EIT-2A.mp3`: 720 s
- `038015_EIT-1A.mp3`: 150 s

These offsets are a critical part of the current segmentation quality.

## Pipeline Architecture

The implementation in `notebook.ipynb` is organized into 6 stages.

1. Stage 1: Dataset analysis and workbook mapping
2. Stage 2: Whisper transcription and sentence segmentation
3. Stage 3: Minimal transcript cleaning
4. Stage 4: Excel template population
5. Stage 5: Evaluation (WER/CER and cross-model consistency)
6. Stage 6: Visualization and dashboard export

## Technical Methodology

### 1. Environment and Path Resolution

The notebook attempts to auto-detect dataset paths from multiple candidates and falls back to current working directory. This supports both local and notebook-hosted execution patterns.

### 2. ASR Models

Two Whisper model sizes are evaluated:

- `small`
- `medium`

Selected inference settings include:

- `task="transcribe"`
- `language="es"`
- `word_timestamps=True`
- `condition_on_previous_text=False`
- `no_speech_threshold=0.6`
- `logprob_threshold=-1.0`
- `compression_ratio_threshold=2.4`

`condition_on_previous_text=False` is intentionally used to reduce context-driven hallucination loops in repetitive/disfluent learner speech.

### 3. Gap-Based Sentence Segmentation

Whisper returns continuous segments. The notebook converts these into exactly 30 sentence bins using a deterministic algorithm:

1. Filter out non-meaningful segments (punctuation/noise-only)
2. Merge adjacent segments if inter-segment gap < 1.5 s
3. Compute all remaining inter-merge gaps
4. Select the largest 29 gaps as boundaries
5. Group merged segments into 30 utterance bins
6. Pad with empty strings if fewer than 30 bins are produced

This is a constrained alignment strategy tuned for fixed-length EIT sessions.

### 4. Cleaning Strategy (Minimal, Error-Preserving)

Post-processing intentionally avoids grammar correction. It only addresses high-confidence ASR artifacts and formatting noise.

Current accent fix patterns include:

- `despues -> después`
- `tambien -> también`
- `peliculas -> películas`
- `dificil -> difícil`
- `policia -> policía`
- `ladron -> ladrón`
- `exámen -> examen`

Additional normalization:

- trim quotes/whitespace
- collapse repeated spaces
- normalize punctuation spacing
- remove trailing periods
- capitalize initial letter where appropriate

## Data Contracts and Output Schemas

### `results/raw_transcripts.csv`

Row granularity: one row per `model x participant x sentence_num`.

Columns:

- `model`
- `participant_id`
- `eit_version`
- `filename`
- `sentence_num`
- `stimulus`
- `raw_transcript`

### `results/clean_transcripts.csv`

Same key fields as raw output, with cleaned text included.

Additional column:

- `clean_transcript`

### `results/evaluation_metrics.csv`

Evaluation record format:

- `evaluation_type` (`ASR_vs_stimulus` or `cross_model`)
- `model` (`small`, `medium`, or `small_vs_medium`)
- `participant_id`
- `wer`
- `cer`
- `num_sentences`

### `results/final_transcriptions.xlsx`

Workbook generated from template with column C filled using the selected best model (`medium` in current notebook execution logic).

### Plot Outputs

- `dataset_overview.png`
- `model_comparison.png`
- `wer_analysis.png`
- `sentence_analysis.png`
- `results_dashboard.png`

## Current Results in This Repository

Computed from existing `results/evaluation_metrics.csv`.

### ASR vs Stimulus (aggregated over participants)

| Model | Mean WER | Mean CER | Min WER | Max WER |
|------|----------:|----------:|--------:|--------:|
| small | 1.0253 | 0.8032 | 0.8675 | 1.1812 |
| medium | 1.1134 | 0.8824 | 1.0396 | 1.2169 |

### Cross-Model Consistency

- Mean WER (`small` vs `medium`): `1.1878`

### Fill Rate from `clean_transcripts.csv`

| Model | Participant | Filled / 30 | Fill Rate |
|------|-------------|------------:|----------:|
| medium | 038010 | 30 | 100.0% |
| medium | 038011 | 30 | 100.0% |
| medium | 038012 | 28 | 93.3% |
| medium | 038015 | 27 | 90.0% |
| small | 038010 | 30 | 100.0% |
| small | 038011 | 30 | 100.0% |
| small | 038012 | 20 | 66.7% |
| small | 038015 | 28 | 93.3% |

## Interpreting WER in EIT

WER is computed as:

$$
\text{WER} = \frac{S + D + I}{N}
$$

where $S$ is substitutions, $D$ deletions, $I$ insertions, and $N$ reference length.

In this task, `ASR_vs_stimulus` mixes two effects:

1. learner deviation from the stimulus (expected behavior in EIT)
2. ASR recognition/segmentation error

Because learner speech can diverge and can include insertions/disfluencies, WER values greater than 1.0 are expected and not automatically indicative of model failure.

## How to Run

### Prerequisites

- Python 3.9+
- ffmpeg available on PATH (recommended for robust audio decoding)
- Optional GPU acceleration through PyTorch CUDA or Apple MPS

### 1. Create and activate environment

Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 2. Install dependencies

```powershell
pip install openai-whisper pandas openpyxl jiwer matplotlib seaborn torch numpy
```

### 3. Run the notebook

```powershell
jupyter notebook notebook.ipynb
```

Then execute cells in order from Stage 1 through Stage 6.

## Caching and Reproducibility Behavior

The notebook is cache-aware and reuses existing artifacts when available:

- If `results/raw_transcripts.csv` exists, transcription is not re-run.
- If `results/clean_transcripts.csv` exists, cleaning is not re-run.
- If `results/evaluation_metrics.csv` exists, evaluation is not re-run.

To force full recomputation, remove cached result files before execution.

## Implementation Notes and Constraints

- Fixed sentence count assumption: exactly 30 responses per participant.
- Segmentation is gap-based, not alignment-based to known prompts.
- Cleaning uses hand-crafted replacements, not a learned normalization model.
- Workbook mapping assumes naming compatibility between MP3 IDs and template sheet IDs.

## Known Limitations

1. Metrics confounding: `ASR_vs_stimulus` cannot cleanly separate learner error from ASR error.
2. Offset dependence: incorrect lead-in skip time degrades segmentation quality.
3. Gap-threshold sensitivity: the 1.5 s merge criterion may not generalize to all speaking rates.
4. Small sample size: only 4 participants in the current output set.
5. Path portability caveat: `results/dataset_summary.json` may contain absolute paths from prior environments and should not be treated as canonical local paths.

## Extension Directions

1. Replace gap-only segmentation with forced alignment against known stimulus prompts.
2. Add confidence and uncertainty reporting at sentence and token level.
3. Evaluate larger model variants (`large-v3`) and distillation alternatives under the same protocol.
4. Integrate downstream automated EIT scoring as a separate stage that consumes `clean_transcripts.csv`.
5. Add unit tests for segmentation edge cases and cleaning transformations.

## Citation and Usage

If this repository is used in research workflows, report:

- Whisper model version and size
- exact preprocessing offsets
- segmentation and cleaning settings
- whether cached or freshly generated outputs were used

This improves comparability across EIT transcription experiments.