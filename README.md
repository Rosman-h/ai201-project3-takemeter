# TakeMeter

A fine-tuned text classifier that evaluates discourse quality in TV/film fan communities. Given a post or comment from a subreddit like r/breakingbad or r/HouseOfTheDragon, TakeMeter predicts whether it's a `reaction`, `hot_take`, or `analysis`.

---

## Community

**r/breakingbad, r/HouseOfTheDragon, and related threads** (Ozymandias episode discussion, Blood and Cheese megathread, Criston Cole discussion, Hank POV thread, Skyler discussion threads).

TV/film fan spaces are a strong fit for this task because discourse varies enormously in quality — from immediate emotional outbursts to carefully reasoned craft analysis — and the community itself has strong norms around what constitutes a substantive post. The distinctions I'm measuring (`reaction` vs. `hot_take` vs. `analysis`) map to how regulars in these communities actually evaluate each other's contributions.

---

## Label Taxonomy

| Label | Definition |
|---|---|
| `reaction` | An immediate emotional response to a specific episode, scene, or release. Little to no argument — the post expresses a feeling in the moment. |
| `hot_take` | A bold, confident opinion stated without supporting reasoning. Asserts a position but doesn't build a case for it. |
| `analysis` | Makes a structured argument backed by specific evidence — craft observations, comparisons, thematic readings. Evidence is specific and reasoning is legible. |

**Example posts per label:**

*reaction:* "I went from hating him to feeling so sorry for him in just an episode. Bravo Vince."

*hot_take:* "Season 3 is the best season and I will not be taking questions."

*analysis:* "The scene is indicative of a weird choice for the overall writing direction. It seems like Condal is making too many things in the Dance come from miscommunication and happenstance. But that softens our feelings toward the purposeful actions of characters. GOT made us hate iconic characters because they revealed the underbelly of human nature — our own cruelty, malice, greed."

---

## Data Collection

**Source:** Posts and top-level comments collected from r/breakingbad (Felina finale discussion, Ozymandias episode discussion, Skyler unpopular opinion thread, Hank POV thread), r/HouseOfTheDragon (Blood and Cheese megathread, Criston Cole discussion, season 2 criticism thread, episode 2 discussion), and r/HouseOfTheDragon season 2 general threads. All posts are public.

**Labeling process:** Each post was read in full and labeled according to the definitions above. Posts under 15 words, pure questions, and replies-within-replies (which lose context) were excluded. A running notes column tracked edge cases during annotation.

**Label distribution:**

| Label | Count |
|---|---|
| `reaction` | 71 |
| `hot_take` | 61 |
| `analysis` | 70 |
| **Total** | **202** |

**Three difficult-to-label examples:**

1. **"I don't think she went there for peace — it was so he could get credit for making the blue meth all the way to death since he was found dead at a meth lab. I'm blown away."**
   Could be `reaction` (ends with emotional exclamation, live-watch register) or `analysis` (offers a theory about Walt's motivation with specific evidence). Labeled `reaction` because the primary register is excited discovery in the moment — the theory is brief and the emotional "I'm blown away" frames it as a feeling, not an argument.

2. **"Hank is simultaneously the best and worst cop ever. Piecing everything together except the Nobel prize-winning chemistry brother-in-law that suddenly received big bucks counting cards with the initials W.W."**
   Could be `hot_take` (bold ironic verdict) or `analysis` (points to specific plot details). Labeled `hot_take` because the purpose is to deliver a punchline about Hank's blind spot — the details are used for comic effect, not to build an argument.

3. **"Todd is like the type of guy that would make sure you feel comfortable before killing you."**
   Could be `reaction` (vivid immediate impression) or `analysis` (identifies a specific character trait and explains what makes it distinctive). Labeled `analysis` because it makes a claim about what kind of person Todd is and implicitly explains why that quality is unsettling — it's a character reading, not just an emotional response.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

**Training setup:**
- Framework: HuggingFace `transformers` + `datasets`
- Platform: Google Colab (T4 GPU)
- Split: 70% train / 15% validation / 15% test (stratified)
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16

**Key hyperparameter decision:** I kept the default 3 epochs rather than increasing them because with only ~141 training examples, additional epochs would likely increase overfitting rather than improve generalization. The validation loss was the primary signal — with this dataset size, more epochs would memorize training noise rather than learn the underlying distinctions.

---

## Baseline

**Model:** Groq `llama-3.3-70b-versatile`, zero-shot

**Prompt used:**

```
You are classifying posts from TV/film fan communities. Assign exactly one label.

Labels:
- reaction: An immediate emotional response to a specific episode, scene, or release. Little to no argument — expressing a feeling in the moment.
- hot_take: A bold, confident opinion stated without supporting reasoning. Asserts a position but doesn't build a case.
- analysis: Makes a structured argument backed by specific evidence — craft observations, comparisons, thematic readings.

Decision rules:
- A post that cites one fact to support a bold opinion is hot_take, not analysis.
- A post that names a craft device while primarily expressing emotion is reaction, not analysis.
- Label by what the post does, not how it's framed.

Respond with ONLY the label name: reaction, hot_take, or analysis
No punctuation. No explanation. No other words.

Post: {post_text}
```

**How results were collected:** The prompt was run on every example in the test set via the Groq API in Section 5 of the Colab notebook. All 31 test examples returned parseable responses.

---

## Evaluation Report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | 0.806 |
| Fine-tuned DistilBERT | 0.516 |

The fine-tuned model performed significantly worse than the zero-shot baseline — 52% vs 81%. This is the key finding of this project and is explained in detail in the reflection section below.

### Per-class metrics — fine-tuned model

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.50 | 0.82 | 0.62 | 11 |
| `hot_take` | 0.50 | 0.44 | 0.47 | 9 |
| `reaction` | 0.60 | 0.27 | 0.38 | 11 |
| **Macro avg** | **0.53** | **0.51** | **0.49** | **31** |

### Per-class metrics — zero-shot baseline

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 1.00 | 0.82 | 0.90 | 11 |
| `hot_take` | 0.64 | 0.78 | 0.70 | 9 |
| `reaction` | 0.82 | 0.82 | 0.82 | 11 |
| **Macro avg** | **0.82** | **0.80** | **0.81** | **31** |

### Confusion matrix — fine-tuned model

Rows = true label, columns = predicted label.

|  | predicted: analysis | predicted: hot_take | predicted: reaction |
|---|---|---|---|
| **true: analysis** | 9 | 1 | 1 |
| **true: hot_take** | 3 | 4 | 2 |
| **true: reaction** | 5 | 3 | 3 |

The model over-predicts `analysis` — it predicted analysis for 17 of 31 test examples (55%), including 5 that were actually `reaction` and 3 that were actually `hot_take`. The `reaction` class has the worst recall (0.27), meaning the model missed most real reaction posts and called them something else.

### Wrong predictions — analysis

**Example 1:**
> "I can't watch the after episodes anymore because they infuriate me."

True label: `reaction` | Predicted: `hot_take` | Confidence: 0.35

This post is a pure emotional reaction — frustration expressed in the moment. The model predicted `hot_take`, likely because the post makes a negative claim about the show ("they infuriate me") which superficially resembles a critical opinion. The confusion reveals that the model is picking up on negative sentiment as a proxy for `hot_take` rather than reading the emotional register. The near-random confidence (0.35) confirms the model has no real signal here — it's essentially guessing.

**Example 2:**
> "Todd is like the type of guy that would make sure you feel comfortable before killing you."

True label: `analysis` | Predicted: `reaction` | Confidence: 0.34

This is one of the most interesting misclassifications. The post makes a genuine character observation — it identifies a specific quality (false politeness preceding violence) and explains implicitly why it's unsettling. That's analysis. But the model predicted `reaction`, probably because the post is short, uses casual language ("like the type of guy"), and reads like a live observation. The model appears to be using length and formality as proxies for `analysis`, which fails on short but substantive posts.

**Example 3:**
> "Hank is simultaneously the best and worst cop ever. Piecing everything together except the Nobel prize-winning chemistry brother-in-law that suddenly received big bucks counting cards with the initials W.W."

True label: `hot_take` | Predicted: `analysis` | Confidence: 0.36

This post is a `hot_take` delivered through ironic enumeration of plot details — it's a punchline about Hank's blind spot, not a genuine argument. The model predicted `analysis` because the post references multiple specific plot details (initials W.W., counting cards, the brother-in-law connection), which look like evidence-based reasoning on the surface. The model can't distinguish between details used to build an argument and details used for comic rhetorical effect. This is the hardest boundary in the taxonomy and the one the model handles worst.

### Sample classifications

| Post (truncated to 120 chars) | True label | Predicted | Confidence |
|---|---|---|---|
| "I just can't get over how Alicent and Cole were going at it with the door unlocked." | reaction | reaction | 0.41 |
| "The big problem is that we don't spend enough time with Helena to understand her so her reactions..." | analysis | analysis | 0.52 |
| "Better Call Saul is a better show. Largely due to the character development." | hot_take | hot_take | 0.44 |
| "This is the fucking darkest episode of this show to date." | reaction | reaction | 0.48 |
| "Once you get a reputation for breaking the rules that becomes your norm. The GoT tropes are worn out." | hot_take | analysis | 0.34 |

**Correct prediction explained:** The model correctly predicted `analysis` for the Helaena characterization post with 0.52 confidence — the highest confidence in this sample. This makes sense: the post is long, references specific character details, and uses reasoning language ("because it adds," "I like that"). The model has learned that length plus explanatory language signals `analysis`, and this post has both clearly.

---

## Reflection: What the Model Learned vs. What I Intended

My label definitions target three distinct modes of discourse. In practice, the model appears to have learned a weak surface-level proxy: length and explanatory language signal `analysis`, negative evaluative claims signal `hot_take`, and everything else defaults to `analysis` when uncertain.

The boundary the model handles best is recognizing clear `analysis` — posts with multiple sentences, explicit reasoning words ("because," "this is why"), and specific named evidence. Its recall on `analysis` is 0.82, meaning it catches most real analysis posts. But its precision is only 0.50, meaning half its `analysis` predictions are wrong — it's over-applying the label to anything that looks substantive.

The boundary it struggles with most is `reaction` vs. everything else. Recall for `reaction` is only 0.27 — the model missed 8 of 11 real reaction posts, labeling them as `analysis` or `hot_take`. This makes sense: many reaction posts contain specific observations ("that scene where Jesse..."), which look like evidence on the surface. The model can't read emotional register, only token patterns.

The most fundamental problem is that the distinctions I defined depend on pragmatic intent — *why* someone is writing, not *what words* they use. A post that says "Walt died where he did the most of his living" could be a reaction (emotional appreciation in the moment) or analysis (thematic observation about the character arc). DistilBERT fine-tuned on 141 examples cannot learn this. The Groq baseline succeeds because a 70B parameter model has learned enough about human communication to read pragmatic intent from context — something a small model trained on tiny data simply cannot do.

What I would change: collect at least 500 examples before fine-tuning, ensure reaction posts are clearly distinguished from brief analytical observations in the annotation guidelines, and consider adding a `notes` review pass specifically for posts under 50 words where the signal is weakest.

---

## Spec Reflection

**One way the spec helped:** The requirement to write edge case decision rules in planning.md before annotating forced me to confront the hardest classification cases before committing to 200 examples. The rules about "hot take with one supporting fact" and "emotional reaction that names a craft device" directly shaped how I labeled ambiguous posts — without them I would have been inconsistent across the dataset, which would have made the training signal even noisier.

**One way implementation diverged from the spec:** The spec assumes fine-tuning will improve on the baseline, and frames the comparison as confirming how much fine-tuning helped. In this case, fine-tuning made things significantly worse. This outcome is actually more instructive than a clean win — it demonstrates that 200 examples is genuinely insufficient for a task where the label boundaries depend on pragmatic intent rather than vocabulary. The spec hints at this possibility ("if your fine-tuned model performs worse than the baseline, that's a signal worth investigating") but frames it as an edge case. For subjective discourse classification tasks, it may be the norm.

---

## AI Usage

**Instance 1 — Label taxonomy and stress-testing:**
I used Claude to generate the full label taxonomy, edge case decision rules, and planning.md before collecting any data. Claude proposed the three-label system (reaction / hot_take / analysis), wrote definitions for each, and generated specific boundary examples to stress-test the taxonomy. I reviewed and approved the definitions but did not independently derive them — they came from Claude's initial proposal, which I then applied during annotation. The decision rules (particularly "label by what the post does, not how it's framed") shaped every ambiguous annotation decision.

**Instance 2 — Dataset construction:**
I provided Claude with raw Reddit thread text (pasted directly from r/breakingbad, r/HouseOfTheDragon episode discussion threads, and the Blood and Cheese megathread). Claude extracted individual posts, assigned labels, and wrote notes explaining edge case decisions. I reviewed the labeled outputs but did not re-annotate from scratch — Claude did the primary labeling and I verified. This is a significant disclosure: the training data was labeled primarily by an AI tool, not by me independently. Any systematic errors in Claude's labeling judgment would propagate directly into the training signal.

**Instance 3 — Failure analysis:**
After evaluating the fine-tuned model, I worked with Claude to analyze the 15 wrong predictions. Claude identified three patterns: (1) the model uses length and formality as proxies for `analysis`, failing on short substantive posts; (2) the model reads negative evaluative sentiment as `hot_take` regardless of emotional register; (3) the model cannot distinguish details used for comic rhetorical effect from details used as genuine evidence. I verified these patterns by reviewing the wrong predictions myself and confirmed all three are present in the data.
