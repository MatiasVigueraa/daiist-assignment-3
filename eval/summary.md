# Evaluation summary

2026-11-24 · agent `gemini-3.5-flash-lite` · judge `gemini-3.5-flash-lite` · search design `article`

## Retrieval

Answerable questions: 27. A hit means an expected article is among the top k results.

| Design | hit@1 | hit@3 | hit@5 | multi: both passages in top 5 | words added (top 5) |
|---|---|---|---|---|---|
| article | 0.74 | 0.96 | 1.00 | 0.833 | 835 |
| fixed | 0.63 | 0.78 | 0.85 | 0.833 | 745 |

Highest hit@5 (ties broken by hit@3, hit@1, fewer words): **article**.

## End to end

Questions answered: 32 of 32.

| Result | Count |
|---|---|
| Correct (all questions) | 29/32 (91%) |
| Supported (all questions) | 31/32 (97%) |
| Correct answers to answerable questions | 24/27 (89%) |
| Unsupported answers | 1 |
| Correct refusals (unanswerable) | 5/5 (100%) |
| Wrong refusals (answerable) | 0 |
| Blocked by the citation check | 0 |
| Citation check fired (incl. fixed on retry) | 0 |

| Kind | n | Correct | Supported |
|---|---|---|---|
| single | 21 | 18 | 20 |
| multi | 6 | 6 | 6 |
| unanswerable | 5 | 5 | 5 |

## Cost per question

| | Mean | Median | Max |
|---|---|---|---|
| Agent tokens | 5,364 | 4,420 | 10,492 |
| Agent model calls | 2.3 | 2 | 3 |
| Seconds | 16.5 | 4.7 | 161.7 |
| Judge tokens (judged answers) | 2,750 | 2,416 | 4,506 |

## Agreement with your grades

No grades yet in eval/human_labels.csv.

## Failures by suggested cause (check each one yourself)

- retrieval miss: 1 (q21)
- wrong answer from the right passage: 2 (q02, q14)

## Citation check firings

None.
