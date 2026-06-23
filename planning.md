# TakeMeter — Project Plan

A fine-tuned classifier that labels discourse quality in online music communities.
This document is the spec: it defines the community, the label taxonomy, how data is
collected and annotated, how the model is evaluated, what "success" means, and where AI
tools are used in the workflow. It is written before any example is labeled.

---

## 1. Community

**Chosen communities: r/LetsTalkMusic and r/kpop.**

I chose music discussion because the line between a *real argument* and a *hot take* is
something these communities actively police themselves — "source?", "that's just your
opinion", and "low-effort post" are recurring replies. That makes discourse quality a
distinction the community already cares about, not one I'm inventing.

I pair two subreddits on purpose because each anchors a different end of the quality
spectrum:

- **r/LetsTalkMusic** is explicitly discussion-first. It removes low-effort and reaction
  posts and rewards substantive, evidenced argument, so it's a rich source of `analysis`.
- **r/kpop** is comeback- and news-driven. MV drops, chart and sales updates, award shows,
  and "best group/album of the year" debates generate waves of immediate hype and
  disappointment (`reaction`) alongside confident, unbacked ranking claims (`hot_take`).

**Why this is a good fit for a classification task:** the discourse is text-heavy, public,
high-volume (easily 200+ posts), and genuinely varied in quality — the same event (a
comeback, a new album) produces careful breakdowns, bold assertions, and pure emotional
reaction side by side. That variety is exactly what makes the labels meaningful; a community
where every post looked the same would have nothing to classify.

---

## 2. Labels

The taxonomy measures **how a post supports its position**, not whether the opinion is
correct. The three labels are mutually exclusive: every post gets the single label that
describes its dominant mode.

### `analysis` — argument backed by specific, verifiable evidence
The post makes a structured claim about music and supports it with specifics: particular
tracks or moments, production or arrangement details, music-theory observations,
discography/historical context, or concrete lyrical references — such that the reasoning
would still stand if the opinion framing were removed.

- **Example A:** "I think *In Rainbows* is Radiohead's most cohesive record because the
  rhythm section finally drives the songs — '15 Step' is in 5/4, 'Reckoner' rides a
  shuffling hi-hat, 'Weird Fishes' builds on one arpeggio. Compared to the texture-first
  *Kid A*, every track here has a groove you can follow."
- **Example B:** "People reduce this group's title tracks to 'noise', but listen to the
  arrangement — the pre-chorus drops out every instrument except the bass before the drop,
  and they reuse that tension trick on three singles now. It's a deliberate structural
  signature, not random maximalism."

### `hot_take` — bold claim asserted without real support
A confident opinion or value judgment stated with little or no evidence. It might be true,
but it asserts rather than argues. Includes sweeping rankings, "overrated/underrated"
declarations, and claims propped up by one vague or decorative detail rather than genuine
reasoning.

- **Example A:** "This is the best comeback of the year and it's not even close, anyone who
  disagrees just isn't listening properly."
- **Example B:** "Title tracks have gotten worse across the board — companies stopped caring
  about actual songs years ago."

### `reaction` — immediate emotional response to a specific event
An in-the-moment feeling triggered by a specific event: an MV drop, single, comeback, tour
announcement, or artist news. It expresses excitement, disappointment, or hype with little
to no argument or quality claim.

- **Example A:** "THE MV JUST DROPPED I'm literally shaking, today is a good day"
- **Example B:** "Saw the tour dates and there's nothing within 6 hours of me again.
  Devastated. Why do they always skip the middle of the country."

### Decision boundaries (one sentence each)
- **analysis vs. hot_take:** If the post supplies specific, verifiable evidence that would
  genuinely support the claim even with the opinion framing removed → `analysis`; if the
  evidence is absent, vague, cherry-picked, or merely decorative → `hot_take`.
- **hot_take vs. reaction:** If the post makes a standalone judgment/claim about quality →
  `hot_take`; if it's an emotional response tied to a specific event with no real claim →
  `reaction`.
- **analysis vs. reaction:** If the post argues and evaluates → `analysis`; if it just
  expresses a feeling about an event → `reaction`.

---

## 3. Hard edge cases

**1. The decorative-evidence post (the hardest case — `analysis` vs. `hot_take`).**
> "This album is overrated — it's got like two good songs on the whole tracklist."
It *looks* like it cites evidence ("two good songs") but the detail is vague, cherry-picked,
and chosen for effect rather than as part of an argument.
**Handling rule:** evidence must be *load-bearing*. If removing the opinion framing leaves a
claim that still reasons toward a conclusion, label `analysis`; if the "evidence" is just
enough to sound credible but isn't doing argumentative work, label `hot_take`.
→ **`hot_take`**

**2. The excited post that contains a real detail (`reaction` vs. `analysis`).**
> "JUST HEARD THE NEW title track and the way the beat drops into the chorus at 1:40 is
> INSANE, been waiting years for this!!!"
It contains a specific moment ("beat at 1:40") but is dominated by event-driven hype.
**Handling rule:** label by the post's *dominant mode* — if the specific detail is part of an
argument/evaluation → `analysis`; if it's decoration inside an emotional, event-triggered
outburst → `reaction`. → **`reaction`**

**3. The discussion-prompt question (common on r/LetsTalkMusic).**
> "Why does this genre get dismissed as 'just noise'?"
Posts often open with a question in the title.
**Handling rule:** classify by the *body*, not the title. If the body builds an evidenced
case → `analysis`; if it just states the OP's strong opinion → `hot_take`; if it's a short
emotional vent → `reaction`. A title-only question with no substantive body is labeled by
its closest mode (usually `hot_take` if it carries an implied claim).

**General tie-breaker for any ambiguous post:** pick the *dominant mode* — the function the
post mostly serves. When still 50/50 between `analysis` and `hot_take`, apply the
load-bearing-evidence test; when 50/50 between `hot_take` and `reaction`, ask whether there
is a quality *claim* (→ `hot_take`) or only a *feeling* (→ `reaction`). These rules are
written down so the same call is made consistently across all 200 examples.

---

## 4. Data collection plan

**Source.** Public posts and top-level comments from r/LetsTalkMusic and r/kpop, collected
via the Reddit API / PRAW (or saved `.json` listings). I'll pull from a mix of `hot`, `top`
(past month/year), and active megathreads so the sample spans both careful threads and
real-time reaction threads. Each row stored as `text`, `label`, plus `source_subreddit` and
`permalink` for traceability.

**Target counts.** At least **200 examples total**, aiming for a balanced split of roughly
**70–80 per label** (≥20% per label, per the project guidance). Expected natural skew:
r/LetsTalkMusic supplies most `analysis`; r/kpop comeback/news threads supply most
`reaction`; `hot_take` is drawn from both.

**Workflow.** Read 30–40 posts first to confirm the labels apply cleanly to real text and
tighten definitions if needed (see §6). Then collect a larger raw pool (~300+), and annotate
toward the per-label targets rather than labeling blindly in feed order.

**If a label is underrepresented after 200:** I will (a) targeted-collect for the thin
label — e.g., if `analysis` is short, scrape more from r/LetsTalkMusic threads and
"unpopular opinion / let's discuss" posts; if `hot_take` is short, pull from r/kpop ranking
and "hot take" threads — until each label clears the 20% floor. I will **not** fix imbalance
by downsampling so aggressively that total drops below 200, and I'll report the final
per-label counts in the README. If a label is structurally rare (<10% even after targeted
collection), I'll reconsider merging it rather than ship a class the model can't learn.

---

## 5. Evaluation metrics

Accuracy alone is misleading here because the classes will not be perfectly balanced and the
three errors are not equally interesting. I'll report, on the locked test set, for **both**
the fine-tuned model and the Groq zero-shot baseline:

- **Overall accuracy** — headline number, and the simplest way to compare to the baseline.
- **Macro-averaged F1** — the primary metric. It weights each label equally, so a model that
  wins on the majority class but fails a minority class is penalized. With likely imbalance,
  macro-F1 is a more honest summary than accuracy.
- **Per-class precision, recall, and F1** — to see *which* labels work. I specifically expect
  the `analysis` ↔ `hot_take` boundary to be the weakest, and per-class numbers will show it.
- **Confusion matrix** — to read the *direction* of errors. For a community tool, confusing
  `hot_take` → `analysis` (giving an unbacked take false credibility) is a more costly error
  than `reaction` → `hot_take`, and the matrix is what surfaces that pattern.

**Baseline comparison framing:** random guessing on 3 classes ≈ 0.33 accuracy. The
zero-shot LLM baseline is the real bar — fine-tuning is only "worth it" if it beats the
baseline by a meaningful margin (see §6 reflection), not just beats random.

---

## 6. Definition of success

Success criteria are stated as thresholds I can objectively check at the end, not "it works
well."

**Minimum bar (project is a success):**
- Fine-tuned model **macro-F1 ≥ 0.65** on the test set, **and**
- It **beats the zero-shot baseline's macro-F1 by ≥ 0.05**, **and**
- **No single class has F1 < 0.55** (the model must actually learn all three labels, not
  punt on the hard one).

**Reasoning:** on a genuinely subjective 3-class task, even human annotators won't agree
100% of the time, so chasing >0.90 would signal a leak or labels that are too easy. ~0.65
macro-F1 that clearly beats a strong 70B zero-shot model is a real, defensible result for
this difficulty.

**Good-enough-for-deployment bar (would I put this in a real community tool):**
- **Macro-F1 ≥ 0.75**, **and**
- **`hot_take` precision ≥ 0.70** specifically — because the tool's value is flagging takes
  that *assert without evidence*, and a tool that frequently mislabels real `analysis` as
  `hot_take` would annoy exactly the high-quality posters you want to keep.
- Confidence behaves sensibly (high-confidence predictions are right more often than
  low-confidence ones) — checked as the optional calibration stretch feature.

If the minimum bar is met but the deployment bar isn't, the honest conclusion is "useful as
an assistive triage signal, not an autonomous labeler" — and I'll say so in the report.

---

## 7. AI Tool Plan

This project has no implementation code to generate, so AI tools are used at three specific
points in the *design and analysis* workflow. I've made an explicit decision about each.

**a. Label stress-testing — WILL DO, before annotating.**
I'll give an LLM my label definitions and edge-case rules and ask it to generate 8–10 posts
that deliberately sit on the `analysis` ↔ `hot_take` and `hot_take` ↔ `reaction`
boundaries. If I can't classify a generated post cleanly using my own rules, that's a signal
the definition is too loose, and I'll tighten it *before* labeling 200 real examples. I'll
record any definition changes this prompts in the version history of this file.

**b. Annotation assistance — WILL USE, with disclosure and tracking.**
I'll use an LLM (the same Groq `llama-3.3-70b` baseline prompt, or a comparable model) to
**pre-label** batches, then review every pre-label myself and correct it — the LLM proposes,
I decide. To keep this honest and avoid contaminating the baseline comparison: I'll add a
`pre_labeled` boolean and a `final_label_changed` flag to the dataset so I can report what
share of pre-labels I overrode, and I'll disclose the tool and process in the README's AI
usage section. The held-out **test set will be labeled by hand only** (no LLM pre-labeling)
so the baseline isn't evaluated against labels it effectively helped write.

**c. Failure analysis — WILL DO, with my own verification.**
After evaluation, I'll give the AI the list of wrong predictions (text, true label, predicted
label, confidence) and ask it to surface *systematic* patterns — e.g., "misclassifies short
posts", "confuses sarcasm for analysis", "fails on posts with one decorative stat". I'll
treat its output as hypotheses only: for each claimed pattern I'll pull the specific examples
and confirm the pattern actually holds (and isn't cherry-picked) before putting it in the
evaluation report.

---

## 8. Notes / version history

- **v1 (Milestone 1):** community = r/LetsTalkMusic + r/indieheads; 3-label taxonomy
  (analysis / hot_take / reaction).
- **v2 (Milestone 1 revision):** swapped r/indieheads → r/kpop; examples re-grounded in
  K-pop discourse.
- **v3 (Milestone 2):** full spec — added data collection plan, evaluation metrics,
  definition of success (with thresholds), and AI Tool Plan.
- **Before each stretch feature:** update this file first.

> **Reminder:** the example posts in §2–§3 are representative of the communities' real
> discourse. Validate them against the 30–40 real posts read during the §4 first-pass before
> committing to annotating 200.
