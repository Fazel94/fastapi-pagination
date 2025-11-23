# fastapi-pagination

Python library for FastAPI pagination with 20+ database integrations. Supports both sync and async, cursor and page-based pagination.

**Links:** [Repo](https://github.com/uriyyo/fastapi-pagination) | [Docs](https://uriyyo-fastapi-pagination.netlify.app/)

## Tech Stack

Python 3.10+ • FastAPI • Pydantic • uv • ruff • mypy • pytest • mkdocs

## Key Directories

- `fastapi_pagination/` - Core library (`api.py`, `bases.py`, `flow.py`, `flows.py`)
- `fastapi_pagination/ext/` - 20+ database integrations (SQLAlchemy, Beanie, Motor, Django, etc.)
- `tests/` - Unit tests and integration tests (requires PostgreSQL, MongoDB, Cassandra)

## Development

**Setup:**
```bash
uv sync --dev --all-extras    # Install dependencies
uv run pre-commit install     # Install hooks
```

**Code Quality:**
```bash
uv run pre-commit run --all-files   # Format, lint, type check
uv run ruff format fastapi_pagination tests
uv run mypy fastapi_pagination
```

**Testing:**
```bash
./scripts/ci-prepare.sh              # Prepare environment
uv run pytest tests                  # Run all tests
uv run pytest tests --unit-tests     # Unit tests only
```

## Code Style

**Ruff:** 120 char lines, all rules except D/ANN/TD/ARG (no docstrings/annotations required)
**Mypy:** Strict mode, Python 3.10+
**Coverage:** Excludes `pragma: no cover`, `@abstractmethod`, `@overload`, `TYPE_CHECKING`

## Adding Database Extensions

Create `fastapi_pagination/ext/your_db.py`:
```python
async def paginate(query, params=None, *, transformer=None, additional_data=None):
    params, raw_params = verify_params(params, "limit-offset")
    total = await query.count()
    items = await query.limit(raw_params.limit).offset(raw_params.offset).all()
    t_items = await apply_items_transformer(items, transformer, async_=True)
    return create_page(t_items, total=total, params=params, **(additional_data or {}))
```
Pattern: `verify_params()` → get total/items → `apply_items_transformer()` → `create_page()`

## Core Files

- `api.py` - Core API (`create_page()`, `apply_items_transformer()`)
- `flow.py` - Flow system (`run_sync_flow()`, `run_async_flow()`)
- `flows.py` - Generic pagination flow
- `bases.py` - Abstract base classes
- `ext/sqlalchemy.py` - Reference implementation for extensions

## Sync/Async Architecture

See `SYNC_ASYNC_GUIDE.md` for details. Key technique: generator-based flows that execute the same logic in either sync or async mode. The async runner conditionally awaits yielded values, while the sync runner doesn't.

## Quick Reference

**Version:** 0.15.0 • **License:** MIT • **Python:** 3.10-3.14
