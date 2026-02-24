+++
title = "Intro to Isosceles, part 2"
date = 2026-02-23
description = "Toward a better model for linguistic annotation of 19th-century French literature"

[taxonomies]
tags = ["annotation", "methodology", "tools", "Universal Dependencies"]
+++

*This is the second in a series of posts about Isosceles, a corpus of 19th-century French and English literature. The first post described the corpus itself. This post covers the annotation methodology: how the French texts are converted to CoNLL-U format, what goes wrong, and how cross-pipeline comparison and targeted LLM review bring the annotations to training-data quality. Readers are assumed to be familiar with Universal Dependencies and CoNLL-U format.*

# Toward a better model for linguistic annotation of 19th-century French literature
## Overview
It's well known that NLP tools trained on modern corpora degrade in practice when used on historical literary text (Gladstone et al., 2025). In my experience attempting to annotate this corpus of ~7 million tokens of French literary text from 1840 to 1920, I found that Stanza, spaCy, and CoreNLP all exhibited systematic errors on *passé simple* constructions, rare subjunctive forms, and irregular verb morphology. Manual correction would be impractical for a corpus of this size, and an expensive round of LLM annotation would tend to introduce error patterns of its own (Gladstone et al., 2025 describe a related approach using full-sentence LLM annotation, but acknowledge that their synthetic ground truth likely still contains systematic errors).

This post describes an annotation correction pipeline built around several highly targeted per-task classifiers running on consumer hardware (MacBook M4 Pro with 24 GB RAM). The pipeline combines three layers of detection:

First, rule-based classifiers handle cases where the errors can be spotted deterministically with high likelihood:
- Dictionary lookup against Littré's _Dictionnaire de la langue française_ (1872–1877) catches fabricated lemmas (non-words invented by Stanza, like `trouvâmer` or `buver`)
- Suffix rules resolve unambiguous verb forms
- Cross-pipeline comparison between Stanza and spaCy flags lemma, tense/mood, and UPOS disagreements
- A subject–verb person agreement checker flags mismatches between unambiguous subject pronouns and the verb's Person feature

Next, ensembles of local LLM classifiers (`mistral-small3.2:24b`, `qwen3:8b`, or `gemma3:12b`) flag likely errors in ambiguous or context-dependent cases:
- Lemma errors where both candidates are real words (e.g., prier vs prendre)
- Tense/mood mistagging
- AUX/VERB confusion
- ADJ/ADV misclassification
- `que` disambiguation (relative pronoun vs complementizer vs restrictive adverb)

Finally, tokens flagged by at least one of the above layers are escalated to a stronger model (`claude-sonnet-4-5`) for final adjudication with full sentence context. This layered approach is cheaper and more accurate than sending raw Stanza output to a single, more powerful LLM.

I've applied this pipeline to a subset of the ELTeC-fra corpus (about 600,000 tokens from 80 novels) to produce corrected CoNLL-U that will serve as training data for a domain-adapted Stanza model for 19th-century literary French. If the resulting model generalizes well, I will then use it to annotate the remaining millions of tokens in the Isosceles corpus. The resulting model should also be useful to anyone working computationally with 19th-century French literature who wants UD v2-compliant annotation. I will release the model publicly when training completes sometime around the middle of March, 2026.

## Choosing an annotation stack
My goal is to get high-quality, UD v2-compliant CoNLL-U with reliable tokenization, lemmas, UPOS tags, and morphological features. I evaluated three tools for the purpose: spaCy (`fr_dep_news_trf`), Stanza (GSD treebank model), and Stanford CoreNLP.

For sentence segmentation, I chose CoreNLP for its consistency across French and English texts. Stanza had two problems that made it less than ideal in this task for my use case: it splits sentences at semicolons in French but not in English, and, in French only, occasionally stumbles on paragraph-final quotation marks. spaCy had pervasive segmentation problems in French that made it entirely unusable for this purpose.

For annotation, Stanza's GSD model produces the most consistent UD v2 output. I initially tried the default "combined" model, which was trained on multiple treebanks, but the mixture of annotation conventions there tended to produce inconsistent results, as Stanza itself warns in [the documentation](https://stanfordnlp.github.io/stanza/combined_models.html).

spaCy was initially attractive for its stronger dependency parsing. But, in the end, I found that none of the available parsers could produce satisfactory structural parses for my purposes, and I decided to leave dependency parsing out of scope. With this decision out of the way, Stanza's richer inventory of morphological features, and its native UD v2 compliance, made it the better choice.

CoreNLP's French lemmatizer turned out not to be competitive with these more recent tools; for most tokens, it simply returns the surface form or its lowercased variant.

## What Stanza gets wrong
### Passé simple tagged as present tense
This is the largest single error class. In much first-person narration in French, *passé simple* is the primary narrative tense. But many *passé simple* forms are homographic with the present tense, and Stanza frequently defaults to present.

### Fabricated infinitives
Stanza's lemmatizer sometimes invents infinitives that don't exist in French. These are plausible-looking but wrong: `trouvâmes` → `trouvâmer` (should be `trouver`), `parurent` → `pararer` (should be `paraître`), `haïssez` → `haïsser` (should be `haïr`), `buvaient` → `buver` (should be `boire`), `entrâmes` → `entrâmer` (should be `entrer`). The *passé simple* and literary past forms are the most affected.

### Cross-verb confusion
A related problem: Stanza assigns forms to the wrong real verb, attracted by character-level prefix overlap. The particularly systematic case is `prier`/`prendre`: I found that every short form of *prier* (`prie`, `pria`, `priez`) is lemmatized as `prendre`. The `pri-` prefix is shared with high-frequency *prendre* forms (`pris`, `prit`, `prend`), and presumably the character model can't distinguish them. Longer forms where the `-ier` conjugation pattern is visible (`priait`, `prié`, `priant`) are correctly assigned.

A few other confirmed cross-verb confusions: `durerait` → `devoir`, `dirent` → `devoir`, `pourrions` → `pourrir`.

### Conditional and subjunctive misanalysis
Conditional forms are tagged as indicative present, imperfect subjunctive forms are tagged as indicative. These are morphologically distinct — `retournerait` is not a present-tense form under any analysis.

## Cross-pipeline comparison
Although I found Stanza more reliable overall, it proved useful to also have outputs from spaCy on hand for comparison. In particular, cases where Stanza and spaCy disagree have turned out to provide a valuable error-detection signal.

To leverage this signal, I built a comparison tool that aligns Stanza and spaCy outputs by token (handling MWT expansion differences) and checks whether the two parsers converge on the same annotation or not. Across the 600k tokens of my training set (i.e. chunks from 80 ELTeC-Fra novels):

| Category | Count |
|----------|-------|
| Substantive lemma disagreements | 41,092 |
| Form-as-lemma (Stanza uses surface form) | 7,202 |
| Case-only differences | 3,471 |
| **Total Stanza lemma outliers** | **51,765** |

Not all of these are errors. Many are convention differences: Stanza follows GSD conventions for pronoun lemmas (`lui` for 3sg subject, `moi` for 1sg), while spaCy normalizes differently (`il`, `je`). Gendered noun lemmas, ligature normalization (`sœur`/`soeur`), and the participle-adjective boundary account for much of the rest.

After filtering conventional disagreements such as these, roughly 20,000 substantive disagreements remain. These are flagged as candidates for correction.

### Tense and mood
The comparison extends to morphological features. Across the 600k tokens of the test set, almost 17k tokens show tense or mood disagreements between the two pipelines. The most frequent patterns:

| Count | Stanza | spaCy |
|-------|--------|-------|
| 976 | Ind Pres | Ind Past |
| 728 | Ind Pres | Ind Fut |
| 649 | Ind Pres | Ind Imp |
| 481 | Ind Pres | Cnd Pres |

The top three patterns alone flag 2,353 tokens — the same *passé simple*, future, and imparfait errors found during manual review. SpaCy is not always right (it over-corrects toward *passé simple* on stative verbs like `s'agir`, and mishandles imperfect subjunctive), but it often catches tense errors that were missed entirely by the local LLM models used later in the pipeline.

## Model selection for local LLM review
The local LLM review is a set of binary classifiers, each tuned to one error type, each using the model best suited to that task.

I evaluated four models — Mistral-Nemo (12B), Qwen 3 (8B), Gemma 3 (12B), and Mistral-Small 3.2 (24B, Q4 quantized to fit 24 GB RAM) — across five classifier tasks on test chunks from several ELTeC authors (Audoux, Allais, Balzac, Flaubert, Montagne, Rolland).

### Lemma classification
The lemma classifier asks whether a lemma is correct _in context_, given GSD treebank conventions. Mistral-Small 3.2 was most promising in this domain.

The Littré + Mistral-Small combination is complementary: Littré catches non-word lemmas, Mistral-Small catches real-word confusions.

### Tense/mood classification
Qwen had the best precision here but a low recall rate. Gemma, on the other hand, serves as a recall-oriented backup: it catches some errors Qwen misses but produces more false positives. But some false positives are tolerable, as flagged tokens will go to Sonnet for a final review, anyway. 

The union of Qwen and Gemma flags, after filtering, gives the best candidate set for detecting tense/mood errors.

### AUX/VERB classification
Qwen was excellent on this classification task across all evaluated chunks — perfect or near-perfect precision and recall.

### `que` classification
Gemma was perfect — zero false flags across all evaluated chunks. Qwen produced false PRON→SCONJ flags on relative `que` in multi-clause sentences. Mistral-Nemo consistently misclassified in both directions.

### ADJ↔ADV classification
A fifth classifier was added after the initial evaluation revealed that Stanza mistagged French words that swing between ADJ and ADV depending on context. This classifier uses a closed word list of 28 target lemmas (23 from the GSD treebank, 5 common 19th-century additions) with per-word usage guidance interpolated into the prompt. Qwen handles this well.

### Self-contradiction filter
As a post-processing step before final Sonnet review, I discarded any flag where the model's proposed correction equals the current value. This addresses a frequent deficiency in all LLM models observed: a model flags a token, generates reasoning about why it might be wrong, but then proposes no actual change.

### Summary
The resulting pipeline architecture, after three rounds of evaluation and refinement:

| Classifier | Best model | Notes |
|---|---|---|
| Lemma | Littré + Mistral-Small 3.2 24B | Dictionary for non-words, LLM for real-word confusions |
| Tense/mood | Qwen 3 8B + Gemma 3 12B (union) | Qwen for precision, Gemma for recall backup |
| AUX/VERB | Qwen 3 8B | Only model to resist the "eut" trap |
| `que` | Gemma 3 12B | Only model with zero false positives |
| ADJ↔ADV | Qwen 3 8B | Closed word list + syntactic judgment |

## Final review with Sonnet
The flags from the local classifiers are submitted to Claude Sonnet (`claude-sonnet-4-5`). Each prompt contains the sentence, the flagged token with its current annotation, a targeted set of instructions about GSD treebank conventions, and the proposed correction (which it's free to change if necessary). Sonnet then returns a structured JSON response: either `"action": "correct"` with updated fields, or `"action": "no_change"` with a reason.

Sonnet's review is consistently very good but not perfect. It tends to fail, for example, on forms that are genuinely ambiguous between two tenses. The 3sg `-it` ending of second-group verbs is identical in present and *passé simple* (`envahit`, `grandit`, `finit`). When a classifier or spaCy flags one of these, Sonnet sometimes accepts the change even when context clearly favors present tense. This failure is consistent across models (Opus produces the same error) and across temperature settings. For these ambiguous forms, a contextual heuristic upstream may be needed: if all other finite verbs in the sentence are present tense.

## What's next
Once processing completes in early March, 2026, my training set of ELTeC French (~600,000 words across 80 novels) will be used to train a domain-adapted Stanza model for literary French. The question is how much the corrected training data improves tagging accuracy on held-out literary text compared to the stock GSD model. I'll report on that when the time comes.

The corpus, annotation scripts, and comparison tools are available at [github.com/myersm0/isosceles](https://github.com/myersm0/isosceles).

## References
- Gladstone, C., Fang, Z., and Stewart, S.D. (2025). "Ground Truth Generation for Multilingual Historical NLP using LLMs." *Proceedings of the Conference of the Alliance of Digital Humanities Organizations*.

