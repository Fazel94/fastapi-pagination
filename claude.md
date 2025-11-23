# fastapi-pagination

## Project Overview

`fastapi-pagination` is a Python library that simplifies pagination in FastAPI applications. It provides utility functions and data models to help paginate database queries and return paginated responses to clients.

**Key Features:**
- Supports multiple pagination strategies (cursor-based and page-based pagination)
- Works with a wide range of SQL and NoSQL database frameworks
- Supports async/await syntax
- Python 3.10+ compatible
- Extensive ORM/ODM integrations through extension modules

**Repository:** https://github.com/uriyyo/fastapi-pagination
**Documentation:** https://uriyyo-fastapi-pagination.netlify.app/

## Tech Stack

- **Language:** Python 3.10+
- **Framework:** FastAPI
- **Dependencies:** fastapi, pydantic, typing-extensions
- **Package Manager:** uv (modern Python package manager)
- **Linting/Formatting:** ruff
- **Type Checking:** mypy
- **Testing:** pytest, pytest-cov, pytest-asyncio
- **Documentation:** mkdocs with mkdocs-material theme

## Project Structure

```
fastapi-pagination/
├── fastapi_pagination/          # Main package
│   ├── __init__.py             # Public API exports
│   ├── api.py                  # Core pagination API
│   ├── bases.py                # Abstract base classes
│   ├── customization.py        # Page customization utilities
│   ├── cursor.py               # Cursor-based pagination
│   ├── limit_offset.py         # Limit-offset pagination
│   ├── ext/                    # Database-specific extensions
│   │   ├── sqlalchemy.py       # SQLAlchemy support
│   │   ├── sqlmodel.py         # SQLModel support
│   │   ├── tortoise.py         # Tortoise ORM support
│   │   ├── beanie.py           # Beanie (MongoDB) support
│   │   ├── motor.py            # Motor (async MongoDB) support
│   │   ├── django.py           # Django ORM support
│   │   ├── piccolo.py          # Piccolo ORM support
│   │   └── ...                 # Many other integrations
│   └── links/                  # Pagination links support
├── tests/                      # Test suite
│   ├── base/                   # Base test utilities
│   ├── ext/                    # Integration tests for extensions
│   └── test_*.py               # Unit tests
├── docs/                       # Documentation (mkdocs)
├── examples/                   # Example applications
└── scripts/                    # CI/build scripts

```

## Development Workflow

### Setup

1. **Install dependencies:**
   ```bash
   # Install all dependencies (including all extras)
   uv sync --dev --all-extras

   # Or install only basic dependencies
   uv install --dev
   ```

2. **Install pre-commit hooks (recommended):**
   ```bash
   uv run pre-commit install
   ```

### Code Quality Tools

**Pre-commit hooks run automatically before each commit:**
- `ruff format` - Code formatting
- `ruff check` - Linting with auto-fix
- `mypy` - Type checking

**Manual execution:**
```bash
# Run all pre-commit hooks
uv run pre-commit run --all-files

# Run individual tools
uv run ruff format fastapi_pagination tests
uv run ruff check --fix fastapi_pagination tests
uv run mypy fastapi_pagination --show-error-codes
```

### Testing

**Prepare test environment:**
```bash
./scripts/ci-prepare.sh
```

**Run tests:**
```bash
# All tests
uv run pytest tests

# With coverage
uv run pytest tests --cov=fastapi_pagination

# Unit tests only
uv run pytest tests --unit-tests

# Integration tests only (requires PostgreSQL, MongoDB, Cassandra)
uv run pytest tests/ext

# Full CI test suite
./scripts/ci-test.sh
```

## Code Style and Conventions

### Ruff Configuration

- **Line length:** 120 characters
- **Target Python:** 3.10
- **Select:** ALL rules by default
- **Notable ignored rules:**
  - No docstring requirements (D)
  - No function annotation requirements (ANN)
  - No TODO/FIXME checks (TD, FIX)
  - No unused argument checks (ARG)
  - Allows magic numbers (PLR2004)
  - Allows access to private members (SLF001)
  - Allows more than 5 function arguments (PLR0913)

### Mypy Configuration

- **Strict mode:** Enabled
- **Python version:** 3.10
- **Implicit reexport:** Allowed (no_implicit_reexport = false)
- **Missing imports:** Ignored

### Coverage

- **Source:** `fastapi_pagination/`
- **Excluded patterns:**
  - `pragma: no cover`
  - `@abstractmethod`
  - `@overload`
  - `if TYPE_CHECKING:`

## Extension Pattern

When adding a new database integration, create a module in `fastapi_pagination/ext/` following this pattern:

```python
from typing import Any, Optional

from fastapi_pagination.api import apply_items_transformer, create_page
from fastapi_pagination.bases import AbstractParams
from fastapi_pagination.types import AdditionalData, AsyncItemsTransformer
from fastapi_pagination.utils import verify_params


async def paginate(
    query: Any,
    params: Optional[AbstractParams] = None,
    *,
    transformer: Optional[AsyncItemsTransformer] = None,
    additional_data: Optional[AdditionalData] = None,
) -> Any:
    params, raw_params = verify_params(params, "limit-offset")

    total = await query.count()
    items = await query.limit(raw_params.limit).offset(raw_params.offset).all()
    t_items = await apply_items_transformer(items, transformer, async_=True)

    return create_page(
        t_items,
        total=total,
        params=params,
        **(additional_data or {}),
    )
```

**Key considerations:**
- Use `verify_params()` to validate pagination parameters
- Support both sync and async operations where appropriate
- Apply item transformers for post-processing
- Include total count for page metadata
- Use `create_page()` to construct the response

## Important Files

- **`fastapi_pagination/api.py`** - Core pagination API with `paginate()`, `create_page()`, etc.
- **`fastapi_pagination/bases.py`** - Abstract base classes for params and pages
- **`fastapi_pagination/customization.py`** - Page customization and configuration
- **`pyproject.toml`** - Project configuration, dependencies, and tool settings
- **`.pre-commit-config.yaml`** - Pre-commit hook configuration
- **`pytest.ini`** - Pytest configuration
- **`mkdocs.yml`** - Documentation configuration

## Documentation

Documentation is built with mkdocs and hosted on Netlify:

```bash
# Install docs requirements
uv pip install -r docs_requirements.txt

# Serve docs locally
mkdocs serve

# Build docs
mkdocs build
```

Documentation files are in `docs/` and written in Markdown.

## Common Tasks

### Adding a new feature
1. Create an issue first to describe the idea
2. Implement the feature following existing patterns
3. Add tests in `tests/`
4. Update documentation in `docs/` if needed
5. Run pre-commit hooks: `uv run pre-commit run --all-files`
6. Run tests: `uv run pytest tests`
7. Create a pull request

### Adding a new database extension
1. Create a new module in `fastapi_pagination/ext/`
2. Use `fastapi_pagination/ext/sqlalchemy.py` as a reference
3. Add the extension to `project.optional-dependencies` in `pyproject.toml`
4. Add integration tests in `tests/ext/`
5. Update documentation

### Updating dependencies
The project uses uv for dependency management. Dependencies are declared in `pyproject.toml` and locked in `uv.lock`.

## Version Information

- **Current version:** 0.15.0
- **License:** MIT
- **Python:** 3.10, 3.11, 3.12, 3.13, 3.14
- **FastAPI:** >=0.93.0
- **Pydantic:** >=1.9.1

## CI/CD

The project uses GitHub Actions for CI:
- Runs tests on multiple Python versions
- Checks code coverage with codecov
- Validates formatting and linting
- Runs type checking

## Additional Notes

- The project uses limit-offset and cursor-based pagination strategies
- Each database extension follows a consistent API pattern
- Tests are split into unit tests and integration tests
- Integration tests require actual database instances (PostgreSQL, MongoDB, Cassandra)
- The library is designed to be framework-agnostic but optimized for FastAPI
- Use `add_pagination(app)` to register pagination with a FastAPI application
- The `Page` type is used as a return type annotation for endpoints
