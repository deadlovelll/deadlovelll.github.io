---
layout: post
title: The task that failed to start and left asyncio stuck
subtitle: Failed context entry during eager start left the child recorded as the current task
tags: [cpython, asyncio, contextvars, debugging]
---

Here is a program that catches the error from a failed task constructor, then
hangs at its next `await`:

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
hangs with the following traceback:

```text
Exception in callback <_asyncio.TaskStepMethWrapper object at 0x1059db9d0>()
handle: <Handle <_asyncio.TaskStepMethWrapper object at 0x1059db9d0>()>
Traceback (most recent call last):
  File "/opt/homebrew/Cellar/python@3.14/3.14.6/Frameworks/Python.framework/Versions/3.14/lib/python3.14/asyncio/events.py", line 94, in _run
    self._context.run(self._callback, *self._args)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
RuntimeError: Invalid attempt to leave task <Task pending name='Task-1' coro=<main() running at /tmp/eager-start-article-20261116/repro.py:18> cb=[_run_until_complete_cb() at /opt/homebrew/Cellar/python@3.14/3.14.6/Frameworks/Python.framework/Versions/3.14/lib/python3.14/asyncio/base_events.py:181]> while task <Task pending name='Task-2' coro=<child() done, defined at /tmp/eager-start-article-20261116/repro.py:4>> is entered.
```

The `RuntimeError` we caught came from trying to enter the same `Context`
twice. `ctx.run` has already entered it, and `eager_start=True` makes the task
try to enter it again before `ctx.run` can return. Python
[forbids that][context-docs], so the constructor is right to raise.

The second error occurs when leaving `main`, but the runtime has `child`
recorded as current. The child's constructor never returned, and its
coroutine has not even reached `pass`.

## Eager start changes the current task before entering the context

An eager task starts running its coroutine inside the constructor, until it
finishes or actually suspends. That means the parent can still be inside
`Task(...)` while the child is executing, and `asyncio.current_task()` has
to report the child during that interval.

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

`_PyContext_Enter` fails. The function decrefs the parent and returns, leaving
the child current and registered. Decrementing `prevtask`'s reference count
does nothing to the field where the child was installed.

The code to restore the parent was further down, after the first coroutine
step. The context failure returned before any of it could run.

The constructor has failed, but the state change it made is still visible
to the code that called it. Save `parent = asyncio.current_task()` before
the `try` and put this in the handler:

```python
except RuntimeError:
    print(asyncio.current_task() is parent, flush=True)
```

It prints `False` even though the parent is executing the print. The child
is still recorded as current. The enclosing `ctx.run` has unwound its own
context change correctly, but asyncio has not restored the current task.

## The next await fails in `leave_task`

The parent can execute its exception handler because it has not suspended
yet. The C call raised into the Python frame that was already running, and
the `except` block is just more code in that frame.

Then `await asyncio.sleep(0)` yields. Asyncio finishes the parent's step and
calls `leave_task`, which checks this:

```c
if (ts->asyncio_running_task != task) {
    // raise RuntimeError: Invalid attempt to leave task ...
}
```

With `task` pointing to the parent and `asyncio_running_task` to the child,
the guard raises, and the event loop logs the callback error. On the next
attempt to resume the parent, `enter_task` finds that a task is already
current and refuses to enter another one:

```text
Exception in callback <_asyncio.TaskStepMethWrapper object at 0x1059dbbb0>()
handle: <Handle <_asyncio.TaskStepMethWrapper object at 0x1059dbbb0>()>
Traceback (most recent call last):
  File "/opt/homebrew/Cellar/python@3.14/3.14.6/Frameworks/Python.framework/Versions/3.14/lib/python3.14/asyncio/events.py", line 94, in _run
    self._context.run(self._callback, *self._args)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
RuntimeError: Cannot enter into task <Task pending name='Task-1' coro=<main() running at /tmp/eager-start-article-20261116/repro.py:18> cb=[_run_until_complete_cb() at /opt/homebrew/Cellar/python@3.14/3.14.6/Frameworks/Python.framework/Versions/3.14/lib/python3.14/asyncio/base_events.py:181]> while another task <Task pending name='Task-2' coro=<child() done, defined at /tmp/eager-start-article-20261116/repro.py:4>> is being executed.
```

The loop still dispatches ordinary callbacks, but attempts to resume `main`
fail because `asyncio_running_task` still points to the child. This leaves
`asyncio.run` waiting for `main` to finish. A timeout implemented as another
coroutine also depends on task stepping, so the reproduction needs an
external watchdog.

Catching the constructor's exception does not restore the current task,
and the exception handler has no reason to expect that it has changed.

## Restore the parent on the way out

My [original report][issue] also had a shorter way to reach this path:
`context=2.14`. Checking the argument's type catches that at construction.
The example above passes a `Context` instance, though, and still fails to enter
it. Validation alone leaves the bug in place.

The error path has to undo the setup it has already performed:

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

Swap the parent back in, release the displaced reference to the child,
unregister the child, then return the original error. The log flag also
suppresses the pending-task warning for the task that could not be
constructed. The caller still gets `RuntimeError`, with its own task
current again.

The Python task implementation already had `finally` blocks around the
current-task swap. The missing restoration was in the C implementation.
The Python-side changes add the context type check and suppress the warning
on failed eager construction.

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
