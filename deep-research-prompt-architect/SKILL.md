---
name: deep-research-prompt-architect
description: Guides the programmatic construction of high-precision prompts for autonomous deep research agents. Enforces strict XML framing, negative boundaries, rigorous investigative question decomposition, anti-hallucination protocols, and explicit epistemic gap handling while banning persona fluff and information prescription. Use this skill whenever dispatching tasks to autonomous research sub-agents, orchestrating background multi-step research loops, compiling research directives, or automating deployment via the GitHub Deep Research Controller.
license: MIT
metadata:
  version: "1.1.0"
  architecture: "agent-skills-spec"
---

# Deep Research Prompt Architect

Use this skill whenever you need to formulate an execution prompt for an autonomous deep research agent or background multi-step retrieval pipeline. 

Your objective when invoking this skill is **not** to conduct the research yourself, nor to answer the query. Your sole objective is to engineer a rigorous, un-biased, and structurally constrained directive that steers an autonomous research agent into deep, verifiable, and multi-angled discovery.

---

## 1. Foundational Operating Invariants

When composing a prompt for a deep research agent, strictly enforce the following rules:

### A. Never Prescribe Information
- **Do not leak answers, conclusions, or your internal training priors into the prompt.** 
- If the research topic involves an open debate, an obscure technical bug, an empirical quantity, or an unfolding situation, do not state what you think the outcome is.
- Formulate every objective as an open investigative target or an adversarial hypothesis test (e.g., instead of writing *"Document how System X outperforms System Y due to lower latency,"* write *"Determine the empirical latency, throughput, and operational trade-offs between System X and System Y across varied workloads; identify conditions under which System Y outperforms System X"*).
- Prescribing conclusions biases the research agent's query generation, forcing it into confirmation-bias retrieval loops where it searches only for evidence supporting your seeded assertion.

### B. Never Direct the Research Agent to Interact with the End User
- The deep research agent is an asynchronous retrieval and analytical engine, **not** a conversational chatbot.
- **Never** write instructions such as *"Explain this clearly to the user,"* *"Format your response to be friendly and accessible,"* *"Ask the user follow-up questions,"* or *"Provide recommendations for the user's situation."*
- The research agent produces an unvarnished, high-density **technical research dossier** delivered directly back to your pipeline or context window. You are the entity that synthesizes or presents findings to the end user later; the research agent's only audience is the research task itself.

### C. Eliminate Persona Fluff and Roleplay
- **Never include persona priming phrases** like *"You are a world-class investigative researcher,"* *"Act as an expert data scientist,"* or *"You are a brilliant historian."*
- Roleplay priming wastes attention tokens, degrades calibration, and induces sycophantic narrative generation.
- Ground the model strictly through operational parameters, precise definitions of scope, algorithmic execution phases, and deterministic output schemas.

### D. Steer Through Granular Decomposition, Not Broad Commands
- Autonomous research agents degrade rapidly when given broad, shallow directives like *"Research quantum error correction."*
- Steer the agent by decomposing the overarching goal into 15 to 30+ highly technical, orthogonal sub-questions categorized under distinct investigative axes. Force the agent to investigate edge cases, counter-arguments, failure modes, historical divergences, and empirical benchmarks.

---

## 2. Standard XML Prompt Architecture

Every prompt you compile for an autonomous deep research agent must be encapsulated in structured XML tags. This creates an unambiguous choice architecture and cleanly isolates instructions from user queries and dynamic variables.

Structure the prompt using this exact hierarchy:

```xml
<research_directive>
  <context_and_scope>
    <!-- Define the subject boundaries, temporal boundaries, and operational horizon. -->
  </context_and_scope>

  <primary_objective>
    <!-- State the un-biased, core investigative mission in 1-3 direct sentences. -->
  </primary_objective>

  <negative_constraints>
    <!-- Hard behavioral boundaries, explicit exclusions, and anti-hallucination rules. -->
  </negative_constraints>

  <investigative_axes>
    <!-- Multi-dimensional decomposition: dense thematic axes packed with granular questions. -->
    <axis name="...">
      <!-- Targeted questions, edge cases, competing hypotheses, and specific metrics. -->
    </axis>
  </investigative_axes>

  <search_and_retrieval_protocol>
    <!-- Source hierarchy, verification requirements, triangulation procedures. -->
  </search_and_retrieval_protocol>

  <epistemic_gaps_protocol>
    <!-- Explicit instructions on what to do when information is unlocatable, contradictory, or absent. -->
  </epistemic_gaps_protocol>

  <required_dossier_structure>
    <!-- Exact structural markdown schema for the returned research artifact. -->
  </required_dossier_structure>
</research_directive>
```

---

## 3. Detailed Drafting Specifications

### 3.1. Negative Constraints Section

You must always generate an explicit `<negative_constraints>` block in the prompt. Include these mandatory restrictions:

1. **No Presumed Knowledge or Hallucinated Grounding:** The agent must never infer or fabricate dates, metrics, citations, version numbers, or legal quotes. Every factual assertion must link to an active retrieval result.
2. **No Conversational Preamble or Postamble:** Ban all conversational filler (e.g., *"Sure, here is the research you requested,"* *"In conclusion, I hope this helps"*). The response must start immediately with the first dossier header.
3. **No Redundant Superficiality:** Ban generic Wikipedia-style summaries, hand-waving generalities, and high-level listicles. Demand granular, code-level, clause-level, or data-level specificity.
4. **No Premature Search Termination:** Forbid the agent from terminating a search after a single query if primary sources disagree or if only secondary blogs have been consulted.
5. **No Speculative Synthesis Without Identification:** The agent must never blend confirmed empirical evidence with theoretical extrapolation without explicitly demarcating the boundary.

### 3.2. Instructions for Missing, Ambiguous, or Contradictory Information

A major failure mode of autonomous agents is fabricating answers or glossing over gaps when queries yield dry results. You must force the agent to rigorously audit negative space via the `<epistemic_gaps_protocol>`:

1. **Acknowledge the Vacuum:** If a metric, document, or confirmation cannot be found after exhaustive query permutation, the agent must document the search space covered and state: *"Exhaustive search yielded no public primary documentation."*
2. **Distinguish Non-Existence from Inaccessibility:** Explicitly differentiate between:
   - Evidence proving a claim is false / does not exist.
   - Information that is proprietary, classified, paywalled, or non-indexed.
   - Ambiguous or conflicting reporting among primary sources.
3. **Audit Search Dead Ends:** The agent must log specific failed query strings and explain why the data was unobtainable (e.g., *"API changelog omitted commit sha 4f1a9; internal schema undocumented in public SDK repos"*).
4. **Preserve Contradictions:** When two authoritative sources conflict, the agent must not force a compromise or pick a favorite. It must present the contradictory claims side-by-side, detail the pedigree and methodology of each source, and highlight the unresolved tension.

### 3.3. Specification of the Research Process

In `<search_and_retrieval_protocol>`, mandate a systematic multi-step verification sequence:

1. **Source Hierarchy:**
   - *Tier 1 (Authoritative Primary):* Peer-reviewed empirical literature, official statutory/regulatory text, SEC/regulatory filings, direct repository source code/commits, official vendor API documentation, signed institutional statements.
   - *Tier 2 (Secondary Corroborative):* Technical whitepapers, industry engineering blogs, investigative journalism with named sources, statistical clearinghouses.
   - *Tier 3 (Contextual / Tertiary):* Forum threads, community discussions, opinion editorials (use solely for discovering edge cases, practitioner sentiment, or unindexed bug reports, never as primary factual proof).
2. **Lateral Triangulation:** Require that non-trivial claims be verified by at least two independent Tier 1 or Tier 2 sources.
3. **Root-Cause Tracing:** When a secondary source quotes an original study, statement, or court case, instruct the agent to track down and inspect the original primary document rather than citing the secondary report.

---

## 4. Required Structure of the Final Research Dossier

To ensure the research agent outputs an artifact that is immediately parseable and dense with information, prescribe the following standardized report schema in `<required_dossier_structure>`:

```markdown
# RESEARCH DOSSIER: [TOPIC TITLE]

## 1. Executive Summary & Epistemic Matrix
- High-density factual digest (no conversational framing).
- Tabular Epistemic Ledger:
  | Claim / Finding | Verification Status (Verified / Contested / Unresolved) | Confidence Level (High / Med / Low) | Primary Source Type |

## 2. Granular Thematic Findings
[Subdivided by the designated investigative axes. Must detail technical mechanisms, exact figures, dates, verbatim statutory/technical nomenclature, and comparative trade-offs.]

## 3. Discrepancies, Dissenting Evidence & Competing Models
[Detailed inventory of points where authoritative sources diverge, including the underlying assumptions causing each divergence.]

## 4. Epistemic Gaps & Search Dead Ends
[Explicit ledger of data points that could not be verified, specific query angles that failed, and identification of proprietary or unindexed barriers.]

## 5. Primary Source Ledger
[Exhaustive index of consulted sources, accompanied by author/organization pedigree, publication date, and specific extraction coordinates.]
```

---

## 5. Step-by-Step Prompt Construction Workflow

When a topic is given to you, follow this internal sequence to author the research prompt:

1. **Deconstruct the Ingestion Target:** Identify the core technical, historical, legal, or empirical core of the topic.
2. **Formulate Orthogonal Investigative Axes:** Break the topic into 3–6 mutually exclusive, collectively exhaustive thematic tracks (e.g., Architectural Mechanics, Empirical Benchmarks, Vulnerability & Failure Modes, Legal/Regulatory Framework, Economic Trade-offs).
3. **Generate Dense Sub-Questions:** Under each axis, write 4–8 deeply specific, interrogative prompts. Focus on mechanisms ("how does X prevent Y"), thresholds ("at what latency does Z fail"), and edge cases ("what occurs during concurrent network partitions").
4. **Draft Negative Constraints:** Identify the most likely hallucination paths, generic cliches, or superficial traps for this specific topic, and explicitly forbid them.
5. **Inject Epistemic Protocols & Schema:** Append the mandatory handling rules for unlocatable data and wrap everything in valid XML tags.

---

## 6. Complete Production-Grade Reference Blueprint

Below is a complete, illustrative example of a prompt written according to this skill:

```xml
<research_directive>
  <context_and_scope>
    Investigation target: The technical, operational, and financial trade-offs of deploying Local Key Management Systems (KMS) with Hardware Security Modules (HSMs) vs. Cloud-Native Managed KMS (AWS KMS / Google Cloud KMS) in high-throughput financial transaction processing environments.
    Temporal scope: 2024 to present architectures.
  </context_and_scope>

  <primary_objective>
    Conduct an exhaustive, evidence-based investigation into the latency profiles, compliance boundaries, total cost of ownership (TCO), and cryptographic failure modes distinguishing on-premises dedicated HSMs from cloud-native managed KMS architectures.
  </primary_objective>

  <negative_constraints>
    - Never include conversational filler, meta-announcements, or introductory pleasantries.
    - Never write for a consumer or non-technical audience; format exclusively as a rigorous engineering and cryptographic research dossier.
    - Do not state generalities such as "cloud is usually cheaper" or "on-prem offers more control." Provide exact network hop overhead, throughput thresholds, and pricing formulas.
    - Forbid reliance on vendor promotional marketing brochures unless corroborated by third-party latency audits or independent cryptographic reviews.
  </negative_constraints>

  <investigative_axes>
    <axis name="1. Latency & Throughput Envelope">
      - What are the documented P50, P99, and P99.9 latency figures for AES-256 envelope decryption operations over dedicated on-premises PCIe/network HSMs (e.g., Thales payShield 10K, Luna PCIe) compared to AWS KMS and Google Cloud KMS via DirectConnect / Cloud Interconnect?
      - How do round-trip network hops, TLS negotiation, and IAM authorization checks within cloud KMS impact transaction throughput per second (TPS) during burst events exceeding 50,000 TPS?
      - What caching or envelope encryption patterns (e.g., local data key caching with cryptographic wear-out limits) are deployed to mitigate cloud KMS latency, and what are their specific security trade-offs?
    </axis>

    <axis name="2. Regulatory Compliance & Cryptographic Boundaries">
      - Contrast FIPS 140-2/140-3 Level 3 physical and logical boundary requirements with FIPS 140-2 Level 2 shared multi-tenant cloud hardware environments.
      - How do PCI DSS v4.0 Requirement 3 and PCI PIN Security guidelines treat key custodian dual-control and split-knowledge procedures across cloud KMS vs. dedicated on-premises HSMs?
      - Under what exact regulatory conditions (e.g., FedRAMP High, DORA, regional data sovereignty statutes) is a purely cloud-native KMS legally disqualifying for Tier 1 transaction clearers?
    </axis>

    <axis name="3. Failure Modes, Redundancy, and Cryptographic Drift">
      - Document historical public outage post-mortems involving cloud KMS service degradation and their cascading impacts on authorization pipelines.
      - What are the operational failure modes of physical HSM battery depletion, firmware zeroization, and synchronization drift in dual-datacenter topologies?
      - How is key destruction (cryptographic erasure) audited and provably verified across multitenant cloud object storage compared to physical degaussing/sanitization?
    </axis>
  </investigative_axes>

  <search_and_retrieval_protocol>
    - Prioritize NIST publications, FIPS certification validation certificates, PCI Security Standards Council guidance, vendor security whitepapers, and engineering post-mortems.
    - Every quantitative latency claim must specify benchmark hardware, test harness setup, and network topology.
    - Triangulate cloud performance numbers across multiple independent engineering benchmarks and post-mortems.
  </search_and_retrieval_protocol>

  <epistemic_gaps_protocol>
    - If specific sub-millisecond benchmarking figures for proprietary bank setups are not publicly released, explicitly document the absence of open disclosure rather than interpolating synthetic numbers.
    - For proprietary cloud KMS hardware details (such as custom Nitro-enclosed security chips), distinguish publicly confirmed whitepaper disclosures from independent third-party reverse engineering.
    - Log any search pathways that yielded exclusively SEO-marketing landing pages without verifiable technical data.
  </epistemic_gaps_protocol>

  <required_dossier_structure>
    Follow the 5-part research dossier schema:
    1. Executive Summary & Epistemic Matrix (with verification status and confidence scores)
    2. Granular Thematic Findings (by investigative axis)
    3. Discrepancies, Dissenting Evidence & Competing Models
    4. Epistemic Gaps & Search Dead Ends
    5. Primary Source Ledger
  </required_dossier_structure>
</research_directive>
```

---

## 7. Automated Deployment via GitHub Deep Research Controller (Optional)

The environment may provide the **GitHub Deep Research Controller** tool suite connected to the `IsaiahDePorto/Deepresearch` repository.

### 7.1. Execution Precondition & Intent Rules
- **NEVER assume tool execution by default.** If the user asks to "write a prompt", "design a research directive", or "review research questions", output the structured XML prompt directly into the conversation. DO NOT invoke any controller tools unless specifically instructed.
- **Trigger Condition (Full Deployment):** If and only if the user explicitly instructs you to push/deploy the prompt AND launch/run the research (e.g., *"push the prompt and start the research"*, *"deploy this to GitHub and trigger the deep research"*, *"send it through the plugin to replace the prompt and kick off the workflow"*), execute the **two-stage tool chain** in direct sequential order:

```
[User instructs: Deploy & Run]
          │
          ▼
1. overwrite_prompt_markdown(commit_message, content)
          │
          ▼ (Verify commit success)
2. trigger_deep_research_action(branch="main")
          │
          ▼
Report commit SHA & GitHub Actions monitoring link to user
```

### 7.2. Tool Execution Specifications

1. **Stage 1: Commit Directive (`overwrite_prompt_markdown`)**
   - **Target File:** Overwrites `Prompt.md` in `IsaiahDePorto/Deepresearch` on branch `main`.
   - **Parameter `content`:** The complete, unescaped markdown document containing the generated `<research_directive>` XML block.
   - **Parameter `commit_message`:** A precise, semantic Git commit message describing the target topic (e.g., `feat: deploy deep research directive for [topic]`).

2. **Stage 2: Workflow Dispatch (`trigger_deep_research_action`)**
   - **Target Workflow:** `deep_research.yml` ("Deep Research") on branch `main`.
   - **Invocation:** Execute immediately after Stage 1 resolves successfully. Do not wait for user prompting between stages when full deployment was requested.

3. **Stage 3: Verification & Reporting**
   - Confirm both calls succeeded.
   - Provide the user with the direct link to the commit and the GitHub Actions dashboard: `https://github.com/IsaiahDePorto/Deepresearch/actions`.
```
