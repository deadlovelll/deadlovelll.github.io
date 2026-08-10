---
layout: post
title: Two lines of Python that segfault the interpreter
subtitle: SET_ADD trusted the compiler, and PEP 749 quietly handed user code a way to break that trust
tags: [cpython, annotations, bytecode, compiler]
---

Here is a complete Python program that, until last week, crashed every CPython that
has annotations lazily evaluated — 3.14, 3.15, and 3.16:

```python
__conditional_annotations__ = 0
a: 1
```

No C extension, no `ctypes`, no threads. Two lines of plain Python, and the process
dies with a signal instead of a traceback:

```console
$ python3.14 crash.py
$ echo $?
138           # 128 + 10, SIGBUS on this macOS build
```

The reporter saw a plain segfault. I get a bus error. Which signal you land on depends
on what happens to be sitting in memory next to the object you just corrupted — the
interesting part is that there is no traceback at all.

`a: 1` is a nonsense annotation, but that is not the problem — annotations are not
evaluated eagerly anymore. The problem is the first line, which reassigns a name
most people have never heard of.

## The set nobody told you about

Python 3.14 added [PEP 649](https://peps.python.org/pep-0649/) and
[PEP 749](https://peps.python.org/pep-0749/): annotations are no longer evaluated at
definition time; they are moved into a lazily-called `__annotate__` function.

That created a bookkeeping problem. Libraries write this all the time:

```python
if TYPE_CHECKING:
    someattr: SpecialType
```

`__annotate__` runs later, long after the `if` has been decided, so it has no idea
whether that branch was taken. PEP 749 solves it with a side channel: every annotated
assignment at module level — and every conditional one in a class body — gets a unique
integer, and the enclosing body maintains a set of the ones it actually executed.

You can see the whole mechanism in the disassembly of a two-annotation module:

```python
if True:
    a: int
b: str
```

```console
$ python3.14 -m dis mod.py
   1           ...
               BUILD_SET                0
               STORE_NAME               0 (__conditional_annotations__)

   2           LOAD_NAME                0 (__conditional_annotations__)
               LOAD_SMALL_INT           0
               SET_ADD                  1
               POP_TOP

   3           LOAD_NAME                0 (__conditional_annotations__)
               LOAD_SMALL_INT           1
               SET_ADD                  1
               POP_TOP
```

and on the other side, inside `__annotate__`:

```console
  2           LOAD_SMALL_INT           0
              LOAD_GLOBAL              0 (__conditional_annotations__)
              CONTAINS_OP              0 (in)
              POP_JUMP_IF_FALSE       10 (to L2)
```

Note the opcodes: `STORE_NAME` and `LOAD_NAME`. This is not a hidden stack slot — it is an ordinary module global, 
sitting in `globals()` next to everything else you wrote, under a name anyone can assign to. That is the whole bug.

## Why SET_ADD never checked

Before 3.14, every `SET_ADD` the compiler emitted — in set comprehensions and in set
displays alike — took its operand straight from a `BUILD_SET` a few instructions
earlier. The set sits on the stack and never gets a name, so no Python code can reach
it, let alone rebind it. The compiler knows the operand is a set because the compiler
is what put it there. So the opcode skips the check.

```c
inst(SET_ADD, (set, unused[oparg-1], v -- set, unused[oparg-1])) {
    int err = _PySet_AddTakeRef((PySetObject *)PyStackRef_AsPyObjectBorrow(set),
                                PyStackRef_AsPyObjectSteal(v));
    ERROR_IF(err);
}
```

That cast to `PySetObject *` is unchecked, and `_PySet_AddTakeRef` goes straight for
the internals — `so->mask`, `so->table`, `so->used`. Those fields sit well past the end
of a small `int`, so the interpreter reads garbage as a hash table pointer and
dereferences it. Classic type confusion.

PEP 749 gave the opcode a second caller, and that one loads its operand by name. 
Nobody removed a check; the check was never needed until the ground moved.

The issue was reported by [Lydxn](https://github.com/python/cpython/issues/154902) on
July 30. I picked it up and ended up writing two different fixes for it, which turned
out to be the actually interesting part.

## The fix that couldn't be backported

On `main` the right answer is not to make a generic opcode defensive — it is to stop
using a generic opcode for a specific job. Conditional annotations got their own
intrinsic:

```c
static PyObject *
add_conditional_annotation(PyThreadState* tstate, PyObject *conditional_annotations,
                           PyObject *index)
{
    if (!PySet_CheckExact(conditional_annotations)) {
        _PyErr_Format(tstate, PyExc_TypeError,
                      "__conditional_annotations__ must be a set, not %T",
                      conditional_annotations);
        return NULL;
    }
    if (PySet_Add(conditional_annotations, index) < 0) {
        return NULL;
    }
    Py_RETURN_NONE;
}
```

and the code generator stopped emitting `SET_ADD` for annotations entirely:

```c
ADDOP_I(c, loc, CALL_INTRINSIC_2, INTRINSIC_ADD_CONDITIONAL_ANNOTATION);
```

Set comprehensions and displays keep their fast, unchecked opcode. Annotations get an operation
that owns its own invariant and can name the variable in the error message, because
that intrinsic exists for nothing else.

The catch: this adds an intrinsic, changes emitted bytecode, and bumps the pyc magic
number from 3704 to 3705. That is a non-starter on a released branch —
[#155026](https://github.com/python/cpython/pull/155026) could only go to `main`.

## The fix for 3.14 and 3.15

3.14 and 3.15 are already out. No new opcodes, no new intrinsics, no magic number
bump. So those branches get the boring fix instead: teach `SET_ADD` to check.

```c
PyObject *set_o = PyStackRef_AsPyObjectBorrow(set);
// gh-154902: user code can rebind __conditional_annotations__
if (!PySet_CheckExact(set_o)) {
    _PyErr_Format(tstate, PyExc_TypeError,
                  "'%T' object is not a set", set_o);
    PyStackRef_CLOSE(v);
    ERROR_IF(true);
}
int err = _PySet_AddTakeRef((PySetObject *)set_o,
                            PyStackRef_AsPyObjectSteal(v));
```

Note that the message stays generic: `SET_ADD` is a shared opcode, so it cannot blame a
variable the user may have never written. Naming `__conditional_annotations__` is a
luxury only the dedicated intrinsic can afford.

Both landed on August 4–5: [#155071](https://github.com/python/cpython/pull/155071) for
3.15 and [#155072](https://github.com/python/cpython/pull/155072) for 3.14.

## The takeaway

Every "the compiler guarantees this" comment is a contract between two pieces of code
that may not stay neighbors. `SET_ADD` was written for set comprehensions and
displays, and was correct for as long as those were its only callers. Years later a new feature needed to
add integers to a set, `SET_ADD` was sitting right there, and the invariant that made
it safe — *the operand is unreachable from Python* — was not written down anywhere the
new caller would trip over it.

The interesting part of this fix is not the guard. It is that a released branch and a
development branch deserved genuinely different answers to the same crash: one
defensive, one structural. Backport policy is not an obstacle to routing around — it
is a constraint that tells you which of the two fixes you are allowed to be proud of.
