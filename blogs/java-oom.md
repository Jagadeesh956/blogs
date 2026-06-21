---
title: "Memory Management in JVM"
date: 2026-06-21
tags: [Java, JVM, SRE]
summary: "How Java manages memory under the hood and why OOM errors happen — a practical guide for developers and SREs."
---

## Overview

Java's memory model is one of the most important things to understand as a backend engineer or SRE. This post breaks down the JVM heap, garbage collection, and how to diagnose OutOfMemoryError in production.

## The JVM Memory Model

The JVM divides memory into several regions:

| Region | Purpose |
|---|---|
| Heap | Objects allocated by your application |
| Metaspace | Class metadata (replaced PermGen in Java 8+) |
| Stack | Per-thread method frames and local variables |
| Code Cache | JIT-compiled native code |

## Why OOM Happens

`OutOfMemoryError` is thrown when the JVM cannot allocate an object because:

1. **Heap exhaustion** — too many live objects, GC can't free enough
2. **Memory leaks** — objects held in static collections, caches, listeners
3. **Metaspace overflow** — too many dynamically loaded classes

```java
// Classic memory leak — list grows forever
static List<byte[]> leak = new ArrayList<>();

void allocate() {
    leak.add(new byte[1024 * 1024]); // 1 MB per call
}
```

## How to Diagnose

```bash
# Enable heap dump on OOM (add to JVM flags)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# Analyze with Eclipse MAT or VisualVM
```

> **Tip:** Always run with `-Xmx` set explicitly in production. Never rely on the default heap size.

## Summary

Understanding JVM memory lets you write applications that are both faster and more predictable under load.
