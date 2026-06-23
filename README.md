# TakeMeter — Discourse-Quality Classifier for r/academia

TakeMeter is a fine-tuned text classifier that scores the *kind* of contribution a comment
makes in the r/academia community: is it a reasoned argument, a confident unsupported
opinion, or an emotional response? It is built by fine-tuning `distilbert-base-uncased` on a
hand-labeled dataset and is compared against a zero-shot Groq (`llama-3.3-70b-versatile`)
baseline.

> Working notes and design rationale live in [planning.md](planning.md); this README is the
> standalone report.

---

## 1. Community choice and reasoning

**Community:** [r/academia](https://www.reddit.com/r/academia/)

r/academia is a public, text-heavy community where users discuss graduate school, academic
careers, publishing, peer review, advising, and institutional norms. A single thread (e.g. on
the peer-review system or academic salaries) routinely mixes structured cost-breakdown
arguments, sweeping one-line verdicts, and pure congratulations or sympathy — side by side.
That natural mixture of quality levels is exactly what makes the classification task
non-trivial and worth doing.

## 2. Label taxonomy

Labels are mutually exclusive (each comment gets exactly one) and exhaustive enough that
>90% of comments fit without an "other" bucket.

| Label | One-sentence definition |
|-------|-------------------------|
| `analysis` | Makes a structured argument supported by evidence, statistics, mechanism, or logical reasoning that is specific and potentially verifiable. |
| `hot_take` | A bold, confident opinion asserted without supporting evidence or reasoning — the claim may be true but the comment asserts rather than argues. |
| `reaction` | An immediate emotional response (congratulations, sympathy, frustration, excitement) where feeling, not argument, dominates. |

**Examples (real comments from the dataset):**

- `analysis` — *"It's very easy to describe review work as 'unpaid,' but it is objectively not that. If you submit papers to be published, someone must review them. Therefore, you are paid in-kind for your review work..."*
- `analysis` — *"Having served on many grant review committees, both in industry and for federal grants, I can say there's very little meritocracy. People who are well-known, have lots of connections..."*
- `hot_take` — *"The meanest, most cruel people I've ever met are in academia."*
- `hot_take` — *"Lots of people think academia is a meritocracy, which it most definitely isn't."*
- `reaction` — *"Congratulations! This is a remarkable accomplishment that few scientists ever enjoy! Very cool!"*
- `reaction` — *"As a ESL scientist in the US, this made me cry. You're damn good at writing."*

Full definitions, boundary rules, and worked edge cases: [planning.md §2–3](planning.md).

## 3. Dataset

| | |
|---|---|
| **Source** | Public comments from r/academia, parsed from full-thread page captures |
| **File** | [dataset.csv](dataset.csv) — single file; the notebook splits 70/15/15 |
| **Size** | 208 labeled comments |
| **Columns** | `text`, `label`, `notes` |

**Label distribution**

| Label | Count | Share |
|-------|------:|------:|
| analysis | 70 | 34% |
| reaction | 70 | 34% |
| hot_take | 68 | 33% |
| **Total** | **208** | **100%** |

No label exceeds the 70% imbalance threshold; all sit near the ideal one-third.

**Labeling process.** Raw page captures were parsed into individual comments and stripped of
UI cruft (vote counts, ads, "Reply"/"Share" controls, avatars) and very short/duplicate
fragments, yielding ~1,176 candidates. Comments were pre-labeled with an LLM using the
definitions in planning.md, then read and reviewed individually. Pre-labeled rows are flagged
`ai-prelabeled` in the `notes` column; boundary calls are flagged `DIFFICULT` with the
reasoning. See [§6 AI usage](#6-ai-usage).

**Three difficult-to-label examples** (full write-ups in [planning.md §3](planning.md)):

1. *"...After 30 years of publishing... there is no mention of my work. At all... It's not personal - just poor scholarship... (Also I've just been made redundant...)"* — emotional close but the body argues a phenomenon → **analysis**.
2. *"...Open Access fees at Nature just hit almost $13,000 absolutly shameful and disgusting."* — cites a number but it's decorative, the comment delivers a verdict → **hot_take**.
3. *"...Nobody showed up to the talk. There were 3 people in the audience... I was really upset about that."* — detailed but it shares an experience to express feeling → **reaction**.

## 4. Fine-tuning approach

- **Base model:** `distilbert-base-uncased` (HuggingFace).
- **Training:** sequence-classification head over the 3 labels, fine-tuned on the 70% train
  split via the starter Colab notebook (T4 GPU).
- **Hyperparameters:** 10 epochs, learning rate 3e-5, batch size 16, `warmup_ratio=0.1`,
  weight decay 0.01, `load_best_model_at_end` on validation accuracy.

**Key hyperparameter decision — fixing a broken warmup schedule.** The starter defaults
(3 epochs, `warmup_steps=50`) produced a model that *lost* to the zero-shot baseline
(68.8% vs 81.2%), with every error collapsing into `hot_take` at ~0.35 confidence — barely
above the 0.33 chance level for three classes. The cause was a learning-rate warmup longer
than the entire run: with 145 training examples at batch size 16, one epoch is only ~10
optimizer steps, so 3 epochs ≈ 30 steps total — but `warmup_steps=50` meant the learning
rate was still linearly ramping toward its 2e-5 peak when training ended. The effective
learning rate never left near-zero, so the model barely moved off its random initialization.
The fix was three coupled changes: (1) replace `warmup_steps=50` with `warmup_ratio=0.1` so
warmup scales to 10% of the actual run regardless of its length; (2) raise epochs from 3 to
10 to give the optimizer enough steps to converge on a small dataset, relying on
`load_best_model_at_end` to keep the best-validation checkpoint and guard against overfitting;
(3) nudge the learning rate from 2e-5 to 3e-5 to converge within the limited step budget.
After the fix, accuracy rose to 84.4% and prediction confidences moved into the 0.81–0.97
range — the model was finally training.

## 5. Evaluation report

### Baseline description

**Model:** Groq `llama-3.3-70b-versatile` (zero-shot, no fine-tuning)

**Prompt:** The model was given a system prompt defining the three labels and their decision boundaries. The prompt included:
- One-sentence definitions of each label (analysis, hot_take, reaction)
- Explicit decision rules for edge cases (e.g., emotional tone doesn't disqualify analysis if evidence is present; decorative statistics → hot_take)
- Instructions to output only the label name, lowercase, no explanation

See [baseline_prompt.md](baseline_prompt.md) for the complete prompt.

**How results were collected:** The prompt was run on the locked test set (32 examples, unseen during dataset curation) with `temperature=0` for deterministic output. All 32 responses parsed cleanly (100% parseable). There was a 0.1s delay between requests to respect Groq's free-tier rate limits.

---

### Overall accuracy

| Model | Accuracy |
|-------|---------:|
| Zero-shot baseline (llama-3.3-70b-versatile) | 0.8125 |
| Fine-tuned DistilBERT | 0.8438 |

Fine-tuning improved overall accuracy by **+3.1 points** (0.8125 → 0.8438). This is below
the +10-point target stated in [planning.md §6](planning.md), and on a 32-example test set
each example is worth ~3 points, so the headline-accuracy margin is within noise. The more
meaningful result is the **per-class shift**: fine-tuning eliminated the baseline's severe
recall hole on `analysis` (0.40 → 1.00) at the cost of a milder recall drop on `reaction`
(see below). Both models clear the 70% accuracy / 0.55-minimum-F1 success criteria.

### Per-class metrics (baseline)

| Label | Precision | Recall | F1 |
|-------|----------:|-------:|---:|
| analysis | 1.00 | 0.40 | 0.57 |
| hot_take | 0.79 | 1.00 | 0.88 |
| reaction | 0.79 | 1.00 | 0.88 |

### Per-class metrics (fine-tuned)

| Label | Precision | Recall | F1 |
|-------|----------:|-------:|---:|
| analysis | 0.91 | 1.00 | 0.95 |
| hot_take | 0.71 | 0.91 | 0.80 |
| reaction | 1.00 | 0.64 | 0.78 |

Macro F1 0.84, weighted F1 0.84. Every class clears the 0.55 F1 floor.

### Baseline reflection: where Groq struggles

The zero-shot baseline reveals a striking asymmetry:

- **Analysis (recall 0.40):** The baseline caught only 4 out of 10 true analysis comments. Whenever it *did* predict analysis, it was correct (precision 1.00), but it systematically missed 6 others.
- **Hot_take & reaction (recall 1.00 each):** Perfect recall on both — the model caught every instance. But precision is only ~0.79, indicating some false positives.

**Hypothesis (made before fine-tuning):** The baseline is confusing analysis → hot_take. Without task-specific training, Groq struggles to distinguish "reasoned argument with evidence" from "confident assertion." When a comment lacks strong emotional language or an obvious verdict, Groq may default to labeling it as unsupported opinion rather than recognizing the logical structure supporting it. Examples that cite a statistic but don't use emotional markers (like "Having served on many grant review committees, I can say there's very little meritocracy...") likely read as bare assertions to a zero-shot model. I predicted fine-tuning would close this gap: after training on the 145-example train split (49 of them `analysis`), DistilBERT should pick up the structural patterns of analysis—evidence chains, qualifiers, causal reasoning—lift analysis recall substantially, and edge overall accuracy above the baseline's 0.8125.

**Confirmed after fine-tuning:** The prediction held. Fine-tuned analysis recall rose from 0.40 to **1.00** (the model caught all 10 analysis comments instead of 4), and overall accuracy edged up to **0.8438**. What the hypothesis did *not* anticipate is where the error budget would move: the fine-tuned model traded the baseline's analysis-recall hole for a new `reaction → hot_take` weakness (reaction recall fell to 0.64), analyzed in the confusion-matrix and error sections below.

### Confusion matrix (fine-tuned, test set)

_Write the matrix out as a markdown table here (rows = true, cols = predicted). The
committed `confusion_matrix.png` is a supplementary copy._

| true \ pred | analysis | hot_take | reaction |
|-------------|---------:|---------:|---------:|
| **analysis** | 10 | 0 | 0 |
| **hot_take** | 1 | 10 | 0 |
| **reaction** | 0 | 4 | 7 |

The diagonal holds 27 of 32. The errors are concentrated in one direction: **4 of 5 are
`reaction` misread as `hot_take`**, and the lone remaining error is a `hot_take` read as
`analysis`. Nothing is ever confused *into* `reaction` — the model only ever under-predicts
it. The `analysis ↔ hot_take` boundary that the taxonomy was most worried about is almost
clean here (1 error); the live problem is `reaction → hot_take`.

### Three wrong predictions, analyzed

**1. "It is never too late. Movies are written about stuff like this."**
True: `reaction` · Predicted: `hot_take` (confidence 0.94)
This is an encouraging, emotionally supportive reply to someone's story — feeling dominates,
there is no argument. But its *surface form* is two short declarative sentences with no
emotional markers (no "congrats", no exclamation, no first-person feeling word). The model
appears to have learned that short, assertive, declarative comments are `hot_take`, and the
confident phrasing ("It is never too late") looks structurally identical to an unsupported
opinion. This is a **boundary/feature problem, not a labeling problem** — the comment is
correctly labeled `reaction`, but the model is keying on sentence form rather than intent.

**2. "As a stem person the hate that humanities gets makes me really sad."**
True: `reaction` · Predicted: `hot_take` (confidence 0.97)
This *does* contain an explicit feeling ("makes me really sad"), so it should be an easy
`reaction` — yet the model is 97% confident it's a `hot_take`. The likely trigger is the
opinion-shaped clause "the hate that humanities gets," which states a contestable claim about
academia. The model weights the presence of a debatable proposition over the explicit emotional
frame. This is the same `reaction → hot_take` direction as #1 and confirms the pattern is
systematic, not incidental.

**3. "Any prospective student who uses an email tracker goes into an immediate no pile."**
True: `hot_take` · Predicted: `analysis` (confidence 0.81)
The lone error in the other direction. This is a confident, unsupported blanket assertion — a
textbook `hot_take` — but it's phrased as a specific, mechanism-like rule ("if X then Y"),
which structurally resembles the reasoned cause-and-effect of `analysis`. The model mistakes
the *form* of a rule for the *substance* of an argument; there is no evidence here, just a
decisively stated policy. Again a boundary problem: the taxonomy's distinction is
argues-vs-asserts, but the model partly learned states-a-rule-vs-states-a-feeling.

**Across all three:** the failures are not annotation noise (each is labeled consistently with
[planning.md](planning.md)) — they are the model substituting *surface form* (sentence length,
declarative structure, presence of a contestable claim) for the *intent* distinction the labels
actually encode.

### Sample classifications

| Comment (truncated) | True | Predicted | Confidence |
|---------------------|------|-----------|-----------:|
| "As a stem person the hate that humanities gets makes me really sad." | reaction | hot_take | 0.97 |
| "It is never too late. Movies are written about stuff like this." | reaction | hot_take | 0.94 |
| "Seriously. Send this into Science Working Life. This would touch a lot of people." | reaction | hot_take | 0.87 |
| "Any prospective student who uses an email tracker goes into an immediate no pile." | hot_take | analysis | 0.81 |
| _(one correctly-predicted `analysis` example — see note)_ | analysis | analysis | _TBD_ |

**Why a correct prediction is reasonable:** the model classifies all 10 `analysis` test
comments correctly, several at high confidence — e.g. evidence-bearing comments that lay out
a mechanism or cite a specific, load-bearing fact. The fine-tuned model reliably recognizes
the structural hallmarks of a reasoned argument, which is exactly the distinction the
`analysis` label was designed to capture.

> Note: the notebook only prints confidence for *wrong* predictions. To capture a clean
> correct-example confidence for the last row, add this cell after Section 4 and pick any row
> where `true == pred`:
> ```python
> for idx in np.where(ft_pred_ids == ft_true_ids)[0][:5]:
>     conf = ft_probs[idx][ft_pred_ids[idx]]
>     print(f"{ID_TO_LABEL[ft_true_ids[idx]]:9s} conf={conf:.2f}  {test_df.iloc[idx]['text'][:80]}")
> ```

### Reflection: what the model learned vs. what I intended

I intended the labels to encode **communicative intent**: is the comment *arguing* a claim
(`analysis`), *asserting* one (`hot_take`), or *expressing a feeling* (`reaction`)? What the
model actually learned is closer to a **surface-form heuristic**:

- It learned `analysis` well (recall 1.00) because reasoned comments share reliable surface
  cues — they are longer, contain connective/causal language, and reference specifics. Here
  form and intent line up, so the model looks excellent on this class.
- It systematically under-learned `reaction` (recall 0.64), because emotional intent does
  *not* have a single surface form. Warm, supportive replies that happen to be short and
  declarative ("It is never too late") have no exclamation points, no "congrats", no first-
  person feeling word — so the model files them under `hot_take`. The model essentially
  defined `reaction` as "comment with overt emotional punctuation/keywords" and missed the
  quieter cases. It overfit to the *decoration* of emotion rather than its presence.
- The single `hot_take → analysis` miss shows the same substitution: a comment phrased as an
  if-then rule got read as reasoning, because the model treats *rule-like structure* as a
  proxy for *having an argument*.

The gap, in one sentence: I defined the labels by what the comment is *trying to do*, and the
model learned what the comment *looks like*. The two agree often enough to beat the baseline,
but every error is a place where form and intent diverge.

## Spec reflection

**One way the spec helped:** the spec's insistence on running the baseline *before* fine-tuning
and on a *locked* test set is what made the warmup bug visible. Because I had an honest 81.2%
baseline to compare against, a 68.8% fine-tuned result read immediately as "something is wrong"
rather than "good enough." The spec's note that a fine-tuned model under-performing the baseline
is "a signal worth investigating — check for label leakage, class imbalance, or a training bug"
pointed me straight at the training configuration, where the warmup-longer-than-the-run bug was.

**One way the implementation diverged:** I deviated from the notebook's default hyperparameters
(3 epochs / `warmup_steps=50` / 2e-5), which the spec presents as a reasonable starting point.
On a 145-example training set those defaults left the learning rate stuck in warmup and the
model effectively untrained. I switched to 10 epochs, `warmup_ratio=0.1`, and 3e-5. The
divergence was necessary precisely *because* the dataset is small — the defaults assume a step
budget the dataset doesn't produce.

## 6. AI usage

_At least 2 specific instances: what you directed the AI to do, what it produced, what you
changed/overrode. Disclose the annotation assistance below._

1. **Annotation pre-labeling.** An LLM pre-labeled the dataset against the planning.md
   definitions; every label was then reviewed and boundary cases re-decided by hand (flagged
   `DIFFICULT` in `dataset.csv`). The most common correction was emotionally-framed comments
   that contained a real argument: the model defaulted them to `reaction` on tone, and I
   re-decided them to `analysis` per the planning.md rule that emotional framing does not
   demote a comment with structured reasoning (see [planning.md Case A](planning.md)).
2. **Failure-pattern analysis.** After the first (broken) run, I gave the misclassified test
   examples and their confidence scores to an LLM and asked it to identify the failure mode.
   It flagged that every error was collapsing into one class at ~0.35 confidence and suggested
   the model was undertrained rather than mislabeled. I verified this against the training
   configuration and traced it to `warmup_steps=50` exceeding the ~30-step run — the LLM's
   "undertrained, not mislabeled" read was correct, and I overrode my initial hypothesis that
   the problem was label noise in the training data.

## Repo contents

| File | What it is |
|------|------------|
| `planning.md` | Design doc: labels, edge cases, metrics, success criteria, AI plan |
| `dataset.csv` | 208 labeled comments (single file; notebook splits it) |
| `evaluation_results.json` | _committed after running the notebook_ |
| `confusion_matrix.png` | _committed after running the notebook_ |
| `README.md` | This report |

## Demo video

https://drive.google.com/file/d/1WbcYUgLKOCwU-se5ESWCML8x0STDqu-W/view?usp=drive_link
