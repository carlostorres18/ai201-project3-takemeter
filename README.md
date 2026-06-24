# TakeMeter

AI text classifier for Reddit posts from r/soccer. Assigns each post to one of four intent labels: `analysis`, `hot_take`, `reaction`, or `news`.

---

## Community

I chose r/soccer because I follow the sport closely and the 2026 World Cup is actively happening during this project, making the subreddit especially rich with varied post types. The subreddit is a strong fit for a multi-class classification task because its discourse is genuinely diverse — on any given day you will find breaking transfer news, statistical deep dives, emotional reactions to goals, and contrarian opinions about players or tournament format. That natural variety across intent and tone makes it possible to define distinct, non-overlapping labels rather than splitting on subtle stylistic differences.

---

## Labels

| Label | Definition |
|-------|-----------|
| `analysis` | A post that makes a claim supported by statistics, historical precedent, or structured reasoning. |
| `hot_take` | A post expressing an opinion that goes against the mainstream view held by the r/soccer community, likely to generate strong disagreement. |
| `reaction` | A post that captures an immediate emotional response to something that just happened, prioritizing feeling over reasoning. |
| `news` | A post reporting factual information such as a transfer, injury update, managerial change, or official announcement. |

**Edge case tiebreaker:** When a post blends categories, label by primary intent — structured around evidence → `analysis`; core purpose is to stake an unpopular position → `hot_take`; dominant register is emotional → `reaction`; reporting a fact or announcement → `news`.

The hardest overlap is `reaction` vs `hot_take`: an emotionally phrased opinion can look like both. The tiebreaker is whether the post would exist even without the immediate event (hot take) or only makes sense in the moment it was written (reaction).

---

## Data Collection

Posts were collected directly from r/soccer by browsing Hot, Top (week), and New to capture a range of post styles. The target was 50 examples per label (200 total). Collection was skewed toward the active World Cup period, which naturally surfaced more `reaction` and `news` posts — `analysis` and `hot_take` required deliberate targeted collection using search terms like `stats`, `historically`, `unpopular opinion`, and `change my mind`.

The `text` field in `posts.csv` stores the Reddit post URL. This was a practical shortcut during collection — the URL slug encodes the post title in a machine-readable format — but it turned out to be the most consequential failure in the pipeline (see Evaluation Report).

---

## Model

**Baseline:** Zero-shot classification using a Groq-hosted LLM with a structured system prompt. The prompt defines all four labels with examples, source attribution signals, and the tiebreaker rule. No training data is consumed — the model classifies based on the instruction alone.

**Fine-tuned:** `distilbert-base-uncased` fine-tuned on the labeled `posts.csv` dataset. DistilBERT is a lightweight transformer pre-trained on English web text, making it a reasonable starting point for Reddit post classification. Fine-tuning adapts the model's weights to the specific label vocabulary using the labeled examples as supervision signal.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy | Test Set Size |
|-------|----------|--------------|
| Zero-shot baseline (Groq + system prompt) | **46.7%** | 15 |
| Fine-tuned DistilBERT | **26.7%** | 15 |

The fine-tuned model performed **worse** than the zero-shot baseline by 20 percentage points — a regression, not an improvement. This outcome is explained in detail below.

---

### Per-Class Metrics — Baseline (Zero-Shot)

| Label | Precision | Recall | F1-Score | Support |
|-------|-----------|--------|----------|---------|
| analysis | 0.50 | 0.40 | 0.44 | 5 |
| hot_take | 0.00 | 0.00 | 0.00 | 3 |
| reaction | 1.00 | 0.50 | 0.67 | 4 |
| news | 0.33 | 1.00 | 0.50 | 3 |
| **macro avg** | **0.46** | **0.47** | **0.40** | 15 |

The baseline defaulted toward `news` when uncertain — it caught every real news post (recall 1.00) but also mislabeled other posts as news (precision 0.33). `hot_take` was a complete failure (F1 0.00), which makes sense because identifying a contrarian community opinion requires cultural context a zero-shot prompt alone cannot reliably supply.

### Per-Class Metrics — Fine-Tuned DistilBERT

| Label | Precision | Recall | F1-Score | Support |
|-------|-----------|--------|----------|---------|
| analysis | 0.00 | 0.00 | 0.00 | 5 |
| hot_take | 0.00 | 0.00 | 0.00 | 3 |
| reaction | 0.27 | 1.00 | 0.42 | 4 |
| news | 0.00 | 0.00 | 0.00 | 3 |
| **macro avg** | **0.07** | **0.25** | **0.11** | 15 |

---

### Confusion Matrix — Fine-Tuned Model (Test Set)

|  | Pred: analysis | Pred: hot_take | Pred: reaction | Pred: news |
|--|:-:|:-:|:-:|:-:|
| **True: analysis** | 0 | 0 | 5 | 0 |
| **True: hot_take** | 0 | 0 | 3 | 0 |
| **True: reaction** | 0 | 0 | 4 | 0 |
| **True: news** | 0 | 0 | 3 | 0 |

Every single prediction is `reaction`. The fine-tuned model collapsed to a single-class output — it predicted `reaction` for all 15 test examples regardless of true label. The 26.7% accuracy reflects exactly the 4 posts that were genuinely `reaction` and happened to be correctly labeled by a model that never made a real judgment.

---

### Sample Classifications — Fine-Tuned Model

All five predictions below are `reaction` with confidence 0.27. The uniform output and near-random confidence confirm the model did not learn any meaningful decision boundary. The confidence of 0.27 ≈ 1/4 is indistinguishable from chance on a 4-class problem.

| Post Title (from URL slug) | True Label | Predicted | Confidence | Correct? |
|---|---|---|---|:-:|
| Norway players and fans doing Viking Row | `reaction` | `reaction` | 0.27 | ✓ |
| OptaStats: Belgium had 23 shots vs IR Iran | `analysis` | `reaction` | 0.27 | ✗ |
| Xavi: he is the Michael Jordan of football in [his era] | `hot_take` | `reaction` | 0.27 | ✗ |
| Sky Sports: Real Madrid will activate their [option] | `news` | `reaction` | 0.27 | ✗ |
| Lionel Messi becomes the all-time leading [scorer] | `analysis` | `reaction` | 0.27 | ✗ |

**On the correctly predicted example:** "Norway players and fans doing Viking Row" is a reasonable `reaction` label — it describes a celebratory crowd ritual happening in the moment after a match result, with no statistical claim or factual announcement, and no contrarian argument. Even though the model predicted this by defaulting to `reaction` for everything rather than by making a real judgment, the prediction is defensible: the post is about collective emotional expression tied to an immediate event.

---

### Wrong Examples — Analysis

All three examples below were predicted as `reaction` with identical confidence (0.27). The shared confidence score signals the model had no meaningful information — it was not making a judgment call, it was defaulting.

---

**Example 1**
- **Post:** `OptaStats: Belgium had 23 shots vs IR Iran`
- **True label:** `analysis`
- **Predicted:** `reaction` (confidence: 0.27)

*Which labels are being confused?* `analysis → reaction`. This is the most frequent error pair — two of the three analyzed examples are this exact swap.

*Why is the boundary hard?* The post cites a specific statistic from an external analytics source (OptaStats), which should be a clear `analysis` signal. But from a URL slug alone, "Belgium had 23 shots" reads like an excited fan observing a match stat in real time — the phrasing is short and tied to a live game. The model cannot distinguish "analyst citing a figure" from "fan exclaiming a figure" without recognizing OptaStats as a data source or seeing the post body.

*Labeling problem or data problem?* Data problem. The model's input is a URL slug, not post text. The slug strips source attribution, surrounding discussion, and framing — exactly the signals that distinguish `analysis` from `reaction`.

*What would need to change?* Fetch and use the actual post title and body as the `text` field. URL slugs are not a viable input format for this task.

---

**Example 2**
- **Post:** `Do the new World Cup tiebreaker rules make the [game worse]?`
- **True label:** `hot_take`
- **Predicted:** `reaction` (confidence: 0.27)

*Which labels are being confused?* `hot_take → reaction`. This is the hardest boundary in the label set, flagged as a known edge case during planning.

*Why is the boundary hard?* This post is a contrarian opinion framed as a rhetorical question — it implies "yes, these rules are bad" without asserting it. The model likely saw no `hot_take` examples phrased interrogatively during training, so it defaulted to `reaction`. The slug is also truncated, leaving the model with an incomplete sentence.

*Labeling problem or data problem?* Both. The training data lacked rhetorical-question hot takes. The truncated slug compounded the problem by removing the last few words that would clarify intent.

*What would need to change?* Add training examples of hot takes phrased as provocative questions. Update the `hot_take` definition to explicitly cover this framing. Fix the input pipeline to use full post text.

---

**Example 3**
- **Post:** `ESPN FC: History for Curaçao as they earn their [first World Cup win]`
- **True label:** `analysis`
- **Predicted:** `reaction` (confidence: 0.27)

*Which labels are being confused?* `analysis → reaction`, same as Example 1. This pair dominates the error set.

*Why is the boundary hard?* "History" and "earn their first..." carry a strong celebratory framing that reads like a fan reacting to a milestone. The ESPN FC source attribution should push toward `analysis` or `news`, but source names carry no weight in a URL slug. This post was labeled `analysis` rather than `news` because the body discusses historical precedent rather than just reporting the result — a distinction invisible from the slug.

*Labeling problem or data problem?* This also exposes a secondary boundary risk between `analysis` and `news`. Historical milestone posts ("X is the first Y to do Z") sit at that boundary depending on whether the body adds context or just states the fact. Annotation inconsistency here would directly confuse training.

*What would need to change?* Use full post text. Audit historical-milestone posts in the training set for label consistency. If the `analysis`/`news` distinction is ambiguous for this post type, add it as an explicit example in both label definitions.

---

### Error Patterns

**Pattern 1 — The fine-tuned model collapsed to a single label.**
Every prediction is `reaction`. This is not confusion between similar labels — it is a complete failure to learn any boundary. The macro F1 of 0.11 reflects this: only `reaction` has nonzero scores because it is the only class ever predicted.

**Pattern 2 — `reaction` is both models' uncertainty default.**
The baseline defaulted to `news` when uncertain (recall 1.00, precision 0.33). The fine-tuned model defaulted to `reaction` for everything. The catch-all label switched between models because fine-tuning shifted the model's prior without teaching it real distinctions. In both cases, the catch-all inflates that label's recall while destroying every other class's recall.

**Pattern 3 — `analysis → reaction` is the dominant confusion pair.**
Two of three analyzed wrong examples are this swap. Stat posts and historical milestone headlines are short and punchy in a way that sounds reactive. Without the post body, there is no signal to separate "analyst citing evidence" from "fan exclaiming a surprising fact."

**Pattern 4 — `hot_take → reaction` is the hardest boundary.**
Contrarian opinions phrased as questions or emotional complaints look identical to reactions at the surface level. Both models failed on this pair. The boundary is real in the definitions but the training data did not contain enough examples of the hard cases — especially hot takes that sound emotionally charged.

**Pattern 5 — All errors share identical low confidence (0.27).**
A confidence of 0.27 on a 4-class problem is indistinguishable from random chance (0.25). The model was producing a near-flat output distribution and picking the argmax. This means the input text carried essentially zero signal for the model.

**Pattern 6 — The input format is not viable.**
URL slugs are lowercase, underscore-joined, percent-encoded fragments of a post title, often truncated. Feeding them to a text classifier is equivalent to feeding a compressed and partially corrupted version of the data. Fixing the input pipeline to fetch actual post titles and body text is the highest-leverage improvement available — more impactful than any label or architecture change.

---

### What the Model Captured vs. What Was Intended

This section is distinct from the wrong examples above — it addresses the gap between the label definitions and the model's actual decision boundary.

**What was intended:** The model was supposed to learn intent. An `analysis` post exists to make an argument; a `reaction` post exists to express a feeling; a `news` post exists to inform; a `hot_take` exists to provoke disagreement. These are fundamentally about *why* the person posted, not *what words* they used.

**What the model actually captured:** Nothing — the model collapsed to a degenerate solution and did not learn any feature boundary. But the direction of the collapse is informative. It converged on `reaction` rather than another label, which suggests `reaction` posts were the most consistent with the informal, short-form text patterns DistilBERT already associates with web text from pretraining. The model overfit to the implicit assumption that "short informal text = reaction," which is exactly what URL slugs look like regardless of true label.

**The core gap:** The label definitions are intent-based and require reading the full post in context. The model received URL slugs — lowercase, underscored, truncated fragments that strip every cue that makes intent classifiable (source attribution, sentence structure, register, emotional markers, argument chains). The model was asked to learn something the input data made structurally impossible to learn.

**What this means going forward:** Even with more training data, fine-tuning on URL slugs would not produce a useful classifier. The fix is to change the input, not the model. With actual post text, the `analysis`/`reaction` boundary becomes much clearer (stat tables vs. exclamation marks), and `news` posts become identifiable from source-name patterns (Fabrizio Romano, Sky Sports, BBC Sport). The hardest remaining boundary — `hot_take` vs `reaction` — would still require community-cultural context that DistilBERT was not pre-trained on, and would benefit from more training examples and possibly a larger model.

---

## Baseline vs. Fine-Tuned Summary

| | Baseline (Zero-Shot) | Fine-Tuned DistilBERT |
|--|--|--|
| Accuracy | 46.7% | 26.7% |
| Macro F1 | 0.40 | 0.11 |
| `hot_take` F1 | 0.00 | 0.00 |
| `reaction` recall | 0.50 | 1.00 (catch-all) |
| Default label | `news` | `reaction` |
| Learns boundaries? | Partially | No |

The zero-shot baseline outperforms fine-tuning. The most likely reasons: (1) the training set is too small for DistilBERT to generalize on a 4-class problem, (2) URL slugs stripped the features the model needed to learn, and (3) `hot_take` had too few training examples for a label that requires community-cultural context to identify.

---

## Spec Reflection

**One way the spec helped:** The planning document required writing explicit tiebreaker rules before collecting any data. That constraint turned out to be genuinely useful during annotation. Without the rule "primary intent determines the label," posts like "I can't believe these tiebreaker rules are real" would have been ambiguous between `reaction` (the tone is outrage) and `hot_take` (the content implies a contrarian position on FIFA's rule design). Having the tiebreaker written down in advance meant I applied the same logic consistently across similar edge cases, rather than deciding case by case.

**One way the implementation diverged:** The spec set a target of 200 labeled examples (50 per label) and defined success as ≥ 70% macro F1. The implementation ran with roughly 100 examples total and produced 26.7% accuracy on the fine-tuned model. The divergence was not just about scale — the more significant departure was storing Reddit URLs as the `text` field instead of fetching actual post text. The planning document described classifying "Reddit posts," implicitly meaning post content. The implementation shortcut of saving URLs instead of text was never flagged as a risk during planning, and it turned out to be the single largest cause of model failure. In retrospect, the data collection plan should have included a step to fetch and store the post title and body at collection time.

---

## AI Usage

**Instance 1 — System prompt design**

I described the community (r/soccer), the four label definitions, and the tiebreaker rules to Claude and asked it to rewrite a generic classification prompt template to match the project. Claude produced a complete prompt with the label definitions, two examples per label drawn directly from the planning document, and the tiebreaker rule formatted as a decision guide. I kept the overall structure but changed the example posts to match real entries from the planning document rather than Claude's generated ones, and I moved the tiebreaker rule to appear between the examples and the valid-labels block so it reads as context before the instruction rather than as an afterthought. The final system prompt used in the baseline evaluation reflects those edits.

**Instance 2 — Pre-labeling annotation assistance**

Before finalizing labels, I pasted batches of Reddit URL slugs along with the four label definitions into Claude and asked it to return a single label and a one-sentence reason for each post. I recorded Claude's output as an `AI_prelabel` column and then reviewed every row independently before entering a `final_label`. The disagreement rate was approximately 20%, concentrated on two pairs: `analysis` vs `news` for source-attributed stat posts, and `hot_take` vs `reaction` for emotionally phrased contrarian opinions. Every case where I disagreed with Claude's pre-label was used as a signal to look more carefully at whether that post type was ambiguous in the definitions. All pre-labeled examples are included in the final training data with my reviewed labels, not Claude's.

**Instance 3 — Error pattern analysis**

After the fine-tuned model evaluation, I pasted the three wrong examples into Claude and asked it to identify patterns in why the model failed. Claude identified that all three had identical confidence scores (catch-all signal), that `analysis → reaction` was the dominant confusion pair, and that URL slugs were a structural problem for the input pipeline. I verified each pattern by checking it against all 15 test examples before including it in this report. The URL-slug diagnosis matched what I observed in the confusion matrix — uniform prediction regardless of true label — and I verified it held for all 15 cases, not just the three analyzed examples.
