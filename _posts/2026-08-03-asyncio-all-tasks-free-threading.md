---
layout: post
title: When asyncio.all_tasks() started forgetting tasks under free-threading
subtitle: How a task hid from the function whose only job is to list it
tags: [cpython, asyncio, free-threading, concurrency]
---

The job of `asyncio.all_tasks()` is to return all active tasks in the event loop. In builds with the GIL disabled, it could return fewer tasks than there actually are. But how is this possible? Let's take a look.

## The function that's supposed to see everything

... 

## Two new things, and a bug that only lives where they meet

...


## Finding it

...


## What was actually going on

...

## The fix

...

## The takeaway

...