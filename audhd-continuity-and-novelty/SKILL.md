---
name: audhd-continuity-and-novelty
description: Guides the agent whenever AuDHD, ADHD, or Autism Spectrum Disorder (ASD) is present in the user's prompt or will be substantively addressed in the response (ignoring casual mentions in background tool outputs). Enforces a mandatory MemoryPlugin pre-check to eliminate retreading heavily discussed concepts, suppresses ubiquitous boilerplate (e.g., DSM-5 comorbidity history, 'more than the sum of its parts', 'community term' disclaimers), and pivots directly into novel facts, mechanisms, and undiscussed scientific angles.
metadata:
  version: "1.0.0"
  author: "IsaiahDePorto"
---

# AuDHD Continuity & Novelty Engine

## 1. Activation & Scope Boundaries

### When to Activate
Activate this skill when **either** of the following conditions is met:
1. The user's prompt explicitly or implicitly inquires about, discusses, or references **AuDHD**, **ADHD**, or **Autism Spectrum Disorder (ASD)**.
2. The agent determines that its final response must substantively address, analyze, or explain aspects of AuDHD, ADHD, or ASD.

### When NOT to Activate
- **Passive Tool Output Matches:** Do **not** load or execute this skill if the terms "ADHD", "ASD", or "AuDHD" only appear incidentally inside third-party web search snippets, file dumps, or background tool logs without being the focus of the user's inquiry or the final generation.
- **Unrelated Clinical Contexts:** Do not activate for purely general neurological/psychiatric questions unless ADHD or ASD is directly involved.

---

## 2. Core Protocol: MemoryPlugin Pre-Generation Audit

Before generating explanations, analyses, or thematic overviews about AuDHD/ADHD/ASD, the agent must perform a rapid verification pass across the user's past chat history and stored memories to guarantee continuity and avoid repetition.

### Step 1: Pre-Formulation Query
Identify the primary claims, theoretical frameworks, or phenomena intended for the response. Query MemoryPlugin:
- Call `recall_chat_history` using focused queries (e.g., `"AuDHD [specific topic/mechanism]"`).
- Call `search_memories` if verifying specific diagnostic, physiological, or personal preferences.

### Step 2: Negative Filtering & Novelty Assessment
- **Has this specific angle or fact been extensively discussed?** If the retrieval results indicate the concept has already been thoroughly covered, **drop the detailed explanation**.
- **Can it be referenced in passing?** If the concept is necessary as context for a newer point, state it in a single clause or sentence without explanatory buildup.
- **What is unexamined?** Pivot cognitive effort and token budget entirely toward unexplored literature, deeper neurobiological mechanisms, divergent clinical perspectives, or counterintuitive angles.

### Step 3: Seamless Integration
Never break immersion with meta-commentary such as *"Based on our past chats, I know you already know..."* or *"According to your memory records..."* Simply deliver the fresh information directly and naturally.

---

## 3. Repetition Filter: The Retread Registry

The user is deeply informed on core AuDHD foundations. Repeating entry-level history, semantics, or well-worn metaphors wastes tokens and degrades the conversational experience.

Treat the following concepts as **settled context**. They must never receive standalone explanatory paragraphs:

| Settled Concept | Permitted (Passing Mention Only) | Prohibited (Explanatory Bloat) |
| :--- | :--- | :--- |
| **DSM Comorbidity History**<br>*(Prior to DSM-5 in 2013, ADHD and ASD could not be co-diagnosed)* | *"Since the DSM-5 opened up dual diagnosis, research has shifted toward..."* | *"For decades, the DSM-IV strictly prohibited clinicians from diagnosing both conditions together. Because of this artificial diagnostic divide, many people went unrecognized..."* |
| **Non-Additive Interaction**<br>*(AuDHD is not just ADHD + Autism; it creates an emergent, distinct profile)* | *"Given the unique interactive phenotype of AuDHD..."* | *"It is crucial to realize that having both conditions is more than just the sum of its parts. When combined, the traits constantly pull against each other in complex ways..."* |
| **Nomenclature Status**<br>*(AuDHD is a community-coined umbrella term, not an official ICD/DSM diagnosis)* | *"While AuDHD remains an informal clinical/community descriptor..."* | *"It's worth noting that AuDHD is actually a colloquial term created by the neurodivergent community and is not an officially recognized diagnostic category in psychiatric manuals..."* |
| **Common Paradoxes**<br>*(Craving routine vs. needing novelty; executive paralysis vs. hyperfocus)* | Brief contextual anchor when introducing an underlying neurological substrate. | Long introspect-style lists explaining how the autistic side wants order while the ADHD side wants dopamine. |

---

## 4. Execution Directives for Generation

1. **Assume Total Foundational Literacy:** Speak directly to a high-comprehension reader. Never explain basic definitions (e.g., do not define executive dysfunction, masking, stimming, interoception, or sensory gating from scratch).
2. **Prioritize Deep Neurobiology & Emerging Research:** When discussing symptoms or phenotypes, jump straight to the operational mechanisms:
   - Thalamocortical gating, TRN hyperexcitability, and sensory filtering failures.
   - Neurometabolic friction, predictive coding mismatches, and active inference costs.
   - Functional connectivity networks (Salience, Default Mode, Central Executive Network interactions).
   - Emerging multimodal biomarkers and 2025–2026 clinical literature.
3. **Calibrate for High Information Density:** Avoid patronizing accessibility caveats, performative empathy buffers, or tone-softening fluff. Deliver precise, intellectually rigorous, and structured analyses.
4. **Preserve Continuity Across Chats:** Treat the user's neurodivergent understanding as an ongoing, evolving tapestry rather than an isolated, stateless interaction.

---

## 5. Edge Cases & Fallbacks

- **Direct User Prompt on a Settled Topic:** If the user explicitly asks a direct factual question about a settled topic (e.g., *"What year did the DSM allow both ADHD and ASD?"* or *"Is AuDHD an official medical term?"*), answer concisely and accurately in 1–2 sentences, then immediately offer an advanced or novel follow-up angle rather than elaborating on the basic answer.
- **MemoryPlugin Unavailability:** If memory retrieval tools return empty or fail, err on the side of novelty and technical depth rather than retreating into explanatory generalities.
