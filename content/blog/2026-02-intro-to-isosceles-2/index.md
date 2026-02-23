+++
title = "Intro to Isosceles, part 2"
date = 2026-02-18
description = "Toward a better model for linguistic annotation of 19th-century French literature"

[taxonomies]
tags = ["annotation", "methodology", "tools", "Universal Dependencies"]
+++

*This is the second in a series of posts about Isosceles, a corpus of 19th-century French and English literature. The first post described the corpus itself. This post covers the annotation methodology: how the French texts are converted to CoNLL-U format, what goes wrong, and how cross-pipeline comparison and targeted LLM review bring the annotations to training-data quality. Readers are assumed to be familiar with Universal Dependencies and CoNLL-U format.*

# Toward a better model for linguistic annotation of 19th-century French literature
## Introduction
It's well known that NLP tools trained on modern corpora degrade in practice when used on historical literary text (Piotrowski, 2012; Gladstone et al., 2025). In my experience attemping to annotate this corpus of ~7 million tokens of French literary text from 1840 to 1920, I found that Stanza, spaCy, and CoreNLP all exhibited systematic errors on *passé simple* constructions, rare subjunctive forms, and irregular verb morphology. Manual correction would be impractical for a corpus of this size, and an expensive round of LLM annotation would tend to introduce own error patterns of its own (Gladstone et al., 2025 describe a related approach using full-sentence LLM annotation, but acknowledge that their synthetic ground truth likely still contains systematic errors).

This post describes a targeted correction pipeline built around per-task classifiers running on consumer hardware (MacBook M4 Pro with 24 GB RAM). The pipeline combines three layers of detection:

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

Finally, tokens flagged by at least one of the above layers are escalated to a stronger model (`claude-sonnet-4-5` via Anthropic API) for final adjudication with full sentence context. This layered approach is both cheaper and more accurate than sending raw Stanza output to a single, more powerful LLM.

I applied this pipeline to a subset of the ELTeC-fra corpus (80 novels, ~600,000 tokens) to produce corrected CoNLL-U that will serve as training data for a domain-adapted Stanza model for 19th-century literary French. If the resulting model generalizes well, it can annotate the remaining millions of tokens in the Isosceles corpus without per-sentence LLM review. Furthermore, it should be useful to anyone working computationally with 19th-century French literature who wants UD v2-compliant annotation. I will release the model publicly when training completes sometime around the middle of March, 2026.

## Choosing an annotation stack
The goal is UD v2-compliant CoNLL-U with reliable tokenization, lemmas, POS tags, and morphological features. I evaluated three tools for the purpose: spaCy (`fr_dep_news_trf`), Stanza (GSD single-treebank model), and Stanford CoreNLP.

For sentence segmentation, I chose CoreNLP for its consistency across French and English texts. Stanza had two problems that made it less than ideal for my use case: it splits sentences at semicolons in French but not in English, and, in French only, it occasionally stumbles on paragraph-final quotation marks. spaCy had pervasive segmentation problems in French that made it entirely unusable for this purpose.

For annotation, Stanza's GSD model produces the most consistent UD v2 output. I initially tried the default "combined" model, which was trained on multiple treebanks, but it bleeds in annotation conventions from other corpora.

spaCy was initially attractive for its stronger dependency parsing. But, in the end, I found that none of the available parsers could produce acceptable structural parses for complex literary prose, and I decided to leave dependency parsing out of scope for my purposes. With this decision out of the way, Stanza's richer inventory of morphological features, and its native UD v2 compliance, made it the better choice.

CoreNLP's French lemmatizer turned out not be competitive with these more recent tools; for most tokens, it simply returns the surface form or its lowercased variant. Therefore I found that it adds no useful signal for lemma or feature correction, and dropped it after an initial evaluation.

## What Stanza gets wrong
### Passé simple tagged as present tense
This is the largest single error class. In much first-person narration in French, *passé simple* is the primary narrative tense. But many *passé simple* forms are homographic with the present tense, and Stanza systematically defaults to present: `voulus`, `partis`, `sautai`, `retrouvai`, `soulevai`, `plongeai`, `passai`, `ouvris` are all tagged `Tense=Pres`.

Some of these are unambiguous from the suffix alone: `-èrent` is always *passé simple*, `-ât`/`-ît`/`-ût` are always imperfect subjunctive, `-aient` with `Mood=Ind` is always imparfait. These can be caught deterministically. The ambiguous cases — 3sg `-it` forms of second-group verbs, where present and *passé simple* are identical — require context.

### Fabricated infinitives
Stanza's character-level lemmatizer sometimes invents infinitives that don't exist in French. These are plausible-looking but wrong: `trouvâmes` → `trouvâmer` (should be `trouver`), `parurent` → `pararer` (should be `paraître`), `haïssez` → `haïsser` (should be `haïr`), `buvaient` → `buver` (should be `boire`), `entrâmes` → `entrâmer` (should be `entrer`). The *passé simple* and literary past forms are the most affected.

### Cross-verb confusion
A related problem: Stanza assigns forms to the wrong real verb, attracted by character-level prefix overlap. The particularly systematic case is `prier`/`prendre`: I found that every short form of *prier* (`prie`, `pria`, `priez`) is lemmatized as `prendre`. The `pri-` prefix is shared with high-frequency *prendre* forms (`pris`, `prit`, `prend`), and presumably the character model can't distinguish them. Longer forms where the `-ier` conjugation pattern is visible (`priait`, `prié`, `priant`) are correctly assigned.

A few other confirmed cross-verb confusions: `durerait` → `devoir`, `dirent` → `devoir`, `pourrions` → `pourrir`.

### Conditional and subjunctive misanalysis
Conditional forms are tagged as indicative present, imperfect subjunctive forms are tagged as indicative. These are morphologically distinct — `retournerait` is not a present-tense form under any analysis.











