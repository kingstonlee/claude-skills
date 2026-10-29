---
name: asyncio-blocking-diagnosis
description: Use when a Python asyncio/uvicorn service shows jobs running sequentially despite being submitted concurrently, requests stalling for the duration of another request, or S3/external calls appearing to serialize across coroutines
---

# asyncio Blocking Diagnosis

## Overview

When a sync function runs inside an `async def` without `asyncio.to_thread`, it seizes the event loop for its full duration. Every other coroutine freezes until it returns. The diagnostic signal is that concurrent job 2's stall period equals concurrent job 1's blocking call duration — this lets you identify the exact offending call from logs alone.

## When to Use

- Two concurrent requests complete sequentially (job 2 finishes right after job 1)
- Request latency for job N ≈ latency for job 1 + job N's own work
- S3 downloads, file reads, or HTTP calls are sync libraries used inside `async def`
- Service is Python + uvicorn/FastAPI/aiohttp (single event loop)

## Diagnostic Pattern

### Step 1: Read log timestamps across concurrent jobs

Look for two jobs submitted close together. Find the gap in job 2's log activity:

```
# Job 1 logs — timestamps spaced normally
10:00:00 [job_1] started
10:00:01 [job_1] downloaded input file
10:05:32 [job_1] processing complete        ← 5m31s blocking call
10:05:33 [job_1] uploaded result

# Job 2 logs — big silence, then suddenly active
10:00:05 [job_2] started
10:05:33 [job_2] downloaded input file       ← 5m28s silence = job_1's blocking duration
10:07:40 [job_2] processing complete
```

**The correlation:** job 2's silence (5m28s) ≈ job 1's blocking call (5m31s). This tells you exactly when the loop was held and approximately which call caused it.

### Step 2: Identify the blocking call

Look at what job 1 was doing during that exact window. Common culprits in Python services:
- S3 client calls (`boto3` is sync — `upload`, `download`, `get_object`)
- PDF/file processing libraries (`pdfminer`, `pikepdf`, `reportlab`)
- HTTP clients (`requests.get` instead of `aiohttp`)
- Any third-party SDK that isn't explicitly async

### Step 3: Confirm with placeholder state

If the service writes a placeholder to S3 before processing, check its byte count at the time of the stall. A placeholder that stays at its initial size (e.g. 122 bytes) for the entire stall duration while the other job's process runs confirms the event loop was never free to update it.

## Fix

Wrap every sync blocking call with `asyncio.to_thread`:

```python
# Before (blocks the event loop)
result = s3_client.download_file(bucket, key, dest)

# After (runs in thread pool, loop stays free)
result = await asyncio.to_thread(
    s3_client.download_file,
    bucket, key, dest,
)
```

`asyncio.to_thread` requires Python 3.9+. For 3.8 use `loop.run_in_executor(None, fn, *args)`.

**Audit all call sites:** Search for every call to the sync client in the file — there are often multiple paths (happy path, error path, exception handler) each needing the wrap.

## Quick Reference

| Signal | Meaning |
|--------|---------|
| Job 2 silence ≈ Job 1 blocking call | Sync call holding event loop |
| S3 placeholder stays small for N seconds | Loop not free during that window |
| Jobs complete in series, not parallel | Single-threaded blocking somewhere |
| `PYTHONASYNCIODEBUG=1` warnings | Coroutine blocking loop >100ms |

## Common Mistakes

- **Wrapping only the happy path** — error paths and exception handlers also call sync clients and need `asyncio.to_thread`
- **Assuming `async def` means non-blocking** — the function is async but the calls inside may not be
- **Looking at CPU/memory first** — the symptom is serialization, not resource exhaustion; timestamp correlation is faster than profiling
