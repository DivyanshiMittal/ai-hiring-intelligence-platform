# ADR-018 — model_version Format and Value Object

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-2, TDD §0 |

## Context
`TDD.md §0` defines `model_version` as a "pinned tuple (provider, model_id, prompt_template_version)" stamped on every AI-derived artifact. Every AI result table has a `model_version` column but no type was specified. Without a canonical format, values will diverge across developers and become unqueryable for A/B comparison and re-scoring.

## Decision
`model_version` is stored as **`VARCHAR(128)`** with the format:
```
{provider}:{model_id}:{prompt_template_version}
```

Examples:
```
google:gemini-2.5-pro-05-06:resume-parser-v1
google:gemini-2.5-pro-05-06:match-scorer-v2
google:gemini-2.5-pro-05-06:answer-evaluator-v1
```

A **`ModelVersion`** value object in `app/core/model_version.py` constructs and parses this string:
```python
from dataclasses import dataclass
import re

_PATTERN = re.compile(r'^[a-z0-9_-]+:[a-z0-9._-]+:[a-z0-9_-]+-v\d+$')

@dataclass(frozen=True)
class ModelVersion:
    provider: str
    model_id: str
    prompt_template_version: str

    def __str__(self) -> str:
        return f"{self.provider}:{self.model_id}:{self.prompt_template_version}"

    @classmethod
    def parse(cls, value: str) -> "ModelVersion":
        if not _PATTERN.match(value):
            raise ValueError(f"Invalid model_version format: {value!r}")
        provider, model_id, prompt_version = value.split(":", 2)
        return cls(provider=provider, model_id=model_id, prompt_template_version=prompt_version)
```

A Postgres check constraint on every AI-result table enforces the format:
```sql
CONSTRAINT chk_model_version_format
    CHECK (model_version ~ '^[a-z0-9_-]+:[a-z0-9._-]+:[a-z0-9_-]+-v[0-9]+$')
```

## Consequences
- A prompt template rename requires a version bump (e.g., `resume-parser-v1` → `resume-parser-v2`). The new version writes new rows; old rows retain the old version. This is the intended behaviour for A/B comparison.
- The `ModelVersion` value object is the only permitted way to construct a `model_version` string. Direct string construction is a code review failure.
- The LLM gateway reads the current model version from config (`settings.llm.model_version`) and passes it to every call. The gateway constructs the `ModelVersion` object; domain code receives it as a typed value.

## Revisit Trigger
If a prompt template versioning system with semantic versioning (major.minor.patch) is adopted, update the format and pattern accordingly.
