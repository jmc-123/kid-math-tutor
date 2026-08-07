# Publication Readiness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `math-tutor` easier to discover, trust, install, and contribute to before public release.

**Architecture:** Keep the repository dependency-free. A standard-library Python validator owns data-integrity checks; `unittest` fixtures prove its pass and fail paths; GitHub Actions invokes the same command. Documentation is split by purpose: README for conversion and use, CONTRIBUTING for collaboration, LICENSE for rights.

**Tech Stack:** Markdown, JSON, Python 3.11 standard library, GitHub Actions.

## Global Constraints

- Preserve user-authored uncommitted changes in `README.md` and `skills/math-tutor/SKILL.md`; reread both immediately before modifying them.
- Do not alter files under `skills/math-tutor/data/**`.
- Do not add third-party runtime dependencies.
- README uses bilingual Chinese-English only in the landing section; detailed documentation remains Chinese.
- Do not make claims about platform-specific skill installation unless verified in the repository.
- The validator reads only `skills/math-tutor/data/{7-up,7-down,8-up,8-down,9-up,9-down}`.

---

### Task 1: Add data-validator test coverage

**Files:**
- Create: `tests/test_validate_data.py`
- Create: `scripts/__init__.py`
- Test: `tests/test_validate_data.py`

**Interfaces:**
- Produces: `scripts.validate_data.validate_dataset(dataset_dir: pathlib.Path) -> list[str]`
- Produces: `scripts.validate_data.main(argv: Sequence[str] | None = None) -> int`

- [ ] **Step 1: Write failing tests for valid data and a cyclic dependency graph**

```python
from __future__ import annotations

import json
import tempfile
import unittest
from pathlib import Path

from scripts.validate_data import validate_dataset


def write_dataset(directory: Path, dependencies: list[dict[str, str]]) -> None:
    (directory / "topics.json").write_text(
        json.dumps({"topics": [{"id": "a"}, {"id": "b"}]}), encoding="utf-8"
    )
    (directory / "dependencies.json").write_text(
        json.dumps({"dependencies": dependencies}), encoding="utf-8"
    )
    (directory / "clusters.json").write_text(json.dumps({"clusters": []}), encoding="utf-8")
    (directory / "manifest.json").write_text(
        json.dumps({"stats": {"topics": 2, "dependencies": len(dependencies), "clusters": 0}}),
        encoding="utf-8",
    )


class ValidateDatasetTests(unittest.TestCase):
    def test_accepts_valid_acyclic_dataset(self) -> None:
        with tempfile.TemporaryDirectory() as temporary_directory:
            dataset_dir = Path(temporary_directory)
            write_dataset(dataset_dir, [{"topicId": "b", "prerequisiteId": "a"}])
            self.assertEqual(validate_dataset(dataset_dir), [])

    def test_reports_dependency_cycle(self) -> None:
        with tempfile.TemporaryDirectory() as temporary_directory:
            dataset_dir = Path(temporary_directory)
            write_dataset(
                dataset_dir,
                [
                    {"topicId": "a", "prerequisiteId": "b"},
                    {"topicId": "b", "prerequisiteId": "a"},
                ],
            )
            errors = validate_dataset(dataset_dir)
            self.assertTrue(any("cycle" in error for error in errors), errors)
```

- [ ] **Step 2: Run the tests to verify the expected RED state**

Run: `python -m unittest tests.test_validate_data -v`

Expected: import failure because `scripts.validate_data` does not exist.

- [ ] **Step 3: Add the package marker only**

Create `scripts/__init__.py` as an empty file.

- [ ] **Step 4: Commit the failing-test baseline**

Run:

```text
git add tests/test_validate_data.py scripts/__init__.py
git commit -m "test: define data validation expectations"
```

### Task 2: Implement the dependency-free data validator

**Files:**
- Create: `scripts/validate_data.py`
- Modify: `tests/test_validate_data.py`
- Test: `tests/test_validate_data.py`

**Interfaces:**
- Consumes: grade directories containing `topics.json`, `dependencies.json`, `clusters.json`, and `manifest.json`.
- Produces: a list of human-readable validation errors per dataset and process exit status `0` on success or `1` on validation failure.

- [ ] **Step 1: Implement JSON loading, ID/reference checks, count checks, and cycle detection**

```python
from __future__ import annotations

import argparse
import json
from collections import defaultdict, deque
from pathlib import Path
from typing import Sequence

GRADE_NAMES = ("7-up", "7-down", "8-up", "8-down", "9-up", "9-down")
REQUIRED_FILES = ("topics.json", "dependencies.json", "clusters.json", "manifest.json")


def load_json(path: Path) -> object:
    with path.open("r", encoding="utf-8") as source:
        return json.load(source)


def validate_dataset(dataset_dir: Path) -> list[str]:
    errors: list[str] = []
    for filename in REQUIRED_FILES:
        if not (dataset_dir / filename).is_file():
            errors.append(f"{dataset_dir}: missing {filename}")
    if errors:
        return errors

    try:
        topics_document = load_json(dataset_dir / "topics.json")
        dependencies_document = load_json(dataset_dir / "dependencies.json")
        clusters_document = load_json(dataset_dir / "clusters.json")
        manifest_document = load_json(dataset_dir / "manifest.json")
    except (OSError, json.JSONDecodeError) as error:
        return [f"{dataset_dir}: invalid JSON: {error}"]

    topics = topics_document.get("topics", []) if isinstance(topics_document, dict) else []
    dependencies = (
        dependencies_document.get("dependencies", []) if isinstance(dependencies_document, dict) else []
    )
    clusters = clusters_document.get("clusters", []) if isinstance(clusters_document, dict) else []
    if not all(isinstance(value, list) for value in (topics, dependencies, clusters)):
        return [f"{dataset_dir}: topics, dependencies, and clusters must be arrays"]

    topic_ids = [topic.get("id") for topic in topics if isinstance(topic, dict)]
    if len(topic_ids) != len(topics) or any(not isinstance(topic_id, str) or not topic_id for topic_id in topic_ids):
        errors.append(f"{dataset_dir}: every topic requires a non-empty string id")
    if len(topic_ids) != len(set(topic_ids)):
        errors.append(f"{dataset_dir}: topic ids must be unique")

    known_ids = set(topic_ids)
    graph: dict[str, list[str]] = defaultdict(list)
    in_degree = {topic_id: 0 for topic_id in known_ids}
    for index, dependency in enumerate(dependencies):
        if not isinstance(dependency, dict):
            errors.append(f"{dataset_dir}: dependency {index} must be an object")
            continue
        topic_id = dependency.get("topicId")
        prerequisite_id = dependency.get("prerequisiteId")
        if topic_id not in known_ids or prerequisite_id not in known_ids:
            errors.append(f"{dataset_dir}: dependency {index} references an unknown topic")
            continue
        graph[prerequisite_id].append(topic_id)
        in_degree[topic_id] += 1

    queue = deque(topic_id for topic_id, degree in in_degree.items() if degree == 0)
    visited = 0
    while queue:
        current = queue.popleft()
        visited += 1
        for next_topic in graph[current]:
            in_degree[next_topic] -= 1
            if in_degree[next_topic] == 0:
                queue.append(next_topic)
    if known_ids and visited != len(known_ids):
        errors.append(f"{dataset_dir}: prerequisite graph contains a cycle")

    stats = manifest_document.get("stats", {}) if isinstance(manifest_document, dict) else {}
    if isinstance(stats, dict):
        actual_counts = {"topics": len(topics), "dependencies": len(dependencies), "clusters": len(clusters)}
        for field, actual in actual_counts.items():
            declared = stats.get(field)
            if declared is not None and declared != actual:
                errors.append(f"{dataset_dir}: manifest stats.{field} is {declared}, expected {actual}")
    return errors


def main(argv: Sequence[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Validate math-tutor knowledge-graph data.")
    parser.add_argument("--data-root", type=Path, default=Path("skills/math-tutor/data"))
    arguments = parser.parse_args(argv)
    errors: list[str] = []
    for grade_name in GRADE_NAMES:
        errors.extend(validate_dataset(arguments.data_root / grade_name))
    if errors:
        print("Knowledge graph validation failed:")
        print("\n".join(f"- {error}" for error in errors))
        return 1
    print(f"Validated {len(GRADE_NAMES)} grade datasets successfully.")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Run unit tests and verify GREEN**

Run: `python -m unittest tests.test_validate_data -v`

Expected: both tests pass.

- [ ] **Step 3: Validate all real grade datasets**

Run: `python scripts/validate_data.py`

Expected: `Validated 6 grade datasets successfully.` and exit code 0. If a manifest has intentional data differences, correct the manifest rather than weakening the validator.

- [ ] **Step 4: Commit the validator**

Run:

```text
git add scripts/validate_data.py scripts/__init__.py tests/test_validate_data.py
git commit -m "feat: validate knowledge graph data"
```

### Task 3: Run data validation in GitHub Actions

**Files:**
- Create: `.github/workflows/validate-data.yml`

**Interfaces:**
- Consumes: repository checkout on `push` and `pull_request`.
- Produces: a status check named `Validate knowledge graph data`.

- [ ] **Step 1: Create the workflow**

```yaml
name: Validate knowledge graph data

on:
  push:
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Validate datasets
        run: python scripts/validate_data.py
      - name: Run validator tests
        run: python -m unittest tests.test_validate_data -v
```

- [ ] **Step 2: Validate YAML and local command equivalence**

Run: `python scripts/validate_data.py; python -m unittest tests.test_validate_data -v`

Expected: validator reports six datasets and all `unittest` cases pass.

- [ ] **Step 3: Commit workflow configuration**

Run:

```text
git add .github/workflows/validate-data.yml
git commit -m "ci: validate knowledge graph data"
```

### Task 4: Add licensing, contribution guidance, and issue templates

**Files:**
- Create: `LICENSE`
- Create: `CONTRIBUTING.md`
- Create: `.github/ISSUE_TEMPLATE/data-correction.yml`
- Create: `.github/ISSUE_TEMPLATE/feature-request.yml`

**Interfaces:**
- Consumes: GitHub issue forms and contributor pull requests.
- Produces: clear licensing boundaries and structured feedback.

- [ ] **Step 1: Add CC BY-SA 4.0 license text**

Use the unmodified Creative Commons Attribution-ShareAlike 4.0 International legal code in `LICENSE`. Do not rewrite the legal text.

- [ ] **Step 2: Add contributor guide**

`CONTRIBUTING.md` must contain these sections: `Before you start`, `Data changes`, `Documentation and skill changes`, `Validation`, and `Pull requests`. Require contributors to cite the source of new or corrected content, keep pull requests focused, and run both commands:

```text
python scripts/validate_data.py
python -m unittest tests.test_validate_data -v
```

- [ ] **Step 3: Add data-correction issue form**

```yaml
name: Data correction
about: Report an incorrect, missing, or inconsistent knowledge-graph entry
labels: [data]
body:
  - type: input
    id: grade
    attributes:
      label: Grade dataset
      placeholder: 8-up
    validations:
      required: true
  - type: input
    id: topic
    attributes:
      label: Topic ID or name
    validations:
      required: true
  - type: textarea
    id: correction
    attributes:
      label: Proposed correction and source
      description: Include the textbook section, curriculum standard, or other verifiable source.
    validations:
      required: true
```

- [ ] **Step 4: Add feature-request issue form**

```yaml
name: Feature request
about: Suggest an improvement to the tutoring skill, data tooling, or visualization
labels: [enhancement]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem to solve
    validations:
      required: true
  - type: textarea
    id: proposal
    attributes:
      label: Proposed change
    validations:
      required: true
  - type: dropdown
    id: audience
    attributes:
      label: Primary audience
      options: [Student, Parent, Teacher, Developer]
    validations:
      required: true
```

- [ ] **Step 5: Check files and commit**

Run: `git diff --check; git status --short`

Expected: no whitespace errors; only the four new community files are staged for this task.

Run:

```text
git add LICENSE CONTRIBUTING.md .github/ISSUE_TEMPLATE
git commit -m "docs: add contribution and issue guidance"
```

### Task 5: Improve skill discovery metadata

**Files:**
- Modify: `skills/math-tutor/SKILL.md:1-4`

**Interfaces:**
- Produces: an Agent Skills-compatible frontmatter description with observable triggers.

- [ ] **Step 1: Preserve the user’s current change and replace only the description value**

Reread the first 10 lines before editing. Set the frontmatter description to:

```yaml
description: "Use when tutoring Chinese junior-high mathematics, diagnosing incorrect answers, generating adaptive exercises, planning self-study paths, or tracing prerequisite knowledge with a local PEP 2022 math knowledge graph. 适用于初中数学辅导、错题诊断、出题练习、数学自学与知识图谱学习路径。"
```

- [ ] **Step 2: Verify frontmatter shape and key discovery terms**

Run:

```text
python -c "from pathlib import Path; text=Path('skills/math-tutor/SKILL.md').read_text(encoding='utf-8'); assert text.startswith('---\nname: math-tutor\ndescription: \"Use when'); assert '错题诊断' in text; assert '知识图谱' in text"
```

Expected: exit code 0.

- [ ] **Step 3: Commit only the skill metadata change**

Run:

```text
git add skills/math-tutor/SKILL.md
git commit -m "docs: improve math tutor skill discovery"
```

### Task 6: Rebuild the README landing section and correct stale content

**Files:**
- Modify: `README.md`

**Interfaces:**
- Produces: bilingual landing content; Chinese detailed documentation; verified repository links and workflows.

- [ ] **Step 1: Reread the entire current README and preserve valuable Chinese reference content**

Confirm the existing user change from `CodeBuddy` to `WorkBuddy` remains intact before further edits.

- [ ] **Step 2: Replace the top landing section through the first horizontal rule**

Use this content, then retain the existing detailed sections after the first horizontal rule:

```markdown
# 📚 人教版初中数学知识图谱与 AI 辅导技能

> **Offline AI tutoring skill and prerequisite knowledge graph for PEP 2022 junior-high mathematics.**

为学生、家长、教师和 AI 智能体提供可追溯的错题诊断、个性化练习与自学路径；全部知识图谱数据随技能包离线分发。

| 169 个微知识点 | 237 条前置依赖 | 覆盖 6 册教材 | 100% 离线可用 |
| :---: | :---: | :---: | :---: |

## 从错题到补强，只需一个提示

**输入 / Prompt**

> 「我把 $0.1010010001\ldots$ 判断成了有理数，为什么错了？」

**辅导路径 / What the skill returns**

1. 定位直接薄弱点：无理数与实数的概念；
2. 追溯前置理解：有理数的定义与小数表示；
3. 给出先补前置、再理解概念、最后递进练习的学习顺序；
4. 根据答案继续调整练习难度。

## 快速开始 / Quick start

1. 将完整的 `skills/math-tutor/` 文件夹复制到你的智能体或项目所使用的技能目录；
2. 保持 `SKILL.md`、`data/`、`references/` 与 `schema/` 的相对目录结构不变；
3. 用自然语言开始，例如：「帮我诊断这道八年级分式题为什么做错了」。

> 本技能不依赖在线服务；运行时只读取随包提供的本地数据。

查看 [中文功能说明](#智能体技能math-tutor)、[交互式八年级上册图谱](visualization_8up.html) 或在 [Issues](../../issues) 中反馈问题与建议。

---
```

- [ ] **Step 3: Correct stale and unsupported claims in remaining README content**

Apply these exact replacements:

- Every `visualization_8up_simple.html` becomes `visualization_8up.html`.
- Replace `输出符合本仓库 schema 的 JSON 文件，可直接 push。` with `输出符合本仓库 schema 的 JSON 文件，供人工复核后纳入版本控制。`.
- Remove all lines containing `[项目维护者邮箱]`.
- Replace platform-specific installation instructions with a platform-neutral paragraph that directs readers to copy the complete skill folder and consult their tool’s official documentation for its skills directory.
- Replace the existing license section with a short link to `[LICENSE](LICENSE)` and a separate note that upstream sources and textbook-derived material remain subject to their own terms.

- [ ] **Step 4: Validate Markdown references and prohibited stale wording**

Run:

```text
python -c "from pathlib import Path; text=Path('README.md').read_text(encoding='utf-8'); assert 'visualization_8up_simple.html' not in text; assert '[项目维护者邮箱]' not in text; assert '可直接 push' not in text; assert '(visualization_8up.html)' in text; assert '[LICENSE](LICENSE)' in text"
```

Expected: exit code 0.

- [ ] **Step 5: Commit README improvements**

Run:

```text
git add README.md
git commit -m "docs: improve release landing page"
```

### Task 7: Final publication-readiness verification

**Files:**
- Verify: `README.md`, `skills/math-tutor/SKILL.md`, `LICENSE`, `CONTRIBUTING.md`, `.github/`, `scripts/`, `tests/`

- [ ] **Step 1: Run the full local verification suite**

Run:

```text
python scripts/validate_data.py
python -m unittest tests.test_validate_data -v
python -c "from pathlib import Path; text=Path('README.md').read_text(encoding='utf-8'); assert 'visualization_8up_simple.html' not in text; assert '[项目维护者邮箱]' not in text; assert '可直接 push' not in text; assert Path('LICENSE').is_file(); assert Path('CONTRIBUTING.md').is_file()"
git diff --check
git status --short
git log --oneline -8
```

Expected: validator reports six valid datasets; all unit tests pass; README checks exit 0; `git diff --check` emits nothing; status has no unintended files.

- [ ] **Step 2: Inspect requirement coverage**

Verify against `docs/superpowers/specs/2026-08-07-publication-readiness-design.md`:

- bilingual landing section present;
- Chinese detailed documentation retained;
- skill frontmatter is discovery-oriented;
- license and contribution paths are present;
- both issue forms exist;
- validator and CI workflow exist;
- data files remain unchanged.

- [ ] **Step 3: Create a single release-readiness commit only if tasks were not individually committed**

Run:

```text
git add README.md skills/math-tutor/SKILL.md LICENSE CONTRIBUTING.md .github scripts tests
git commit -m "chore: prepare math tutor for publication"
```

Do not run this step if the task-specific commits already contain all expected changes.
