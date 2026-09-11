---
name: novel-subtopic-explorer
description: Identifies, verifies, and explores previously undiscussed subtopics within a broader subject area when the user requests something new or unexamined. Enforces a multi-stage negative filtering protocol using MemoryPlugin and past chat history tools (recall_chat_history, search_memories) to guarantee topics have not been substantively explored in previous conversations.
compatibility: Requires active MemoryPlugin tools (recall_chat_history, search_memories, get_memories_and_buckets)
metadata:
  version: "1.0.0"
  author: "agent-architecture"
---

# Operational Guide: Novel Subtopic Exploration & Verification

## 1. Overview & Trigger Conditions

Activate this skill whenever:
1. The user requests a topic, deep dive, or subtopic they have **not discussed before** with the agent (e.g., *"Tell me something we haven't discussed before regarding urban planning in detail"* or *"Give me a fresh angle on cognitive science we haven't covered"*).
2. The user requests novelty within an established field of interest while **MemoryPlugin** tools are available.

The core objective of this skill is **negative verification**: rather than assuming an LLM's internal recall is accurate, the model must systematically probe conversation archives, map previously covered semantic ground, eliminate known territory, generate candidates, and independently verify each candidate against past chats prior to selection.

---

## 2. Core Execution Protocol

The workflow follows a 4-phase deterministic pipeline:

```
[User Request for Novel Topic]
           │
           ▼
[Phase 1: Broad Domain Deconstruction & 3+ Memory Queries]
           │
           ▼
[Phase 2: Archive Triage & Candidate Generation (2–3 Subtopics)]
           │
           ▼
[Phase 3: Targeted Verification Probes (1 Probe per Candidate)]
           │
           ├──────────────────────────────┐
           ▼                              ▼
[All Candidates Eliminated]      [≥1 Candidate Passes]
           │                              │
           ▼                              ▼
[Loop Re-entry: Regenerate      [Phase 4: Final Selection
 Candidates (Skip Phase 1)]      & Detailed In-Depth Response]
```

---

### Phase 1: Context Deconstruction & Multi-Query Baseline Probing

1. **Analyze Context & Scope:**
   - Identify the macro-domain specified by the user (e.g., Urban Planning, Neurobiology, Distributed Systems).
   - Deconstruct the domain into its fundamental sub-disciplines, common conversational tangents, and adjacent terminology.

2. **Execute Minimum of 3 Distinct Memory Probes:**
   - The agent MUST issue at least **three separate, non-overlapping queries** to the memory system (`recall_chat_history` or `search_memories`).
   - Query 1: Direct macro-topic query targeting central discourse.
   - Query 2: Core structural or institutional sub-domain query.
   - Query 3: Concrete, practical, or edge application query.
   *Do not concatenate these into a single query; run distinct search calls to uncover disparate chat threads.*

---

### Phase 2: Memory Triage & Initial Candidate Generation

1. **Classify Retrieved Memory Evidence:**
   Analyze every excerpt, transcript snippet, and stored memory returned from Phase 1. Categorize all mentioned concepts into two tiers:
   - **Substantively Discussed (ELIMINATED):** The concept was the primary focus of an explanation, received dedicated analytical paragraphs, involved multi-turn dialogue, or was evaluated in depth.
   - **Idle Mention / In Passing (PERMISSIBLE):** The concept was merely listed as an example in a bullet point, mentioned off-hand in a sentence without follow-up, or appeared strictly as an analogy. These remain eligible, though entirely unmentioned topics are prioritized.

2. **Generate Candidate Subtopics:**
   - Formulate candidate subtopics within the macro-domain that sit completely outside the "Substantively Discussed" zone.
   - Select a minimum of **2 to 3 distinct candidate subtopics** that appear absent or only idly referenced in the Phase 1 search results.

---

### Phase 3: Targeted Per-Candidate Verification Probes

Before presenting or selecting any subtopic, the agent MUST run an independent verification check on **each individual candidate**:

1. **Individual Targeted Search:**
   - For candidate subtopic $A$, call `recall_chat_history(query="[specific terminology for subtopic A]")`.
   - For candidate subtopic $B$, call `recall_chat_history(query="[specific terminology for subtopic B]")`.
   - For candidate subtopic $C$, call `recall_chat_history(query="[specific terminology for subtopic C]")`.

2. **Strict Verification Audit:**
   - Inspect the raw transcript results returned for each subtopic.
   - If the specific query returns prior chat history showing prior substantive analysis, **immediately eliminate that candidate**.
   - A candidate is deemed **VERIFIED NOVEL** if and only if:
     - The tool returns zero relevant conversation hits, OR
     - The results show only an incidental, single-phrase mention without conceptual unpacking.

---

### Phase 4: Resolution or Loop Re-entry

- **Case A: One or more candidates pass verification:**
  - Select the strongest, highest-depth candidate among the verified options (or present the vetted shortlist if the user requested choices).
  - Proceed directly to deliver the deep-dive analysis requested by the user.

- **Case B: All 2–3 candidates fail verification (all were previously discussed):**
  - **Do NOT re-run Phase 1.** (The broad baseline is already established in your working context).
  - Return immediately to Phase 2: synthesize **2 to 3 completely new candidate subtopics** in alternate branches of the macro-domain.
  - Re-execute Phase 3 (individual targeted verification calls) for the new batch.
  - Repeat until at least one candidate successfully passes the verification threshold.

---

## 3. Heuristic Standards: "Idle Mention" vs. "Substantive Discussion"

To maintain rigorous epistemic boundaries without prematurely disqualifying every noun ever mentioned in chat history, apply these definitions:

| Feature | Idle Mention (Eligible) | Substantive Discussion (Disqualified) |
| :--- | :--- | :--- |
| **Turn Depth** | Present in 1 turn, never referenced again. | Spans 2+ turns or forms the subject of an extensive single-turn essay. |
| **Contextual Role** | Syntactic filler, parenthetical note, or item in a broad illustrative list. | The direct object of an inquiry, breakdown, comparison, or problem-solving step. |
| **Mechanistic Detail** | Named without explaining how it functions or why it matters. | Underlying mechanics, trade-offs, historical background, or data points were articulated. |

---

## 4. End-to-End Walkthrough Example

### User Query:
> *"Tell me something we haven't discussed before regarding urban planning in detail."*

### Execution Sequence:

1. **Phase 1: Broad Memory Probing (3 Calls):**
   - Call 1: `recall_chat_history(query="urban planning cities infrastructure design")`
   - Call 2: `recall_chat_history(query="American street design transit stroads walkability")`
   - Call 3: `recall_chat_history(query="zoning laws single family housing land use reform")`

2. **Phase 2: Evidence Audit & Candidate Formulation:**
   - *Audit of Call Results:*
     - Call 1 returned discussions on "15-minute cities" and "congestion pricing in NYC" (Substantively Discussed $\rightarrow$ **Disqualified**).
     - Call 2 returned multi-turn debates on "stroad remediation" and "curb extensions" (Substantively Discussed $\rightarrow$ **Disqualified**).
     - Call 3 returned discussions on "parking minimums" and "accessory dwelling units" (Substantively Discussed $\rightarrow$ **Disqualified**). An idle mention of "district heating utilities" appeared in a single bullet point.
   - *Candidate Generation:*
     - Candidate 1: *District geothermal heating & pneumatic solid waste networks* (Idly mentioned, never detailed).
     - Candidate 2: *Superblock (Superilles) modal filtering mechanics in Barcelona*.
     - Candidate 3: *Subsurface utility tunneling (utilidors) vs. direct burial economics*.

3. **Phase 3: Targeted Verification Probes (3 Individual Calls):**
   - Call 1: `recall_chat_history(query="pneumatic waste collection Envac district heating")` $\rightarrow$ Result: 0 matches. (**PASSED**)
   - Call 2: `recall_chat_history(query="Barcelona superblocks superilles modal filter")` $\rightarrow$ Result: Found a 3-turn discussion from 2 months ago analyzing Poblenou's superblocks. (**DISQUALIFIED**)
   - Call 3: `recall_chat_history(query="utilidor utility tunnel subsurface urban infrastructure")` $\rightarrow$ Result: 0 matches. (**PASSED**)

4. **Phase 4: Delivery:**
   - Candidates 1 and 3 are verified novel.
   - Agent selects Candidate 1 (or 3) and begins the comprehensive, high-resolution breakdown.

---

## 5. Safety & Operational Constraints

1. **Zero Hallucinated Novelty:** Never say *"As we haven't discussed X before..."* without executing the verification calls. The verification trail in the tool log is the sole ground truth.
2. **Deterministic Query Differentiation:** Phase 1 queries must target divergent terminology to prevent redundant retrieval of the same top 5 memories.
3. **No Infinite Loops:** If candidate generation loops 4 consecutive times without finding an untouched topic, explicitly surface the boundary to the user: outline the areas already covered according to chat records, present the near-novel candidates, and allow the user to select the directional boundary.
