# TakeMeter

A fine-tuned text classifier that evaluates discourse quality in TV/film fan communities. Given a post or comment from a subreddit like r/television or r/TrueFilm, TakeMeter predicts whether it's a `reaction`, `hot_take`, or `analysis`.

---

## Community

**r/television, r/TrueFilm, and show-specific subreddits** (r/TheLastOfUs, r/Succession, r/BreakingBad).

TV/film fan spaces are a strong fit for this task because discourse varies enormously in quality — from immediate emotional outbursts to carefully reasoned craft analysis — and the community itself has strong norms around what constitutes a substantive post. The distinctions I'm measuring (`reaction` vs. `hot_take` vs. `analysis`) map to how regulars in these communities actually evaluate each other's contributions.

---

## Label Taxonomy

| Label | Definition |
|---|---|
| `reaction` | An immediate emotional response to a specific episode, scene, or release. Little to no argument — the post expresses a feeling in the moment. |
| `hot_take` | A bold, confident opinion stated without supporting reasoning. Asserts a position but doesn't build a case for it. |
| `analysis` | Makes a structured argument backed by specific evidence — craft observations, comparisons, thematic readings. Evidence is specific and reasoning is legible. |

**Example posts per label:**

*reaction:* "Just finished the finale and I'm not okay. That ending destroyed me."

*hot_take:* "Season 3 is the best season and I will not be taking questions."

*analysis:* "The color palette in episode 6 mirrors the pilot almost shot-for-shot — blue-green tones when she feels safe, harsh ambers when she's losing control. It's not subtle but it works because they're consistent."

---

## Data Collection

**Source:** Posts and top-level comments collected manually from r/television, r/TrueFilm, r/TheLastOfUs, r/Succession, and r/BreakingBad. All posts are public.

**Labeling process:** Each post was read in full and labeled according to the definitions in planning.md. Posts under 20 words, pure questions, and non-commentary posts (links, image posts without text) were excluded. A running list of difficult cases was maintained during annotation.

**Label distribution:**

| Label | Count |
|---|---|
| `reaction` | ___ |
| `hot_take` | ___ |
| `analysis` | ___ |
| **Total** | **___** |

**Three difficult-to-label examples:**

1. **"The pacing in act 2 is terrible — they wasted 20 minutes on the subplot."**  
   Could be `hot_take` or `analysis` — it cites a specific observation but doesn't build an argument. Labeled `hot_take`: a single decorative fact doesn't make analysis; the opinion came first.

2. **"That finale WRECKED me — the way they brought back the season 1 motif in the last scene was such a perfect callback."**  
   Could be `reaction` or `analysis` — it names a craft device but is primarily emotional. Labeled `reaction`: the reasoning explains the feeling, not the other way around.

3. **"Hot take: [followed by 3 paragraphs of craft observations with specific citations]"**  
   Framed as a hot take but functions as analysis. Labeled `analysis`: I label by what the post does, not how it's framed.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

**Training setup:**
- Framework: HuggingFace `transformers` + `datasets`
- Platform: Google Colab (T4 GPU)
- Split: 70% train / 15% validation / 15% test
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16

**Key hyperparameter decision:** [describe one decision you made here — e.g. "I kept the default 3 epochs rather than increasing to 5 because the validation loss was already plateauing after epoch 2, suggesting additional epochs would overfit on 200 examples."]

---

## Baseline

**Model:** Groq `llama-3.3-70b-versatile`, zero-shot

**Prompt used:**

```
You are classifying posts from TV/film fan communities. Assign exactly one label to the post below.

Labels:
- reaction: An immediate emotional response to a specific episode, scene, or release. Little to no argument — expressing a feeling in the moment.
- hot_take: A bold, confident opinion stated without supporting reasoning. Asserts a position but doesn't build a case.
- analysis: Makes a structured argument backed by specific evidence — craft observations, comparisons, thematic readings.

Respond with ONLY the label name (reaction, hot_take, or analysis). No punctuation, no explanation.

Post: {post_text}
```

**How results were collected:** The prompt was run on every example in the test set via the Groq API in Section 5 of the Colab notebook. Responses that didn't exactly match a label name were logged as unparseable and treated as incorrect.

---

## Evaluation Report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq) | ___ |
| Fine-tuned DistilBERT | ___ |

### Per-class metrics (fine-tuned model)

| Label | Precision | Recall | F1 |
|---|---|---|---|
| `reaction` | ___ | ___ | ___ |
| `hot_take` | ___ | ___ | ___ |
| `analysis` | ___ | ___ | ___ |
| **Macro avg** | ___ | ___ | ___ |

### Per-class metrics (zero-shot baseline)

| Label | Precision | Recall | F1 |
|---|---|---|---|
| `reaction` | ___ | ___ | ___ |
| `hot_take` | ___ | ___ | ___ |
| `analysis` | ___ | ___ | ___ |
| **Macro avg** | ___ | ___ | ___ |

### Confusion matrix (fine-tuned model)

Rows = true label, columns = predicted label.

|  | predicted: reaction | predicted: hot_take | predicted: analysis |
|---|---|---|---|
| **true: reaction** | ___ | ___ | ___ |
| **true: hot_take** | ___ | ___ | ___ |
| **true: analysis** | ___ | ___ | ___ |

### Wrong predictions — analysis

**Example 1:**
> [paste the post text here]

True label: `___` | Predicted: `___` | Confidence: ___%

Analysis: [explain why the model got this wrong — which labels are being confused, why that boundary is hard, whether this is a labeling inconsistency or a data distribution problem]

**Example 2:**
> [paste the post text here]

True label: `___` | Predicted: `___` | Confidence: ___%

Analysis: [explain]

**Example 3:**
> [paste the post text here]

True label: `___` | Predicted: `___` | Confidence: ___%

Analysis: [explain]

### Sample classifications

| Post (truncated) | True label | Predicted | Confidence |
|---|---|---|---|
| [post 1] | ___ | ___ | ___% |
| [post 2] | ___ | ___ | ___% |
| [post 3] | ___ | ___ | ___% |
| [post 4] | ___ | ___ | ___% |
| [post 5] | ___ | ___ | ___% |

**Correct prediction explained:** [pick one correct prediction and explain in a sentence why the model's prediction is reasonable given the post content]

---

## Reflection: What the Model Learned vs. What I Intended

[Write this after evaluating. Example structure to fill in:]

My label definitions target three distinct modes of discourse. In practice, the model appears to have learned ___. The boundary it handles best is ___ vs. ___, likely because ___. The boundary it struggles with most is ___ vs. ___, and I think this is because ___.

One thing the model seems to have overfit to is ___. For example, posts containing words like ___ tend to get predicted as ___ even when the actual content is different. This suggests the model is partly using surface vocabulary as a proxy for the discourse mode I intended.

What I'd change: ___.

---

## Spec Reflection

**One way the spec helped:** The requirement to define edge case decision rules in planning.md before annotating forced me to confront the hardest classification cases before I'd committed to labeling 200 examples. The "hot take with one supporting fact" rule saved me from annotating inconsistently across that boundary.

**One way implementation diverged from the spec:** [describe your divergence — e.g. "The spec assumes a roughly equal label distribution, but my natural collection skewed toward reaction. I compensated by deliberately seeking analysis posts from r/TrueFilm, which may have introduced a slight domain shift in that class — r/TrueFilm posts use more formal critical vocabulary than typical r/television comments."]

---

## AI Usage

**Instance 1 — Label stress-testing:**
I gave Claude my three label definitions and asked it to generate 15 posts that sit at the boundary between two labels. It produced several posts in the "hot take with evidence" gray zone that I couldn't cleanly classify. This revealed that my original `analysis` definition was too loose — I revised it to require that evidence be "specific and verifiable" rather than just "present." I discarded all generated posts and used only real collected data for training.

**Instance 2 — Failure analysis:**
After evaluating the fine-tuned model, I pasted all misclassified test examples into Claude and asked it to identify patterns. It suggested that many errors involved short posts (under 40 words) where the signal was ambiguous. I verified this by filtering my wrong predictions by length and confirmed that ___ of my ___ errors came from posts under 40 words. I added this finding to my error analysis above.

[Add a third instance if you used annotation assistance — disclose exactly which examples were pre-labeled and how you reviewed them.]
