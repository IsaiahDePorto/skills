---
name: tts-mode
description: Alters the agent's output style to optimize for Text-to-Speech (TTS) voice synthesizers. Completely strips structural markdown, complex layouts, and formal citations, replacing them with fluid prose paragraphs and natural-sounding spoken attributions. Activate this skill whenever the user requests TTS mode or indicates that responses will be read aloud.
license: MIT
metadata:
  version: "1.0.0"
  author: "Agent Architecture Group"
---

# Operational Guide: TTS Mode

This skill instructs the agent on how to reformat all communication to be perfectly optimized for auditory consumption. When active, the primary objective is to generate text that reads aloud naturally, smoothly, and without cognitive friction for a human listener or a voice synthesizer.

## 1. Core Objectives
- Eliminate all visual formatting that causes stuttering, reading errors, or awkward pauses in TTS engines.
- Convert dense, structured information (tables, lists, bullet points) into highly coherent, narrative prose.
- Replace technical or bracketed citations with seamless, spoken-style attributions embedded directly into the flow of speech.

---

## 2. Structural Formatting Constraints

To ensure compatibility with TTS engines (like Gemini TTS), the output must adhere to a strict set of formatting rules:

| Visual Element | Prohibited Format | TTS-Compliant prose replacement |
| :--- | :--- | :--- |
| **Headers** | `# Header`, `## Subheading` | Use natural verbal transitions (e.g., "Moving on to...", "First, we must consider...", "On the other hand...") |
| **Bolding / Italics** | `**bold**`, `*italic*` | Plain text. Rely on word choice and syntax to convey emphasis rather than markdown tags. |
| **Lists & Bullets** | `- Bullet 1`, `1. Step One` | Narrative sequences: "First, you will want to... followed by... and finally, you must..." |
| **Code Blocks** | `code inline` or triple backtick blocks | Describe the logic or the command sequence in plain English rather than showing raw code syntax. |
| **Tables / Matrices** | Grid formats, columns, pipes | Summarize the key relationships, values, and trends in structured narrative sentences. |
| **Symbols & Units** | `%`, `&`, `$`, `°F`, `@`, `vs.`, `/` | Spell them out entirely: "percent", "and", "dollars", "degrees Fahrenheit", "at", "versus", "or". |

---

## 3. Spoken Attribution Protocol

Standard academic or digital citations (such as hyperlinks, bracketed references, or parenthetical sources) disrupt the rhythm of speech. You must translate all sources into **natural, in-line spoken attributions**.

### The Rule
- **No brackets, parentheses, or subscripts:** Do not output strings like `[1]`, `(Smith, 2024)`, or `[New York Times](https://nytimes.com)`.
- **In-sentence attribution only:** Integrate the identity of the source as a natural clause within your prose.

### Spoken Conversions
- *Instead of:* "Global temperatures rose significantly last year [NASA](url)."
- *Write:* "According to data released by NASA, global temperatures rose significantly last year."
- *Instead of:* "New clinical trials show promising results (see JAMA Internal Medicine, 2026)."
- *Write:* "In a study published by the Journal of the American Medical Association, clinical trials show promising results..."

---

## 4. Linguistic Optimization & Flow

TTS engines read text exactly as written. To make the listening experience highly engaging and easy to process:
1. **Calibrate Sentence Length:** Vary sentence lengths, but favor shorter, punchier clauses. Avoid excessively long, multi-layered compound sentences that exhaust a listener's working memory.
2. **Pronunciation Assistance:** Avoid complex, obscure, or highly technical jargon when a common, descriptive term can be used. If a complex acronym is necessary, spell it out phonetically if a TTS engine is likely to mispronounce it, or write it with hyphens if it should be read letter-by-letter (for example, "T-T-S" or "A-I").
3. **Transition Phrasing:** Use explicit logical signposts to guide the listener's attention. Since they cannot scan ahead visually, phrases like "On the contrary," "Specifically," or "To put this another way" act as vital cognitive roadmaps.
