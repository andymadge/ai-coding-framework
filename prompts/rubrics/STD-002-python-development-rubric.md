# Python Development Rubric

## Overview

This rubric provides reusable patterns, standards, and best practices for developing high-quality Python code that is:
- Type-safe with comprehensive type hints (PEP 484)
- Pythonic and idiomatic (PEP 8, PEP 20)
- Well-tested with clear, maintainable test coverage
- Properly documented with clear docstrings and comments where necessary
- Secure and performant by default
- Easy to maintain and extend

Use this document as a reference when writing new Python code, reviewing implementations, or establishing team standards.

---

## Table of Contents

1. [When to Apply This Rubric](#when-to-apply-this-rubric)
2. [Core Principles](#core-principles)
3. [Type Hints and Static Analysis](#type-hints-and-static-analysis)
4. [Pythonic Code Patterns](#pythonic-code-patterns)
5. [Code Organization and Structure](#code-organization-and-structure)
6. [Error Handling and Exceptions](#error-handling-and-exceptions)
7. [Testing Standards](#testing-standards)
8. [Documentation Standards](#documentation-standards)
9. [Security Best Practices](#security-best-practices)
10. [Performance Considerations](#performance-considerations)
11. [Dependencies Management](#dependencies-management)
12. [Self-Assessment Checklist](#self-assessment-checklist)

---

## When to Apply This Rubric

### Apply Strictly When:
- Writing production code that will be maintained by multiple developers
- Creating shared libraries or packages
- Building applications with security or performance requirements
- Any code that processes user input or external data
- Code that handles sensitive operations (authentication, payments, data processing)

### Core Expectations (Always):
- Type hints on all function signatures
- PEP 8 compliance (linted with `ruff`, `pylint`, or `flake8`)
- Documentation only where code intent is non-obvious (public APIs, complex logic)
- Tests for all non-trivial logic

### May Be Relaxed For:
- Quick scripts or prototypes (mark as such with `# pragma: no-typing` or similar)
- Throwaway code in exploratory notebooks (mark clearly as temporary)
- Internal utilities with single developer and clear scope

---

## Core Principles

| Principle | Description | Examples |
|-----------|-------------|----------|
| **Type-First Design** | Type hints are required, not optional. They serve as executable documentation and enable static analysis. | `def process(data: list[str]) -> dict[str, int]:` instead of `def process(data):` |
| **Explicit Over Implicit** | Code clarity and explicitness trump cleverness. Use readable code even if it's longer. | Use `for item in items:` over confusing list comprehensions |
| **Fail Fast** | Validate inputs early, raise exceptions for invalid states, don't silently continue with bad data. | Check preconditions at function start; raise `ValueError` immediately |
| **Single Responsibility** | Each function/class should have one clear purpose. If a name has "and" in it, split it. | `parse_config()` not `parse_and_validate_config_and_apply_defaults()` |
| **Defensive at Boundaries** | Validate all external inputs (user input, API responses, file content). Trust internal code. | Use `try`/`except` for file I/O; assume internal function returns valid data |
| **Standard Library First** | Prefer Python's standard library before adding dependencies. Evaluate new packages carefully. | Use `pathlib` instead of adding `upath`; use `json` before `ujson` |
| **Tests Are Code** | Tests must be as clean and maintainable as production code. Use factories, fixtures, and helpers. | Build reusable test fixtures; avoid test code duplication |
| **Clear Error Messages** | Exceptions and logs should be actionable by the next developer reading them. | `raise ValueError(f"Expected port 1-65535, got {port}")` not `ValueError("invalid port")` |
| **Code Over Documentation** | Clear naming and types eliminate the need for documentation. Document only the *why*, never the *what*. | Type hints show parameters; names show intent; only document complex algorithms |

---

## Type Hints and Static Analysis

### Required Type Hints

Every function signature must have type hints. This is non-negotiable for production code.

```python
# ❌ WRONG - Missing type hints
def calculate_total(items):
    return sum(item['price'] for item in items)

# ✅ CORRECT - Full type hints
from typing import TypedDict

class Item(TypedDict):
    price: float
    quantity: int

def calculate_total(items: list[Item]) -> float:
    return sum(item['price'] * item['quantity'] for item in items)
```

### Type Hint Patterns

#### Basic Types
```python
def process(
    name: str,
    count: int,
    ratio: float,
    enabled: bool,
) -> str:
    """Process data with given parameters."""
    return f"{name}:{count}"
```

#### Collections
```python
from typing import Sequence, Mapping, Set

def analyze(
    items: list[str],
    config: dict[str, int],
    tags: set[str],
    values: Sequence[float],
) -> Mapping[str, list[int]]:
    """Analyze items and return results mapping."""
    ...
```

#### Optional and Union
```python
from typing import Optional, Union

def get_user(user_id: int) -> Optional[dict[str, str]]:
    """Get user or None if not found."""
    ...

def parse_value(value: Union[int, str]) -> float:
    """Parse numeric value from int or string."""
    if isinstance(value, str):
        return float(value)
    return float(value)
```

#### Callables
```python
from typing import Callable

def apply_transform(
    data: list[int],
    transformer: Callable[[int], int],
) -> list[int]:
    """Apply transformation function to each item."""
    return [transformer(x) for x in data]
```

#### TypedDict for Structured Data
```python
from typing import TypedDict

class UserProfile(TypedDict):
    user_id: int
    name: str
    email: str
    verified: bool

def create_user(profile: UserProfile) -> None:
    """Create a new user from profile."""
    ...
```

### Generic Types
```python
from typing import Generic, TypeVar

T = TypeVar('T')

class Cache(Generic[T]):
    def __init__(self, capacity: int) -> None:
        self._capacity = capacity
        self._items: dict[str, T] = {}

    def get(self, key: str) -> Optional[T]:
        return self._items.get(key)

    def set(self, key: str, value: T) -> None:
        self._items[key] = value
```

### Static Analysis Tools

Configure and use these tools in your CI/CD pipeline:

```ini
# pyproject.toml or setup.cfg
[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true  # Enforce type hints
ignore_missing_imports = false

[tool.pylint]
disable = [
    "missing-docstring",  # Handled by tools like darglint
    "line-too-long",      # Handled by formatters
]

[tool.ruff]
select = ["E", "F", "W", "I", "UP"]  # Errors, undefs, warnings, imports, upgrades
```

Validate types before committing:
```bash
mypy src/
pylint src/
ruff check src/
```

---

## Pythonic Code Patterns

### List Comprehensions and Generator Expressions

Use comprehensions for simple transformations. Keep them readable.

```python
# ✅ GOOD - Clear intent, readable
squares = [x**2 for x in range(10) if x % 2 == 0]
unique_names = {user['name'] for user in users}

# ❌ AVOID - Too complex, hard to follow
result = [
    process(x)
    for x in items
    if condition1(x) and condition2(x)
    for y in x.children
    if condition3(y)
]

# ✅ BETTER - Use loops for complex logic
result = []
for x in items:
    if condition1(x) and condition2(x):
        for y in x.children:
            if condition3(y):
                result.append(process(x))
```

### Context Managers

Use `with` statements for resource management. Always.

```python
# ✅ CORRECT - Context manager ensures cleanup
with open('file.txt') as f:
    data = f.read()

with database.transaction():
    update_user(user_id, new_data)

# ❌ WRONG - May leak resources if exception occurs
f = open('file.txt')
data = f.read()
f.close()
```

Define context managers for your own resource management:

```python
from contextlib import contextmanager
from typing import Generator

@contextmanager
def database_transaction() -> Generator[Connection, None, None]:
    """Context manager for database transactions."""
    conn = create_connection()
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

### String Operations

Use modern string formatting. No old-style formatting.

```python
# ✅ CORRECT - f-strings (Python 3.6+)
message = f"User {user_id} logged in at {timestamp}"
debug = f"Value: {value!r}, Length: {len(value)}"

# ⚠️ ACCEPTABLE - .format() if f-strings not suitable
message = "User {} logged in at {}".format(user_id, timestamp)

# ❌ WRONG - Old %-formatting
message = "User %s logged in at %s" % (user_id, timestamp)
```

### Avoid Anti-Patterns

```python
# ❌ WRONG - Mutable default arguments
def add_item(item, items=[]):
    items.append(item)
    return items

# ✅ CORRECT - Use None and initialize inside
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items
```

```python
# ❌ WRONG - Bare except catches system exits
try:
    process_data(data)
except:
    log_error("Failed")

# ✅ CORRECT - Catch specific exceptions
try:
    process_data(data)
except ValueError as e:
    log_error(f"Invalid data: {e}")
except IOError as e:
    log_error(f"I/O error: {e}")
```

### Iterator and Generator Usage

```python
# ✅ GOOD - Generators for memory efficiency
def read_large_file(file_path: str) -> Generator[str, None, None]:
    """Yield lines from a large file without loading all at once."""
    with open(file_path) as f:
        for line in f:
            yield line.rstrip('\n')

# Use it
for line in read_large_file('huge_file.txt'):
    process(line)
```

---

## Code Organization and Structure

### Module Organization

```
project/
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── models/              # Data models
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── product.py
│       ├── services/            # Business logic
│       │   ├── __init__.py
│       │   ├── user_service.py
│       │   └── payment_service.py
│       ├── repositories/        # Data access
│       │   ├── __init__.py
│       │   └── user_repository.py
│       ├── utils/               # Utilities
│       │   ├── __init__.py
│       │   ├── validators.py
│       │   └── formatters.py
│       └── config.py            # Configuration
├── tests/
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
├── pyproject.toml
└── README.md
```

### Class Organization

Organize class members in this order:

```python
class User:
    """A user in the system."""

    # Class variables
    MAX_RETRIES = 3
    DEFAULT_TIMEOUT = 30

    # Special/magic methods
    def __init__(self, user_id: int, name: str) -> None:
        """Initialize a new user."""
        self.user_id = user_id
        self.name = name
        self._cache: dict[str, str] = {}

    def __str__(self) -> str:
        return f"User({self.user_id}, {self.name})"

    def __repr__(self) -> str:
        return f"User(user_id={self.user_id!r}, name={self.name!r})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, User):
            return NotImplemented
        return self.user_id == other.user_id

    # Properties
    @property
    def display_name(self) -> str:
        """Get the display name for this user."""
        return self.name.title()

    # Public methods
    def update_profile(self, name: str, email: str) -> None:
        """Update user profile information."""
        self.name = name
        self.email = email

    # Private methods
    def _validate_email(self, email: str) -> bool:
        """Validate email format."""
        return "@" in email

    # Static methods (rarely used)
    @staticmethod
    def from_dict(data: dict[str, str]) -> 'User':
        """Create a User from a dictionary."""
        return User(int(data['id']), data['name'])
```

### Naming Conventions

```python
# ✅ CORRECT naming
class UserService:              # Classes: PascalCase
    MAX_RETRIES = 3             # Constants: UPPER_CASE

    def process_user(self) -> None:  # Functions: snake_case
        self._private_attr = 1   # Private: leading underscore
        user_list = []           # Variables: snake_case

# ❌ WRONG naming
class user_service:             # Should be UserService
    maxRetries = 3              # Should be MAX_RETRIES

    def processUser(self):       # Should be process_user
        self.__name = "private"  # Double underscore is too private
```

---

## Error Handling and Exceptions

### Raising Exceptions

Be specific about what went wrong. Provide actionable error messages.

```python
# ❌ WRONG - Generic exception, unclear message
def parse_port(port_str: str) -> int:
    try:
        port = int(port_str)
    except:
        raise Exception("Bad port")

# ✅ CORRECT - Specific exception with context
def parse_port(port_str: str) -> int:
    try:
        port = int(port_str)
    except ValueError as e:
        raise ValueError(
            f"Port must be a valid integer, got {port_str!r}"
        ) from e

    if not (1 <= port <= 65535):
        raise ValueError(
            f"Port must be in range 1-65535, got {port}"
        )

    return port
```

### Creating Custom Exceptions

```python
class ValidationError(ValueError):
    """Raised when input validation fails."""
    pass

class ConfigurationError(Exception):
    """Raised when configuration is invalid or missing."""
    pass

class DatabaseError(Exception):
    """Raised when database operation fails."""
    pass

# Usage
try:
    config = load_config()
except FileNotFoundError:
    raise ConfigurationError(
        "Configuration file not found at ~/.myapp/config.yaml"
    ) from None
```

### Exception Handling Strategy

```python
# ✅ GOOD - Catch specific exceptions
def fetch_user(user_id: int) -> dict[str, str]:
    try:
        response = requests.get(f"https://api.example.com/users/{user_id}")
        response.raise_for_status()
        return response.json()
    except requests.ConnectionError as e:
        raise RuntimeError("Failed to connect to API") from e
    except requests.TimeoutError as e:
        raise RuntimeError("API request timed out") from e
    except ValueError as e:
        raise RuntimeError(f"Invalid JSON response: {e}") from e

# ❌ WRONG - Catching everything
def fetch_user(user_id: int) -> dict[str, str]:
    try:
        response = requests.get(f"https://api.example.com/users/{user_id}")
        return response.json()
    except Exception as e:
        print(f"Error: {e}")
        return {}  # Silent failure
```

---

## Testing Standards

### Test Organization

```python
# tests/unit/test_user_service.py
import pytest
from myproject.services.user_service import UserService
from myproject.models.user import User

class TestUserService:
    """Tests for UserService."""

    @pytest.fixture
    def service(self) -> UserService:
        """Create a fresh service instance for each test."""
        return UserService()

    @pytest.fixture
    def sample_user(self) -> User:
        """Create a sample user for testing."""
        return User(user_id=1, name="Alice")

    def test_create_user(self, service: UserService, sample_user: User) -> None:
        """Test that a user can be created."""
        result = service.create(sample_user)
        assert result.user_id == sample_user.user_id
        assert result.name == sample_user.name

    def test_get_user_returns_none_if_not_found(self, service: UserService) -> None:
        """Test that get_user returns None when user doesn't exist."""
        result = service.get(9999)
        assert result is None

    @pytest.mark.parametrize("invalid_id", [-1, 0, "abc"])
    def test_get_user_raises_on_invalid_id(
        self,
        service: UserService,
        invalid_id: int | str,
    ) -> None:
        """Test that get_user raises ValueError for invalid IDs."""
        with pytest.raises(ValueError, match="user_id must be positive"):
            service.get(invalid_id)
```

### Test Naming

```python
# ✅ GOOD - Clear test names
def test_calculate_total_with_empty_list_returns_zero() -> None:
def test_parse_port_raises_valueerror_for_out_of_range() -> None:
def test_user_service_creates_user_with_correct_attributes() -> None:

# ❌ WRONG - Vague names
def test_calculate() -> None:
def test_error() -> None:
def test_works() -> None:
```

### Assertions

```python
# ✅ GOOD - Specific assertions with context
assert user.name == "Alice", f"Expected name 'Alice', got {user.name!r}"
assert len(items) == 5, f"Expected 5 items, got {len(items)}"
assert user.email in valid_emails, f"Email {user.email} not in {valid_emails}"

# Use pytest assertions for better error messages
def test_list_operations(self) -> None:
    result = process_data([1, 2, 3])
    assert result == [1, 4, 9]  # pytest shows difference
    assert isinstance(result, list)
    assert all(x > 0 for x in result)
```

### Mocking

```python
from unittest.mock import Mock, patch, MagicMock

def test_user_service_calls_database(self) -> None:
    """Test that service correctly calls database layer."""
    mock_db = Mock()
    mock_db.get_user.return_value = {"id": 1, "name": "Alice"}

    service = UserService(db=mock_db)
    result = service.get_user(1)

    mock_db.get_user.assert_called_once_with(1)
    assert result["name"] == "Alice"

# ✅ Use patch for dependencies
@patch('myproject.services.api.requests.get')
def test_fetch_user_with_api_error(self, mock_get: Mock) -> None:
    """Test error handling when API is down."""
    mock_get.side_effect = ConnectionError("Network unreachable")

    service = UserService()
    with pytest.raises(RuntimeError, match="Failed to connect"):
        service.fetch_remote_user(1)
```

### Test Coverage

Aim for 80%+ coverage on core logic. Some rules:

```python
# ✅ Test all branches
def validate_age(age: int) -> bool:
    if age < 0:
        raise ValueError("Age cannot be negative")
    return age >= 18

# Tests:
# - test_raises_on_negative_age()
# - test_returns_true_for_adult()
# - test_returns_false_for_minor()

# ✅ Test happy path and error cases
# - Happy path: valid input, expected output
# - Error cases: invalid input, edge cases, exceptions
# - Boundary cases: min, max, empty, None

# Skip testing:
# - Third-party library behavior
# - Simple property getters/setters
# - Code that delegates entirely to dependencies
```

---

## Documentation Standards

### Philosophy: Code Speaks for Itself

Type hints, clear naming, and small focused functions eliminate most documentation needs. **Only document when code clarity is impossible.**

### When to Document (Sparingly)

```python
# ✅ NO DOCSTRING NEEDED - Type hints and name are clear
def calculate_total(items: list[Item]) -> float:
    return sum(item.price * item.quantity for items in items)

def get_user(user_id: int) -> User | None:
    return self._users.get(user_id)

# ✅ ONE-LINER - Simple public function
def create_user(name: str, email: str) -> User:
    """Create a new user."""
    return User(name, email)

# ✅ DOCUMENT COMPLEX LOGIC ONLY - Non-obvious algorithms
def calculate_score(user: User) -> float:
    """Score calculation using exponential decay weighted by recency.

    Recent activities (< 30 days) weighted more heavily.
    Older activities exponentially decrease in weight.
    """
    decay_rate = 0.693 / (30 * 24)  # Half-life: 30 days
    return sum(
        activity.points * math.exp(-decay_rate * hours_ago(activity))
        for activity in user.activities
    )

# ✅ DOCUMENT PUBLIC APIS - External interfaces only
def apply_damage(damage_info: DamageInfo, target: Character) -> Result:
    """Apply damage to target after calculating modifiers.

    Applies in order: critical multiplier → armor reduction →
    elemental resistance.

    Args:
        damage_info: Source damage and type.
        target: Character receiving damage.

    Returns:
        Result.ok() if damage applied, Result.fail() if target immune.

    Raises:
        ValueError: If damage_info or target is invalid.
    """
    ...
```

### Module Docstrings

Only for public packages. Keep brief:

```python
"""Combat system for damage calculations and state transitions."""

# Implementation follows...
```

Avoid:
```python
# ❌ WRONG - Redundant, duplicates code
"""User management services and models.
This module provides the core functionality for user account management,
including creation, authentication, and profile updates."""
```

### Inline Comments

Only explain *why*, never *what*. The code shows what it does.

```python
# ✅ GOOD - Explains design decision
def cache_users(users: list[User]) -> None:
    # Use user ID as key for O(1) lookups instead of list traversal
    for user in users:
        self._cache[user.id] = user

# ❌ WRONG - Restates the code
def cache_users(users: list[User]) -> None:
    # Loop through users and add to cache
    for user in users:
        self._cache[user.id] = user
```

### Type Hints Are Documentation

Never use comments to explain what type hints already show:

```python
# ✅ CORRECT - Types eliminate need for comments
def process_data(
    items: list[Item],
    multiplier: float,
    callback: Callable[[Item, float], None],
) -> dict[str, list[Item]]:
    ...

# ❌ WRONG - Type hints already document this
def process_data(items, multiplier, callback):
    # items: list of Item objects
    # multiplier: float to multiply each item by
    # callback: function that takes (Item, float)
    # Returns: dict mapping string keys to lists of Items
    ...
```

### When Code is Self-Documenting

Clear naming and structure eliminate documentation:

```python
# ✅ NO DOCS NEEDED - Function signature is clear
@property
def can_attack(self) -> bool:
    return not self.is_dead and not self.is_stunned and self.attack_cooldown <= 0

def validate_spawn_position(position: tuple[float, float, float]) -> bool:
    return 0 <= position[0] < MAP_WIDTH and 0 <= position[1] < MAP_HEIGHT

def apply_critical_multiplier(damage: DamageInfo) -> DamageInfo:
    if not damage.is_critical:
        return damage
    return DamageInfo(
        amount=int(damage.amount * 2),
        type=damage.type,
        source_id=damage.source_id,
        is_critical=damage.is_critical
    )
```

---

## Security Best Practices

### Input Validation

Always validate at system boundaries. Never trust external input.

```python
# ✅ SECURE - Validate all external input
def create_account(username: str, email: str) -> Account:
    # Validate username
    if not username or len(username) > 50:
        raise ValueError("Username must be 1-50 characters")
    if not username.isalnum():
        raise ValueError("Username must be alphanumeric")

    # Validate email
    if "@" not in email or "." not in email:
        raise ValueError("Invalid email format")

    # Now proceed with creation
    return Account.create(username, email)

# ❌ INSECURE - No validation
def create_account(username: str, email: str) -> Account:
    # Directly use user input without checking
    return Account.create(username, email)
```

### SQL Injection Prevention

Never use string formatting for SQL queries. Always use parameterized queries.

```python
# ❌ VULNERABLE - SQL injection risk
def get_user(user_id: int) -> dict:
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return db.execute(query)

# ✅ SECURE - Parameterized query
def get_user(user_id: int) -> dict:
    query = "SELECT * FROM users WHERE id = ?"
    return db.execute(query, (user_id,))

# ✅ ALSO SECURE - Using ORM
def get_user(user_id: int) -> User:
    return User.query.filter_by(id=user_id).first()
```

### Credential and Secret Management

Never hardcode secrets. Use environment variables or secret management systems.

```python
# ❌ WRONG - Hardcoded secrets
API_KEY = "sk-12345abcde"
DATABASE_PASSWORD = "admin123"

# ✅ CORRECT - Load from environment
import os
from typing import NoReturn

def get_secret(name: str) -> str:
    """Get secret from environment, raising if missing."""
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Required secret {name} not found in environment")
    return value

API_KEY = get_secret("API_KEY")
DATABASE_PASSWORD = get_secret("DATABASE_PASSWORD")
```

### Dependency Pinning

Always pin dependency versions to avoid unexpected breaking changes.

```ini
# ❌ WRONG - Loose versioning
[project]
dependencies = [
    "django",           # Could be 5.0 or 2.0!
    "requests>=2.0",   # Could break with 3.0
]

# ✅ CORRECT - Pinned versions
[project]
dependencies = [
    "django>=4.2,<5",
    "requests>=2.31,<3",
]
```

---

## Performance Considerations

### Algorithmic Efficiency

Choose the right algorithm for the task.

```python
# ❌ SLOW - O(n²) lookup in list
def find_user(user_id: int, users: list[User]) -> User | None:
    for user in users:
        if user.id == user_id:
            return user
    return None

# ✅ FAST - O(1) lookup with dict
users_by_id: dict[int, User] = {user.id: user for user in users}
user = users_by_id.get(user_id)
```

### Generator Usage for Large Data

```python
# ❌ WRONG - Loads all 1M records into memory
def process_all_users() -> list[dict]:
    users = database.query("SELECT * FROM users")
    return [process(u) for u in users]

# ✅ CORRECT - Stream data
def process_all_users() -> Generator[dict, None, None]:
    for user in database.iter_query("SELECT * FROM users"):
        yield process(user)

# Usage
for processed in process_all_users():
    save(processed)
```

### Caching

Use caching for expensive operations that have stable results.

```python
from functools import lru_cache

# ✅ GOOD - Cache computed results
@lru_cache(maxsize=128)
def calculate_user_score(user_id: int) -> float:
    """Calculate user score (cached)."""
    user = get_user(user_id)
    return sum(activity.points for activity in user.activities)

# For class methods, use custom caching
class UserService:
    def __init__(self) -> None:
        self._cache: dict[int, User] = {}

    def get_cached(self, user_id: int) -> User:
        """Get user, returning cached version if available."""
        if user_id not in self._cache:
            self._cache[user_id] = self._fetch_from_db(user_id)
        return self._cache[user_id]

    def invalidate_cache(self, user_id: int) -> None:
        """Invalidate cache for a user."""
        self._cache.pop(user_id, None)
```

### Profiling

Profile before optimizing. Don't guess.

```python
import cProfile
import pstats

# Profile your code
profiler = cProfile.Profile()
profiler.enable()

# ... code to profile ...

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)  # Print top 10
```

---

## Dependencies Management

### Declaring Dependencies

Use `pyproject.toml` for modern Python projects:

```toml
[project]
name = "myproject"
version = "1.0.0"
description = "My awesome project"
requires-python = ">=3.9"

dependencies = [
    "requests>=2.31,<3",
    "pydantic>=2.0,<3",
    "sqlalchemy>=2.0,<3",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "mypy>=1.0",
    "ruff>=0.1",
    "black>=23.0",
]

docs = [
    "sphinx>=5.0",
    "sphinx-rtd-theme>=1.0",
]
```

### Dependency Evaluation Checklist

Before adding a dependency, ask:

- [ ] Is this needed, or can we use the standard library?
- [ ] Is the project actively maintained?
- [ ] Does it have good test coverage and documentation?
- [ ] What's the security history? Any known vulnerabilities?
- [ ] Does it have minimal transitive dependencies?
- [ ] Will it lock us into a specific Python version?
- [ ] Can we easily replace it if needed?

```python
# ✅ Good dependency additions
import json  # stdlib, always available
from pathlib import Path  # stdlib
import requests  # Popular, well-maintained, minimal deps

# ⚠️ Evaluate carefully
import numpy  # Heavy dependency, adds compile step
import tensorflow  # Large download, complex dependencies

# ❌ Avoid
import some_obscure_pkg  # No recent updates, unclear maintenance
```

### Avoiding Dependency Hell

Use version pinning and lock files:

```bash
# Generate requirements.txt with exact versions
pip install pip-tools
pip-compile pyproject.toml

# Use uv for faster dependency resolution
uv pip install --python=3.11 -e ".[dev]"
```

---

## Self-Assessment Checklist

Use this checklist when reviewing code (yours or others'):

### Type Safety (8 items)
- [ ] All function parameters have type hints
- [ ] All return types are explicitly declared
- [ ] Complex types use TypedDict, Generic, or Protocol
- [ ] No `Any` types except where explicitly justified
- [ ] MyPy runs cleanly with `disallow_untyped_defs = true`
- [ ] Union types prefer `X | Y` over `Union[X, Y]`
- [ ] Optional types use `X | None` or `Optional[X]`
- [ ] Type hints accurately reflect actual behavior

### Pythonic Code (8 items)
- [ ] Code uses list comprehensions for simple transformations
- [ ] All resource management uses context managers (`with` statements)
- [ ] f-strings used for all string formatting
- [ ] No mutable default arguments
- [ ] Specific exceptions caught, not bare `except:`
- [ ] Names follow PEP 8 conventions (snake_case, PascalCase)
- [ ] No code duplication; functions have single responsibility
- [ ] Imports organized: stdlib, third-party, local

### Error Handling (5 items)
- [ ] All function preconditions validated or documented
- [ ] Specific exceptions raised with clear messages
- [ ] Context provided in error messages (expected vs actual)
- [ ] No silent failures (exceptions never caught and ignored)
- [ ] Exceptions chain using `from` when re-raising

### Testing (6 items)
- [ ] All public functions have tests
- [ ] All branches/conditions are tested
- [ ] Edge cases covered (empty, None, boundary values)
- [ ] Both success and error cases tested
- [ ] Test names clearly describe what they test
- [ ] Fixtures/factories used to reduce duplication

### Documentation (5 items)
- [ ] Minimal docs—only where code intent isn't obvious from names/types
- [ ] Public APIs have brief docstrings (one-liner or explanation of complex logic)
- [ ] Complex algorithms documented (the *why*, not the *what*)
- [ ] No comments restating what code clearly shows
- [ ] Type hints eliminate need for parameter documentation

### Security (4 items)
- [ ] All external input is validated
- [ ] No hardcoded secrets or credentials
- [ ] No SQL injection vulnerabilities (parameterized queries)
- [ ] Dependencies are pinned to specific versions

### Performance (3 items)
- [ ] Appropriate data structures used (dict for lookup, list for order)
- [ ] No unnecessary iteration or computation
- [ ] Large data sets handled with generators/streaming

### Organization (4 items)
- [ ] Files organized in logical module structure
- [ ] Related functionality grouped together
- [ ] Classes have clear responsibilities
- [ ] Public vs private clearly indicated

### Total: 43 verification items

---

## Quick Reference

### Essential Commands

```bash
# Format code
black src/
ruff check --fix src/

# Type checking
mypy src/

# Linting
pylint src/
flake8 src/

# Testing
pytest tests/ -v --cov=src/

# Run all checks
black src/ && ruff check src/ && mypy src/ && pytest tests/
```

### Type Hints Quick Guide

```python
str | int              # Union type (Python 3.10+)
list[str]             # List of strings
dict[str, int]        # Dict with string keys, int values
tuple[int, str, bool] # Tuple with specific types
Sequence[int]         # Any sequence (list, tuple, etc.)
Optional[str]         # Equivalent to str | None
Callable[[int], str]  # Function: takes int, returns str
```

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `except:` | Catches system exits, too broad | Catch specific exceptions |
| Mutable defaults | State shared between calls | Use `None` as default |
| `type(x) == str` | Ignores subclasses | Use `isinstance(x, str)` |
| `dict(x=1, y=2)` over `{'x': 1, 'y': 2}` | Hard to use as config dict | Use dict literal |
| `== True` | Redundant for booleans | Just use `if value:` |
| No type hints | Can't catch errors early | Add types to all functions |

