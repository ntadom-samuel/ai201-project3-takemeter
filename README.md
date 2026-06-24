# ai201-project3-takemeter

A four-class topic classifier for **African / Pan-African podcast episodes**. Given an
episode's description, it predicts one of `history`, `arts`, `economy`, `politics`.

[Open the Colab notebook](https://colab.research.google.com/drive/13fHE3iM5Fd9D604y2k4eGTq8D0UmsACh#scrollTo=fd7f5b6e)

> **Note on artifacts.** All metrics below are the **canonical 0.733 run** (full
> `classification_report` output for both models). ⚠️ The committed
> [confusion_matrix.png](confusion_matrix.png) is **stale** — it is from a _different_ 0.733
> run and is inconsistent with these per-class numbers (its cells imply `history` precision
> 1.00 / `arts` 0.67, but this run is 0.67 / 0.80). **Regenerate and re-commit
> `confusion_matrix.png` from the run that produced the report below**, and paste the raw
> confusion-matrix counts so the four off-diagonal cells in the table below can be filled in
> exactly (per-class metrics alone do not determine them).

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

| Label      | One-line definition                                                                                                   |
| ---------- | --------------------------------------------------------------------------------------------------------------------- |
| `history`  | The pre-present past: events, eras, kingdoms, colonialism, archaeology, heritage.                                     |
| `arts`     | Creative/intellectual production: theatre, film, music, art, literature, storytelling, scholarship.                   |
| `economy`  | Money and the material world: business, finance, trade, plus the built environment (cities, housing, infrastructure). |
| `politics` | How people are governed and live together: current affairs, geopolitics, security, identity, migration, media.        |

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

| Hyperparameter     | Default | Changed to | Why                                                                                                                                                                                                                                                                        |
| ------------------ | ------: | ---------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `learning_rate`    |  `2e-6` |     `2e-5` | `2e-6` was too small to move the weights, so the freshly initialized classification head never adapted. `2e-5` is the standard fine-tuning rate for BERT-family models — large enough to train the new head, small enough not to wash away pre-trained language knowledge. |
| `warmup_steps`     |    `50` |        `2` | With 70 examples and `batch_size=16`, an epoch is only 5 steps, so an 8-epoch run is ~40 steps total. A 50-step warmup never finished ramping before training ended. `2` stabilizes the first gradients, then unlocks the full LR for the remaining ~38 steps.             |
| `num_train_epochs` |     `3` |        `8` | 3 epochs is too few iterations to map 4 classes from so little data. 8 epochs gives room to learn the boundaries; `load_best_model_at_end=True` keeps the best-validation checkpoint.                                                                                      |

Validation loss fell steadily across the run (`0.76 → 0.62`): the model kept generalizing
better each epoch — **no overfitting was actually observed** (overfitting would show val loss
_rising_ while train loss falls). `load_best_model_at_end=True` simply guarantees we keep the
lowest-validation-loss checkpoint as insurance.

---

# Evaluation report

## Overall accuracy

| Model                                |     Test accuracy |
| ------------------------------------ | ----------------: |
| Zero-shot **Llama 3.3** (baseline)   | **0.867** (13/15) |
| Fine-tuned **DistilBERT** (this run) | **0.733** (11/15) |
| _Improvement_                        |          _−0.133_ |

The fine-tuned model **underperforms** the zero-shot baseline. This is the central result and
is expected: fine-tuning a 66M-parameter model to a brand-new 4-way taxonomy from **70
examples (~17 per class)** is data-starved, whereas a large instruction-tuned model can read
the label definitions and reason zero-shot.

## Per-class metrics — fine-tuned DistilBERT

From the `classification_report` on the 15-example test set.

| Label            | Precision |   Recall |       F1 | Support |
| ---------------- | --------: | -------: | -------: | ------: |
| history          |      0.67 |     0.50 |     0.57 |       4 |
| arts             |      0.80 |     1.00 |     0.89 |       4 |
| economy          |      0.75 |     0.75 |     0.75 |       4 |
| politics         |      0.67 |     0.67 |     0.67 |       3 |
| **macro avg**    |  **0.72** | **0.73** | **0.72** |      15 |
| **weighted avg** |  **0.72** | **0.73** | **0.72** |      15 |
| **accuracy**     |           |          | **0.73** |      15 |

Macro-F1 ≈ **0.72**, below the **0.80** target in [planning.md](planning.md). `arts` is the
strongest class (F1 0.89) and `history` the weakest (F1 0.57, recall 0.50 — below the
per-class floor of 0.70).

## Per-class metrics — baseline (zero-shot Llama 3.3)

From the `classification_report` on the same 15 examples (15/15 responses parseable).

| Label            | Precision |   Recall |       F1 | Support |
| ---------------- | --------: | -------: | -------: | ------: |
| history          |      0.75 |     0.75 |     0.75 |       4 |
| arts             |      1.00 |     1.00 |     1.00 |       4 |
| economy          |      0.80 |     1.00 |     0.89 |       4 |
| politics         |      1.00 |     0.67 |     0.80 |       3 |
| **macro avg**    |  **0.89** | **0.85** | **0.86** |      15 |
| **weighted avg** |  **0.88** | **0.87** | **0.86** |      15 |
| **accuracy**     |           |          | **0.87** |      15 |

The baseline beats the fine-tune on **every** class. It is perfect on `arts` (1.00 F1) and has
perfect recall on `economy`. Its one soft spot mirrors the fine-tune's: `politics` recall 0.67
(it misses 1 of 3 politics episodes) — evidence that `politics` is the genuinely hardest
boundary in this taxonomy, not just an artifact of small training data.

## Confusion matrix — fine-tuned model (test set, n=15)

Rows = true label, columns = predicted label. Reconstructed exactly from the 4 misclassified
examples (below) and cross-checked against the per-class precisions above.

| True ↓ \ Pred →  | history | arts  | economy | politics | Total (support) |
| ---------------- | :-----: | :---: | :-----: | :------: | :-------------: |
| **history**      |  **2**  |   0   |    1    |    1     |        4        |
| **arts**         |    0    | **4** |    0    |    0     |        4        |
| **economy**      |    0    |   1   |  **3**  |    0     |        4        |
| **politics**     |    1    |   0   |    0    |  **2**   |        3        |
| **Total (pred)** |    3    |   5   |    4    |    3     |       15        |

Correct = diagonal = 2+4+3+2 = **11/15**. The four errors: `history→politics`,
`history→economy`, `economy→arts`, `politics→history`. Note `arts` has **zero** false
negatives (recall 1.00) but one false positive (an `economy` episode), and `history` loses
**half** its episodes (recall 0.50).

(The committed `confusion_matrix.png` differs in one cell — it shows `politics→arts` instead of
the actual `politics→history` — confirming it is from a different run and should be regenerated.)

## Failure analysis

### AI-assisted pattern surfacing (and what I corrected)

I pasted the 4 misclassified examples (true label, predicted label, confidence, text) into an
LLM (Claude) and asked it to surface common themes, then **verified each against the examples**:

- **Pattern 1 — most errors are on episodes whose _gold label is itself debatable_. Verified.**
  3 of 4 errors are episodes from the _African History_ podcast (labeled `history` by source)
  whose text is actually present-tense economics/politics — and the model is **highly confident**
  it's right (0.99, 1.00). The model isn't malfunctioning on these; it's arguably _more correct
  than the gold label_ (see #1, #3 below). This is **annotation noise from labeling-by-podcast**,
  not pure model error.
- **Pattern 2 — `history` has the weakest recall (0.50) and `arts` is an over-prediction magnet.
  Verified.** History loses half its episodes; `arts` takes one extra (`economy→arts`) while never
  missing a true `arts` episode (recall 1.00). Both follow from the model keying on surface
  vocabulary rather than the abstract "pastness" / work-vs-vehicle distinctions.
- **Pattern I discarded.** The LLM first proposed "low confidence drives the errors." **False** —
  3 of 4 errors are at ≥0.98 confidence. Only the genuinely ambiguous case (#2) was low-confidence
  (0.54). I dropped the claim; the data shows confident-but-wrong is the dominant failure, which is
  worse and points at labels, not model uncertainty.

### Three analyzed failures (real misclassifications, with confidence)

**1. `history → politics`, confidence 0.99** — _"Azikiwe and the Global Defense of Liberian
Sovereignty"_ (_African History_). Text: "the geopolitical and economic struggles of Liberia…
tension between sovereignty and foreign exploitation."

- **Which boundary & why hard:** history↔politics. The text is dominated by _politics_ vocabulary
  (sovereignty, exploitation, geopolitical). Nothing signals "past" except the show's framing.
- **Labeling or data problem:** **labeling.** The episode was labeled `history` because of its
  _podcast_, but the description reads as political analysis — the model's `politics` call is
  defensible. The gold label is the weak link.
- **Fix:** relabel by _content_ not source, or add a "historical-politics" cue (explicit dates).

**2. `politics → history`, confidence 0.54** — _"Simukai Chigudu, associate professor of African
Politics… the legacy of empire and how to reckon with the past"_ (_The Interview_).

- **Which boundary & why hard:** politics↔history. The episode is _about_ history (empire, the
  past) but framed as present-day reckoning, so it legitimately straddles both labels.
- **Labeling or data problem:** **a genuine boundary case**, and the model knew it — this is the
  _only_ low-confidence error (0.54, almost a coin flip). The decision order says present-day
  reckoning → `politics`, but "legacy of empire / the past" pulls `history`.
- **Fix:** add an explicit edge-case rule ("a _current_ conversation about the past → `politics`")
  and training examples that show it.

**3. `history → economy`, confidence 1.00** — _"The Dangote Petroleum Refinery in Nigeria…
transitioning the region from a fuel importer to a strategic international supplier"_ (_African
History_).

- **Which boundary & why hard:** history↔economy. This is a **business/energy** story (refinery,
  importer, supplier); there is essentially nothing historical in the text.
- **Labeling or data problem:** **labeling**, unambiguously — at 1.00 confidence the model says
  `economy`, and by our own taxonomy (trade/industry → `economy`) it is _correct_; the `history`
  gold label is wrong. This single mislabel costs `history` recall and inflates the error count.
- **Fix:** relabel to `economy`.

**Fourth error — `economy → arts`, confidence 0.98** — _"…shifting from consuming to creating…
if you're not ready to create…"_ (_Business of African Fashion_, labeled `economy`).

- **Why hard:** the text is generic creator-mindset language ("creating", from a _fashion_ show);
  the financial/entrepreneurial framing is implicit. The arts vocabulary wins.
- **Labeling or data problem:** **data + boundary** — a real `economy`↔`arts` confusion (the
  arts-magnet effect). The label is reasonable (it's the _business_ of fashion) but the text gives
  the model little economic signal.
- **Fix:** add `economy` examples that use creative vocabulary (the _business_ of music/fashion/film).

**Bottom line:** at least 2 of 4 "errors" (#1, #3) are really **gold-label errors** caused by
labeling episodes by their podcast rather than their content. The true model error rate on
_clean_ labels is closer to 2/15. The single biggest fix is **relabeling the _African History_
podcast episodes by content**, not collecting more data.

## Sample classifications

Real predictions from the fine-tuned model on the test set (the 4 misclassified episodes; max
softmax shown as confidence).

| Episode (excerpt)                                                                            | True       | Predicted  | Confidence | Correct? |
| -------------------------------------------------------------------------------------------- | ---------- | ---------- | ---------: | :------: |
| "…geopolitical and economic struggles of Liberia… sovereignty and foreign exploitation"      | `history`  | `politics` |       0.99 |    ✗     |
| "Simukai Chigudu… legacy of empire and how to reckon with the past"                          | `politics` | `history`  |       0.54 |    ✗     |
| "The Dangote Petroleum Refinery… from a fuel importer to a strategic international supplier" | `history`  | `economy`  |       1.00 |    ✗     |
| "…shifting from consuming to creating… if you're not ready to create"                        | `economy`  | `arts`     |       0.98 |    ✗     |

**Reading a _reasonable_ prediction (even though it's scored "wrong").** The Dangote Refinery
episode → `economy` at **1.00** is the model behaving _correctly_: the text is purely about trade
and industry (refinery, importer, supplier), which our taxonomy maps to `economy`. The high
confidence is appropriate because topic and vocabulary agree with no competing signal — the only
reason it counts as an error is that the gold label (`history`, assigned by podcast) is wrong.

**Reading an _unreasonable_ prediction.** The "consuming to creating" episode → `arts` at **0.98**
is over-confident on a thin signal: the model latched onto "create/creating" and ignored that the
episode is the _business_ of fashion. This is the arts-vocabulary magnet, and the high confidence
shows the model has no calibrated sense of the `economy`↔`arts` boundary.

> **One gap remains:** all 4 examples above are the model's _errors_. To show a confidently
> **correct** prediction with its score (e.g. a clear `arts` episode about a musician), paste 1–2
> correctly-classified test rows with confidence and I'll add them here.

---

## Reflection: intended vs. learned decision boundary

**What I intended.** The taxonomy defines labels by _what the episode is fundamentally about_,
with a priority order (`arts` > `history` > `economy` > `politics`) meant to resolve episodes
that mix themes — e.g. a historical _artist_ is `arts`, a present-day _city_ is `economy`.

**What the model actually learned.** With only 70 examples, DistilBERT learned a **vocabulary
classifier, not a topic classifier.** Episodes with cultural words (art, music, fashion, create)
drift toward `arts` (recall 1.00, precision 0.80), and `history` collapses whenever the text
lacks explicit past-tense markers (recall 0.50). It **overfit to surface tokens** — the cultural
keywords dense in the small `arts` set — and **missed the abstract cues** the taxonomy encodes:
the difference between a creative _work_ and culture used as a _vehicle_, and the "pastness" that
separates `history` from current affairs. The priority order, which the Llama baseline applies
directly from the definitions (and scores 0.87), is exactly what the fine-tune failed to
internalize from so few examples.

**But the gap is also a label gap, not only a model gap.** The failure analysis shows the model
captured something the _labels_ did not: it sorts by _what the text is about_, while the gold
labels sort the _African History_ podcast by _source_. On the two refinery/Liberia episodes the
model's "wrong" answer matches the taxonomy better than the human/LLM label does. So the decision
boundary the model learned is partly _more_ faithful to the intended definitions than the training
labels were — the intended-vs-captured gap cuts both ways.

## Spec reflection

- **How the spec helped.** The decision order in [planning.md](planning.md)/[taxonomy.md](data/taxonomy.md)
  made annotation _consistent_: every "historical artist" (Ben Enwonwu, Sipho Mabuse) went to
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
5. **Failure-analysis pattern surfacing + this report.** I pasted the 4 misclassified examples
   into Claude to surface common themes. It reconstructed the exact confusion matrix from those
   examples, **cross-checked it against the `classification_report`**, and caught that the committed
   `confusion_matrix.png` is from a different run (one cell differs). I **discarded** its first
   proposed pattern ("low confidence drives the errors" — false, 3 of 4 errors were ≥0.98) and kept
   the verified finding that most "errors" are debatable gold labels. Earlier it refused to invent
   the baseline per-class / confidence numbers it lacked, flagging them as needed until I provided
   them — which I then did.

## Submission checklist

- [x] Evaluation report with overall accuracy for both models
- [x] Per-class metrics — both models (full `classification_report`)
- [x] Confusion matrix written out as a markdown table (exact, reconstructed from the 4 errors)
- [x] ≥3 analyzed failures with real text + confidence (labeling-vs-data diagnosis)
- [x] Sample Classifications with confidence scores (4 real predictions)
- [x] Reflection on intended vs. learned boundary
- [x] Spec reflection (one help, one divergence)
- [x] AI usage section with annotation disclosure
- [ ] Add ≥1 confidently-**correct** example w/ confidence to Sample Classifications (paste 1–2 rows)
- [ ] Regenerate + re-commit `confusion_matrix.png` from the canonical run (current PNG is stale)
