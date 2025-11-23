# Sync/Async Pattern Guide

How fastapi-pagination provides unified sync/async APIs from a single codebase.

## Core Technique: Generator-Based Flows

**The Flow:** Generator that yields sync or async values, executed differently based on mode.

```python
from typing import Generator, Awaitable, TypeVar

R = TypeVar('R')
Flow = Generator[Awaitable[Any] | Any, Any, R]

# Sync runner - fails if coroutines encountered
def run_sync_flow(gen: Flow[R]) -> R:
    res = gen.send(None)
    while True:
        try:
            if inspect.iscoroutine(res):
                raise RuntimeError("Cannot await in sync context")
            res = gen.send(res)
        except StopIteration as e:
            return e.value

# Async runner - conditionally awaits
async def run_async_flow(gen: Flow[R]) -> R:
    res = gen.send(None)
    while True:
        try:
            res = await res if inspect.isawaitable(res) else res  # Key difference
            res = gen.send(res)
        except StopIteration as e:
            return e.value
```

**Usage:**
```python
def pagination_flow(count_fn, items_fn):
    total = yield count_fn()   # May be int or Awaitable[int]
    items = yield items_fn()   # May be list or Awaitable[list]
    return Page(items=items, total=total)

# Sync
page = run_sync_flow(pagination_flow(sync_count, sync_items))

# Async
page = await run_async_flow(pagination_flow(async_count, async_items))
```

---

## Pattern 1: Type Overloads

Provide type-safe sync and async signatures for the same function.

```python
from typing import overload
from sqlalchemy.orm import Session
from sqlalchemy.ext.asyncio import AsyncSession

@overload
def paginate(db: Session, query) -> Page: ...

@overload
async def paginate(db: AsyncSession, query) -> Page: ...

def paginate(db: Session | AsyncSession, query) -> Page:
    if isinstance(db, AsyncSession):
        return await _async_impl(db, query)
    return _sync_impl(db, query)
```

**Files:** `ext/sqlalchemy.py:381-464`, `ext/sqlmodel.py:77-123`

---

## Pattern 2: Runtime Detection

Inspect callables at runtime to determine sync vs async.

```python
import inspect
import functools

def is_async_callable(func) -> bool:
    """Check if callable is async."""
    while isinstance(func, functools.partial):
        func = func.func
    return (inspect.iscoroutinefunction(func) or
            (callable(func) and inspect.iscoroutinefunction(func.__call__)))

async def await_if_async(func, *args):
    """Call function and await if async."""
    if is_async_callable(func):
        return await func(*args)
    return func(*args)
```

**Files:** `utils.py:64-101`

---

## Pattern 3: Context Variables

Propagate request-scoped state without parameter threading.

```python
from contextvars import ContextVar
from contextlib import contextmanager

_params: ContextVar[Params] = ContextVar('_params')

@contextmanager
def set_params(params):
    token = _params.set(params)
    try:
        yield
    finally:
        _params.reset(token)

# Use anywhere in call stack
def deep_function():
    params = _params.get()  # No parameter passing needed
```

**Files:** `api.py:51-58`

---

## Pattern 4: Conditional Transformer

Support both sync and async transformers with validation.

```python
from typing import overload, Literal

@overload
def apply_items_transformer(
    items: list,
    transformer: SyncTransformer | None = None,
    *,
    async_: Literal[False] = False,
) -> list: ...

@overload
async def apply_items_transformer(
    items: list,
    transformer: AsyncTransformer | None = None,
    *,
    async_: Literal[True],
) -> list: ...

def apply_items_transformer(items, transformer=None, *, async_=False):
    if transformer is None:
        return items

    is_async = is_async_callable(transformer)

    if is_async and not async_:
        raise ValueError("Async transformer requires async_=True")

    if is_async:
        return transformer(items)  # Returns awaitable
    return transformer(items)
```

**Files:** `api.py:164-206`

---

## Minimal Example

Complete paginator in ~40 lines:

```python
from typing import TypeVar, Generic, overload
from sqlalchemy.orm import Session
from sqlalchemy.ext.asyncio import AsyncSession
import inspect

T = TypeVar('T')

class Page(Generic[T]):
    items: list[T]
    total: int

def pagination_flow(count_fn, items_fn):
    """Generator yielding sync or async operations."""
    total = yield count_fn()
    items = yield items_fn()
    return Page(items=items, total=total)

def run_sync(gen):
    res = gen.send(None)
    while True:
        try:
            res = gen.send(res)
        except StopIteration as e:
            return e.value

async def run_async(gen):
    res = gen.send(None)
    while True:
        try:
            res = await res if inspect.isawaitable(res) else res
            res = gen.send(res)
        except StopIteration as e:
            return e.value

@overload
def paginate(db: Session, query) -> Page: ...

@overload
async def paginate(db: AsyncSession, query) -> Page: ...

def paginate(db, query):
    if isinstance(db, AsyncSession):
        return run_async(pagination_flow(
            lambda: db.execute(query.count()),
            lambda: db.execute(query)
        ))
    return run_sync(pagination_flow(
        lambda: db.execute(query.count()).scalar(),
        lambda: db.execute(query).scalars().all()
    ))
```

---

## Key Architecture Files

| File | Purpose | Key Functions |
|------|---------|---------------|
| `flow.py:44-74` | Flow runners | `run_sync_flow()`, `run_async_flow()` |
| `flows.py:80-145` | Generic flow | `generic_flow()` with `async_` flag |
| `api.py:164-206` | Transformer | `apply_items_transformer()` |
| `utils.py:64-101` | Detection | `is_async_callable()`, `await_if_async()` |
| `ext/sqlalchemy.py` | Reference impl | Overloaded `paginate()` |

---

## When to Use This Pattern

✅ **Use when:**
- Building libraries for both sync and async frameworks
- Complex multi-step pipelines to avoid duplication
- Supporting 20+ integrations (like fastapi-pagination)

❌ **Don't use when:**
- Simple single-purpose libraries
- Application code (pick one mode)
- Performance-critical hot paths (5-10% overhead)

---

## Common Pitfalls

```python
# ❌ Wrong: Forgetting await
page = paginate(async_db, query)

# ✅ Correct
page = await paginate(async_db, query)

# ❌ Wrong: Sync transformer in async mode
def sync_trans(items): return items
await paginate(async_db, query, transformer=sync_trans)

# ✅ Correct
async def async_trans(items): return items
await paginate(async_db, query, transformer=async_trans)
```

---

## Testing Strategy

```python
import pytest

def test_sync_mode():
    page = paginate(sync_session, query, params)
    assert page.total >= 0

@pytest.mark.asyncio
async def test_async_mode():
    page = await paginate(async_session, query, params)
    assert page.total >= 0

def test_mode_mismatch():
    with pytest.raises(ValueError):
        await paginate(async_db, query, transformer=sync_transformer)
```

---

## Quick Implementation Checklist

1. ✅ Define flow type: `Generator[Awaitable[T] | T, T, R]`
2. ✅ Create sync runner (no await)
3. ✅ Create async runner (conditional await)
4. ✅ Add type overloads for API
5. ✅ Implement runtime detection helpers
6. ✅ Test both modes thoroughly

**Result:** Single codebase, dual execution modes, type-safe APIs.
