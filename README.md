# DuSE-Bench

**Benchmarking Joint Reasoning over Evolving Dialogue and Documents**

DuSE-Bench (Dual-Source Evolution Benchmark) evaluates how assistants combine applicable document facts with user requirements established and revised across sessions. It contains **1,000 tasks across five scenarios**, including **700 jointly grounded tasks** whose required premises originate in both dialogue and documents.

Published document versions or reports are paired with constructed multi-session dialogue. An evolving **Support Graph** guides task construction and preserves links to original turns and passages through updates and reuse. Task annotations distinguish **grounding source** from **state dependence** and support separate evaluation of **response content, response type, and source attribution**.

## Data overview

| Scenario | Domain | Documents | Passages | Sessions | Turns | Tasks |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| [Scenario 1](Scenario_1_News/) | News | 38 | 211 | 17 | 810 | 200 |
| [Scenario 2](Scenario_2_Science/) | Science | 30 | 442 | 18 | 700 | 200 |
| [Scenario 3](Scenario_3_Software/) | Software | 22 | 781 | 20 | 690 | 200 |
| [Scenario 4](Scenario_4_Corporate/) | Corporate | 22 | 1,998 | 20 | 816 | 200 |
| [Scenario 5](Scenario_5_Environment/) | Environment | 14 | 3,943 | 28 | 808 | 200 |
| **Total** | **Five domains** | **126** | **7,375** | **103** | **3,824** | **1,000** |

Turns count individual user or assistant messages. Documents include the released versions and reports; passages are the original document units used for evidence attribution.

## Release files

Each scenario directory contains three JSON files:

| File | Contents |
| --- | --- |
| `scenario.json` | Scenario metadata, original documents and passages, recorded dialogue sessions, and annotated tasks. |
| `support_graph.json` | Document and dialogue facts, intermediate conclusions, and their grounding, support, evolution, and task-dependency relations. |
| `alternative_evidence_sets.json` | Accepted alternative complete evidence sets for tasks with additional valid support. Other tasks use their default evidence set. |

The top-level fields in `scenario.json` are `scenario`, `documents`, `sessions`, and `tasks`. Identifiers are scoped to each scenario:

- Resolve a dialogue turn using `session_id` and `turn_id` in `sessions`.
- Resolve a document passage using `document_id` and `passage_id` in `documents`.
- Document records include `introduced_at` and `previous_document_id` to identify introduction points and predecessor relations.

## Task annotations

| Field | Meaning |
| --- | --- |
| `triggered_by` | The session and user turn containing the current request. |
| `task_dependencies` | Grounding source, state dependence, and integration mode where applicable. |
| `reference_content` | The reviewed reference response content. |
| `required_response_type` | The response type required by the request and its applicable premises. |
| `default_evidence_set` | A complete set of original dialogue turns and document passages supporting the required response. |

### Task dependencies

- **Grounding source** (`grounding_source`): `dialogue`, `document`, or `joint`, according to where the necessary premises originate. A document fact retains its document origin when repeated in dialogue.
- **State dependence** (`state_dependence`): `static`, `dialogue`, `document`, or `dual`. A recorded change is relevant when restoring its prior state alters the required result, a necessary qualification, or the response type. Dual dependence requires this effect for either source while the other is held fixed.
- **Integration structure** (`integration_mode`, jointly grounded tasks only): **Direct** (`parallel` in JSON) or **Chained** (`chained`). Direct tasks combine dialogue and document premises without requiring a prior conclusion derived within one source. Chained tasks combine a conclusion derived from at least two jointly necessary premises of one source with premises from the other; the conclusion may be reused or reconstructed.

Task counts pooled across the five scenarios are:

| Grounding source | Static | Dialogue dependence | Document dependence | Dual dependence | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| Dialogue | 50 | 100 | — | — | 150 |
| Document | 50 | — | 100 | — | 150 |
| Joint | 300 | 100 | 100 | 200 | 700 |
| **Total** | **400** | **200** | **200** | **200** | **1,000** |

Dashes indicate inapplicable combinations. Each scenario has the same task allocation, including 140 jointly grounded tasks split equally between Direct and Chained integration.

### Response types

| JSON value | Required behavior | Tasks |
| --- | --- | ---: |
| `answer` | Give the supported result and its necessary qualifications. | 700 |
| `clarification` | Ask for an essential missing user condition or resolve an ambiguous reference. | 100 |
| `conflict` | Explain incompatible applicable claims that remain unresolved. | 100 |
| `abstention` | State any supported partial result, identify the factual gap, and withhold unsupported conclusions. | 100 |

Each scenario contains 140 answer tasks and 20 tasks of each other response type.

### Complete evidence sets

A complete evidence set contains original turns and passages that jointly justify the required response, with every entry necessary. It includes updates needed to establish which information applies. Accepted alternative sets must independently justify the same response under the same request conditions.

Evidence references are stored under `turns` and `passages`. The Support Graph preserves links to these original sources when document facts are restated or earlier conclusions are reused.

## Evaluation

The paper evaluates three response aspects separately:

| Aspect | Metric | Definition |
| --- | --- | --- |
| Response content | Content accuracy | Fraction of tasks whose responses supply all essential results and qualifications without material error. |
| Response content | Content quality | Independently assigned 0–5 grade divided by five before averaging. |
| Response type | Macro-F1 | Macro-F1 over the four required response types. |
| Source attribution | F1 | Identifier-level F1 against one complete reference evidence set. |

For source attribution, select the default or accepted alternative complete set that maximizes overall F1 for that response. Do not combine entries from different reference sets. Content judging uses the request, reference content, and original evidence from the fixed default set.

The paper compares 11 main baseline configurations using GPT-4.1-mini: long context, retrieval, raw dialogue history, and dialogue memory. Content accuracy on jointly grounded tasks reaches at most 56.7% across these configurations. Additional LLM comparisons, retrieval interventions, and matched evidence substitutions examine performance across models and the interaction between dialogue and document support.

## Reading the data

The following example inspects one task using Python's standard library. Run it from the repository root:

```python
import json
from pathlib import Path

path = Path("Scenario_1_News/scenario.json")
data = json.loads(path.read_text(encoding="utf-8"))

task_id, task = next(iter(data["tasks"].items()))
location = task["triggered_by"]
request = data["sessions"][location["session_id"]]["turns"][location["turn_id"]]["text"]

print(task_id, request)
print(task["task_dependencies"])
print(task["required_response_type"])
```

For evaluation, use `triggered_by` to locate the current request and supply only the preceding recorded dialogue and documents introduced by that turn. Keep later material, reference content, required response types, evidence annotations, and the Support Graph out of model inputs. Use the reference annotations and alternative evidence sets for scoring and analysis.
