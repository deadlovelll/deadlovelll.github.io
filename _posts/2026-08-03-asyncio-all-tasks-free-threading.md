---
layout: post
title: When asyncio.all_tasks() started forgetting tasks under free-threading
subtitle: Under free-threading, a task could be running and missing from all_tasks() at the same time
tags: [cpython, asyncio, free-threading, concurrency]
---

The job of `asyncio.all_tasks()` is to return all active tasks in the event loop. 
In builds with the GIL disabled, called from a thread other than the one running
the loop, it could return fewer tasks than there actually are.

## What the list is used for

It is used for: graceful shutdown, waiting for in-flight work to finish, 
monitoring (counting running tasks), and test suites (checking for leaks).

If a task is not in this list, a shutdown routine driving the loop from another thread can silently drop it.

## Two new things, and a bug that only lives where they meet

An **eager task** runs synchronously until the first real suspension point, 
and if the coroutine completes without being suspended, it never gets scheduled on the event loop at all.

The free-threading build eliminates the GIL, so a task created by the loop's thread
can be examined by another thread in parallel - and to hand it out safely,
the runtime has to incref an object it doesn't own.

None of this is a problem in itself. With regular tasks, everything works fine.
With **eager tasks** under GIL, the tasks behave as intended. The bug needed both -
and it was not a race: with both in place, the task went missing every single time.

## Finding it

I encountered this issue when running asynchronous code in a free-threaded build. 
The `asyncio.all_tasks()` function returns an incomplete number of active tasks. 
My first guess was that eager tasks might disappear: a task that completes without ever being suspended is 
unregistered again, so never appearing in `asyncio.all_tasks()` is correct behavior, not a bug. 
But that wasn't the case: the task was still running.
The other obvious suspect, a registry being modified from one thread while another is iterating it, 
didn't hold water either — the registry was fine, and the task was in it. Whatever was dropping it happened on the way out.
So I stopped guessing and went into `task_init`.

## The bug, and the fix

In the free-threaded build, `asyncio.all_tasks()` collects tasks from a linked
list of borrowed references, and `_Py_TryIncref` from a non-owning thread only
works if the task has its maybe-weakref bit set. The
old code set it *after* the eager-start logic:

```c
if (eager_start) {
    // ... check is_running ...
    if (is_loop_running) {
        if (task_eager_start(ts, state, self)) {
            return -1;
        }
        return 0;          // <-- early return, skips the code below
    }
}
task_call_step_soon(state, self, NULL);
#ifdef Py_GIL_DISABLED
    _PyObject_SetMaybeWeakref((PyObject *)self);   // <-- never reached
#endif                                             //     for eager tasks
register_task(ts, self);
```

An eager task started on a running loop took that `return 0` and left the
function without ever having its maybe-weakref bit set - so another thread
calling `all_tasks()` couldn't safely see it.

The fix is one move: set the bit up front, before any eager-start path can
return early.

```c
#ifdef Py_GIL_DISABLED
    _PyObject_SetMaybeWeakref((PyObject *)self);   // <-- now unconditional
#endif
if (eager_start) {
    // ... unchanged ...
}
```

The fix landed in 3.16 and was also backported to 3.15 and 3.14. The PR is
[python/cpython#152022](https://github.com/python/cpython/pull/152022).

## The GIL build cannot catch this

Free-threading removes the GIL and adds per-object bookkeeping that has no
counterpart in the default build. `_PyObject_SetMaybeWeakref` compiles to nothing under the GIL,
so a missing call is invisible to every test that runs there and nothing catches it at runtime
either: `_Py_TryIncref` returns 0 and the task is skipped without an error.
One path through `task_init` had the call, the other returned before reaching it.
Every early return in a function like this is a chance to skip setup that only matters
to other threads, and this one will not be the last.