# TakeMeter

A fine-tuned text classifier that evaluates **discourse quality** in online music
communities. Given a post, TakeMeter labels *how* it supports its position — a reasoned
argument, a bold unsupported claim, or a pure emotional reaction.

This README documents the label taxonomy and the annotated dataset. The full project spec
(community rationale, evaluation plan, success criteria, AI-tool plan) lives in
[`planning.md`](./planning.md). Model training and evaluation results are added in later
milestones.

## Community

TakeMeter is built on two music-discussion subreddits, chosen because each anchors a
different end of the quality spectrum:

- **r/LetsTalkMusic** — discussion-first; rewards substantive, evidenced argument. Main
  source of `analysis`.
- **r/kpopthoughts** — the K-pop opinions/discussion sub; comeback and fandom debates produce
  bold takes and emotional reactions. (r/kpop itself was tried first and rejected: its feed
  is news/MV headlines, not *takes*.)

"Is this a real argument or just a hot take?" is a distinction these communities already
police themselves, which is what makes the taxonomy meaningful.

## Label taxonomy

Three mutually exclusive labels. Every post gets the single label describing its **dominant
mode**. The taxonomy measures *how a post supports its position*, not whether the opinion is
correct.

### `analysis` — argument backed by specific, verifiable evidence
A structured claim supported by specifics (particular tracks/moments, production or
arrangement details, music-theory observations, discography/historical context, or concrete
lyrics) — such that the reasoning would still stand if the opinion framing were removed.

> *"I think In Rainbows is Radiohead's most cohesive record because the rhythm section
> finally drives the songs — '15 Step' is in 5/4, 'Reckoner' rides a shuffling hi-hat,
> 'Weird Fishes' builds on one arpeggio."*

> *"People reduce this group's title tracks to 'noise', but the pre-chorus drops every
> instrument except the bass before the drop, and they reuse that tension trick on three
> singles now — a deliberate structural signature, not random maximalism."*

### `hot_take` — bold claim asserted without real support
A confident opinion or value judgment stated with little or no evidence. It might be true,
but it asserts rather than argues — sweeping rankings, "overrated/underrated" declarations,
or claims propped up by one vague or decorative detail.

> *"This is the best comeback of the year and it's not even close, anyone who disagrees just
> isn't listening properly."*

> *"Title tracks have gotten worse across the board — companies stopped caring about actual
> songs years ago."*

### `reaction` — immediate emotional response to a specific event
An in-the-moment feeling triggered by a specific event (an MV drop, single, comeback, tour
announcement, or artist news), with little to no argument or quality claim.

> *"THE MV JUST DROPPED I'm literally shaking, today is a good day"*

> *"Saw the tour dates and there's nothing within 6 hours of me again. Devastated."*

### Decision boundaries
- **analysis vs. hot_take:** specific, load-bearing evidence that would support the claim
  even without the opinion framing → `analysis`; absent, vague, cherry-picked, or decorative
  evidence → `hot_take`.
- **hot_take vs. reaction:** a standalone quality *claim* → `hot_take`; an emotional response
  to a specific event with no real claim → `reaction`.
- **analysis vs. reaction:** argues and evaluates → `analysis`; just expresses a feeling →
  `reaction`.

## Dataset

The labeled dataset is a single file, [`takemeter_dataset.csv`](./takemeter_dataset.csv)
(not pre-split — the notebook handles the 70/15/15 train/val/test split). Columns:

| column | description |
|---|---|
| `id` | stable row id (maps to the raw scrape pool) |
| `text` | post title + body, cleaned (URLs stripped) |
| `label` | `analysis`, `hot_take`, or `reaction` |
| `notes` | annotation notes on hard cases |
| `source` | `LetsTalkMusic` or `kpopthoughts` |
| `post_url` | permalink to the original Reddit post |
| `pre_labeled` | `yes` — every label was LLM-pre-labeled (see AI usage) |
| `reviewed` | left blank for human review/correction |

### Where the data came from
Public top-level posts (title + body) scraped from r/LetsTalkMusic and r/kpopthoughts via an
Apify Reddit actor (full-subreddit pull of the recent feed). Raw pool: **450 posts** (300
r/LetsTalkMusic + 150 r/kpopthoughts), cleaned of URLs, empties, and near-duplicates.

### Labeling process
Posts were pre-labeled by an LLM applying the definitions and decision rules above, with the
final labels to be human-reviewed (see AI usage). **Non-take posts** — questions,
recommendation requests, weekly threads, and news re-posts — don't fit the taxonomy and were
filtered out rather than forced into a label, scoping TakeMeter to viewpoint-expressing posts.

### Label distribution (221 examples)

| label | count | share |
|---|---|---|
| analysis | 105 | 47.5% |
| hot_take | 62 | 28.1% |
| reaction | 54 | 24.4% |

No class exceeds 70% and each class clears 20%. `analysis` is the plurality because reasoned,
evidenced posts dominate r/LetsTalkMusic; `reaction` is scarcest because both subs reward
substance over pure emotion. r/LetsTalkMusic supplies most `analysis`; r/kpopthoughts
supplies a larger share of `reaction` and `hot_take`.

### Three examples that were genuinely difficult to label
(ids reference `takemeter_dataset.csv`.)

1. **id 308 — "KATSEYE's Gnarly is easily the worst song I've heard in a long time."**
   `analysis` vs `hot_take`: it quotes specific lyrics, which looks like evidence — but the
   quotes are mockery, not reasoning toward a musical conclusion, so the evidence isn't
   load-bearing. → **`hot_take`**

2. **id 350 — "Fifty Fifty's photobook: 123 of 195 pages are BLANK."**
   `reaction` vs `analysis`: it cites a hard number, but the dominant mode is incredulous
   reaction to a product ("this is actually insane???"), not a built argument. →
   **`reaction`**

3. **id 61 — "The #1 most 'American' band of all time is the Grateful Dead. Fight me."**
   `hot_take` vs `analysis`: the title/"Fight me" framing reads as a hot take, but the body
   lays out numbered, substantive reasons. Classifying by the body, not the title →
   **`analysis`**

## AI usage

Per the project's disclosure requirements:

- **Pre-labeling.** Every row in `takemeter_dataset.csv` was pre-labeled by an LLM (Claude)
  applying the taxonomy in this README. These are **proposed** labels (`pre_labeled = yes`);
  the `reviewed` column is for human verification. Each label must be read and confirmed or
  corrected before training — that review is the real annotation. Because the notebook splits
  train/val/test randomly, the full set is reviewed by hand so the held-out test labels are
  human-verified before the zero-shot baseline is evaluated against them.
- **Data collection.** Posts were scraped via an Apify Reddit actor (no manual copy-paste).

## Repository

| file | purpose |
|---|---|
| `planning.md` | full project spec: community, labels, edge cases, data plan, metrics, success criteria, AI-tool plan |
| `takemeter_dataset.csv` | 221 labeled examples (single file; notebook splits it) |
| `ai201_project3_takemeter.ipynb` | fine-tuning + evaluation pipeline (DistilBERT) and Groq zero-shot baseline |
| `README.md` | this file |

## Status

- [x] Milestone 1 — community + label taxonomy
- [x] Milestone 2 — project spec (`planning.md`)
- [x] Milestone 3 — collect + annotate dataset (221 examples)
- [ ] Milestone 4 — fine-tune, run baseline, evaluation report
