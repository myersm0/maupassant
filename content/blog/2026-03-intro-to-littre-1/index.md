+++
title = "Introducing Deep-Littré"
date = 2026-03-10
description = "A quick introduction to the Deep-Littré: a deeply structured, computationally enriched edition of Émile Littré's French dictionary, available as TEI Lex-0 XML and SQLite."

[taxonomies]
tags = ["tools"]
+++

# Introduction
The four quarto volumes and supplement of Émile Littré's _Dictionnaire de la langue française_, published between 1862 and 1877, come to over 11 million words in total. That's longer than all the hundred novels of the ELTeC-Fra corpus combined.

Thankfully we have digital versions. François Gannaz's [XMLittré](https://www.littre.org/) project provides both a plain-text and an XML digitization of the full dictionary. In the XML edition, the internal structure of entries is represented by roughly 87,000 sub-entry blocks encoded as "\<indent\>" elements.

But the variety and structure of Littré's actual text are not fully captured by this flat, undifferentiated _indent_ structure. Specifially, a typical entry in Littré contains a hierarchy of several kinds of information:
- **Numbered senses** — the core definitions, ordered from literal to abstract.
- **Figurative extensions** — senses marked _Fig._ or _Figurément_, where a word's meaning shifts metaphorically. Often nested under the literal sense they derive from.
- **Domain labels** — _Terme de marine_, _Terme de musique_, _Terme de botanique_ — marking definitions specific to a technical field.
- **Register labels** — _Familièrement_, _Populairement_, _Par extension_ — indicating the social or stylistic context of a usage.
- **Locutions** — fixed expressions defined under a headword (for example, "Tomber sous la main de quelqu'un" under headword **main**)
- **Literary citations** — dated quotations from authors spanning the 10th to the 19th century, serving as evidence for each sense.
- **Etymologies and historical notes** — tracing each word back through Old French, Provençal, and Latin with dated attestations.

While XMLittré preserves the text, it does not distinguish the different structural roles those elements play within an entry.

This is not a criticism of Gannaz’s work. Converting Littré into XML was a major undertaking—one that reportedly required hundreds or even thousands of hours. My own work, still in progress, has been a similarly meticulous task that builds directly on that effort.

My project introduced here, [Deep-Littré](https://github.com/myersm0/deep-littre), is an ongoing attempt to recover the internal structure of Littré’s entries and represent it explicitly. The goal is to make the dictionary easier to analyze computationally and more navigable for readers.

[Work in progress. To be continued.]

![UMAP visualization of the embedding space](umap.png)



