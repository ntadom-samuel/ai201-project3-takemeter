# Planning — African Podcast Episode Classifier

A text classifier that labels African / Pan-African podcast episodes by topic, using the
episode title, podcast name, and description as input.

---

## 1. Community

**Who:** Listeners of independent **African and Pan-African podcasts** — people across the
continent and its diaspora who use these shows to follow the history, ideas, power, and
economic life shaping Africa today. The corpus is drawn from shows like *African
Renaissance Podcast*, *Focus on Africa*, *African History*, *The African Business Podcast*,
*African Cities*, and *Pan-African Frequency*.

**Why this community:** It is a coherent, real audience whose content is *not* served well
by generic podcast directories — Apple Podcasts files most of these under one flat
"Society & Culture" or "News" genre, which collapses an enormous range of conversation into
one undifferentiated feed. A topic classifier is genuinely useful here: it gives the
catalogue browsable shelves.

**Why it's a good fit for classification:** The discourse is varied enough to be
interesting but bounded enough to be tractable. A single week of episodes can span a
medieval Swahili city, a bank CEO's growth roadmap, a prime-minister's resignation, and a
musician's life story — clearly distinct topics — yet they share a regional frame, so the
classifier can't lean on cheap geographic keywords ("Africa", country names) and must
actually learn topical signal. The varied register (academic interviews, breaking news,
business analysis, cultural conversation) is exactly the kind of real-world messiness that
makes the task non-trivial.

---

## 2. Labels

Four mutually-exclusive labels. Each episode gets **exactly one**; the decision order in
§3 resolves episodes that touch more than one theme.

### `history` — History & Heritage
> *The episode's subject is the pre-present past: historical events, eras, kingdoms,
> colonialism, archaeology, or cultural heritage examined to understand what came before.*

- **t009 — "The Map that wasn't: How the 1884 Berlin Conference drew Africa's borders"**
  (*The Scramble for Africa*) — the mechanics of colonial border-drawing.
- **t011 — "The Birth of Mobutu and the Collapse of Congo"** (*African History*) — the rise
  and fall of a historical dictator across the Cold War era.

### `arts` — Arts, Culture & Ideas
> *The episode is about creative or intellectual production: theatre, film, music, visual
> art, literature, storytelling, or scholarship and ideas about culture.*

- **t001 — "John Kani: Arts, Hollywood, Pan-Africanism, and the future of Storytelling"**
  (*African Renaissance Podcast*) — a conversation with an actor/playwright about theatre.
- **t013 — "Ben Enwonwu: Titan of Modern African Art"** (*African History*) — the life and
  cultural impact of a pioneering Nigerian visual artist.

### `economy` — Economy & Development
> *The episode is about money or the material world: business, finance, banking,
> entrepreneurship, and trade, or the built environment — urban planning, housing,
> infrastructure, and climate adaptation.*

- **t015 — "Ecobank CEO Jeremy Awori on Africa's Banking Future"** (*The African Business
  Podcast*) — a bank's growth strategy and the role of development finance.
- **t018 — "Urban planning with Shuaib Lwasa"** (*African Cities*) — rethinking how African
  cities are planned and built.

### `politics` — Politics, Power & Society
> *The episode is about how people are governed and how they live together: current
> affairs, elections, diplomacy, geopolitics, security, social dynamics, identity,
> migration, leadership, or the information/media space.*

- **t003 — "What UK prime minister resignation means for Africans"** (*Focus on Africa*) —
  current-affairs analysis of a political event and its diaspora impact.
- **t006 — "Who is really speaking? ... protest and foreign manipulation"** (*Pan-African
  Frequency*) — digital sovereignty and the information space.

---

## 3. Hard edge cases

The label boundaries are clean for most episodes, but four recurring ambiguities exist.
The fix is a **priority decision order (top-down, first match wins)** applied during
annotation:

> **1. Arts** (creative/intellectual work or maker) → **2. History** (subject set in the
> past) → **3. Economy** (money / built environment) → **4. Politics** (everything else:
> governance, society, media)

**Edge case A — Historical artist (`arts` vs `history`).** An episode about a historical
*figure* who happens to be an artist (e.g. t013 Ben Enwonwu, t020 Sipho Mabuse). *Rule:*
Arts wins — if the episode is about the *work and creative legacy*, it's `arts`, even when
the person is long dead. History is reserved for events/eras/movements.

**Edge case B — Heritage site vs built environment (`history` vs `economy`).** An episode
naming a city or building (t008 "The Stone City of Songo Mnara"). *Rule:* if it's
archaeology/the past, it's `history`; if it's present-day planning/housing/infrastructure
(t002 densification, t023 Lagos housing), it's `economy`. The deciding question is
**tense**: is the city being excavated or being built?

**Edge case C — Disinformation / media (`politics` vs `economy`/tech).** Episodes about the
information space (t006, t019). *Rule:* the information ecosystem and propaganda are a form
of power → `politics`, not `economy`, even when "digital sovereignty" sounds techy.

**Edge case D — Social policy with an economic hook (`economy` vs `politics`).** e.g.
xenophobia driven by economic stagnation (t014 Afrophobia), or carbon markets framed as
jobs. *Rule:* if the subject is the *market/enterprise/infrastructure*, it's `economy`
(t024 carbon markets); if the subject is *how people are treated or governed*, it's
`politics` (t014).

When a genuinely 50/50 case appears during annotation, I will (a) apply the decision order,
(b) log the episode id in an `edge_cases.md` notes file with the two candidate labels and
the reason for the call, so the boundary stays consistent across the whole dataset.

---

## 4. Data collection plan

**Source:** Apple Podcasts, via the public **iTunes Search API**
(`https://itunes.apple.com/search?entity=podcastEpisode`), which returns real episode
title, podcast name, and description — the exact fields the classifier uses. The current
25 episodes (`data/train_episodes.json`, `data/test_episodes.json`) were collected this
way.

**Target volume:** ~**200–240 labeled episodes**, aiming for **~50–60 per label** so even
the smallest class has enough examples to train and evaluate on. Collection is by querying
topic-aligned terms ("African history", "African business podcast", "Pan-African",
"African cities urban planning", "African disinformation media", etc.) and de-duplicating
by (podcast, title).

**Split:** roughly **80/20 train/test**, stratified so each label appears in the test set.

**If a label is underrepresented after 200 examples:** (in this corpus `arts` is the likely
laggard)
1. **Targeted top-up first** — run additional searches biased toward the thin label
   (African film/music/literature/theatre podcasts) to collect ~20 more real examples
   before changing anything structural.
2. **If real examples are genuinely scarce**, document the floor and either (a) accept the
   imbalance and rely on macro-averaged metrics (see §5) so the small class still counts
   equally, or (b) merge the thin label into its nearest neighbour (`arts` → fold scholarship
   into a broader culture class) and re-state the taxonomy.
3. **Never synthesize fake episodes** into the gold set — synthetic data is used only for
   label stress-testing (§7), never as labeled training/eval data.

---

## 5. Evaluation metrics

**Accuracy is not enough** because the classes are imbalanced (in the seed set, `politics`
has 8 episodes and `arts` only 4). A model that ignores `arts` entirely could still post a
respectable accuracy while being useless for a third of the catalogue. I will therefore
report:

| Metric | Why it's needed |
|--------|-----------------|
| **Macro-averaged F1** *(primary)* | Averages F1 across the four classes with equal weight, so the smallest label (`arts`) counts as much as the largest. This is the headline number. |
| **Per-class precision & recall** | Tells me *where* the model fails — e.g. low `arts` recall (missed) vs low `arts` precision (over-predicted). The fix differs in each case. |
| **Confusion matrix** | Reveals *which pairs* get confused (I expect `arts`↔`history` and `economy`↔`politics`, matching the §3 edge cases). Already part of the repo (`confusion_matrix.png`). |
| **Accuracy** *(secondary)* | Reported for completeness and comparison to the majority-class baseline. |
| **Majority-class baseline** | Always predicting `politics` ≈ 32% accuracy on the seed set — the floor any real model must clear. |

For annotation quality I will also track **inter-annotator / human-vs-LLM agreement
(Cohen's κ)** on any pre-labeled batch (see §7), since the metrics above are only
trustworthy if the gold labels themselves are consistent.

---

## 6. Definition of success

Success is defined against the **macro-F1** primary metric with explicit, checkable
thresholds:

- **Good enough to ship in a community browsing tool:** **macro-F1 ≥ 0.80**, **with no
  single class below 0.70 F1**, and overall accuracy comfortably beating the majority
  baseline (≥ 0.80 vs. ~0.32). The per-class floor matters: an 0.80 macro-F1 that hides a
  0.45 `arts` class is *not* acceptable, because the tool would mis-shelve cultural episodes.
- **Genuinely useful / strong:** **macro-F1 ≥ 0.88** and every class ≥ 0.80, i.e. the
  classifier roughly matches a careful human annotator on this taxonomy.
- **Not acceptable:** macro-F1 < 0.70, or any class < 0.60, or failure to beat the baseline
  on accuracy.

**Are these criteria objectively checkable?** Yes — each is a single number computed from
the held-out test set's confusion matrix (`evaluation_results.json`), with a fixed
threshold and a fixed baseline. At the end I can state unambiguously, per class and overall,
whether the bar was cleared. The only subjective input is the gold labels themselves, which
is why §5 tracks annotation agreement.

---

## 7. AI Tool Plan

This is a *labeling* project, not an implementation project — there is no application code
to generate. AI tools help in three specific places:

### 7.1 Label stress-testing (before annotation)
I will give an LLM (Claude) the four label definitions and the §3 edge-case descriptions and
ask it to generate **5–10 synthetic episode blurbs that deliberately sit on the boundary**
between two labels (e.g. a historical-musician episode straddling `arts`/`history`; a
carbon-market-jobs episode straddling `economy`/`politics`). I then try to classify each
using only the decision order.
- **Pass:** every synthetic blurb resolves to exactly one label via the rules → definitions
  are tight enough to start annotating.
- **Fail:** if I can't cleanly classify one, the definition or the decision order is leaky →
  I tighten it **now**, before labeling 200 real episodes. These synthetic blurbs are a
  *test of the rules only* and never enter the gold dataset.

### 7.2 Annotation assistance (during labeling)
I **will** use an LLM to **pre-label** episodes in batches, then review every pre-label
myself (human-in-the-loop) rather than trusting it.
- **Tool:** Claude, prompted with the §2 definitions + §3 decision order, one batch at a
  time, returning `{id, label, confidence, one-line rationale}`.
- **Tracking for disclosure:** every record carries a `pre_labeled_by: "claude"` and
  `reviewed_by_human: true/false` flag (and the original model label is kept even when I
  override it), so the AI-usage section can report exactly how many labels were
  model-suggested, how many I changed, and the human–LLM **Cohen's κ**. Low-confidence or
  edge-case items get manual priority.

### 7.3 Failure analysis (after evaluation)
After scoring, I will hand the LLM the **list of misclassified test episodes** (true label,
predicted label, text) and ask it to **cluster the errors into patterns** — e.g. "`arts`
episodes about historical figures get pulled into `history`," or "short descriptions are
under-informative."
- **What I look for:** systematic confusions that map back to the §3 edge cases, length/text
  effects, and any single podcast that's consistently misfiled.
- **Verification:** I treat the AI's patterns as *hypotheses only*. I confirm each by going
  back to the confusion matrix and re-reading the actual episodes it cites — a pattern is
  only reported in the write-up if I can point to the specific misclassified rows that
  support it.
