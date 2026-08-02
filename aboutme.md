---
layout: page
title: About
subtitle: CPython, asyncio, and chasing runtime performance
---

I'm Timofei Ivankov, a Python backend engineer who tends to end up where systems hit their limits — making ML inference faster, keeping async services stable under load, and reading the interpreter's source instead of guessing how it behaves.

I contribute to **CPython**, mostly around **asyncio** and the **free-threaded build** — fixing data races, resource leaks, and correctness bugs, some of which landed in 3.14 and 3.15. You can see the work [here](https://github.com/python/cpython/pulls?q=is%3Apr+author%3Adeadlovelll).

This blog is where I write up the bugs I chase: what the symptom looked like, how I tracked it down, what was actually going on underneath, and what the fix was. Mostly CPython internals, concurrency, and runtime performance.

### What I work on

- **CPython contributions** — asyncio, free-threading, small JIT and runtime fixes.
- **Runtime performance** — I run measurement studies on how the different ways of speeding up Python (JIT, native extensions, alternative runtimes, build-level optimizations) actually compare when measured under one protocol.
- **Backend & ML infrastructure** — production services around LLMs and in-house models: serving, inference optimization, and keeping async systems stable under load.

I also write [in Russian on Habr](https://habr.com/ru/users/ivankov_timofei) about distributed systems, fault tolerance, and highload.

### Elsewhere

- GitHub: [deadlovelll](https://github.com/deadlovelll)
- LinkedIn: [timofei-ivankov](https://linkedin.com/in/timofei-ivankov)

If you want to talk about CPython internals, async, or runtime performance, reach out.
