# Python Tooling And Project Setup

## Project Layout And Packaging

- Use a `src/` layout and a single `pyproject.toml` for metadata, dependencies and tool configuration.
- Pin the supported Python range with `requires-python`, and test every version you claim to support.
- Use **uv** (or Poetry/PDM) to manage environments and lockfiles; commit the lockfile; install with the frozen/locked option in CI.

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["httpx>=0.27", "pydantic>=2.7"]

[dependency-groups]
dev = ["pytest>=8", "pytest-cov", "ruff", "pyright"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "S", "ASYNC", "RUF"]
ignore = ["S101"]   # asserts are fine in tests; narrow per-file below

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S"]

[tool.pytest.ini_options]
addopts = "-ra --strict-markers --strict-config"
testpaths = ["tests"]

[tool.pyright]
typeCheckingMode = "strict"
include = ["src"]
```

## Everyday Commands

```bash
uv sync                      # create/update the environment from the lockfile
uv run ruff check --fix .    # lint (and safe autofixes)
uv run ruff format .         # format
uv run pyright               # type check
uv run pytest                # tests
```

- **Ruff** replaces flake8, isort, pyupgrade and Black for most projects: one fast tool for lint and format.
- **Type-check** with pyright or mypy in strict mode for new code; add annotations to public functions at minimum.
- Run the same commands in CI with the lockfile frozen (`uv sync --frozen`).

## Modern Python Idioms

```python
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import Protocol


@dataclass(frozen=True, slots=True)
class User:
    id: int
    email: str
    tags: tuple[str, ...] = field(default_factory=tuple)


class UserRepo(Protocol):
    def find(self, user_id: int) -> User | None: ...


def read_names(path: Path) -> list[str]:
    # pathlib, context managers and comprehensions over manual loops
    with path.open(encoding="utf-8") as handle:
        return [line.strip() for line in handle if line.strip()]


def describe(value: int | str | None) -> str:
    match value:
        case int():
            return f"int {value}"
        case str() as text if text:
            return f"str {text}"
        case _:
            return "empty"
```

- Use `X | None` unions, built-in generics (`list[str]`), `match`, `dataclass(slots=True, frozen=True)` and `Protocol` for structural typing.
- Always pass `encoding="utf-8"` when opening text files. Use `pathlib` for paths and `subprocess.run([...], check=True)` (list form, no `shell=True`) for processes.
- Validate external data with Pydantic (or similar) at boundaries; keep domain code on plain typed objects.
- Use `logging` with structured context rather than `print`; never log secrets.

## Testing

- `pytest` with fixtures, `parametrize` and `tmp_path`; keep tests independent of order and the network.
- Measure coverage with `pytest-cov`, but gate on meaningful tests rather than a percentage alone.
- Use `hypothesis` for parsers and invariants.

## Security And Dependencies

- Run `pip-audit` (or `uv` equivalents) in CI; see the supply-chain security guideline.
- Do not use `pickle`/`yaml.load` on untrusted data; use `json` or `yaml.safe_load`.
- Use `secrets` (not `random`) for tokens; use parameterised queries.

## Quick Checklist

- [ ] `pyproject.toml` is the single source of metadata and tool config
- [ ] Lockfile committed; CI installs frozen
- [ ] `ruff check`, `ruff format --check`, type checker and `pytest` run in CI
- [ ] Public functions are annotated; external data is validated
