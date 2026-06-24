# ai201-project3-takemeter

A four-class topic classifier for **African / Pan-African podcast episodes**. Given an
episode's description, it predicts one of `history`, `arts`, `economy`, `politics`.

[Open the Colab notebook](https://colab.research.google.com/drive/13fHE3iM5Fd9D604y2k4eGTq8D0UmsACh#scrollTo=fd7f5b6e)

> **Note on which run this report describes.** The committed artifacts
> ([evaluation_results.json](evaluation_results.json) + [confusion_matrix.png](confusion_matrix.png))
> are from the run with **fine-tuned accuracy 0.733**. A later, *uncommitted* run in my
> Downloads reports 0.800. This report documents the **committed 0.733 run**, because that is
> the one whose confusion matrix is reproducible here. If 0.800 is the canonical run, re-commit
> its `evaluation_results.json` and `confusion_matrix.png` and the per-class numbers below
> should be regenerated from that matrix.

---

## Project overview

**Community.** Listeners of independent African and Pan-African podcasts — people across the
continent and its diaspora who follow African history, ideas, power, and economic life
through these shows. Apple Podcasts files most of them under one flat "Society & Culture"
or "News" genre; this classifier gives the catalogue browsable topical shelves.

**Task.** Single-label text classification from the episode description (title + blurb) into
four mutually-exclusive labels. Full label definitions, the decision order that keeps them
exclusive, and the hard edge cases are in [data/taxonomy.md](data/taxonomy.md); the
community, data, and evaluation rationale are in [planning.md](planning.md).

| Label | One-line definition |
|-------|---------------------|
| `history` | The pre-present past: events, eras, kingdoms, colonialism, archaeology, heritage. |
| `arts` | Creative/intellectual production: theatre, film, music, art, literature, storytelling, scholarship. |
| `economy` | Money and the material world: business, finance, trade, plus the built environment (cities, housing, infrastructure). |
| `politics` | How people are governed and live together: current affairs, geopolitics, security, identity, migration, media. |

## Dataset

100 real episodes pulled from the Apple Podcasts (iTunes Search) API, descriptions cleaned
of promo/credits boilerplate. Labels live in [data/my_labels.json](data/my_labels.json);
the model trains on [data/train_episodes.csv](data/train_episodes.csv) (`text,label`).
Class balance is near-uniform: history 26, economy 25, politics 25, arts 24.

**Split:** 70 train / 15 validation / 15 test.

> **Annotation disclosure (see also [AI usage](#ai-usage)):** the labels were **pre-assigned
> by an LLM (Claude)** using the taxonomy's decision order, not hand-annotated from scratch.
> This is relevant to the failure analysis below — some errors may reflect label noise rather
> than only model error.

## Model & training

A fine-tuned `distilbert-base-uncased` classifier. The default training config was broken
(it collapsed to predicting a single class); three changes, each tuned to the tiny
70-example training set, fixed it.

| Hyperparameter | Default | Changed to | Why |
|----------------|--------:|-----------:|-----|
| `learning_rate` | `2e-6` | `2e-5` | `2e-6` was too small to move the weights, so the freshly initialized classification head never adapted. `2e-5` is the standard fine-tuning rate for BERT-family models — large enough to train the new head, small enough not to wash away pre-trained language knowledge. |
| `warmup_steps` | `50` | `2` | With 70 examples and `batch_size=16`, an epoch is only 5 steps, so an 8-epoch run is ~40 steps total. A 50-step warmup never finished ramping before training ended. `2` stabilizes the first gradients, then unlocks the full LR for the remaining ~38 steps. |
| `num_train_epochs` | `3` | `8` | 3 epochs is too few iterations to map 4 classes from so little data. 8 epochs gives room to learn the boundaries; `load_best_model_at_end=True` keeps the best-validation checkpoint. |

Validation loss fell steadily across the run (`0.76 → 0.62`): the model kept generalizing
better each epoch — **no overfitting was actually observed** (overfitting would show val loss
*rising* while train loss falls). `load_best_model_at_end=True` simply guarantees we keep the
lowest-validation-loss checkpoint as insurance.

---

# Evaluation report

## Overall accuracy

| Model | Test accuracy |
|-------|--------------:|
| Zero-shot **Llama 3.3** (baseline) | **0.867** (13/15) |
| Fine-tuned **DistilBERT** (this run) | **0.733** (11/15) |
| _Improvement_ | _−0.133_ |

The fine-tuned model **underperforms** the zero-shot baseline. This is the central result and
is expected: fine-tuning a 66M-parameter model to a brand-new 4-way taxonomy from **70
examples (~17 per class)** is data-starved, whereas a large instruction-tuned model can read
the label definitions and reason zero-shot.

## Per-class metrics — fine-tuned model

Derived directly from the committed confusion matrix below.

| Label | Precision | Recall | F1 | Support |
|-------|----------:|-------:|---:|--------:|
| history | 1.00 | 0.50 | 0.67 | 4 |
| arts | 0.67 | 1.00 | 0.80 | 4 |
| economy | 0.75 | 0.75 | 0.75 | 4 |
| politics | 0.67 | 0.67 | 0.67 | 3 |
| **macro avg** | **0.77** | **0.73** | **0.72** | 15 |
| **accuracy** | | | **0.73** | 15 |

Macro-F1 ≈ **0.72**, below the **0.80** target set in [planning.md](planning.md); `history`
recall (0.50) is also below the per-class floor of 0.70.

## Per-class metrics — baseline (Llama 3.3)

⚠️ **Needs data.** Only the baseline's *overall* accuracy (0.867 = 13/15) was saved. To report
per-class precision/recall/F1 I need the baseline's per-example predictions exported from the
notebook (true label + predicted label for each of the 15 test episodes). Paste those and I
will fill this table; I will not estimate them.

## Confusion matrix — fine-tuned model (test set, n=15)

Rows = true label, columns = predicted label.

| True ↓ \ Pred → | history | arts | economy | politics | Total |
|-----------------|:-------:|:----:|:-------:|:--------:|:-----:|
| **history** | **2** | 0 | 1 | 1 | 4 |
| **arts** | 0 | **4** | 0 | 0 | 4 |
| **economy** | 0 | 1 | **3** | 0 | 4 |
| **politics** | 0 | 1 | 0 | **2** | 3 |
| **Total** | 2 | 6 | 4 | 3 | 15 |

Correct = diagonal = 2+4+3+2 = **11/15**. Four errors, in four distinct cells.

## Failure analysis

### AI-assisted pattern surfacing (and what I corrected)

I gave the confusion matrix to an LLM (Claude) and asked it to surface error patterns, then
**verified each against the raw cell counts**:

- **Pattern 1 — `arts` is an over-prediction "magnet." Verified.** The `arts` *column* sums to
  6 predictions but `arts` has only 4 true members; the two extra are an `economy` episode and
  a `politics` episode pulled into `arts`. Precision 0.67, recall 1.00 — the model never misses
  a real arts episode but wrongly grabs others. **2 of 4 errors are `X → arts`.**
- **Pattern 2 — `history` has the weakest recall (0.50). Verified.** Of 4 true-history episodes,
  one was sent to `economy` and one to `politics`; only 2 stayed. The other two error cells are
  both `history → (economy|politics)`.
- **Pattern I discarded.** The LLM initially proposed "short/low-information posts drive the
  errors." I could **not** verify this — descriptions in the dataset are uniformly multi-sentence
  and I have no per-example length-vs-error data, so I dropped this claim rather than assert it.

**Net directional story:** every single error is either *something → arts* or *history → something*.
The model has not learned (a) the boundary between cultural *topic* and cultural *vocabulary*,
and (b) the "pastness" cue that separates `history` from present-tense `economy`/`politics`.

### Three analyzed failures

The committed artifacts give the four error **transitions** but not the specific test-episode
IDs/text/confidences (the 70/15/15 split is random in the notebook). Below I analyze each real
error using a **representative dataset episode that sits on exactly that boundary** to explain
*why* it is hard. To replace these illustrations with the exact misclassified rows + confidence
scores, see [the data request](#sample-classifications) at the end.

**1. `economy → arts` (1 error).** Representative: *"Global Art Law and Trends in the African
Art Market"* (`economy` — it's about an art *market*).
- **Which boundary:** economy↔arts.
- **Why hard:** the text is saturated with arts vocabulary ("art", "market", "gallery"), but the
  topic is commerce. DistilBERT keys on surface tokens, so "art" overwhelms the financial framing.
- **Labeling or data problem:** a *data/boundary* problem — the label is consistent with the
  taxonomy (money/markets → `economy`), but with ~17 `economy` examples the model never saw
  enough "art-as-business" cases to learn that art words ≠ `arts`.
- **Fix:** add `economy` examples whose vocabulary leans cultural (art markets, music industry
  revenue, fashion *business*) so the boundary is shown explicitly.

**2. `history → politics` (1 error).** Representative: *"How Nigeria's Mega Refinery Dictates
European Energy"* (labeled `history`, but it reads like current geopolitics).
- **Which boundary:** history↔politics.
- **Why hard:** the framing is present-tense and geopolitical; the only thing making it `history`
  is the show's retrospective lens. The model has no strong "pastness" signal to grab.
- **Labeling or data problem:** partly a *labeling* problem — this episode is genuinely borderline
  and the LLM-assigned label may itself be debatable (see annotation disclosure). Present-tense
  geopolitical phrasing legitimately points at `politics`.
- **Fix:** tighten the label (or relabel borderline `history` episodes), and add historical
  episodes with explicit temporal markers (dates, "in the 19th century") so the cue is learnable.

**3. `politics → arts` (1 error).** Representative: a culture/identity episode (e.g. African
fashion or storytelling framed around social tensions).
- **Which boundary:** politics↔arts.
- **Why hard:** identity/society episodes often discuss culture, music, or fashion as their
  *vehicle*, so the lexical surface looks like `arts` even though the subject is social/political.
- **Labeling or data problem:** a *data* problem — the same arts-magnet effect as #1; the decision
  order says "creative *work/maker* → arts, social *dynamics* → politics," but the model learned
  the keyword shortcut instead of the abstract distinction.
- **Fix:** more `politics` examples that use cultural vocabulary, and/or a longer/more explicit
  prompt for the inference path that restates the work-vs-vehicle rule.

**The history→economy error** rounds out the four; same root cause as #2 (no pastness cue, with
resource/economy vocabulary tipping it toward `economy`).

## Sample classifications

⚠️ **Needs data (predicted label + confidence per example).** The committed artifacts contain
aggregate metrics and the confusion matrix, but **not** per-episode predictions or softmax
confidences, so I cannot honestly print confidence scores here. **To complete this section,
export from the notebook a small table for ~5 test episodes** — `text`, `true_label`,
`predicted_label`, `confidence` (max softmax) — and paste it. I will then format it as below and
narrate one correct and one incorrect case:

```
| Episode (title) | True | Predicted | Confidence | Correct? |
|-----------------|------|-----------|-----------:|:--------:|
| …               | arts | arts      | 0.xx       | ✓        |
| …               | history | economy | 0.xx      | ✗        |
```

For the correct case I will explain *why the prediction is reasonable* (e.g. "an episode about a
playwright's theatre work is confidently `arts` because both the topic and the vocabulary point
the same way — there is no competing signal"). This is the one piece I will not fabricate.

---

## Reflection: intended vs. learned decision boundary

**What I intended.** The taxonomy defines labels by *what the episode is fundamentally about*,
with a priority order (`arts` > `history` > `economy` > `politics`) meant to resolve episodes
that mix themes — e.g. a historical *artist* is `arts`, a present-day *city* is `economy`.

**What the model actually learned.** With only 70 examples, DistilBERT learned a **vocabulary
classifier, not a topic classifier.** The confusion matrix shows the boundary it captured is
lexical: any episode with cultural words (art, music, fashion, story) drifts toward `arts`
(recall 1.00, precision 0.67), and `history` collapses whenever the text lacks explicit
past-tense markers (recall 0.50). It **overfit to surface tokens** — the exact cultural keywords
that are dense in the tiny arts training set — and **missed the abstract cues** the taxonomy
actually encodes: the difference between a creative *work* and culture used as a *vehicle*, and
the "pastness" that distinguishes `history` from current affairs. The priority order, which a
reasoning model (the Llama baseline) can apply directly from the definitions, is precisely what
the fine-tuned model failed to internalize from so few examples. The gap is intent ("classify by
topic and structure") vs. reality ("classify by keyword frequency").

## Spec reflection

- **How the spec helped.** The decision order in [planning.md](planning.md)/[taxonomy.md](data/taxonomy.md)
  made annotation *consistent*: every "historical artist" (Ben Enwonwu, Sipho Mabuse) went to
  `arts`, every present-day city episode to `economy`. Without that written tie-break rule the
  100-episode label set would have drifted, and the confusion matrix would have mixed model error
  with annotation noise, making this analysis impossible to interpret.
- **Where I diverged and why.** The spec planned **~200–240 examples (50–60/label)**; the actual
  dataset is **100 (70 train)**. I diverged to keep collection to clean, verifiable,
  continent-focused episodes rather than pad the set with marginal or non-African results. The
  cost is visible in the results — the data-starved fine-tune (0.73, macro-F1 0.72) missed the
  spec's 0.80 success bar — which directly validates the spec's original 200-example target.

## AI usage

This project used Claude (Anthropic) substantially. Specific instances:

1. **Taxonomy design.** I asked Claude to propose a mutually-exclusive taxonomy from the seed
   episodes. It produced an 8-label scheme; I **overrode** it to require exactly **4 labels**, and
   Claude re-derived the four-label version plus the priority decision order. The final taxonomy is
   in [data/taxonomy.md](data/taxonomy.md).
2. **Data collection + annotation (disclosed).** I directed Claude to pull ~75 real episodes from
   the Apple Podcasts API, clean the descriptions, and **pre-label** them by topic using the
   decision order. The labels in [data/my_labels.json](data/my_labels.json) are therefore
   **LLM-assigned**, not independently hand-annotated. I reviewed the candidate list and had Claude
   correct mislabeled items (e.g. Ancient-Egypt "sleep history" episodes wrongly placed in `arts`),
   but the labels remain AI-generated — a limitation noted in the failure analysis.
3. **Classification prompt.** Claude wrote the zero-shot `SYSTEM_PROMPT` (definitions + one example
   per label + the decision order + strict label-only output). I kept the output-format constraints
   as written.
4. **Hyperparameter writeup correction.** I gave Claude a draft analysis claiming the falling
   validation loss showed overfitting; Claude **corrected** that (falling val loss = still
   generalizing) and produced the training table above. I accepted the correction.
5. **This evaluation report.** Claude read the committed `confusion_matrix.png`, transcribed it to
   the markdown table, derived the per-class metrics, and surfaced/verified the error patterns. I
   **discarded** one unverifiable pattern it proposed ("short posts cause errors") and required it
   to mark — not invent — the baseline per-class and confidence numbers it lacked.

## Submission checklist

- [x] Evaluation report with overall accuracy for both models
- [x] Per-class metrics — fine-tuned (baseline pending per-example export)
- [x] Confusion matrix written out as a markdown table
- [x] ≥3 analyzed failures (boundary-level, grounded in the real matrix)
- [x] Reflection on intended vs. learned boundary
- [x] Spec reflection (one help, one divergence)
- [x] AI usage section with annotation disclosure
- [ ] Sample Classifications with confidence scores — **needs per-example export**
- [ ] Baseline per-class metrics — **needs baseline predictions export**
- [ ] 3–5 min demo video (classify 3–5 posts w/ confidence, narrate one correct + one wrong, walk through this report)
