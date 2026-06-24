# TakeMeter — Planning Document

## Community

I chose the TV/film fan community, drawing primarily from r/television, r/TrueFilm, and show-specific subreddits (e.g. r/TheLastOfUs, r/Succession, r/BreakingBad). This community is an excellent fit for a classification task because:

- Discourse varies enormously in quality, from immediate emotional outbursts to carefully reasoned critical analysis
- The community itself has norms around what constitutes a "good" post — regulars distinguish between hot takes and actual arguments
- Text is the primary medium (minimal images/memes), which keeps the classification signal in the text itself
- Posts are long enough to classify reliably (most are 30–300 words) but short enough to annotate efficiently

## Label Taxonomy

### `reaction`
**Definition:** An immediate emotional response to a specific episode, scene, film, or release. Little to no argument — the post expresses a feeling in the moment without building a case.

**Examples:**
1. "Just finished the finale and I'm not okay. That ending destroyed me."
2. "The jump scare in episode 4 made me physically throw my phone. I screamed."

**Key signals:** emotional language, present tense, no argument structure, references to a specific moment or scene

---

### `hot_take`
**Definition:** A bold, confident opinion stated without supporting reasoning. The post asserts a position — often contrarian or provocative — but doesn't build a case for it.

**Examples:**
1. "Season 3 is the best season and I will not be taking questions."
2. "Unpopular opinion: the theatrical cut is objectively better than the director's cut. The extra 47 minutes add nothing."

**Key signals:** assertive framing ("unpopular opinion", "hot take", "controversial but"), no evidence, often contrarian, confident declarative statements

---

### `analysis`
**Definition:** Makes a structured argument backed by specific evidence — craft observations (cinematography, writing, pacing, score), comparisons to other works, thematic readings, or historical context. Evidence is specific and the reasoning is legible.

**Examples:**
1. "The color palette in episode 6 mirrors the pilot almost shot-for-shot — blue-green tones when she feels safe, harsh ambers when she's losing control. It's not subtle but it works because they're consistent."
2. "This follows the same structural problem as the second season of Succession — once the writers had to resolve the central power dynamic, they didn't know what to replace it with. The tension deflates in act 2 every time."

**Key signals:** specific evidence, craft vocabulary, comparisons to other works, "because" / "notice how" / "this is why", structured multi-sentence reasoning

---

## Hard Edge Cases

### Edge case 1: Hot take with one supporting fact
**Example post:** "The pacing in act 2 is terrible — they wasted 20 minutes on the subplot."

This sits between `hot_take` and `analysis` because it references a specific observation (20 wasted minutes) but doesn't build a real argument.

**Decision rule:** A single observation used to justify a pre-formed opinion is not analysis. Ask: if you removed the opinion framing, would what remains be a standalone argument? If not — label `hot_take`. Analysis builds a case; hot takes cite a fact to sound credible.

### Edge case 2: Emotional reaction with some reasoning
**Example post:** "That finale WRECKED me — the way they brought back the season 1 motif in the last scene was such a perfect callback."

This sits between `reaction` and `analysis` because it names a specific craft device (callback motif) while being primarily emotional.

**Decision rule:** If the primary register is emotional and the reasoning explains *why* they feel something (rather than making an independent argument), label it `reaction`. Analysis is the primary intent; emotion can be present but it's not the driver.

### Edge case 3: "Hot take:" framing with actual analysis in the body
**Example post:** "Hot take: [3 paragraphs of craft observations with specific evidence]"

**Decision rule:** Label by what the post *does*, not how it's framed. If the body contains specific evidence and structured reasoning, it's `analysis` — the "hot take" framing is rhetorical positioning, not the actual mode of discourse.

---

## Data Collection Plan

**Sources:**
- r/television — high volume, varied discourse quality, many different shows
- r/TrueFilm — skews toward analysis; useful for getting enough `analysis` examples
- Show-specific subreddits (r/TheLastOfUs, r/Succession, r/BreakingBad) — more emotional reactions and hot takes, useful for balancing

**Collection method:** Manual copy-paste into a CSV. I will collect 240–250 posts total to allow for a buffer (some will be unlabelable — too short, pure questions, non-English, etc.).

**Target distribution:** Approximately 75–80 per label (~33% each). Since r/television skews toward `reaction` and `hot_take`, and r/TrueFilm skews toward `analysis`, I'll balance by source.

**If a label is underrepresented after 200 examples:** Deliberately seek posts from the underrepresented community. For `analysis`, go to r/TrueFilm or search for posts with high engagement on show finale threads. For `reaction`, look at episode discussion threads posted within 24 hours of release.

**What I will NOT include:**
- Posts under 20 words (too little signal)
- Pure questions ("Does anyone know if season 4 is confirmed?")
- Posts that are primarily quoting or linking without commentary
- Non-English posts

---

## Evaluation Metrics

**Primary metrics:**
- **Per-class F1 score** — because class imbalance is a real risk, and accuracy alone would hide a model that just predicts the majority class. F1 balances precision and recall per class.
- **Macro-averaged F1** — gives equal weight to each class regardless of size; the right summary metric for a 3-class task with a goal of learning all three distinctions equally.
- **Confusion matrix** — shows exactly which labels are being confused and in which direction. Essential for diagnosing failure modes (e.g. "the model predicts `hot_take` when the true label is `analysis`").

**Why accuracy alone is not enough:** If 50% of my test set is `reaction`, a model predicting `reaction` for everything achieves 50% accuracy without learning anything. Per-class F1 catches this; overall accuracy hides it.

**Also tracking:**
- Baseline (zero-shot Groq) vs. fine-tuned comparison — to confirm fine-tuning actually helped
- Confidence scores — to check whether high-confidence predictions are actually more accurate

---

## Definition of Success

The classifier is **genuinely useful** if:
- Macro F1 ≥ 0.70 across all three classes on the test set
- No single class has F1 < 0.55 (a class that bad means the model hasn't learned that boundary at all)
- Fine-tuned model outperforms zero-shot Groq baseline by at least 10 percentage points in macro F1

**Minimum acceptable for deployment:** Macro F1 ≥ 0.65, with no class below F1 = 0.50. Below this, the classifier would mislead more than it helps in a real community moderation tool.

If the fine-tuned model performs worse than or equal to the zero-shot baseline, I will treat this as a data quality problem and investigate annotation inconsistency before drawing conclusions.

---

## AI Tool Plan

### Label stress-testing (Milestone 1)
Before annotating 200 examples, I will give Claude my label definitions and ask it to generate 10–15 posts that sit at the boundary between two labels — specifically the three edge cases documented above. If I can't cleanly classify those generated posts using my decision rules, the definitions need tightening. I will do this before collecting any real data.

### Annotation assistance (Milestone 3)
I may use an LLM to pre-label a first batch of ~50 posts before reviewing them myself. If I do, I will:
- Provide the full label definitions from this document (not a summary)
- Require the model to output only the label name
- Review and correct every pre-assigned label myself (no skimming)
- Track which examples were pre-labeled in a `prelabeled` column in the CSV
- Disclose this in the AI usage section of the README

I will not pre-label more than 25% of the dataset this way, to keep the annotation signal primarily from my own judgment.

### Failure analysis (Milestone 6)
After evaluating the fine-tuned model, I will paste all misclassified test examples into Claude and ask it to identify patterns — similar post length, use of sarcasm, which label pair is most confused, etc. I will then verify those patterns by re-reading the examples myself. I will include what I found and what I had to correct or discard in the evaluation report.

---

## Stretch Features (planned)

- [ ] **Deployed interface** — Gradio app that accepts a post, runs it through the fine-tuned classifier, and displays the label and confidence score
- [ ] **Error pattern analysis** — systematic analysis of failure modes beyond individual wrong predictions
- [ ] **Confidence calibration** — does a 90% confident prediction get it right more often than a 60% one?

*This document will be updated before starting any stretch features.*
