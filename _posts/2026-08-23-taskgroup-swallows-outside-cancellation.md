---
layout: post
title: The TaskGroup that ate your timeout
subtitle: Cancellation in asyncio is a counter, not a flag — and TaskGroup only knew how to check it for zero
tags: [cpython, asyncio, cancellation, taskgroup]
---

Here is a program that wraps a task group in a half-second timeout, and then does not
time out:

```python
import asyncio

async def worker():
    try:
        await asyncio.sleep(3600)
    finally:
        await asyncio.sleep(1)

async def main():
    async with asyncio.timeout(0.5):
        async with asyncio.TaskGroup() as tg:
            tg.create_task(worker())
            await asyncio.sleep(0.1)
            tg.cancel()
    print("no error")

asyncio.run(main())
```

Expected: `TimeoutError`. Actual: `no error`. Nothing here is exotic — `asyncio.timeout`
around a `TaskGroup` is the ordinary way to bound a fan-out, and `tg.cancel()`, new in
3.15, is the intended way to stop a group early — replacing the old boilerplate of raising
a sentinel exception from a throwaway task. It is the combination: after `tg.cancel()`, the
group would swallow any cancellation coming from outside it.

## Cancellation is a counter

`Task.cancel()` does not set a flag. It increments one:

```python
def uncancel(self):
    """Decrement the task's count of cancellation requests."""
    if self._num_cancels_requested > 0:
        self._num_cancels_requested -= 1
        ...
    return self._num_cancels_requested
```

This exists because cancellation has an owner. `asyncio.timeout` cancels the task it
guards; so does `TaskGroup`; so does whoever called `task.cancel()` from outside. When one
of them catches the resulting `CancelledError`, it has to know whether that was delivery of
*its own* request — which it converts into something meaningful, like `TimeoutError`, and
stops there — or someone else's, which has to keep travelling. A boolean cannot answer that
when two parties ask at once. A counter can.

## Where the second cancellation goes

`tg.cancel()` cancels the children and the group's own parent task, then sets
`_parent_cancel_requested`. The body has nothing left to await, so `__aexit__` runs and
parks on the future that waits for the children:

```python
try:
    await self._on_completed_fut
except exceptions.CancelledError as ex:
    if not self._aborting:
        propagate_cancellation_error = ex
        self._abort()
```

`self._aborting` is already true — that is what `tg.cancel()` did — so the
`CancelledError` is caught and dropped. Correct, for the group's own cancellation.

But `worker` has `await asyncio.sleep(1)` in a `finally`, so it takes ~1.1s to actually
die, and the 0.5s deadline lands in the middle of that wait. The timeout cancels the
parent a second time, the counter goes to 2, and `__aexit__` catches that `CancelledError`
in the very same `except`, with `_aborting` still true. Dropped again — this time wrongly.

By the time the children are gone, the only surviving trace of the timeout is the counter:

```python
if self._parent_cancel_requested:
    # If this flag is set we *must* call uncancel().
    if self._parent_task.uncancel() == 0:
        # If there are no pending cancellations left,
        # don't propagate CancelledError.
        propagate_cancellation_error = None
```

`uncancel()` returns 1, not 0. So the group correctly declines to clear
`propagate_cancellation_error` — which was never set, because the `except` clause threw the
exception away. There was no `else`. Nothing is raised, `__aexit__` returns cleanly, and
`asyncio.timeout` never sees a `CancelledError` to turn into a `TimeoutError`.

## The fix

Remember the cancellation you dropped, and hand it back when the counter says it was not
yours:

```python
    except exceptions.CancelledError as ex:
        pending_cancellation_error = ex
        ...

if self._parent_cancel_requested:
    if self._parent_task.uncancel() == 0:
        propagate_cancellation_error = None
    elif propagate_cancellation_error is None:
        # gh-155433: the remaining cancellation is not ours, don't drop it
        propagate_cancellation_error = pending_cancellation_error
```

[#155439](https://github.com/python/cpython/pull/155439) landed on `main`. It has not been
backported to 3.15 yet, and there is nothing to backport further: `TaskGroup.cancel()` does
not exist before 3.15, and neither does this bug.

Until the backport ships, the exposure is narrow but sharp: on 3.15, any cancellation that
arrives from outside after `tg.cancel()` and before the children are done is lost. If your
children have slow `finally` blocks and you bound the group with an external timeout, the
pre-3.15 sentinel-exception pattern still behaves correctly.

## A three-state fact read through a two-state test

`uncancel() == 0` does not ask "was this cancellation mine". It asks "is the count clear
now", and the code read a non-zero answer as *no information* when it was the whole signal.
Nobody asked, I asked, somebody else asked too — three states, and a test that can only
tell two of them apart.

And the symptom matches. Nothing raised, nothing logged, no warning. The timeout you wrote
to bound the work is still there, still entered, still expiring on time, and doing nothing
at all.
