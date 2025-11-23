# Sync/Async Pattern - Quick Reference

## How fastapi-pagination Does It (TL;DR)

**The Magic Formula:**
```
Generator-based Flows + Type Overloads + Runtime Detection = Unified Sync/Async API
```

### 5 Core Techniques

#### 1. Generator-Based Flows
Same logic, different execution modes:

```python
# Define flow
Flow = Generator[Awaitable[T] | T, T, R]

# Run sync
def run_sync_flow(gen):
    res = gen.send(None)
    # ... continues synchronously

# Run async
async def run_async_flow(gen):
    res = gen.send(None)
    res = await res if isawaitable(res) else res
    # ... continues asynchronously
```

**Key:** Async runner awaits yielded values, sync runner doesn't.

#### 2. Type Overloads
Guide type checkers, route to same implementation:

```python
@overload
def paginate(db: Session, query) -> Page: ...

@overload
async def paginate(db: AsyncSession, query) -> Page: ...

def paginate(db: Session | AsyncSession, query) -> Page:
    if isinstance(db, AsyncSession):
        return await _async_impl(db, query)
    return _sync_impl(db, query)
```

#### 3. Runtime Detection
Inspect at runtime to dispatch correctly:

```python
def is_async_callable(func):
    while isinstance(func, functools.partial):
        func = func.func
    return inspect.iscoroutinefunction(func)

# Use it
if is_async_callable(transformer):
    result = await transformer(items)
else:
    result = transformer(items)
```

#### 4. Context Variables
Propagate state without parameter threading:

```python
_params_ctx: ContextVar[Params] = ContextVar('_params_ctx')

@contextmanager
def set_params(params):
    token = _params_ctx.set(params)
    try:
        yield
    finally:
        _params_ctx.reset(token)

# Use anywhere in call stack
def deep_function():
    params = _params_ctx.get()  # No need to pass as parameter
```

#### 5. Conditional Awaiting
Handle both sync and async values:

```python
async def await_if_coro(value):
    if isinstance(value, Awaitable):
        return await value
    return value

async def await_if_async(func, *args):
    if is_async_callable(func):
        return await func(*args)
    return func(*args)
```

---

## Real-World Example: Building a Paginator

### Minimal Implementation (50 lines)

```python
from typing import TypeVar, Generic, Awaitable, overload
from sqlalchemy.orm import Session
from sqlalchemy.ext.asyncio import AsyncSession
import inspect

T = TypeVar('T')

class Page(Generic[T]):
    items: list[T]
    total: int

# The key: generator that yields sync or async operations
def pagination_flow(count_fn, items_fn):
    total = yield count_fn()      # May be int or Awaitable[int]
    items = yield items_fn()      # May be list or Awaitable[list]
    return Page(items=items, total=total)

# Sync runner
def run_sync(gen):
    res = gen.send(None)
    while True:
        try:
            res = gen.send(res)
        except StopIteration as e:
            return e.value

# Async runner
async def run_async(gen):
    res = gen.send(None)
    while True:
        try:
            res = await res if inspect.isawaitable(res) else res
            res = gen.send(res)
        except StopIteration as e:
            return e.value

# Overloaded API
@overload
def paginate(db: Session, query) -> Page: ...

@overload
async def paginate(db: AsyncSession, query) -> Page: ...

def paginate(db, query):
    if isinstance(db, AsyncSession):
        async def count(): return (await db.execute(query.count())).scalar()
        async def items(): return (await db.execute(query)).scalars().all()
        return run_async(pagination_flow(count, items))
    else:
        count = lambda: db.execute(query.count()).scalar()
        items = lambda: db.execute(query).scalars().all()
        return run_sync(pagination_flow(count, items))
```

**Usage:**
```python
# Sync
page = paginate(sync_session, query)

# Async
page = await paginate(async_session, query)
```

---

## Key Files in fastapi-pagination

| File | Purpose | Key Functions |
|------|---------|---------------|
| `flow.py` | Flow system core | `run_sync_flow()`, `run_async_flow()` |
| `flows.py` | Generic pagination flow | `generic_flow()` |
| `api.py` | Public API, context vars | `create_page()`, `apply_items_transformer()` |
| `utils.py` | Runtime detection | `is_async_callable()`, `await_if_async()` |
| `ext/sqlalchemy.py` | SQLAlchemy integration | `paginate()`, `apaginate()` |
| `bases.py` | Abstract interfaces | `AbstractParams`, `AbstractPage` |

---

## Design Patterns Summary

### Pattern: Type Overload Dispatch

```python
@overload
def func(x: SyncType) -> Result: ...

@overload
async def func(x: AsyncType) -> Result: ...

def func(x: SyncType | AsyncType) -> Result:
    if isinstance(x, AsyncType):
        return await async_impl(x)
    return sync_impl(x)
```

**When:** You want one function name for both modes

### Pattern: Flow-Based Unification

```python
@flow
def process_flow(data, async_=False):
    result = yield fetch(data)  # May be sync or async
    transformed = yield transform(result)
    return transformed

# Execute
sync_result = run_sync_flow(process_flow(data, async_=False))
async_result = await run_async_flow(process_flow(data, async_=True))
```

**When:** Complex multi-step pipelines that should work in both modes

### Pattern: Context Variable State

```python
_state: ContextVar[State] = ContextVar('_state')

with set_state(my_state):
    # Available throughout call stack
    deep_function()  # Can access via _state.get()
```

**When:** Need request-scoped state without parameter threading

### Pattern: Runtime Callable Detection

```python
if is_async_callable(func):
    result = await func()
else:
    result = func()
```

**When:** Supporting both sync and async callbacks/transformers

### Pattern: Naming Convention

```python
def paginate(...) -> Page:        # Sync version
async def apaginate(...) -> Page:  # Async version (a prefix)
```

**When:** Runtime detection not feasible, keep it explicit

---

## Decision Tree: Which Pattern to Use?

```
Do you need to support both sync and async?
├─ No → Just use async, simpler!
└─ Yes
   ├─ Is the logic complex with multiple steps?
   │  ├─ Yes → Use flow-based pattern
   │  └─ No → Use type overloads with separate implementations
   │
   ├─ Can you detect sync vs async at runtime?
   │  ├─ Yes (e.g., Session type) → Use overloaded dispatch
   │  └─ No → Use naming convention (paginate/apaginate)
   │
   └─ Do you need request-scoped state?
      ├─ Yes → Use context variables
      └─ No → Pass parameters normally
```

---

## Common Gotchas

### 1. Forgetting to Await
```python
# ❌ Wrong
page = paginate(async_session, query)

# ✅ Correct
page = await paginate(async_session, query)
```

### 2. Sync Transformer in Async Context
```python
# ❌ Wrong - will fail at runtime
def sync_transformer(items): return items
await paginate(async_db, query, transformer=sync_transformer)

# ✅ Correct
async def async_transformer(items): return items
await paginate(async_db, query, transformer=async_transformer)
```

### 3. Not Resetting Context Variables
```python
# ❌ Wrong - leaks state
token = ctx_var.set(value)
# ... do work
# forget to reset!

# ✅ Correct - use context manager
@contextmanager
def set_value(value):
    token = ctx_var.set(value)
    try:
        yield
    finally:
        ctx_var.reset(token)
```

### 4. Yielding Coroutines in Sync Flow
```python
# ❌ Wrong - async operation in sync context
def sync_flow():
    result = yield async_function()  # Returns coroutine, not awaited!
    return result

# ✅ Correct - use sync operations in sync flow
def sync_flow():
    result = yield sync_function()
    return result
```

---

## Testing Checklist

- [ ] Test sync mode with sync session/connection
- [ ] Test async mode with async session/connection
- [ ] Test sync transformer in sync mode
- [ ] Test async transformer in async mode
- [ ] Test error on sync transformer in async mode
- [ ] Test error on async transformer in sync mode
- [ ] Test context variable cleanup
- [ ] Test type checking passes for both overloads
- [ ] Test edge cases (empty results, errors, etc.)

---

## Quick Implementation Guide

### Step 1: Define Your Types
```python
SyncTransformer = Callable[[T], T]
AsyncTransformer = Callable[[T], Awaitable[T]]
```

### Step 2: Create Detection Utilities
```python
def is_async_callable(func): ...
async def await_if_coro(value): ...
```

### Step 3: Build Core Logic (Choose One)

**Option A: Flow-based**
```python
@flow
def my_flow(...):
    result = yield operation()
    return result
```

**Option B: Separate implementations**
```python
def sync_impl(...): ...
async def async_impl(...): ...
```

### Step 4: Create Overloaded API
```python
@overload
def api(sync_type) -> Result: ...

@overload
async def api(async_type) -> Result: ...

def api(sync_or_async_type) -> Result:
    # Dispatch based on type
```

### Step 5: Test Both Modes
```python
def test_sync(): assert api(sync_db) == expected
async def test_async(): assert await api(async_db) == expected
```

---

## When NOT to Use This Pattern

- ❌ Simple libraries (overhead not worth it)
- ❌ Async-only applications (just use async)
- ❌ Performance-critical hot paths (adds 5-10% overhead)
- ❌ When users will only use one mode
- ❌ Team not familiar with advanced Python patterns

## When to Use This Pattern

- ✅ Building a library used by many projects
- ✅ Users need both sync and async support
- ✅ Complex logic that would be duplicated otherwise
- ✅ Maintenance burden of two codebases too high
- ✅ ORMs, HTTP clients, cache clients, etc.

---

## Resources

- Full guide: `SYNC_ASYNC_PATTERN_GUIDE.md`
- fastapi-pagination source: https://github.com/uriyyo/fastapi-pagination
- Python async: https://docs.python.org/3/library/asyncio.html
- Type overloads: https://peps.python.org/pep-0484/#function-method-overloading
- Context variables: https://peps.python.org/pep-0567/
