# Taxonomy — African Podcast Episode Classifier

Four mutually-exclusive labels. Each episode gets **exactly one**; the decision order
below resolves episodes that touch more than one theme. Per-episode assignments live in
[my_labels.json](my_labels.json).

## Labels

### `history` — History & Heritage
> The episode's subject is the pre-present past: historical events, eras, kingdoms,
> colonialism, archaeology, or cultural heritage examined to understand what came before.

- **t009 — "The Map that wasn't: How the 1884 Berlin Conference drew Africa's borders"**
  (*The Scramble for Africa*) — the mechanics of colonial border-drawing.
- **t011 — "The Birth of Mobutu and the Collapse of Congo"** (*African History*) — the rise
  and fall of a historical dictator across the Cold War era.

### `arts` — Arts, Culture & Ideas
> The episode is about creative or intellectual production: theatre, film, music, visual
> art, literature, storytelling, or scholarship and ideas about culture.

- **t001 — "John Kani: Arts, Hollywood, Pan-Africanism, and the future of Storytelling"**
  (*African Renaissance Podcast*) — a conversation with an actor/playwright about theatre.
- **t013 — "Ben Enwonwu: Titan of Modern African Art"** (*African History*) — the life and
  cultural impact of a pioneering Nigerian visual artist.

### `economy` — Economy & Development
> The episode is about money or the material world: business, finance, banking,
> entrepreneurship, and trade, or the built environment — urban planning, housing,
> infrastructure, and climate adaptation.

- **t015 — "Ecobank CEO Jeremy Awori on Africa's Banking Future"** (*The African Business
  Podcast*) — a bank's growth strategy and the role of development finance.
- **t018 — "Urban planning with Shuaib Lwasa"** (*African Cities*) — rethinking how African
  cities are planned and built.

### `politics` — Politics, Power & Society
> The episode is about how people are governed and how they live together: current affairs,
> elections, diplomacy, geopolitics, security, social dynamics, identity, migration,
> leadership, or the information/media space.

- **t003 — "What UK prime minister resignation means for Africans"** (*Focus on Africa*) —
  current-affairs analysis of a political event and its diaspora impact.
- **t006 — "Who is really speaking? ... protest and foreign manipulation"** (*Pan-African
  Frequency*) — digital sovereignty and the information space.

## Decision order (apply top-down — first match wins)

This ordering is what guarantees exclusivity when an episode spans themes:

1. About a **creative or intellectual work, artist, or field of ideas** (theatre, film,
   music, art, literature, storytelling, scholarship)? → `arts`
   *(wins even when the artist is a historical figure)*
2. Otherwise, **set in the past** — historical events, eras, kingdoms, colonialism,
   archaeology, heritage? → `history`
3. Otherwise, about **money, enterprise, or the built/physical environment** — business,
   finance, trade, urban planning, housing, infrastructure, climate adaptation? → `economy`
4. Otherwise — **current governance, geopolitics, security, social dynamics, identity, or
   the information space** → `politics`

## Hard edge cases

- **A — Historical artist (`arts` vs `history`).** An episode about a historical *figure*
  who happens to be an artist (e.g. Ben Enwonwu, Sipho Mabuse). *Rule:* `arts` wins — if
  the episode is about the *work and creative legacy*, it's `arts`, even when the person is
  long dead. `history` is reserved for events/eras/movements.
- **B — Heritage site vs built environment (`history` vs `economy`).** An episode naming a
  city or building (e.g. "The Stone City of Songo Mnara"). *Rule:* if it's archaeology/the
  past, it's `history`; if it's present-day planning/housing/infrastructure, it's `economy`.
  The deciding question is **tense**: is the city being excavated or being built?
- **C — Disinformation / media (`politics` vs `economy`).** The information ecosystem and
  propaganda are a form of power → `politics`, not `economy`, even when "digital
  sovereignty" sounds techy.
- **D — Social policy with an economic hook (`economy` vs `politics`).** *Rule:* if the
  subject is the *market/enterprise/infrastructure*, it's `economy` (e.g. carbon markets);
  if the subject is *how people are treated or governed*, it's `politics` (e.g. xenophobia).

When a genuinely 50/50 case appears during annotation, apply the decision order and log the
episode id with the two candidate labels and the reason for the call, so the boundary stays
consistent across the dataset.
