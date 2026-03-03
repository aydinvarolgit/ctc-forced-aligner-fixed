# CTC Forced Aligner

Fixed version of the ctc-forced-aligner package with bug fixes.

## Installation

```bash
git clone https://github.com/aydinvarolgit/ctc-forced-aligner-fixed.git
cd ctc-forced-aligner-fixed
pip install -e .
```

## Usage

```bash
ctc-forced-aligner --audio_path audio.wav --text_path text.txt --language tur --romanize --split_size sentence
```

### Options

- `--audio_path`: Path to the audio file (WAV format, 16-bit mono 24000Hz recommended)
- `--text_path`: Path to the text file containing the transcription
- `--language`: Language code (e.g., "tur" for Turkish, "eng" for English)
- `--romanize`: Enable romanization for non-Latin scripts (required for Turkish without Turkish-specific model)
- `--split_size`: Split size - "sentence", "word", or "char"
- `--alignment_model`: CTC model to use (default: "MahmoudAshraf/mms-300m-1130-forced-aligner")

---

## Bug Fixes

This section documents the bugs found and fixed in the original ctc-forced-aligner library.

### Bug 1: Inverted Boolean Logic (alignment_utils.py:245)

**Issue**: The validation check for target token indices had inverted logic.

```python
# Before (WRONG):
if np.max(targets) >= log_probs.shape[-1] and np.min(targets) >= 0:
    raise ValueError("targets values must be within [0, log_probs.shape[-1])")

# After (FIXED):
if np.max(targets) >= log_probs.shape[-1] or np.min(targets) < 0:
    raise ValueError("targets values must be within [0, log_probs.shape[-1])")
```

**Explanation**: The original condition was:
- `max >= vocab_size AND min >= 0` - This is always wrong because it triggers when BOTH conditions are true
- Should be: `max >= vocab_size OR min < 0` - Triggers when EITHER condition is true

This caused the library to incorrectly raise an error or fail to catch actual out-of-bounds errors.

---

### Bug 2: Out-of-Bounds <star> Token Index (alignment_utils.py:261)

**Issue**: The `<star>` token was added to the vocabulary dictionary at an out-of-bounds index.

```python
# Before (WRONG):
dictionary = tokenizer.get_vocab()  # vocab_size = 32, valid indices: 0-31
dictionary["<star>"] = len(dictionary)  # This becomes 32! (out of bounds)

# After (FIXED):
token_indices = [dictionary[c] for c in " ".join(tokens).split(" ") 
                 if c in dictionary and c != "<star>"]  # Exclude <star> from targets
```

**Explanation**: The default MMS CTC model has a vocabulary size of 32 (characters a-z plus special tokens). Adding `<star>` at index 32 made it out of bounds since valid indices are 0-31.

The fix filters out `<star>` tokens from the target sequence since they're used for alignment accuracy but shouldn't be in the actual target indices passed to the CTC decoder.

---

### Bug 3: Incorrect Whitespace Handling (alignment_utils.py:61)

**Issue**: Using `.split(" ")` instead of `.split()` caused issues with multiple consecutive spaces.

```python
# Before (WRONG):
cur_token = tokens[tokens_idx].split(" ")  
# "i r a n   i l e" -> ['i','r','a','n','','','i','l','e']
# Empty strings from multiple spaces caused index mismatches

# After (FIXED):
cur_token = tokens[tokens_idx].split()
# "i r a n   i l e" -> ['i','r','a','n','i','l','e']
# .split() handles multiple whitespace correctly
```

**Explanation**: When text is tokenized with spaces between characters, multiple consecutive spaces (e.g., between words) result in empty strings when using `.split(" ")`. This causes mismatches between the token sequence and the alignment segments.

---

### Bug 4: <star> Token Pipeline Mismatch (align.py)

**Issue**: The `<star>` tokens were included in text_starred but not handled consistently through the pipeline.

```python
# Before (WRONG):
tokens_starred, text_starred = preprocess_text(...)
segments, scores, blank_token = get_alignments(emissions, tokens_starred, tokenizer)
spans = get_spans(tokens_starred, segments, blank_token)
results = postprocess_results(text_starred, spans, ...)  # Mismatch!

# After (FIXED):
tokens_starred, text_starred = preprocess_text(...)
tokens_no_star = [t for t in tokens_starred if t != "<star>"]
text_no_star = [t for t in text_starred if t != "<star>"]

segments, scores, blank_token = get_alignments(emissions, tokens_no_star, tokenizer)
spans = get_spans(tokens_no_star, segments, blank_token)
results = postprocess_results(text_no_star, spans, ...)  # Consistent!
```

**Explanation**: The `<star>` tokens were added for alignment accuracy but caused index mismatches when processing spans and post-processing results. The fix removes them from the pipeline after alignment is complete.

---

### Bug 5: Empty Token Filtering (align.py)

**Issue**: Numbers and special characters that romanize to empty strings caused IndexError.

```python
# Example text: "3) Lübnan'a 27 Eylül 2024"
# After romanization: token="" for "3)", "27", "2024"
# This caused: IndexError: list index out of range

# After (FIXED):
# Filter out empty tokens while maintaining alignment between tokens and text
pairs = [(t, tx) for t, tx in zip(tokens_starred, text_starred) 
         if t != "<star>" and t.strip() != ""]
tokens_no_star = [t for t, _ in pairs]
text_no_star = [tx for _, tx in pairs]

# Also add validation to give clear error message
if not tokens_no_star:
    raise ValueError(
        f"No valid tokens found after preprocessing. "
        f"This may happen if the text contains only characters not in the model's vocabulary..."
    )
```

**Explanation**: When text contains numbers like "3)", "27", "2024", the romanizer converts them to empty strings. These empty tokens caused IndexError when iterating through characters. The fix filters them out and provides a clear error message for edge cases.

---

## Root Cause Analysis

The MMS-300M forced aligner model expects:
- Character-level tokens (vocab of ~32 characters)
- Token indices within [0, vocab_size)
- The default model is trained for English/Latin scripts

For Turkish, the `--romanize` flag converts Turkish characters (ş, ç, ğ, ü, ö, ı) to ASCII equivalents, which is necessary because the default model vocabulary doesn't include these characters.

---

## Testing

Test with Turkish audio:
```bash
ctc-forced-aligner \
  --audio_path "path/to/audio.wav" \
  --text_path "path/to/text.txt" \
  --language "tur" \
  --romanize \
  --split_size sentence
```

Test with English:
```bash
ctc-forced-aligner \
  --audio_path "path/to/audio.wav" \
  --text_path "path/to/text.txt" \
  --language "eng" \
  --split_size sentence
```

## Known Edge Cases

### Empty Token Issue
If the text contains only characters that romanize to empty strings (e.g., pure numbers "12345" or special characters "@#$"), the aligner will now raise a clear error message:

```
ValueError: No valid tokens found after preprocessing. This may happen if the text 
contains only characters not in the model's vocabulary (e.g., numbers, special 
characters, or non-Latin scripts without --romanize).
```

**Workaround**: Ensure your text contains at least some alphabetic characters, or remove numbers/special characters from the input text.

### Characters Not in Vocabulary
The default MMS-300M model uses a 32-character Latin alphabet. Characters that may cause issues:
- Turkish: ş, ç, ğ, ü, ö, ı (use `--romanize` flag)
- Numbers: 0-9 (romanize to empty)
- Special symbols: @, #, $, %, etc. (romanize to empty)
- Non-Latin scripts: Chinese, Arabic, Japanese, etc. (require `--romanize` or language-specific model)

## License

MIT License - see original repo: https://github.com/MahmoudAshraf97/ctc-forced-aligner
