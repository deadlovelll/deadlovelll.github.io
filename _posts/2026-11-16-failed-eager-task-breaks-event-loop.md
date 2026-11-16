---
layout: post
title: The task that failed to start and left asyncio stuck
subtitle: Failed context entry during eager start left the child recorded as the current task
tags: [cpython, asyncio, contextvars, debugging]
---

I found this bug while fuzzing parts of asyncio. A failed task constructor
could leave the event loop stuck, even if the caller caught the exception.
This program catches the error, then hangs at its next `await`:

```python
import asyncio
import contextvars

async def child():
    pass

async def main():
    loop = asyncio.get_running_loop()
    ctx = contextvars.copy_context()
    coro = child()
    try:
        ctx.run(asyncio.Task, coro, loop=loop,
                context=ctx, eager_start=True)
    except RuntimeError:
        pass
    finally:
        coro.close()
    await asyncio.sleep(0)
    print("done")

asyncio.run(main())
```

Running this with CPython 3.14.6 gets as far as `asyncio.sleep(0)`, then
hangs with the following traceback (file paths shortened here and below):

```text
Exception in callback <_asyncio.TaskStepMethWrapper object at 0x1059db9d0>()
handle: <Handle <_asyncio.TaskStepMethWrapper object at 0x1059db9d0>()>
Traceback (most recent call last):
  File ".../asyncio/events.py", line 94, in _run
    self._context.run(self._callback, *self._args)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
RuntimeError: Invalid attempt to leave task <Task pending name='Task-1' coro=<main() running at .../repro.py:18> cb=[_run_until_complete_cb() at .../asyncio/base_events.py:181]> while task <Task pending name='Task-2' coro=<child() done, defined at .../repro.py:4>> is entered.
```

Trying to enter the same `Context` twice explains the `RuntimeError` we
caught: `ctx.run` has already entered it, and `eager_start=True` makes the
task try to enter it again before `ctx.run` can return. Python
[forbids that][context-docs], so the constructor is right to raise.

But `main` is now leaving while the runtime still has `child` as current,
even though the child's constructor never returned and its coroutine has
not reached `pass`.

## Eager start changes the current task before entering the context

An eager task runs its coroutine inside the constructor until it finishes
or actually suspends, so the parent can still be inside `Task(...)` while
the child is executing. During that interval, `asyncio.current_task()` has
to report the child.

In the C implementation, `task_eager_start` does that with
[`swap_current_task`][before-source]: it saves the parent in `prevtask` and
puts the child in the thread state's `asyncio_running_task` field. Then it
registers the child among the tasks asyncio tracks. Context entry comes
after both operations.

This was the setup and error path before the fix:

```c
PyObject *prevtask = swap_current_task(ts, task->task_loop, (PyObject *)task);
if (prevtask == NULL) {
    return -1;
}
register_task(ts, task);

if (_PyContext_Enter(&ts->base, task->task_context) == -1) {
    Py_DECREF(prevtask);
    return -1;
}
```

When `_PyContext_Enter` fails, cleanup consists of a decref of `prevtask`,
and the function returns with the child still current and registered.
Restoring the parent happens further down, after the first coroutine step,
which this error path never reaches.

Save `parent = asyncio.current_task()` before the `try` and put this in the
handler:

```python
except RuntimeError:
    print(asyncio.current_task() is parent, flush=True)
```

It prints `False`, from inside the parent. By then `ctx.run` has unwound its
own context change, but `asyncio_running_task` still points to the child.

## The next await fails in `leave_task`

So far the parent has not suspended: the C call raised into the Python
frame that was already running, and executing the `except` block just
continues in that frame.

Then `await asyncio.sleep(0)` yields. Asyncio finishes the parent's step and
calls `leave_task`, which checks this:

```c
if (ts->asyncio_running_task != task) {
    // raise RuntimeError: Invalid attempt to leave task ...
}
```

After logging this error, the loop tries to resume the parent, but
`enter_task` also fails because the child is still current:

```text
Exception in callback <_asyncio.TaskStepMethWrapper object at 0x1059dbbb0>()
handle: <Handle <_asyncio.TaskStepMethWrapper object at 0x1059dbbb0>()>
Traceback (most recent call last):
  File ".../asyncio/events.py", line 94, in _run
    self._context.run(self._callback, *self._args)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
RuntimeError: Cannot enter into task <Task pending name='Task-1' coro=<main() running at .../repro.py:18> cb=[_run_until_complete_cb() at .../asyncio/base_events.py:181]> while another task <Task pending name='Task-2' coro=<child() done, defined at .../repro.py:4>> is being executed.
```

Ordinary callbacks still run, but every attempt to resume `main` fails on
the stale child in `asyncio_running_task`, leaving `asyncio.run` waiting
for a task that cannot finish. Even a timeout implemented as another
coroutine needs task stepping to work, so this reproduction needs an
external watchdog.

## Restore the parent on the way out

In my [original report][issue], I also used `context=2.14` to reach this
path. A type check catches that at construction, but the example above
passes a `Context` instance and fails because it is already entered, so
the error path still has to undo the current-task swap and registration:

```c
if (_PyContext_Enter(&ts->base, task->task_context) == -1) {
    task->task_log_destroy_pending = 0;
    PyObject *curtask = swap_current_task(ts, task->task_loop, prevtask);
    Py_XDECREF(curtask);
    unregister_task(task);
    Py_DECREF(prevtask);
    return -1;
}
```

Setting `task_log_destroy_pending` to zero suppresses the pending-task warning for
the task that could not be constructed. Now the caller gets `RuntimeError`
with its own task current again.

On the Python side, `finally` blocks already restored the current task,
so the rollback fix belongs in the C implementation. Python-side changes
add the context type check and suppress the warning on failed eager
construction.

[#157302][pr] landed on 19 September and was backported to
[3.13][backport-313], [3.14][backport-314] and [3.15][backport-315]. The
[merged C code][fixed-source] has both the argument check and the rollback.

[issue]: https://github.com/python/cpython/issues/157301
[pr]: https://github.com/python/cpython/pull/157302
[context-docs]: https://docs.python.org/3/library/contextvars.html#contextvars.Context
[before-source]: https://github.com/python/cpython/blob/1d0f1ad0aa7a933e8d03f855d9b5eed7e00e775d/Modules/_asynciomodule.c
[fixed-source]: https://github.com/python/cpython/blob/5b343636bae9b89cb5632ca07b01e1570e24653d/Modules/_asynciomodule.c
[backport-313]: https://github.com/python/cpython/pull/157845
[backport-314]: https://github.com/python/cpython/pull/157844
[backport-315]: https://github.com/python/cpython/pull/157785
