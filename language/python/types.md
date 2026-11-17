# Python Type Hints

## Rules

- Annotate all public function signatures and use built-in generics (`list[str]`, `dict[str, int]`) and `X | None` instead of `typing.List`, `Optional`, `Union`.
- Prefer `Protocol` for structural interfaces, `TypedDict` or dataclasses for structured data, and `Literal`/`Enum` for fixed choices.
- Avoid `Any`; use `object`, `TypeVar`, or a Protocol, and keep `# type: ignore` narrow with a reason.
- Run a type checker (mypy or pyright) in strict mode in CI and make it part of the gate.
- Use `Self`, `TypeAlias` and `ParamSpec` for precise types instead of casts.

## Basic Type Annotations

```python
from typing import List, Dict, Optional, Union, Tuple

# Variable annotations
name: str = "Alice"
age: int = 30
prices: List[float] = [9.99, 19.99, 29.99]
user_scores: Dict[str, int] = {"alice": 100, "bob": 85}

# Function annotations
def greet(name: str, times: int = 1) -> str:
    return f"Hello, {name}! " * times

def find_user(user_id: int) -> Optional[User]:
    """Returns User or None if not found."""
    return db.get(user_id)
```

## Modern Python 3.10+ Syntax

```python
# ✅ Use built-in types directly (Python 3.9+)
def process(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# ✅ Union syntax with | (Python 3.10+)
def parse(value: str | int | None) -> str:
    if value is None:
        return ""
    return str(value)

# Instead of:
from typing import Union, Optional
def parse(value: Union[str, int, None]) -> str: ...
```

## TypedDict and Protocols

```python
from typing import TypedDict, Protocol

# ✅ TypedDict for structured dictionaries
class UserDict(TypedDict):
    id: int
    name: str
    email: str
    is_active: bool

def create_user(data: UserDict) -> User:
    return User(**data)

# ✅ Protocol for structural typing (duck typing)
class Readable(Protocol):
    def read(self) -> str: ...

def process_file(file: Readable) -> None:
    content = file.read()
    # Works with any object that has read() method
```

## Generics

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class Repository(Generic[T]):
    def __init__(self, model: type[T]) -> None:
        self.model = model

    def find(self, id: int) -> T | None:
        return self.db.get(self.model, id)

    def save(self, entity: T) -> T:
        return self.db.save(entity)

# Usage
user_repo = Repository[User](User)
user = user_repo.find(1)  # Returns User | None
```

## Type Checking with mypy

```bash
# Run type checker
mypy src/

# mypy.ini configuration
[mypy]
python_version = 3.11
strict = True
warn_return_any = True
warn_unused_ignores = True
```
