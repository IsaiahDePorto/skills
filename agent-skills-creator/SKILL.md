---
name: agent-skills-creator
description: Guides the design, formatting, and optimization of custom Agent Skills in full compliance with the Agent Skills system specification. Instructs on directory structural rules, YAML frontmatter constraints, progressive disclosure philosophies, and relative file links. Provides clean formatting blueprints and verification standards.
---

# Creating Custom Agent Skills

This skill instructs the agent on how to write and organize production-ready custom Agent Skills. The Agent Skills specification provides a modular, progressive structure that allows LLMs to load and use contextual instructions, templates, and reference materials on demand, optimizing overall performance and context limits.

## 1. Directory Structure

A compliant skill must reside inside a dedicated parent directory, containing at a minimum a `SKILL.md` instruction file at its root. 

```
skill-name/
├── SKILL.md          # Mandatory: metadata frontmatter + core system instructions
├── scripts/          # Optional: executable scripts (Python, JS, Bash) used by the agent
├── references/       # Optional: focused reference documents (.md, .txt, .json) loaded on demand
└── assets/           # Optional: templates, configurations, graphics, static data sheets
```

---

## 2. `SKILL.md` File Format

The `SKILL.md` file consists of two mandatory components separated by frontmatter boundaries:
1. **YAML Frontmatter:** Delimited by triple hyphens (`---`). Defines name, capabilities, and system compatibility.
2. **Markdown Body:** Full markdown text containing detailed operational steps, execution criteria, edge cases, and lookup instructions.

### Frontmatter Schema Definition

| Field | Required | Constraints |
| :--- | :--- | :--- |
| `name` | Yes | 1-64 characters. Must match the parent directory exactly. Only lowercase letters (`a-z`), numbers (`0-9`), and hyphens (`-`) allowed. Cannot start/end with a hyphen, nor contain double hyphens (`--`). |
| `description` | Yes | 1-1024 characters. Must be a dense, descriptive summary detailing what the skill does and when the agent should activate it. |
| `license` | No | Short name or pointer to a bundled license document (e.g., `MIT`, `Apache-2.0`). |
| `compatibility` | No | Max 500 characters. Environment prerequisites (e.g., `Node.js v20+`, `Python 3.12`, `Internet Access`). |
| `metadata` | No | Arbitrary key-value map containing secondary attributes (e.g., version, author). |
| `allowed-tools` | No | Space-separated list of pre-approved system tools (experimental). |

#### Code Block Example
```yaml
---
name: sample-data-processor
description: Extracts and tidies unstructured financial data matrices. Supports Excel spreadsheets, CSVs, and PDF ledger reports. Use this skill when analyzing income statements, generating tax calculations, or processing raw tables.
license: MIT
metadata:
  version: "1.2.4"
  author: "dev-ops-team"
---
```

---

## 3. Core Architectural Philosophies

### Progressive Disclosure
To prevent overwhelming the LLM's active window context, split skills up by detail:
1. **Metadata Phase:** The system loads only the `name` and `description` fields at startup to register capabilities. Keep descriptions search-optimized.
2. **Instruction Phase:** The complete `SKILL.md` body is loaded only when the agent decides to activate that specific skill. Keep the main body concise (recommended under 500 lines).
3. **Resource Phase:** Detailed checklists, API references, or execution scripts are stored in subfolders (`references/` or `scripts/`) and loaded by the agent on demand.

### File Reference Rules
When referring to supporting resources, always use **relative paths** pointing outward from the skill root directory. Keep paths shallow:

```markdown
Run the verification compiler located at:
scripts/verify-syntax.py

For full configuration constraints, refer to:
references/SCHEMA-REGISTRY.md
```

---

## 4. Verification Checklists

Before saving a new custom skill, ensure you satisfy these conditions:
1. **Directory Matching:** Is the directory name identical to the frontmatter `name` string?
2. **Character Constraints:** Does the `name` contain any capital letters, underscores, or consecutive hyphens? (If yes, replace them with singular lowercase hyphens).
3. **Description Length:** Is your frontmatter description under 1024 characters? Does it contain descriptive verbs telling the agent *when* and *why* to load the skill?
4. **No Deep Nesting:** Are supporting resource files kept exactly one folder deep inside `references/`, `scripts/`, or `assets/`?

---

## 5. Blank Template Blueprint

Use this template as your default starting point:

```markdown
---
name: [lowercase-alphanumeric-name-with-hyphens]
description: [Dense description under 1024 characters detailing what the skill does and exact triggers explaining when to load it]
license: [Optional License Name]
compatibility: [Optional Environment Prerequisites]
metadata:
  version: "1.0.0"
---

# Operational Guide

## 1. Objectives
- Clear list of capabilities.

## 2. Step-by-Step Instructions
- Standard operation procedures.
- Command-line examples or API specifications.

## 3. Edge Cases & Safety Constraints
- Common issues.
- Expected fallbacks and error-handling conditions.
```
```
