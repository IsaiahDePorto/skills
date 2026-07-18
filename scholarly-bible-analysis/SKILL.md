---
name: scholarly-bible-analysis
description: Executes a rigorous, 5-step verification-first hermeneutical research loop to interpret Biblical passages. Use this skill when answering questions about Biblical translation, historical-cultural context, original Greek or Hebrew manuscripts, cross-references, or theological doctrines.
license: MIT
compatibility: Access to TheologAI MCP Server (primary) and Bolls Bible API (fallback)
metadata:
  version: "1.0.0"
  author: "Systematic-Theology-AI"
---

# Operational Guide

## 1. Core Objectives
- Neutralize modern semantic drift and translation bias by auditing original languages and historical context before formulating theological claims.
- Force systematic, multi-step verification using real-time API tools rather than relying on anachronistic neural memory.
- Provide a standardized, transparent, and objective output structure for Bible analysis queries.

---

## 2. Step-by-Step Execution State-Machine

When executing a Bible research query, progress through the following five phases sequentially. Use XML-bounded reasoning blocks to isolate intermediate calculations.

### Phase 1: Literary Macro-Context Scoping
1. Retrieve the entire chapter containing the target verse using `bible_lookup` (e.g., `Romans 12` for `Romans 12:1`).
2. Read a minimum of 10 verses prior to and 10 verses following the target verse.
3. Identify the literary genre (e.g., Epistolary, Prophetic, Wisdom, Narrative, Apocalyptic) and write a 2-sentence summary of the chapter's overarching argument in the final output.

### Phase 2: Lexical Translation Divergence Audit
1. Compare the target verse across a minimum of three distinct translation philosophies (Formal Equivalence, Dynamic Equivalence, and Literal) using `compare_translations` or sequential `bible_lookup` calls. Recommended versions: `NASB` or `ESV` (Formal), `YLT` (Literal), and `NET` or `BSB` (Dynamic).
2. Flag words that diverge significantly in meaning between the dynamic/formal translations and the Young's Literal translation. Identify these as "High-Priority Lexical Targets."

### Phase 3: Original Language & Morphology Auditing
1. For every "High-Priority Lexical Target" identified in Phase 2, retrieve its word-by-word grammatical breakdown using `bible_verse_morphology` (which utilizes STEPBible database rows).
2. Look up the exact Hebrew, Aramaic, or Greek Strong's number (e.g., `G3870` or `H1254`) using `original_language_lookup` or `original_language_study`.
3. Extract the following linguistic parameters:
   - **Etymological Root:** The mechanical action of the base root.
   - **Semantic Range:** How the lemma is used across contemporary literature of the same era.
   - **Grammatical Inflections:** How voice, mood, tense, case, and gender affect the targeted translation.

### Phase 4: Intertextual & Cross-Reference Mapping
1. Track how the passage behaves within the broader biblical canon. Query the Treasury of Scripture Knowledge (TSK) index using `bible_cross_references` to extract highly rated canonical cross-references.
2. If synoptic parallels exist (e.g., Gospels, Samuel/Kings/Chronicles), use `parallel_passages` to analyze synoptic divergence, tracing how different authors structured the same events.

### Phase 5: Historical-Cultural Reconstruction
1. Identify the social, political, legal, and environmental contexts of the passage using `commentary_lookup` (querying chapter-level commentary across historical sources like Matthew Henry, Jamieson-Fausset-Brown, John Gill, or Keil-Delitzsch).
2. Explicitly map the ancient cultural worldview (e.g., Honor-Shame dynamics, Roman Imperial occupation, Ancient Near Eastern covenants) to separate the text from modern Western concepts.

---

## 3. Uncertainty Calibration and Exit Valve

```
[CONFIDENCE CHECK]
Assess the structural viability of this inquiry before answering:
- If the target passage contains a hapax legomenon (a word occurring only once) with no clear semantic consensus in extrabiblical literature: Explicitly state this lexical limitation in the 'Original Language' section.
- If the historical context is highly disputed among modern historians and commentators: Present the two leading academic positions objectively, with zero dogmatism.
- If you lack access to the necessary Greek/Hebrew morphology data for a specific verse: Proceed with the translation comparison, tag the grammar section as [DATA_LIMITATION], and restrict your linguistic assertions to the literal translations.
```

---

## 4. Hard Negation Guardrails

- RESTRICTION: DO NOT write conversational greetings, personal remarks, introductory pleasantries (such as "I would be happy to help with that!"), or conversational conclusions.
- RESTRICTION: DO NOT frame the analysis through any specific modern denominational bias (e.g., Reformed, Catholic, Progressive, Evangelical). Keep the tone strictly academic, linguistic, and historical-critical.
- RESTRICTION: DO NOT synthesize or invent Strong's definitions or grammatical codes. Every linguistic definition must be derived directly from the `original_language_lookup` or `bible_verse_morphology` tools.
