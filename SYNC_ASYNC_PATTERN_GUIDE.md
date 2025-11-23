# Unified Sync/Async API Pattern Guide

## Summary: How fastapi-pagination Achieves Dual Sync/Async APIs

`fastapi-pagination` provides both synchronous and asynchronous APIs from a single codebase using these key techniques:

1. **Generator-based Flow System** - A custom abstraction that can execute the same logic synchronously or asynchronously
2. **Type Overloads** - Multiple function signatures that guide type checkers while routing to the same implementation
3. **Runtime Detection** - Inspecting connection types and callable signatures to dispatch to the appropriate execution mode
4. **Context Variables** - Propagating request-scoped state without threading parameters through every function
5. **Conditional Awaiting** - Helper utilities that await values only when they're awaitable

---

## The Core Pattern: Generator-Based Flows

The most innovative pattern is the **Flow system** - a generator that can yield both sync and async values, then executed differently based on context.

### The Flow Type

```python
from typing import TypeAlias, Generator, Awaitable, TypeVar

TArg = TypeVar('TArg')
R = TypeVar('R')

Flow: TypeAlias = Generator[
    Awaitable[TArg] | TArg,  # Can yield either sync or async values
    TArg,                     # Values sent back into the generator
    R,                        # Final return value
]
```

### Sync Flow Runner

```python
def run_sync_flow(gen: Flow[Any, R]) -> R:
    """Execute a flow synchronously - fails if async values are encountered."""
    try:
        res = gen.send(None)
        _check_not_coro(res)  # Ensure no coroutines in sync mode

        while True:
            try:
                res = gen.send(res)
                _check_not_coro(res)
            except StopIteration:
                raise
            except BaseException as exc:
                res = gen.throw(exc)
    except StopIteration as exc:
        return exc.value

def _check_not_coro(obj: Any) -> None:
    """Raise error if object is a coroutine."""
    if inspect.iscoroutine(obj):
        raise RuntimeError(f"Expected a non-coroutine value, got {obj}")
```

### Async Flow Runner

```python
async def run_async_flow(gen: Flow[Any, R]) -> R:
    """Execute a flow asynchronously - awaits values if they're awaitable."""
    try:
        res = gen.send(None)

        while True:
            try:
                res = await await_if_coro(res)  # Conditionally await
                res = gen.send(res)
            except StopIteration:
                raise
            except BaseException as exc:
                res = gen.throw(exc)
    except StopIteration as exc:
        return exc.value

async def await_if_coro(coro: Awaitable[R] | R) -> R:
    """Await if value is awaitable, otherwise return as-is."""
    if isinstance(coro, Awaitable):
        return await coro
    return coro
```

**Key Insight:** The same generator can be executed synchronously or asynchronously. The async version simply awaits yielded values if they're awaitable.

---

## Pattern 1: Type Overloads for Multiple Signatures

Use `@overload` to provide different type signatures while sharing implementation logic.

### Example: SQLAlchemy-style Pagination

```python
from typing import overload, Union, Optional, Any
from sqlalchemy.orm import Session
from sqlalchemy.ext.asyncio import AsyncSession

# Sync overload
@overload
def paginate(
    session: Session,
    query: Any,
    *,
    transformer: SyncItemsTransformer | None = None,
) -> Page:
    ...

# Async overload
@overload
async def paginate(
    session: AsyncSession,
    query: Any,
    *,
    transformer: AsyncItemsTransformer | None = None,
) -> Page:
    ...

# Actual implementation
def paginate(
    session: Union[Session, AsyncSession],
    query: Any,
    *,
    transformer: Any = None,
) -> Any:
    """Runtime dispatches based on session type."""

    # Detect async session
    if isinstance(session, AsyncSession):
        return _paginate_async(session, query, transformer=transformer)

    # Sync path
    return _paginate_sync(session, query, transformer=transformer)

def _paginate_sync(session: Session, query: Any, *, transformer: Any = None) -> Page:
    """Synchronous implementation."""
    total = session.execute(select(func.count()).select_from(query.subquery())).scalar()
    items = session.execute(query).scalars().all()

    if transformer:
        items = transformer(items)

    return Page(items=items, total=total)

async def _paginate_async(session: AsyncSession, query: Any, *, transformer: Any = None) -> Page:
    """Asynchronous implementation."""
    result = await session.execute(select(func.count()).select_from(query.subquery()))
    total = result.scalar()

    result = await session.execute(query)
    items = result.scalars().all()

    if transformer:
        items = await transformer(items)

    return Page(items=items, total=total)
```

**Benefits:**
- Type checkers see different signatures for sync vs async
- Single function name provides consistent API
- Runtime dispatch based on actual parameter types

---

## Pattern 2: Runtime Callable Detection

Determine if a function is async at runtime to handle it appropriately.

### Async Callable Detection

```python
import inspect
import functools
from typing import Any, Callable

def is_async_callable(obj: Any) -> bool:
    """Check if an object is an async callable."""
    # Unwrap partials to get to the real function
    while isinstance(obj, functools.partial):
        obj = obj.func

    # Check if it's a coroutine function
    if inspect.iscoroutinefunction(obj):
        return True

    # Check if __call__ is a coroutine function (for callable objects)
    if callable(obj) and inspect.iscoroutinefunction(obj.__call__):
        return True

    return False
```

### Conditional Execution

```python
from typing import TypeVar, Awaitable, Callable, ParamSpec

P = ParamSpec('P')
R = TypeVar('R')

async def await_if_async(
    func: Callable[P, R | Awaitable[R]],
    *args: P.args,
    **kwargs: P.kwargs
) -> R:
    """Call function and await result if it's async."""
    if is_async_callable(func):
        return await func(*args, **kwargs)
    return func(*args, **kwargs)
```

### Usage Example

```python
# Works with both sync and async transformers
async def apply_transformer(items: list, transformer: Callable | None) -> list:
    if transformer is None:
        return items

    # Automatically handles sync or async transformers
    return await await_if_async(transformer, items)

# Usage
async def async_transformer(items):
    await asyncio.sleep(0.1)
    return [x * 2 for x in items]

def sync_transformer(items):
    return [x * 2 for x in items]

# Both work!
result1 = await apply_transformer([1, 2, 3], async_transformer)
result2 = await apply_transformer([1, 2, 3], sync_transformer)
```

---

## Pattern 3: Flow-Based Unified Pipeline

Create a single pipeline that works in both sync and async contexts.

### Flow Decorator

```python
from typing import Callable, Generator
from functools import wraps

def flow(func: Callable[..., Generator]) -> Callable:
    """Mark a generator function as a flow."""
    return func

def flow_expr(expr: Callable[P, Awaitable[R] | R]) -> Callable[P, Flow[Any, R]]:
    """Convert a sync/async expression into a flow."""
    @wraps(expr)
    def flow_wrapper(*args: P.args, **kwargs: P.kwargs) -> Flow[Any, R]:
        res = yield expr(*args, **kwargs)  # Yield the expression result
        return res
    return flow_wrapper
```

### Example: Pagination Flow

```python
@flow
def pagination_flow(
    query: Any,
    connection: Any,
    *,
    async_: bool = False,
) -> Flow[Any, Page]:
    """Unified pagination flow for sync and async."""

    # Get total count (may be sync or async)
    total = yield get_total_count(connection, query)

    # Get items (may be sync or async)
    items = yield get_items(connection, query)

    # Apply transformer (may be sync or async)
    transformed = yield apply_transformer(items, async_=async_)

    # Create page (always sync)
    return create_page(items=transformed, total=total)

# Sync execution
def paginate_sync(query, connection):
    return run_sync_flow(
        pagination_flow(query, connection, async_=False)
    )

# Async execution
async def paginate_async(query, connection):
    return await run_async_flow(
        pagination_flow(query, connection, async_=True)
    )
```

**Benefits:**
- Write pagination logic once
- Execute synchronously or asynchronously
- Same code paths reduce bugs
- Easy to test both modes

---

## Pattern 4: Context Variables for State Propagation

Use `contextvars` to pass state through the call stack without explicit parameters.

### Setup Context Variables

```python
from contextvars import ContextVar
from contextlib import contextmanager
from typing import TypeVar, Optional

T = TypeVar('T')

# Define context variables
_current_params: ContextVar[Optional[PaginationParams]] = ContextVar('_current_params', default=None)
_current_transformer: ContextVar[Optional[Callable]] = ContextVar('_current_transformer', default=None)

# Context managers for setting values
@contextmanager
def set_params(params: PaginationParams):
    """Set pagination params in context."""
    token = _current_params.set(params)
    try:
        yield
    finally:
        _current_params.reset(token)

@contextmanager
def set_transformer(transformer: Callable):
    """Set transformer in context."""
    token = _current_transformer.set(transformer)
    try:
        yield
    finally:
        _current_transformer.reset(token)

# Getters
def get_params() -> PaginationParams:
    """Get current pagination params from context."""
    params = _current_params.get()
    if params is None:
        raise RuntimeError("No pagination params in context")
    return params

def get_transformer() -> Optional[Callable]:
    """Get current transformer from context."""
    return _current_transformer.get()
```

### Usage in API

```python
async def paginate_endpoint(db: AsyncSession):
    """FastAPI endpoint that uses context variables."""

    # Set params in context (normally done by dependency injection)
    params = PaginationParams(page=1, size=20)

    with set_params(params):
        # Deep in the call stack, params are accessible
        return await paginate(db, query)

async def paginate(db: AsyncSession, query):
    """Uses params from context instead of parameters."""
    params = get_params()  # Retrieved from context

    offset = (params.page - 1) * params.size
    limit = params.size

    result = await db.execute(query.offset(offset).limit(limit))
    return result.scalars().all()
```

**Benefits:**
- No parameter threading through every function
- Cleaner function signatures
- Request-scoped state in async contexts
- FastAPI dependency injection friendly

---

## Pattern 5: Separate Functions with Naming Convention

For libraries where runtime detection isn't feasible, use naming conventions.

### Example: Motor (MongoDB Async Driver)

```python
# Async-only function with 'a' prefix
async def apaginate(
    collection,
    filter: dict,
    *,
    transformer: AsyncItemsTransformer | None = None,
) -> Page:
    """Async pagination for MongoDB Motor."""
    total = await collection.count_documents(filter)
    cursor = collection.find(filter).skip(offset).limit(limit)
    items = await cursor.to_list(length=limit)

    if transformer:
        items = await transformer(items)

    return Page(items=items, total=total)

# Sync version (if using PyMongo)
def paginate(
    collection,
    filter: dict,
    *,
    transformer: SyncItemsTransformer | None = None,
) -> Page:
    """Sync pagination for MongoDB PyMongo."""
    total = collection.count_documents(filter)
    items = list(collection.find(filter).skip(offset).limit(limit))

    if transformer:
        items = transformer(items)

    return Page(items=items, total=total)
```

**Naming Convention:**
- `paginate()` - sync version
- `apaginate()` - async version
- Clear, explicit differentiation
- No runtime detection needed

---

## Pattern 6: Transformer Strategy Pattern

Support both sync and async transformers with a unified interface.

### Type Definitions

```python
from typing import TypeAlias, Callable, Sequence, Awaitable

# Transformer type aliases
SyncItemsTransformer: TypeAlias = Callable[[Sequence[Any]], Sequence[Any]]
AsyncItemsTransformer: TypeAlias = Callable[[Sequence[Any]], Awaitable[Sequence[Any]]]
ItemsTransformer: TypeAlias = SyncItemsTransformer | AsyncItemsTransformer
```

### Unified Transformer Application

```python
from typing import Literal, overload

@overload
def apply_items_transformer(
    items: Sequence[Any],
    transformer: SyncItemsTransformer | None = None,
    *,
    async_: Literal[False] = False,
) -> Sequence[Any]:
    ...

@overload
async def apply_items_transformer(
    items: Sequence[Any],
    transformer: AsyncItemsTransformer | None = None,
    *,
    async_: Literal[True],
) -> Sequence[Any]:
    ...

def apply_items_transformer(
    items: Sequence[Any],
    transformer: ItemsTransformer | None = None,
    *,
    async_: bool = False,
) -> Any:
    """Apply transformer in sync or async mode."""
    if transformer is None:
        return items

    # Check if transformer is async
    is_async = is_async_callable(transformer)

    # Validate mode matches transformer type
    if is_async and not async_:
        raise ValueError("Async transformer requires async_=True")

    if not is_async and async_:
        raise ValueError("Sync transformer with async_=True")

    # Execute appropriately
    if is_async:
        return transformer(items)  # Returns awaitable
    else:
        return transformer(items)  # Returns value directly
```

### Usage

```python
# In sync context
items = apply_items_transformer([1, 2, 3], lambda x: [i*2 for i in x], async_=False)

# In async context
items = await apply_items_transformer(
    [1, 2, 3],
    async_transformer,
    async_=True
)
```

---

## Complete Example: Building a Dual Sync/Async Library

Let's build a complete pagination library that works with both sync and async ORMs.

### Step 1: Define Types and Bases

```python
# types.py
from typing import TypeAlias, Generic, TypeVar, Sequence, Callable, Awaitable
from pydantic import BaseModel

T = TypeVar('T')

# Transformer types
SyncTransformer: TypeAlias = Callable[[Sequence[T]], Sequence[T]]
AsyncTransformer: TypeAlias = Callable[[Sequence[T]], Awaitable[Sequence[T]]]
Transformer: TypeAlias = SyncTransformer[T] | AsyncTransformer[T]

# Page model
class Page(BaseModel, Generic[T]):
    items: Sequence[T]
    total: int
    page: int
    size: int
    pages: int

# Params model
class PaginationParams(BaseModel):
    page: int = 1
    size: int = 20

    @property
    def offset(self) -> int:
        return (self.page - 1) * self.size

    @property
    def limit(self) -> int:
        return self.size
```

### Step 2: Create Flow System

```python
# flow.py
from typing import Generator, TypeVar, Awaitable, Any
import inspect

R = TypeVar('R')

Flow = Generator[Awaitable[Any] | Any, Any, R]

def run_sync_flow(gen: Flow[R]) -> R:
    """Execute flow synchronously."""
    try:
        res = gen.send(None)
        if inspect.iscoroutine(res):
            raise RuntimeError(f"Cannot await in sync context: {res}")

        while True:
            try:
                res = gen.send(res)
                if inspect.iscoroutine(res):
                    raise RuntimeError(f"Cannot await in sync context: {res}")
            except StopIteration as e:
                return e.value
            except BaseException as exc:
                res = gen.throw(exc)
    except StopIteration as e:
        return e.value

async def run_async_flow(gen: Flow[R]) -> R:
    """Execute flow asynchronously."""
    try:
        res = gen.send(None)

        while True:
            try:
                # Await if result is awaitable
                if inspect.isawaitable(res):
                    res = await res
                res = gen.send(res)
            except StopIteration as e:
                return e.value
            except BaseException as exc:
                res = gen.throw(exc)
    except StopIteration as e:
        return e.value
```

### Step 3: Build Pagination Flow

```python
# core.py
from typing import Any, Callable, Awaitable
from .types import Page, PaginationParams, Transformer
from .flow import Flow, run_sync_flow, run_async_flow
from .utils import is_async_callable

def pagination_flow(
    count_func: Callable[[], int | Awaitable[int]],
    items_func: Callable[[int, int], Any],
    params: PaginationParams,
    transformer: Transformer | None = None,
) -> Flow[Page]:
    """Generic pagination flow that works sync or async."""

    # Get total count
    total = yield count_func()

    # Get items for current page
    items = yield items_func(params.offset, params.limit)

    # Apply transformer if provided
    if transformer is not None:
        items = yield transformer(items)

    # Calculate total pages
    pages = (total + params.size - 1) // params.size

    # Return page
    return Page(
        items=items,
        total=total,
        page=params.page,
        size=params.size,
        pages=pages,
    )

def create_page_sync(
    count_func: Callable[[], int],
    items_func: Callable[[int, int], Any],
    params: PaginationParams,
    transformer: Transformer | None = None,
) -> Page:
    """Create page synchronously."""
    return run_sync_flow(
        pagination_flow(count_func, items_func, params, transformer)
    )

async def create_page_async(
    count_func: Callable[[], int | Awaitable[int]],
    items_func: Callable[[int, int], Any],
    params: PaginationParams,
    transformer: Transformer | None = None,
) -> Page:
    """Create page asynchronously."""
    return await run_async_flow(
        pagination_flow(count_func, items_func, params, transformer)
    )
```

### Step 4: SQLAlchemy Integration

```python
# ext/sqlalchemy.py
from typing import overload, Any
from sqlalchemy import select, func
from sqlalchemy.orm import Session
from sqlalchemy.ext.asyncio import AsyncSession
from ..types import Page, PaginationParams, SyncTransformer, AsyncTransformer
from ..core import create_page_sync, create_page_async

@overload
def paginate(
    session: Session,
    query: Any,
    params: PaginationParams | None = None,
    *,
    transformer: SyncTransformer | None = None,
) -> Page:
    ...

@overload
async def paginate(
    session: AsyncSession,
    query: Any,
    params: PaginationParams | None = None,
    *,
    transformer: AsyncTransformer | None = None,
) -> Page:
    ...

def paginate(
    session: Session | AsyncSession,
    query: Any,
    params: PaginationParams | None = None,
    *,
    transformer: Any = None,
) -> Any:
    """Paginate SQLAlchemy query (sync or async)."""
    if params is None:
        params = PaginationParams()

    # Detect async session
    if isinstance(session, AsyncSession):
        return _paginate_async(session, query, params, transformer)

    return _paginate_sync(session, query, params, transformer)

def _paginate_sync(
    session: Session,
    query: Any,
    params: PaginationParams,
    transformer: SyncTransformer | None = None,
) -> Page:
    """Sync SQLAlchemy pagination."""

    def count_func():
        count_query = select(func.count()).select_from(query.subquery())
        return session.execute(count_query).scalar() or 0

    def items_func(offset: int, limit: int):
        result = session.execute(query.offset(offset).limit(limit))
        return result.scalars().all()

    return create_page_sync(count_func, items_func, params, transformer)

async def _paginate_async(
    session: AsyncSession,
    query: Any,
    params: PaginationParams,
    transformer: AsyncTransformer | None = None,
) -> Page:
    """Async SQLAlchemy pagination."""

    async def count_func():
        count_query = select(func.count()).select_from(query.subquery())
        result = await session.execute(count_query)
        return result.scalar() or 0

    async def items_func(offset: int, limit: int):
        result = await session.execute(query.offset(offset).limit(limit))
        return result.scalars().all()

    return await create_page_async(count_func, items_func, params, transformer)
```

### Step 5: Usage Examples

```python
# Sync usage with SQLAlchemy
from sqlalchemy.orm import Session
from .ext.sqlalchemy import paginate

def get_users_sync(db: Session) -> Page:
    query = select(User).where(User.is_active == True)
    params = PaginationParams(page=1, size=20)

    return paginate(db, query, params)

# Async usage with SQLAlchemy
from sqlalchemy.ext.asyncio import AsyncSession

async def get_users_async(db: AsyncSession) -> Page:
    query = select(User).where(User.is_active == True)
    params = PaginationParams(page=1, size=20)

    # Same function name!
    return await paginate(db, query, params)

# With transformer
async def get_users_with_transform(db: AsyncSession) -> Page:
    query = select(User)

    async def transform(users):
        # Add computed fields
        for user in users:
            user.full_name = f"{user.first_name} {user.last_name}"
        return users

    return await paginate(db, query, transformer=transform)
```

---

## Best Practices and Recommendations

### When to Use This Pattern

✅ **Good Use Cases:**
- Database libraries (ORMs/ODMs)
- HTTP clients
- File I/O abstractions
- Cache clients
- Any library that needs to support both sync and async frameworks

❌ **When NOT to Use:**
- Simple, single-purpose libraries
- When users will only use one mode
- When the complexity outweighs the benefits
- Pure async libraries (just use async)

### Design Guidelines

1. **Keep flow logic simple** - Complex flows are hard to debug
2. **Use type overloads liberally** - Help type checkers catch errors early
3. **Document execution mode clearly** - Users need to know when things are async
4. **Validate mode mismatches** - Fail fast if sync transformer used in async mode
5. **Test both modes** - Easy to break one mode while working on the other
6. **Consider performance** - Generator overhead is usually negligible but measure

### Common Pitfalls

1. **Forgetting to await in async mode** - Use type checkers to catch
2. **Mixing sync and async inappropriately** - Validate at runtime
3. **Context variable cleanup** - Always use context managers
4. **Overcomplicating the flow system** - Start simple, add complexity as needed
5. **Poor error messages** - Make it clear when mode mismatch occurs

---

## Testing Strategy

### Test Both Modes

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.ext.asyncio import create_async_engine

# Sync test
def test_pagination_sync():
    engine = create_engine("sqlite:///:memory:")
    with Session(engine) as session:
        page = paginate(session, select(User), PaginationParams(page=1, size=10))
        assert page.total >= 0
        assert len(page.items) <= 10

# Async test
@pytest.mark.asyncio
async def test_pagination_async():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with AsyncSession(engine) as session:
        page = await paginate(session, select(User), PaginationParams(page=1, size=10))
        assert page.total >= 0
        assert len(page.items) <= 10

# Test transformer
@pytest.mark.asyncio
async def test_async_transformer():
    async def transform(items):
        await asyncio.sleep(0.01)  # Simulate async work
        return [item.upper() for item in items]

    page = await paginate(session, query, transformer=transform)
    assert all(item.isupper() for item in page.items)
```

### Validate Mode Mismatches

```python
def test_sync_transformer_in_async_mode():
    """Sync transformer should fail in async mode."""
    def sync_transform(items):
        return items

    with pytest.raises(ValueError, match="async transformer required"):
        await paginate(async_session, query, transformer=sync_transform)

def test_async_transformer_in_sync_mode():
    """Async transformer should fail in sync mode."""
    async def async_transform(items):
        return items

    with pytest.raises(ValueError, match="sync transformer required"):
        paginate(sync_session, query, transformer=async_transform)
```

---

## Performance Considerations

### Flow Overhead

The generator-based flow system adds minimal overhead:
- **Sync mode:** ~5-10% slower than direct calls (usually microseconds)
- **Async mode:** Overhead is negligible compared to I/O wait times
- **Memory:** Minimal - just the generator frame

### When Performance Matters

If the overhead is unacceptable:
1. **Separate implementations** - Don't use flows, write sync and async separately
2. **Inline hot paths** - Use flows for complex logic, direct calls for simple operations
3. **Profile first** - Measure before optimizing

---

## Conclusion

The fastapi-pagination sync/async pattern demonstrates:

1. **Generator-based flows** enable unified logic that executes in both modes
2. **Type overloads** provide type safety without runtime overhead
3. **Runtime detection** automatically dispatches to the correct mode
4. **Context variables** eliminate parameter threading
5. **Conditional awaiting** handles mixed sync/async callables gracefully

This pattern is production-tested in fastapi-pagination, supporting 20+ database backends with a single unified codebase. It reduces bugs, simplifies maintenance, and provides an excellent developer experience.

### Key Takeaways for Your Projects

- **Start with overloads** if you're adding async to an existing sync API
- **Use flows** when you have complex multi-step pipelines
- **Leverage context variables** to avoid parameter pollution
- **Test both modes** thoroughly - easy to break one while working on the other
- **Document clearly** which mode is being used at any point

The complexity is justified when:
- You're building a library used by many projects
- Supporting both sync and async is a core requirement
- The unified codebase reduces maintenance burden significantly

For application code, prefer picking one mode (usually async) and sticking with it!
