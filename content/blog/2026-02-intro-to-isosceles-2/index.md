+++
title = "Toward a better annotation model for 19th-century French literature"
date = 2026-02-23
description = "Building a layered annotation pipeline that combines rule-based checks, cross-pipeline comparison, and local LLM classifiers to improve CoNLL-U accuracy on literary French."

[taxonomies]
tags = ["annotation", "methodology", "tools", "Universal Dependencies"]
+++

*This is the second in a series of posts about Isosceles, a corpus of 19th-century French and English literature. The first post described the corpus itself. This post covers the annotation methodology: how the French texts are converted to CoNLL-U format, what goes wrong, and how cross-pipeline comparison and targeted LLM review bring the annotations to training-data quality. Readers are assumed to be familiar with Universal Dependencies and CoNLL-U format.*

# Toward a better model for linguistic annotation of 19th-century French literature
## Overview
It's well known that NLP tools trained on modern corpora degrade in practice when used on historical literary text (Gladstone et al., 2025). In my experience attempting to annotate this corpus of ~7 million tokens of French literary text from 1840 to 1920, I found that Stanza, spaCy, and CoreNLP all exhibit systematic errors on *passé simple* constructions, rare subjunctive forms, and irregular verb morphology, among other things. Manual correction would be impractical for a corpus of this size, and an expensive round of LLM annotation would tend to introduce error patterns of its own (Gladstone et al., 2025 describe a related approach using full-sentence LLM annotation, but acknowledge that their synthetic ground truth likely still contains systematic errors).

This post describes an alternative: a layered correction pipeline that combines rule-based checks, cross-pipeline comparison, and targeted local LLM classifiers to flag likely errors, then escalates only the flagged tokens to a stronger model for final adjudication. This approach is cheaper and more accurate than sending raw parser output through a single, more powerful LLM, and can be run with on-device models on consumer hardware (e.g. a MacBook M4 Pro, 24 GB RAM).

I've applied this pipeline to a subset of the ELTeC-fra corpus — about 600,000 tokens from 80 novels — to produce high-quality CoNLL-U that will eventually serve as training data for a domain-adapted Stanza model for 19th-century literary French. If the resulting model generalizes well, I'll use it to annotate the remaining 6 million of tokens in the Isosceles corpus. I plan to release the model publicly when training completes, around mid-March 2026.

## Choosing an annotation stack
My goal is to get high-quality, UD v2-compliant CoNLL-U with reliable tokenization, lemmas, UPOS tags, and morphological features. I evaluated three tools for the purpose: spaCy (`fr_dep_news_trf`), Stanza (GSD treebank model), and Stanford CoreNLP.

For sentence segmentation, I chose CoreNLP for its consistency across French and English texts. Stanza had two problems that made it less than ideal in this task for my use case: it splits sentences at semicolons in French but not in English, and, in French only, occasionally stumbles on paragraph-final quotation marks. spaCy had pervasive segmentation problems in French that made it entirely unusable for this purpose.

For annotation, Stanza's GSD model produces the most consistent UD v2 output. I initially tried the default "combined" model, which was trained on multiple treebanks, but the mixture of annotation conventions there tended to produce inconsistent results, as Stanza itself warns in [the documentation](https://stanfordnlp.github.io/stanza/combined_models.html).

spaCy was initially attractive for its stronger dependency parsing. But, in the end, I found that none of the available parsers could produce satisfactory structural parses for my purposes, and I decided to leave dependency parsing out of scope. With dependency relations off the table, then, Stanza became the better choice given its richer set of morphological features and native UD v2 compliance.

CoreNLP's French lemmatizer turned out not to be competitive with these more recent tools; for most tokens, it simply returns the surface form or its lowercased variant.

## What Stanza gets wrong
### Passé simple tagged as present tense
This is the largest single error class. In much first-person narration in French, *passé simple* is the primary narrative tense. But Stanza frequently cannot distinguish this from present tense.

### Fabricated infinitives
Stanza's lemmatizer sometimes invents infinitives that don't exist in French:
- `trouvâmes` → `trouvâmer` (should be `trouver`)
- `parurent` → `pararer` (should be `paraître`)
- `haïssez` → `haïsser` (should be `haïr`)
- `buvaient` → `buver` (should be `boire`)
- `entrâmes` → `entrâmer` (should be `entrer`).

### Cross-verb confusion
Sometimes Stanza assigns forms to the wrong real verb, attracted by character-level prefix overlap. A particularly systematic case is `prier`/`prendre`: I found that every short form of *prier* (`prie`, `pria`, `priez`) is lemmatized as `prendre`. The `pri-` prefix is shared with high-frequency *prendre* forms (`pris`, `prit`, `prend`), and presumably the character-level lemmatizer model can't distinguish them. Longer forms where the `-ier` conjugation pattern is visible (`priait`, `prié`, `priant`) are correctly assigned.

A few other confirmed cross-verb confusions: 
- `durerait` → `devoir` (should be `durer`)
- `dirent` → `devoir` (should be `dire`)
- `pourrions` → `pourrir` (should be `pouvoir`)

### Conditional and subjunctive misanalysis
Some verb forms are being labeled as the wrong tense. For example, conditional forms are marked as present tense, and literary past subjunctive forms are marked as simple indicative forms. But these forms have distinct endings in French. A word like `retournerait`, for example, cannot be present tense — its ending marks it as conditional.

## The layered correction pipeline
The pipeline combines three layers of detection. Tokens flagged by any layer are escalated to a stronger model (Claude Sonnet) for final adjudication, described in a later section.

### 1. Rule-based filters
The first layer handles cases where errors can be detected deterministically with high likelihood:

- Each lemma is checked against a period-appropriate dictionary, Littré's _Dictionnaire de la langue française_ (1872–1877).

- Some verb forms are unambiguous on their own: the passé simple endings `-âmes`, `-âtes`, `-èrent` can only belong to one tense, regardless of context. A small set of heuristics detects violations of these rules.

- A simple subject-verb agreement checker flags mismatches between unambiguous subject pronouns and the verb's `Person` feature.

### 2. Cross-pipeline comparison
Although I found Stanza more reliable overall, it proved useful to also have outputs from spaCy on hand for comparison. In particular, cases where Stanza and spaCy disagree have turned out to provide a valuable error-detection signal.

To leverage this signal, I built a comparison tool that aligns Stanza and spaCy outputs by token (handling MWT expansion differences) and checks whether the two parsers converge on the same annotation or not. Across the 600k tokens of my training set (i.e. chunks from 80 ELTeC-Fra novels):

| Category | Count |
|----------|-------|
| Substantive lemma disagreements | 41,092 |
| Form-as-lemma (Stanza uses surface form) | 7,202 |
| Case-only differences | 3,471 |
| **Total Stanza lemma outliers** | **51,765** |

Not all of these are errors. Many are convention differences: Stanza follows GSD conventions for pronoun lemmas (`lui` for 3sg subject, `moi` for 1sg), while spaCy normalizes differently (`il`, `je`). Gendered noun lemmas, ligature normalization (`sœur`/`soeur`), and the participle-adjective boundary account for much of the rest.

After applying some heuristics to filter out these merely conventional disagreements, roughly 20,000 substantive disagreements remain.

#### Tense and mood
The comparison extends to morphological features. Across the 600k tokens of the test set, almost 17k tokens show tense or mood disagreements between the two pipelines. The most frequent patterns:

| Count | Stanza | spaCy |
|-------|--------|-------|
| 976 | Ind Pres | Ind Past |
| 728 | Ind Pres | Ind Fut |
| 649 | Ind Pres | Ind Imp |
| 481 | Ind Pres | Cnd Pres |

The top three patterns alone flag 2,353 tokens — the same *passé simple*, future, and imparfait errors found during manual review. SpaCy is not always right (it over-corrects toward *passé simple* on stative verbs like `s'agir`, and mishandles imperfect subjunctive), but it often catches tense errors that were missed entirely by the local LLM models used later in the pipeline.

### 3. Targeted LLM classifiers
For situations requiring contextual judgment that neither above approach can easily provide, I used a set of local LLM classifiers, each tuned to a single error type.

I evaluated four models — Mistral-Nemo (12B), Qwen 3 (8B), Gemma 3 (12B), and Mistral-Small 3.2 (24B, Q4 quantized to fit 24 GB RAM) — across five classifier tasks on test chunks from several ELTeC authors (Audoux, Allais, Balzac, Flaubert, Montagne, Rolland). In general, the Mistral models performed best at the lemma task, while Qwen and Gemma had strengths in grammatical judgments. The task and the selected model are described below:
1. **Lemma correctness** (_Mistral-Small 3.2_). Where both the current lemma and a candidate replacement are real words (e.g., prier vs prendre), a classifier asks whether the current lemma is correct in context given GSD treebank conventions. This complements the Littré dictionary check, which only catches non-words.
2. **Tense/mood tagging** (_Qwen3 + Gemma3_ (union)). A classifier flags verb forms whose tense or mood tag may be wrong, targeting the passé simple/present and conditional/indicative confusions described above.
3. **AUX/VERB distinction** (_Qwen3_). French auxiliary verbs (avoir, être) are tagged AUX in some syntactic contexts and VERB in others. A classifier checks whether the current tag is correct.
4. **`que` disambiguation** (_Gemma3_). The word que can be a relative pronoun (PRON), a complementizer (SCONJ), or a restrictive adverb (ADV). Stanza sometimes assigns the wrong category, and the correct choice depends on syntactic context.
5. **ADJ↔ADV misclassification** (_Qwen3_). A closed list of 28 lemmas that can function as either ADJ or ADV depending on context (23 from the GSD treebank, 5 common 19th-century additions). A classifier checks each occurrence against per-word usage guidance.

As a post-processing step, I discarded any cases of model self-contradiction: any flag where the model's proposed correction is the same as the current value. This addresses a frequent deficiency across all LLM models tested.

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
The flags from the local classifiers are submitted to Claude Sonnet (`claude-sonnet-4-5`). Each prompt contains the sentence, the flagged token with its current annotation, a targeted set of instructions about GSD treebank conventions, and the proposed correction (which Sonnet is free to change if necessary). Sonnet then returns a structured JSON response: either `"action": "correct"` with updated fields, or `"action": "no_change"` with a reason.

Sonnet's review is consistently very good but not perfect. It tends to fail, for example, on forms that are genuinely ambiguous between two tenses. The 3sg `-it` ending of second-group verbs is identical in present and *passé simple* (`envahit`, `grandit`, `finit`). When a classifier or spaCy flags one of these, Sonnet sometimes accepts the change even when context clearly favors present tense. This failure is consistent across models (Opus produces the same error) and across temperature settings. For these ambiguous forms, a contextual heuristic upstream may be needed: if all other finite verbs in the sentence are present tense.

Still, the resulting patched CoNLL-U is considerably more accurate than the raw outputs from Stanza. Soon I'll follow up with some specific numbers.

## What's next
Once processing completes in early March, 2026, my training set of ELTeC French (~600,000 words across 80 novels) will be used to train a domain-adapted Stanza model for literary French. The question is how much the corrected training data improves tagging accuracy on held-out literary text compared to the stock GSD model. I'll report on that when the time comes.

The corpus, annotation scripts, and comparison tools are available at [github.com/myersm0/isosceles](https://github.com/myersm0/isosceles).

## References
- Gladstone, C., Fang, Z., and Stewart, S.D. (2025). "Ground Truth Generation for Multilingual Historical NLP using LLMs." *Proceedings of the Conference of the Alliance of Digital Humanities Organizations*.

