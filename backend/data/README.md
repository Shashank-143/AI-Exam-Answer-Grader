# EngSAF Dataset

This directory contains the processed **EngSAF** dataset used by the AI Exam
Answer Grader to classify student answers as incorrect, partially correct, or
correct.

## Source

- Dataset: [EngSAF](https://huggingface.co/datasets/IsmaelMousa/engsaf)
- Format: UTF-8 CSV
- Task: compare a student answer with a reference answer and assign a
  three-level correctness label

The CSV files are generated from the source dataset by the preprocessing
script. They are checked into the repository so that model training and
evaluation can be reproduced without downloading the dataset first.

## Files and splits

| File | Split | Rows | Score 0 | Score 1 | Score 2 |
| --- | --- | ---: | ---: | ---: | ---: |
| [`engsaf_train.csv`](engsaf_train.csv) | Training | 4,733 | 1,011 | 1,674 | 2,048 |
| [`engsaf_validation.csv`](engsaf_validation.csv) | Validation | 338 | 75 | 128 | 135 |
| [`engsaf_test.csv`](engsaf_test.csv) | Test | 506 | 113 | 191 | 202 |
| **Total** | — | **5,577** | **1,199** | **1,993** | **2,385** |

The training split is used to fit models, the validation split is used for
model and hyperparameter selection, and the test split is reserved for final
evaluation.

## Schema

Each CSV contains the following columns:

| Column | Description |
| --- | --- |
| `question` | Exam question or prompt associated with the answer |
| `student_answer` | Original student response |
| `reference_answer` | Instructor or dataset reference answer |
| `mark_scheme` | Rubric represented as a serialized mapping from score labels to descriptions |
| `score` | Target label: `0` incorrect, `1` partially correct, or `2` correct |
| `rationale` | Explanation of why the student response received its label |
| `student_answer_clean` | Preprocessed version of `student_answer` |
| `reference_answer_clean` | Preprocessed version of `reference_answer` |



The `mark_scheme` column is stored as text in the CSV, commonly in a
dictionary-like format such as
`{'0': 'Incorrect response', '1': 'Partially correct response', '2': 'Correct response'}`.
It should be parsed deliberately if structured rubric values are needed.

## Preprocessing

The repository preprocessing pipeline:

1. Loads the `IsmaelMousa/engsaf` dataset.
2. Converts `student_answer` and `reference_answer` to lowercase.
3. Removes punctuation while preserving word and whitespace characters.
4. Strips leading and trailing whitespace.
5. Removes rows with missing or empty cleaned answers, or with a missing
   `score`.
6. Writes one CSV for each source split.

The original answer columns are retained. The `*_clean` columns are provided
for feature extraction and reproducible model training. The grader may also
re-clean raw answers at inference time.

## Regenerating the files

From the `backend/src` directory, run:

```bash
uv run python preprocessing.py
```

This requires internet access to download the dataset through the Hugging Face
`datasets` library. The command overwrites the three CSV files in this
directory.

## Example

```python
import pandas as pd

train = pd.read_csv("data/engsaf_train.csv")

student_answer = train.loc[0, "student_answer"]
reference_answer = train.loc[0, "reference_answer"]
label = train.loc[0, "score"]
```

## Responsible use

These labels represent the grading decisions and rationales present in the
source dataset. They should be treated as training annotations rather than
absolute measures of correctness. Review model predictions for ambiguous,
domain-specific, or unusually phrased answers, and avoid using automated
scores as the sole basis for high-impact academic decisions.

