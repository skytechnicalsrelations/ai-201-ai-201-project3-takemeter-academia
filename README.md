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
- **Hyperparameters:** _(fill in after training)_ — default starting point is 3 epochs,
  learning rate 2e-5, batch size 16. Document at least one decision you made and why
  (e.g. epochs adjusted to avoid overfitting on a small 200-example set).

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
| Fine-tuned DistilBERT | _TBD_ |

### Per-class metrics (baseline)

| Label | Precision | Recall | F1 |
|-------|----------:|-------:|---:|
| analysis | 1.00 | 0.40 | 0.57 |
| hot_take | 0.79 | 1.00 | 0.88 |
| reaction | 0.79 | 1.00 | 0.88 |

### Per-class metrics (fine-tuned)

| Label | Precision | Recall | F1 |
|-------|----------:|-------:|---:|
| analysis | _TBD_ | _TBD_ | _TBD_ |
| hot_take | _TBD_ | _TBD_ | _TBD_ |
| reaction | _TBD_ | _TBD_ | _TBD_ |

### Baseline reflection: where Groq struggles

The zero-shot baseline reveals a striking asymmetry:

- **Analysis (recall 0.40):** The baseline caught only 4 out of 10 true analysis comments. Whenever it *did* predict analysis, it was correct (precision 1.00), but it systematically missed 6 others.
- **Hot_take & reaction (recall 1.00 each):** Perfect recall on both — the model caught every instance. But precision is only ~0.79, indicating some false positives.

**Hypothesis:** The baseline is confusing analysis → hot_take. Without task-specific training, Groq struggles to distinguish "reasoned argument with evidence" from "confident assertion." When a comment lacks strong emotional language or an obvious verdict, Groq may default to labeling it as unsupported opinion rather than recognizing the logical structure supporting it. Examples that cite a statistic but don't use emotional markers (like "Having served on many grant review committees, I can say there's very little meritocracy...") likely read as bare assertions to a zero-shot model.

**Expected improvement:** Fine-tuned DistilBERT should excel at catching the structural patterns of analysis—evidence chains, qualifiers, causal reasoning—after seeing 49 training examples. The recall on analysis should improve significantly; the fine-tuned model's overall accuracy should exceed the baseline's 0.8125.

### Confusion matrix (fine-tuned, test set)

_Write the matrix out as a markdown table here (rows = true, cols = predicted). The
committed `confusion_matrix.png` is a supplementary copy._

| true \ pred | analysis | hot_take | reaction |
|-------------|---------:|---------:|---------:|
| **analysis** | _TBD_ | _TBD_ | _TBD_ |
| **hot_take** | _TBD_ | _TBD_ | _TBD_ |
| **reaction** | _TBD_ | _TBD_ | _TBD_ |

### Three wrong predictions, analyzed

_Pick 3 misclassified test examples. For each: the text, true vs predicted label, and an
analysis of **why** (which boundary failed, was it ambiguous language / a decorative stat /
short post, and is it a labeling problem or a data problem). The hot_take↔analysis boundary
is the most likely confusion given the taxonomy._

### Sample classifications

| Comment (truncated) | Predicted | Confidence |
|---------------------|-----------|-----------:|
| _TBD_ | _TBD_ | _TBD_ |

_For at least one correct prediction, add a sentence on why it's reasonable._

### Reflection: what the model learned vs. what I intended

_Higher-level than the error list: did the model latch onto surface cues (congratulatory
punctuation/emoji → reaction; question marks; comment length) rather than the
argument-vs-assertion distinction I actually defined? What did it overfit to, what did it
miss?_

## Spec reflection

_One way the spec guided the build, one way the implementation diverged and why._

## 6. AI usage

_At least 2 specific instances: what you directed the AI to do, what it produced, what you
changed/overrode. Disclose the annotation assistance below._

1. **Annotation pre-labeling.** An LLM pre-labeled the dataset against the planning.md
   definitions; every label was then reviewed and boundary cases re-decided by hand (flagged
   `DIFFICULT` in `dataset.csv`). _Note what you corrected._
2. _(e.g. label stress-testing, or failure-pattern analysis on wrong predictions.)_

## Repo contents

| File | What it is |
|------|------------|
| `planning.md` | Design doc: labels, edge cases, metrics, success criteria, AI plan |
| `dataset.csv` | 208 labeled comments (single file; notebook splits it) |
| `evaluation_results.json` | _committed after running the notebook_ |
| `confusion_matrix.png` | _committed after running the notebook_ |
| `README.md` | This report |

## Demo video

_Link to 3–5 min demo: 3–5 posts classified with label + confidence, one correct prediction
narrated, one incorrect narrated, and a walkthrough of this evaluation report._
