# TakeMeter — Project Planning

**Community:** r/soccer (Reddit)
**Task:** Multi-class text classification of Reddit posts into four intent categories

---

## Community

I chose r/soccer because I love the sport and the World Cup is currently happening, which means the subreddit is especially active and rich with varied post types. It is a great fit for a classification task because the discourse is genuinely diverse — on any given day you will find breaking transfer news, heated debates about a player's legacy, emotional reactions to a last-minute goal, and bold contrarian opinions. That natural variety across intent and tone makes it possible to build meaningful, distinct labels rather than forcing a split on subtle differences.

---

## Labels

| Label | Definition |
|-------|-----------|
| `analysis` | A post that makes a claim supported by statistics, historical precedent, or structured reasoning. |
| `hot_take` | A post expressing an opinion that goes against the mainstream view held by the r/soccer community, likely to generate strong disagreement. |
| `reaction` | A post that captures an immediate emotional response to something that just happened, prioritizing feeling over reasoning. |
| `news` | A post reporting factual information such as a transfer, injury update, managerial change, or official announcement relevant to the soccer community. |

### Examples per label

**analysis**
- "Rodri has the highest pass completion rate in the Premier League this season at 94% — statistically the best defensive midfielder in Europe right now."
- "Looking at the last 10 World Cup winners, 8 had a goalkeeper who kept at least 4 clean sheets in the tournament. That's why I think Martinez is Argentina's most important player."

**hot_take**
- "Messi is overrated and Ronaldo would have won more Ballon d'Ors if he played for Barcelona."
- "The Premier League is the most boring top league to watch — Serie A has way better tactical football."

**reaction**
- "I CANNOT BELIEVE THAT GOAL. MY HEART IS STILL RACING. VAMOS ESPAÑA!!!"
- "That referee just ended our season. I'm done with football. Done."

**news**
- "OFFICIAL: Kylian Mbappé signs for Real Madrid on a five-year deal."
- "Confirmed: Jurgen Klopp steps down as Liverpool manager at the end of the season."

---

## Hard Edge Cases

The most genuinely ambiguous overlap is between **reaction** and **hot_take** — an emotional post can also contain a contrarian opinion, and it is hard to tell whether the intent is to vent a feeling or to provoke debate. For example, *"I can't believe people still think Ronaldo is better than Messi — it's embarrassing at this point"* reads as both emotional and contrarian. A second tricky overlap is **analysis vs. hot_take**, where someone builds a statistical argument for an unpopular conclusion.

**Tiebreaker rule:** Label based on primary intent.
- Structured around evidence → `analysis`
- Core purpose is to stake an unpopular position → `hot_take`
- Dominant register is emotional → `reaction`

---

## Data Collection Plan

Examples will be collected directly from r/soccer using Reddit's search and browsing tools — sorting by Hot, Top (week/month), and New to get a range of post styles. The target is **50 examples per label (200 total)**.

If a label (most likely `analysis`) is underrepresented after an initial collection pass, I will search for posts containing statistical keywords (`stats`, `xG`, `per 90`, `historically`) or pull from dedicated analysis threads and match discussion posts to find more candidates.

---

## Evaluation Metrics

Per-class **precision**, **recall**, and **F1-score** will be tracked in addition to overall accuracy. Accuracy alone is misleading because the dataset could be skewed toward `news` posts (the most common type on the subreddit), so a model that predicts `news` most of the time could score deceptively well. Per-class F1 reveals whether the model actually learns to distinguish `analysis` from `hot_take`, or whether it collapses minority classes. A **confusion matrix** will be used to identify which label pairs are being mixed up most often.

---

## Definition of Success

| Threshold | Meaning |
|-----------|---------|
| ≥ 75% macro F1 | Genuinely useful as a community tool |
| ≥ 70% macro F1 | Acceptable for initial deployment with monitoring |
| < 70% macro F1 | Too unreliable — requires heavy human review |

The `news` label must not cannibalize recall on the other three labels for the deployment threshold to count.

---

## AI Tool Plan

### Label Stress-Testing

Before annotating any examples, I will paste all four label definitions and the hard edge case description into Claude and ask it to generate 8–10 synthetic r/soccer posts that sit at the boundary between two labels — specifically targeting the `reaction`/`hot_take` and `analysis`/`hot_take` overlaps. I will then attempt to label each generated post using only the written definitions. Any post I hesitate on is a signal that the definition has a gap. I will revise the definitions and tiebreaker rules until I can classify every stress-test post confidently before moving on to real data collection.

### Annotation Assistance

I will use Claude to pre-label a first batch of 50 real r/soccer posts before reviewing them myself. Workflow:

1. Paste each post title and body into Claude along with the four label definitions.
2. Ask it to output a single label and a one-sentence reason.
3. Record Claude's label in a spreadsheet column labeled `AI_prelabel`.
4. Review every row independently and enter my final label in a `final_label` column.

I will track the disagreement rate between the AI pre-label and my final label — a high disagreement rate on a specific pair of labels is an early warning that those labels are still underspecified. All pre-labeled examples will be disclosed in the AI usage section of the final submission.

### Failure Analysis

After evaluation, I will collect all misclassified examples — post text, true label, and predicted label — and send them as a batch to Claude with the prompt: *"Here are posts my classifier got wrong. What patterns do you notice in why they were misclassified? Group them by type of error."*

I will look specifically for:
1. Consistent confusion between the same two label pairs
2. Surface features the model may be latching onto (e.g., exclamation marks triggering `reaction` when the post is actually a `hot_take`)
3. Posts that were genuinely mislabeled in my training data

Every pattern Claude identifies will be verified by reading the examples myself — if I cannot find at least 3 posts that fit a claimed pattern, it will not be reported as a real finding.
