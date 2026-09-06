---
layout: post
title: Reading __dict__ once permanently deoptimizes attribute access
subtitle: Since 3.11 attribute access has not been a dict lookup, and reading __dict__ once takes the specialized path away for good
tags: [cpython, performance, free-threading, bytecode]
---

Start with the oldest performance tip in Python. Pull the attribute out of the loop:

```python
class Attr:
    def process(self):
        self.value = 0
        for i in range(1_000_000):
            self.value += i

class Hoisted:
    def process(self):
        value = 0
        for i in range(1_000_000):
            value += i
        self.value = value
```

Measured with `pyperf` on an M3, CPython 3.14.6: 33.0 ms against 24.8 ms, a ratio of
1.33 +- 0.04, so the tip works. The explanation attached to it, that attribute access goes
through the instance dict and hashing a string costs something, has been wrong since 3.11,
and it is still the explanation you will find everywhere.

Taking the fast path away takes one line, leaves the loop byte-for-byte identical, and
costs more than the hoisting saves.

## The loop stops using LOAD_ATTR after a few runs

Here is the cold disassembly of `Attr.process`, which does look like a lookup:

```console
  3           RESUME                   0

  4           LOAD_SMALL_INT           0
              LOAD_FAST_BORROW         0 (self)
              STORE_ATTR               0 (value)

  5           LOAD_GLOBAL              3 (range + NULL)
              LOAD_CONST               1 (1000000)
              CALL                     1
              GET_ITER
      L1:     FOR_ITER                28 (to L2)
              STORE_FAST               1 (i)

  6           LOAD_FAST_BORROW         0 (self)
              COPY                     1
              LOAD_ATTR                0 (value)
              LOAD_FAST_BORROW         1 (i)
              BINARY_OP               13 (+=)
              SWAP                     2
              STORE_ATTR               0 (value)
              JUMP_BACKWARD           30 (to L1)

  5   L2:     END_FOR
              POP_ITER
              LOAD_CONST               2 (None)
              RETURN_VALUE
```

The adaptive interpreter rewrites that after a few runs, so ask `dis` what it settled on
once the function has been called a couple of times:

```python
a = Attr()
a.process()
dis.dis(Attr.process, adaptive=True)
```

```console
  3           RESUME_CHECK             0

  4           LOAD_SMALL_INT           0
              LOAD_FAST_BORROW         0 (self)
              STORE_ATTR               0 (value)

  5           LOAD_GLOBAL              3 (range + NULL)
              LOAD_CONST_MORTAL        1 (1000000)
              CALL                     1
              GET_ITER
      L1:     FOR_ITER_RANGE          28 (to L2)
              STORE_FAST               1 (i)

  6           LOAD_FAST_BORROW         0 (self)
              COPY                     1
              LOAD_ATTR_INSTANCE_VALUE 0 (value)
              LOAD_FAST_BORROW         1 (i)
              BINARY_OP_ADD_INT       13 (+=)
              SWAP                     2
              STORE_ATTR_INSTANCE_VALUE 0 (value)
              JUMP_BACKWARD_NO_JIT    30 (to L1)

  5   L2:     END_FOR
              POP_ITER
              LOAD_CONST_IMMORTAL      2 (None)
              RETURN_VALUE
```

`LOAD_ATTR` is gone, and what replaced it never touches a hash table:

```c
op(_LOAD_ATTR_INSTANCE_VALUE, (offset/1, owner -- attr)) {
    PyObject *owner_o = PyStackRef_AsPyObjectBorrow(owner);
    PyObject **value_ptr = (PyObject**)(((char *)owner_o) + offset);
    PyObject *attr_o = FT_ATOMIC_LOAD_PTR_ACQUIRE(*value_ptr);
    DEOPT_IF(attr_o == NULL);
    ...
```

A fixed byte offset into the object, baked into the instruction's inline cache when it
specialized. Since 3.13's `Py_TPFLAGS_INLINE_VALUES`, an ordinary instance keeps its
attributes in an inline values array, and the specialized opcode reads straight out of it
after checking the type's version tag and that the values are still there, so the
`__dict__` everyone is picturing does not exist yet.

To find where the 8 nanoseconds go, I added a control that keeps the loop and the iteration
count but has no attribute in it.

```python
class Floor:
    def process(self):
        value = 0
        for i in range(1_000_000):
            value += i
        return value
```

| variant | GIL | free-threaded |
|---|---|---|
| `attr` | 33.0 ms +- 0.5 | 34.0 ms +- 0.3 |
| `hoisted` | 24.8 ms +- 0.7 | 23.9 ms +- 0.1 |
| `floor` | 24.3 ms +- 0.8 | 23.8 ms +- 0.1 |

`hoisted / floor` is 1.02 +- 0.04 on the GIL build and 1.00 +- 0.01 without it. Take the
attribute out and what is left is the bare loop: that ~24 ms is mostly `FOR_ITER_RANGE` and
`BINARY_OP_ADD_INT`. The 1.33x is 8 ms of attribute round-trips on top of 24 ms of loop
that hoisting does not touch.

## Taking the fast path away

`Materialized` has the same `process` body as `Attr`. The only difference is one line
somewhere else entirely:

```python
o = Materialized()
o.__dict__
```

| variant | GIL | free-threaded |
|---|---|---|
| `attr` | 33.0 ms +- 0.5 | 34.0 ms +- 0.3 |
| `materialized` | 50.6 ms +- 1.2 | 59.4 ms +- 0.5 |
| ratio | **1.54 +- 0.04** | **1.75 +- 0.02** |

Reading `__dict__` materializes it, the new dict wraps the same values array, and the
object is off the specialized path for the rest of its life. The loop that took 33 ms
takes 51 ms. That is an 18 ms penalty, where hoisting saved 8 ms.

Other things materialize it too:

```
o.__dict__
vars(o)   
'x' in o.__dict__
copy.copy(o)
```

`copy.copy` is the one that matters. Nobody reads a shallow copy as a performance decision,
and it permanently makes every subsequent attribute access on that object about three times
dearer.

## LOAD_ATTR_WITH_HINT declines split tables

`dis` says the loop is back to plain `LOAD_ATTR` and `STORE_ATTR`, fully generic. A slower
specialization exists for instances that do have a dict:

```c
try_instance:
    if (specialize_dict_access(owner, instr, type, kind, name, tp_version,
                               LOAD_ATTR, LOAD_ATTR_INSTANCE_VALUE, LOAD_ATTR_WITH_HINT))
```

`LOAD_ATTR_WITH_HINT` caches the index of the key in the dict, so a lookup on a known
object shape stays cheap without inline values. It is never selected here, and on the store
side `specialize_dict_access_hint` says why in its first check:

```c
    if (_PyDict_HasSplitTable(dict)) {
        SPECIALIZATION_FAIL(base_op, SPEC_FAIL_ATTR_SPLIT_DICT);
        return 0;
    }
```

Materializing the `__dict__` of an object that had inline values produces a dict with a
*split* table - keys shared with the type, values in the instance - and the hint
specialization declines split tables outright. So the object ends up with no specialization
at all, and the two opcodes get there by different routes: `STORE_ATTR` is the one that
reaches the hint path and is refused for the split table, while `LOAD_ATTR` never gets that
far - it takes an earlier branch that fails as soon as it sees a non-NULL dict.

I assumed one materialized instance would poison the shared code object for every other
instance of the class, since every instance runs through the same instruction with the same
inline cache. The store deoptimizes while the materialized instance is running, but clean
instances re-specialize it right back, and a clean instance running through a "poisoned"
code object measured 1.02x, which is to say nothing.

`__slots__` was the other thing I had wrong. `slots / attr` came out 0.98 +- 0.03 with the
GIL and 1.00 +- 0.01 without, so a slotted instance and an ordinary one read attributes at
exactly the same speed. What `__slots__` buys is the absent
`__dict__`: there is nothing to materialize, so nothing can take the fast path away.

## Without the GIL, both effects grow

Every benchmark above, rerun on the same machine on a build configured with `--disable-gil`:

```console
+----------------+---------+-----------------------+
| Benchmark      | gil     | ft                    |
+================+=========+=======================+
| attr           | 33.0 ms | 34.0 ms: 1.03x slower |
+----------------+---------+-----------------------+
| hoisted        | 24.8 ms | 23.9 ms: 1.04x faster |
+----------------+---------+-----------------------+
| floor          | 24.3 ms | 23.8 ms: 1.02x faster |
+----------------+---------+-----------------------+
| materialized   | 50.6 ms | 59.4 ms: 1.17x slower |
+----------------+---------+-----------------------+
| slots          | 32.3 ms | 33.9 ms: 1.05x slower |
+----------------+---------+-----------------------+
| Geometric mean | (ref)   | 1.04x slower          |
+----------------+---------+-----------------------+
```

The loop itself is *faster* without the GIL, and what costs more is attribute access: the
specialized round-trip goes from 8.2 to 10.1 ns, and the materialization penalty, on the
generic path, from 17.6 to 25.4 ns.

That `FT_ATOMIC_LOAD_PTR_ACQUIRE` above is an acquire-ordered load where the GIL build has
a plain one, and then there is a step the GIL build does not have at all:

```c
#ifdef Py_GIL_DISABLED
    int increfed = _Py_TryIncrefCompareStackRef(value_ptr, attr_o, &attr);
    if (!increfed) {
        DEOPT_IF(true);
    }
#else
    attr = PyStackRef_FromPyObjectNew(attr_o);
#endif
```

Reading an attribute you do not own means taking a reference another thread could be
dropping underneath you, so the incref is a compare-and-incref that is allowed to *fail*.
It cannot block, so it bails to the unspecialized path instead. Anyone who read
[the `all_tasks()` post]({% post_url 2026-08-03-asyncio-all-tasks-free-threading %}) has met
this primitive's sibling: `_Py_TryIncref` is what skipped tasks whose maybe-weakref bit was
never set. A failed incref there meant a running task never showed up in the list, and here
it means one attribute read goes through the unspecialized path.

On the write side, `STORE_ATTR_INSTANCE_VALUE` takes a lock on the object:

```c
macro(STORE_ATTR_INSTANCE_VALUE) =
    unused/1 +
    _GUARD_TYPE_VERSION_AND_LOCK +
    _GUARD_DORV_NO_DICT +
    _STORE_ATTR_INSTANCE_VALUE;
```

```c
op(_GUARD_TYPE_VERSION_AND_LOCK, (type_version/2, owner -- owner)) {
    PyObject *owner_o = PyStackRef_AsPyObjectBorrow(owner);
    assert(type_version != 0);
    EXIT_IF(!LOCK_OBJECT(owner_o));
    ...
```

```c
#ifdef Py_GIL_DISABLED
#  define LOCK_OBJECT(op) PyMutex_LockFast(&(_PyObject_CAST(op))->ob_mutex)
#else
#  define LOCK_OBJECT(op) (1)
#endif
```

Under the GIL that expands to the literal `1` and the optimizer deletes it. Without the GIL
it is an uncontended mutex acquire, cheap, paid a million times. The asymmetry is deliberate:
the read does not take the lock and the write does. A read that cannot safely take a
reference detects that and falls back through `DEOPT_IF`. A write that tears an inline
values array cannot be undone, so it pays for the mutex.

## vars() and copy.copy() in a hot path

Eight to ten nanoseconds per round-trip only shows up with an attribute in the innermost loop of
something that runs millions of times and does nearly nothing else per iteration, which
describes this benchmark and very little production code. Hoisting a lookup out of a loop
that runs forty times buys nanoseconds and costs you a worse-reading diff.

The materialization half matters more, because nothing about it looks like a performance
decision. Ordinary code with nothing to do with the hot loop triggers it, the function that
got slower does not change, and it costs twice what the famous tip saves. If something in a
hot path calls `vars()` or `copy.copy()` on the objects it is iterating over,
that is worth knowing, and `__slots__` is the fix, since a class with no `__dict__` has no
slow path to fall into.

## The explanation stopped being true in 3.11

Before 3.11 the explanation named the right mechanism, because attribute access really did
go through the instance dict. 3.11 replaced that with a version-guarded read into a
values array, and the free-threaded build adds an atomic load on top of it plus a mutex on
the write. Through all of that the benchmark kept agreeing with the conclusion, so nothing
forced anyone to recheck the reason.

Once something reads `__dict__`, the specialization is gone and the tip is worth three times
what it was. The advice never mentions that case, because at the time there was no other
case to distinguish it from.
