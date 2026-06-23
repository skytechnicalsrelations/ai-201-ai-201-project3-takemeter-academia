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

### Edge Case
"My advisor ignored me for months, and it ruined my PhD. You should switch advisors immediately."

Possible Labels:
- personal_anecdote
- unsupported_take

Decision Rule:
If the comment mainly describes the author's experience, label it personal_anecdote.
If it generalizes from a single experience into a strong recommendation without sufficient reasoning, label it unsupported_take.