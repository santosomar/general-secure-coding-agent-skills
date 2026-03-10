---
name: ambiguity-detector
description: Detects ambiguous, contradictory, or underspecified language in requirements that could lead to misinterpretation. Use when reviewing requirements for clarity, when the user asks for ambiguity detection, or when preparing requirements for formalization.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "requirement-enhancer, nl-to-constraints"
---

# Ambiguity Detector

## Purpose

Detect ambiguous, contradictory, or underspecified language in requirements.

## Workflow

1. **Input**: Requirements document(s).
2. **Analyze**: Check for vague terms, contradictions, missing cases.
3. **Flag**: List ambiguities with location and suggestion.
4. **Report**: Categorized findings (ambiguity, contradiction, underspec).

## Output

- List of ambiguities with location
- Contradictions if any
- Suggestions for clarification
