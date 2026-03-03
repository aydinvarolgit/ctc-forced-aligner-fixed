# CTC Forced Aligner

Fixed version of the ctc-forced-aligner package with bug fixes.

## Bug Fixes

1. Fixed inverted boolean logic in `alignment_utils.py:245`
2. Fixed `<star>` token exclusion from token indices  
3. Fixed space handling in `get_spans` (using `.split()` instead of `.split(" ")`)

## Installation

```bash
pip install -e .
```

## Usage

```bash
ctc-forced-aligner --audio_path audio.wav --text_path text.txt --language tur --romanize
```
