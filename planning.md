# TakeMeter Planning Document

## 1. Community

### Chosen Community
r/AskAcademia

### Why This Community?
This project studies discourse quality in r/AskAcademia, a public community where users discuss graduate school, academic careers, publishing, advising, and institutional norms. The classifier will distinguish between evidence-based advice, personal anecdotes, and unsupported takes. These distinctions matter because community members often rely on these comments to make serious academic and career decisions.

---

## 2. Label Taxonomy

### analysis

Definition:
The post makes a structured argument supported by evidence, statistics, historical comparison, examples, or logical reasoning. Evidence should be specific and potentially verifiable.

Examples:
1. "Faculty hiring has become more competitive over the last decade. NSF data shows PhD production has increased while tenure-track openings have remained relatively stable."
2. "Students who meet weekly with advisors tend to finish sooner because feedback loops are shorter and problems are identified earlier."

Uncertain Example:
"Most successful PhD students publish early because publication builds momentum."

---

### hot_take

Definition:
A bold or confident opinion presented without sufficient evidence or reasoning. The claim may be true, but the author asserts rather than argues.

Examples:
1. "Most PhD programs are a waste of time."
2. "Tenure is an outdated system that should be abolished."

Uncertain Example:
"Faculty jobs are impossible to get nowadays."

---

### reaction

Definition:
An immediate emotional response, expression of frustration, excitement, surprise, or sympathy. The focus is emotion rather than argument.

Examples:
1. "Wow, that's awful. I can't believe your advisor treated you like that."
2. "Congratulations! That's an amazing accomplishment."

Uncertain Example:
"That sounds incredibly frustrating. You should probably leave the program."

## 3. Hard Edge Cases

### Edge Case 1: Experience + Strong Recommendation
"My advisor ignored me for months, and it ruined my PhD. You should switch advisors immediately."

Possible Labels: reaction, hot_take

Decision Rule:
If the comment is primarily expressing emotion or describing lived experience, label it **reaction**. If it pivots from experience into a confident, generalized recommendation without supporting reasoning (e.g., "you should always do X"), label it **hot_take**. The sentence above crosses into hot_take because the prescription is presented with confidence and without qualification.

### Edge Case 2: Stat-backed but framed emotionally
"It's honestly depressing — 80% of PhDs don't end up in academia, and advisors just don't prepare you for that."

Possible Labels: analysis, reaction

Decision Rule:
If a post cites a specific, verifiable figure and uses it to support a claim, label it **analysis** even if the framing is emotional. Emotional tone alone doesn't disqualify a post from being analysis. Label **reaction** only when the emotional framing dominates and there is no structured reasoning or evidence.

### Edge Case 3: Single experience generalized without evidence
"I published two first-author papers before finishing my PhD and it made everything easier — advisors, jobs, confidence."

Possible Labels: analysis, hot_take

Decision Rule:
A personal outcome is not evidence in the analysis sense unless it is framed as part of a broader pattern supported by data or reasoning. If the post is purely reporting what happened to the author and implicitly generalizing from it, label it **hot_take**. Label **analysis** only when the post moves beyond personal experience to explain a mechanism or cite external support.

---

## 4. Data Collection Plan

### Source
Public comments and posts from r/AskAcademia collected manually via Reddit. Only top-level comments and post bodies will be collected (no nested replies, to reduce context-dependency).

### Target Distribution
- analysis: ~70 examples
- hot_take: ~70 examples
- reaction: ~70 examples
- Total: ~210 examples (buffer above the 200 minimum)

### Handling Underrepresentation
After 200 examples, if any label is below 55 examples (~27%), I will do a targeted collection pass — searching for specific post types that tend to produce that label (e.g., congratulatory threads for reaction, debate-style threads for hot_take).

### Imbalance Threshold
If any single label exceeds 70% of the dataset, I will stop and collect more examples for the underrepresented labels before training.

---

## 5. Evaluation Metrics

### Metrics
- **Overall accuracy**: Fraction of test examples correctly classified. Reported for both models.
- **Per-class F1**: Harmonic mean of precision and recall per label. F1 is the primary per-class metric because this task has no strong preference for precision over recall — missing a label is as costly as mislabeling.
- **Confusion matrix**: Rows = true labels, columns = predicted labels. Required to identify directional error patterns (e.g., model predicts hot_take when true label is analysis).

### Why Not Accuracy Alone
Accuracy can be misleading if the dataset is imbalanced. A model that predicts the majority class every time can achieve high accuracy while being useless. Per-class F1 catches this.

### Why F1 Over Precision or Recall Individually
In this task there is no strong asymmetry between false positives and false negatives across labels, so F1 is the appropriate single-number summary per class.

---

## 6. Definition of Success

### Threshold
A fine-tuned classifier is considered useful if:
- Overall accuracy on the test set exceeds 70%
- No single label has an F1 score below 0.55
- The fine-tuned model meaningfully outperforms the zero-shot Groq baseline (at minimum +10 percentage points on overall accuracy)

### Rationale
The task involves genuinely ambiguous posts and subjective distinctions. A threshold of 70% overall accuracy and 0.55 minimum F1 reflects a model that has learned the distinctions and is useful for community tooling — not perfect, but meaningfully better than random (33% on 3 classes) and better than a generic LLM with no training.

### Failure Signal
If the fine-tuned model does not beat the baseline by at least 10 points, I will treat it as a signal to investigate label consistency or training data quality before reporting results.

---

## 7. AI Tool Plan

### Label Stress-Testing
I will give Claude my three label definitions and the three edge case examples above and ask it to generate 8–10 additional posts that sit at the boundary between two labels. If Claude produces posts I cannot confidently classify, I will use those to sharpen my decision rules before starting annotation. This step happens before I label any examples.

### Annotation Assistance
I will use an LLM (Claude) to pre-label a batch of 50–80 examples by providing it with the label definitions from this document and a set of unlabeled posts. I will review and correct every pre-assigned label myself before including it in the dataset. Pre-labeled examples will be flagged in a `notes` column in the CSV so they can be tracked for the AI usage disclosure. I will not skim — each pre-label will be verified against the post text and my decision rules.

### Failure Analysis
After fine-tuning, I will paste all misclassified test examples into Claude and ask it to identify common themes across the errors (e.g., post length, sarcasm, a specific confused label pair). I will verify any patterns Claude identifies by re-reading the examples myself, and I will note in my evaluation report which patterns I confirmed, which I rejected, and why.