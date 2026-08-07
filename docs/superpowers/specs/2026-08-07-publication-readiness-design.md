# Publication Readiness Design

## Goal

Increase installation confidence, GitHub discoverability, and contribution readiness for `math-tutor` without changing the existing knowledge-graph content.

## Scope

- Add a bilingual Chinese-English README landing section and retain Chinese as the detailed documentation language.
- Add clear licensing, contribution guidance, issue templates, and automated local data validation.
- Correct inaccurate or stale documentation references.
- Improve `SKILL.md` frontmatter discoverability while preserving its current local-only runtime constraint.

## Non-goals

- No changes to existing topic, dependency, cluster, or curriculum-standard content.
- No new frontend features and no replacement of the existing visualizations.
- No claim that unverified third-party agent platforms support a particular installation path.
- No automated publishing, remote Git operations, or changes to the user's current uncommitted edits beyond the agreed documentation updates.

## README Experience

The README begins with a bilingual value proposition: an offline AI tutoring skill and prerequisite knowledge graph for PEP 2022 junior-high mathematics. The top section includes four factual metrics: 169 micro-concepts, 237 prerequisite edges, six textbooks, and offline operation.

A concise example demonstrates the user outcome: an incorrect classification of a non-repeating decimal is traced to prerequisite understanding and leads to an ordered remediation sequence with adaptive practice. The example is illustrative rather than a claim of live execution.

The detailed feature, curriculum, data-model, and visualization sections remain Chinese. The obsolete `visualization_8up_simple.html` references are changed to the repository's actual `visualization_8up.html`; email placeholders and unverified platform-specific installation claims are removed. Questions and suggestions are directed to GitHub Issues.

## Skill Discovery

`skills/math-tutor/SKILL.md` frontmatter uses a concise `Use when ...` description. It lists observable triggers in English and Chinese, including junior-high math tutoring, incorrect-answer diagnosis, adaptive exercise generation, self-study paths, prerequisite tracing, and relevant Chinese phrases. The body retains the local-data and no-network constraints.

## Repository Trust and Contribution

- `LICENSE` states the license applicable to original repository material and describes that third-party materials remain under their own terms. The recommended repository-wide license is CC BY-SA 4.0 because this package is primarily educational data and documentation.
- `CONTRIBUTING.md` explains the focused contribution flow: discuss larger additions in an issue, keep source attribution, run validation, and submit small reviewable pull requests.
- Two issue templates gather reproducible reports for knowledge-graph data corrections and feature requests.

## Validation

A dependency-free Python validator checks every grade directory:

1. All expected JSON files parse as UTF-8 JSON.
2. Topic IDs are unique.
3. Dependency endpoints refer to existing topic IDs in their grade data.
4. The directed prerequisite graph contains no cycle.
5. The grade-level `manifest.json` statistics match actual topic, dependency, and cluster counts when the manifest exposes those fields.

A GitHub Actions workflow runs this validator for pushes and pull requests. Test fixtures include one valid mini-dataset and one dataset with a dependency cycle, so the validator has a failing and passing automated test path.

## Acceptance Criteria

- The README begins with a bilingual conversion-focused landing section and uses no stale visualization filename, email placeholder, or direct-push wording.
- `SKILL.md` frontmatter begins with `Use when` and contains the agreed discovery triggers.
- A root `LICENSE`, contribution guide, and two issue templates exist.
- `python scripts/validate_data.py` exits successfully on repository data.
- Automated tests demonstrate valid data passes and a cyclic graph fails.
- GitHub Actions invokes the validator on `push` and `pull_request`.
