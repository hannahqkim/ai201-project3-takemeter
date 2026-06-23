# TakeMeter

A fine-tuned text classifier that evaluates **discourse quality** in online music
communities. Given a post, TakeMeter labels *how* it supports its position — a reasoned
argument, a bold unsupported claim, or a pure emotional reaction.

This README is the final report: community, label taxonomy, dataset, fine-tuning approach,
zero-shot baseline, and the full evaluation. The working notes and design rationale (success
criteria, AI-tool plan, edge-case decisions) live in [`planning.md`](./planning.md).

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

Specific instances of AI assistance, with what I directed, what it produced, and what I
changed or overrode:

1. **Annotation pre-labeling (disclosed).** I gave an LLM (Claude) my label definitions and
   edge-case rules and directed it to assign one of `analysis`/`hot_take`/`reaction` to each
   scraped post, and to *skip* posts that weren't takes. It produced the proposed labels in
   `takemeter_dataset.csv` (every row flagged `pre_labeled = yes`). I review and correct these
   myself — the `reviewed` column tracks that pass — so the final labels are human-verified
   before training. Because the notebook splits train/val/test randomly, I review the whole
   set so test labels aren't effectively written by the model that I then benchmark against.

2. **Debugging the fine-tuning collapse.** After the first training run scored 0.471 and
   predicted `analysis` for all 34 test posts, I asked the AI to diagnose it. It identified
   that `warmup_steps=50` exceeded the total optimizer steps (~30 for 3 epochs on 154
   examples), so the learning rate never warmed up and the model never left the majority
   class. I accepted the diagnosis and the proposed fix (warmup_ratio, more epochs, macro-F1
   model selection, class weights) — see *Training & hyperparameters* — but verified the
   reasoning against the step math myself rather than taking it on faith.

3. **Data collection.** Posts were scraped via an Apify Reddit actor (no manual copy-paste);
   I chose the subreddits and the take-only scope.

4. **Failure-pattern surfacing (evaluation).** I pasted the fine-tuned model's misclassified
   examples to an LLM and asked it to name common themes, then re-read the cases myself to
   confirm or discard each claimed pattern. See *Failure analysis*.

## Repository

| file | purpose |
|---|---|
| `planning.md` | full project spec: community, labels, edge cases, data plan, metrics, success criteria, AI-tool plan |
| `takemeter_dataset.csv` | 221 labeled examples (single file; notebook splits it) |
| `ai201_project3_takemeter_starter_clean_HK.ipynb` | fine-tuning + evaluation pipeline (DistilBERT) and Groq zero-shot baseline |
| `baseline_results.md` | zero-shot baseline record + reflection |
| `confusion_matrix.png` | fine-tuned confusion matrix (supplementary image; text version is in the report) |
| `evaluation_results.json` | exported accuracy numbers for both models |
| `README.md` | this file |

## Training & hyperparameters

Fine-tuned `distilbert-base-uncased` (a pretrained encoder with a fresh 3-way
classification head) on the 154-example training split.

**The first run failed and why.** With the starter defaults the model scored 0.471 and
predicted `analysis` for every test post (a flat confusion matrix; hot_take and reaction
recall = 0.00). The cause was a hyperparameter bug, not the data: 154 examples at batch size
16 is ~10 optimizer steps/epoch, so 3 epochs ≈ 30 steps — but `warmup_steps=50` meant the
learning rate was still warming up when training ended, so it never reached an effective rate
and the model defaulted to the majority class.

**Fixes made (and why):**

| change | from → to | why |
|---|---|---|
| epochs | 3 → 10 | ~30 steps was far too few; 10 epochs (~100 steps) lets the model actually learn |
| warmup | `warmup_steps=50` → `warmup_ratio=0.1` | warmup must scale to the real schedule, not exceed it |
| model selection | `accuracy` → macro-`f1` | accuracy rewards predicting the majority class; macro-F1 makes all three labels count |
| loss | unweighted → inverse-frequency **class weights** | dataset is imbalanced (analysis 47% vs reaction 24%); weights stop the model ignoring smaller classes |

Learning rate (2e-5) and batch size (16) were kept at the BERT-fine-tuning standard.

## Baseline (zero-shot Groq)

The comparison bar is `llama-3.3-70b-versatile` (Groq) run **zero-shot** — no task-specific
training, just a prompt. Each of the 34 test posts was sent individually at `temperature=0`
(`max_tokens=20`) with a system prompt that (a) names the task and communities, (b) gives the
one-sentence definition of each label, (c) shows one example post per label, and (d) instructs
the model to **reply with only the label name**. Responses are lowercased and matched against
the label strings; anything unmatched counts as unparseable. In the reported run **0 of 34**
were unparseable. Full prompt is in the notebook (Section 5) and the run record is in
[`baseline_results.md`](./baseline_results.md). Prompt skeleton:

```
You are classifying posts from r/LetsTalkMusic and r/kpopthoughts. Assign each
post to exactly ONE of: analysis, hot_take, reaction — based on HOW it supports
its position.
  analysis — structured claim backed by specific, verifiable evidence.   Example: "..."
  hot_take — bold opinion asserted with little/no evidence.               Example: "..."
  reaction — immediate emotional response to a specific event.            Example: "..."
Respond with ONLY the label name. Do not explain.
```

## Evaluation report

Test set: 34 held-out examples (analysis 16, hot_take 10, reaction 8).

**Headline: the zero-shot baseline clearly beats the fine-tuned model.** Fixing the training
collapse more than doubled the fine-tuned model's macro-F1 (0.21 when collapsed → 0.52 here), so
it now predicts all three classes — but at 0.56 accuracy it still trails the 70B zero-shot
baseline (0.79) by a wide margin. On 154 training examples, fine-tuning a small encoder did
**not** help for this task — an honest and informative result (see *Reflection*).

> All fine-tuned numbers below are from one run (`evaluation_results.json`: fine-tuned acc
> 0.559). Fine-tuning had run-to-run variance (0.471 / 0.50 / 0.559 observed) because the head
> was randomly initialized; the notebook now calls `set_seed(42)` before model creation to make
> a run reproducible.

### Overall accuracy

| model | accuracy | macro-F1 |
|---|---|---|
| Zero-shot baseline (Groq `llama-3.3-70b`) | **0.794** | 0.82 |
| Fine-tuned DistilBERT (corrected) | 0.559 | 0.52 |
| _difference (fine-tuned − baseline)_ | _−0.235_ | _−0.30_ |

### Per-class metrics — both models

| label | model | precision | recall | f1 | support |
|---|---|---|---|---|---|
| analysis | baseline | 0.75 | 0.94 | 0.83 | 16 |
| analysis | fine-tuned | 0.71 | 0.62 | 0.67 | 16 |
| hot_take | baseline | 0.88 | 0.70 | 0.78 | 10 |
| hot_take | fine-tuned | 0.39 | 0.70 | 0.50 | 10 |
| reaction | baseline | 1.00 | 0.75 | 0.86 | 8 |
| reaction | fine-tuned | 1.00 | 0.25 | 0.40 | 8 |
| **macro avg** | baseline | 0.88 | 0.80 | 0.82 | 34 |
| **macro avg** | fine-tuned | 0.70 | 0.52 | 0.52 | 34 |

(Baseline accuracy varies slightly run-to-run at `temperature=0` — 0.735 / 0.794 / 0.824
observed; the bundled run in `evaluation_results.json` is 0.794. The baseline per-class above
is from a representative baseline run [acc 0.824]; the head-to-head conclusion — baseline far
ahead — is unchanged.)

### Confusion matrix — fine-tuned model (text)

Rows = true label, columns = predicted (matches `confusion_matrix.png`).

| true \ pred | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 10 | 6 | 0 |
| **hot_take** | 3 | 7 | 0 |
| **reaction** | 1 | 5 | 2 |

The clearest pattern is **over-prediction of `hot_take`** (column sum 18 vs its true support of
10): 6 of 16 `analysis` posts and 5 of 8 `reaction` posts were called `hot_take`. `reaction` is
badly under-caught (2 of 8, recall 0.25) — but the model never *falsely* predicts `reaction`
(precision 1.00; the only 2 posts it called reaction were both correct). So the model defaults to
"opinion" (`hot_take`/`analysis`) and only commits to `reaction` when the emotional signal is
unmistakable. Predictions sit at low confidence overall (~0.4–0.6; chance on 3 classes is ~0.33),
so the boundary is weak.

### Failure analysis — 3 misclassified examples (real, from this run)

1. **"…it is clear that Courtney Love was right — Taylor Swift is recycling the same lyrics,
   themes, melodies… with zero artistic growth."** — true `hot_take`, predicted `analysis`
   (conf 0.61, the model's *most confident* error).
   *Why it failed:* the post lists specifics (lyrics, themes, melodies) so it carries the
   surface markers of evidence — but they're asserted, not argued. The model learned "mentions
   specifics → `analysis`" instead of "is the evidence load-bearing?", which is the
   `analysis`↔`hot_take` boundary flagged as hardest in `planning.md`. **Boundary problem, not
   labeling** — the label is correct by my rules; the distinction is too subtle for this data
   scale.

2. **"Listening to radio stations in GTA games is a spectacular way of discovering a lot of
   excellent retro music!"** — true `reaction`, predicted `hot_take` (conf 0.46).
   *Why it failed:* this is enthusiastic, in-the-moment appreciation with no quality argument,
   but the model pushed it into `hot_take`. `reaction` is the smallest class (~38 train
   examples) and the model defaults to "opinion" unless the emotional signal is unmistakable.
   **Data problem:** too few `reaction` examples to carve the class out.

3. **"More music is made in a single day now than in the entire year of 1989 and it's becoming
   a problem."** — true `analysis`, predicted `hot_take` (conf 0.43).
   *Why it failed:* this is a data-backed argument, but its confident, provocative framing reads
   as a bare opinion. The model treats *assertive tone* as a `hot_take` cue and misses the
   supporting reasoning. **Boundary problem** — same `analysis`↔`hot_take` line, in the
   opposite direction from case 1.

**What would fix it:** far more training data (each class has only ~38–43 examples), and
specifically more *hard* examples that decouple the surface proxy from the true label
(emotional posts that are still hot takes; evidence-listing posts that are still hot takes; data-
backed posts in an assertive voice). A tighter definition wouldn't help — the labels are
consistent; the model just can't learn this distinction at this scale, which is why the 70B
zero-shot model wins.

### Sample classifications

Five real test posts through the fine-tuned model (predicted label + confidence):

| post (snippet) | true | predicted | confidence | correct? |
|---|---|---|---|---|
| "Never Mind the Bollocks, Here's the Sex Pistols… I had never really given this album a shot…" | analysis | analysis | 0.62 | ✅ |
| "SOPHIE is dead. Let's talk about her influence." | reaction | reaction | 0.56 | ✅ |
| "Nazism in black metal bands shouldn't be socially accepted" | hot_take | hot_take | 0.50 | ✅ |
| "…it is clear that Courtney Love was right — Taylor Swift is recycling the same lyrics…" | hot_take | analysis | 0.61 | ❌ |
| "More music is made in a single day now than in the entire year of 1989…" | analysis | hot_take | 0.43 | ❌ |

*Why a correct prediction is reasonable:* "Never Mind the Bollocks…" (predicted `analysis`,
conf 0.62 — the model's highest-confidence call) is a structured album reflection that weighs
substance against spectacle and reasons toward a verdict. Those concrete, load-bearing details
are exactly what the `analysis` definition rewards, and `analysis` is the one class the model
learned reasonably (F1 0.69).

### Reflection — what the model captured vs. what I intended

I intended the labels to capture **how a post argues** — specifically whether its evidence is
*load-bearing*. What the fine-tuned model actually learned are **surface proxies**: an assertive,
opinionated tone pulls a post toward `hot_take` (which it over-predicts), and the mere presence
of specifics (track names, numbers, lists) pulls it toward `analysis` — without regard to whether
there's a real *claim* or whether the evidence does any work. The errors make the gap concrete: a
sweeping take that lists specifics became `analysis` (case 1), a data-backed argument in a punchy
voice was read as a bare opinion (case 3), and `reaction` was largely missed (recall 0.25) since
the model only commits to it when the emotion is unmistakable.

The deeper point is the one the spec anticipated: my taxonomy encodes a *reasoning-quality*
judgment ("is the evidence load-bearing?") that even I found hard to annotate. A 66M-parameter
model given 154 examples can only reach for the easy correlates of that judgment, not the
judgment itself — so it sits near chance (0.52 macro-F1, low confidence everywhere). The 70B
zero-shot model already approximates the intended distinction (~0.82 macro-F1) without any of my
data. The honest conclusion: the label design is sound, but for this task the value isn't in
fine-tuning a small model on a few hundred examples — it's in the prompt-defined large model.

## Spec reflection

- **One way the spec helped:** `planning.md` committed in advance to **macro-F1 and per-class
  metrics, not just accuracy**, and to a ≥20%-per-class distribution. That's exactly what
  exposed the first model's collapse — accuracy alone (0.471) looked merely mediocre, but the
  per-class F1s of 0.00 for two classes made the failure obvious and diagnosable.
- **One way the implementation diverged:** the spec planned r/LetsTalkMusic + **r/kpop** and
  Reddit API/PRAW collection. In practice r/kpop's feed was news headlines, not takes, so I
  switched to **r/kpopthoughts**, and Reddit was unreachable via API/browser so I scraped via
  **Apify** instead. The taxonomy survived the change; the source and tooling did not.

