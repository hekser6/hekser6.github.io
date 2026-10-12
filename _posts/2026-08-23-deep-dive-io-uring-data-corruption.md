---
layout: post
title: "Deep Dive: The io_uring Data Corruption Bug"
date: 2026-08-23 00:00:00 +0000
author: Hamza Adaoud
tags: [linux, kernel, io_uring]
---

# io_uring: provided-buffer-ring head under-advance after bundled MSG_WAITALL receive

## Status

This is an application-level data-corruption bug in io_uring provided-buffer ring accounting. It does not provide a kernel memory-corruption primitive, cross-process data leak, or privilege-escalation vector. The maintainer has confirmed that it is not a security-boundary violation and requested a normal public patch. No upstream fix is merged as of this writeup.

## Summary

A logic bug in the Linux kernel io_uring subsystem causes the provided-buffer ring head to under-advance when a bundled receive operation (`IORING_RECVSEND_BUNDLE`) with `MSG_WAITALL` is forced to retry after a short read. The kernel successfully delivers the network data to the application's mapped memory but fails to commit the final segments of the provided buffer ring. Consequently, subsequent receive operations reuse the uncommitted buffer indices, silently overwriting data that the application correctly assumes was already delivered.

## Environment and Affected Versions

The behavior was reproduced on the following specific kernel versions:

- Linux 6.12.90
- Linux 6.12.101
- Vanilla Linux 7.2.0 (linux-next snapshot 20260821, dated August 21, 2026)

`IORING_RECVSEND_BUNDLE` support is present in the tested kernels. The testing confirms the behavior is present in the specific 6.12 stable/longterm releases listed above.

The precise commit that introduced this behavior has not been definitively established.

## Technical Background

When an application uses `IORING_OP_RECV` with `IORING_RECVSEND_BUNDLE` and provided buffers, io_uring maps multiple buffer segments at once to satisfy the requested read length. If `MSG_WAITALL` is specified, the kernel will repeatedly wait for network data until the full requested length is fulfilled, provided no errors occur.

During these operations, the kernel must track how many bytes have been written to which buffers and accurately advance the buffer ring head so the kernel does not reuse filled buffers for subsequent I/O.

## Reproduction Methodology and Observations

A standalone C reproducer was included in the original report to the maintainer and security list. It is not included here; the steps below are sufficient to reproduce the behavior.

The issue is deterministically reproducible using a stream socket and an 8-segment provided-buffer ring (100 bytes per buffer).

Steps:

1. The application issues a bundled `IORING_OP_RECV` requesting 400 bytes with `MSG_WAITALL`.
2. The peer writes a short payload of 150 bytes, pauses, and then writes the remaining 250 bytes.
3. The io_uring request completes with a CQE reporting `res=400`.
4. The application checks the provided-buffer ring head.

Expected Result:

The kernel advances the provided-buffer ring head by 4 (accounting for 400 bytes across four 100-byte buffers).

Observed Result:

The kernel only advances the provided-buffer ring head to 2.

Data Corruption Verification:

Because the head only advanced by 2, the kernel still considers buffer indices 2 and 3 available. If the application subsequently issues a standard 200-byte receive, the kernel maps the new incoming data into buffers 2 and 3. This overwrites the 200 bytes of data that were legitimately delivered during the second half of the initial bundled receive.

## Trace Evidence and Hypothesized Root Cause

Kernel instrumentation traces demonstrate a desynchronization between the buffer selection state and the commit logic during the `MSG_WAITALL` retry path.

When the initial short read (150 bytes) occurs, the request recycles. The trace shows a partial commit of the two buffers that absorbed the data:

```c
[   64.205642] IOURING-TRACE RECYCLE: len=150 done_io=150 flags=10900426020 iter_count=250 nbufs=2 bl=ffff9699474f3680
[   64.205649] IOURING-TRACE COMMIT: len=150 nr=2 flags=10d00426020 bl=ffff9699474f3680
[   64.205651] IOURING-TRACE COMMIT-DONE: len=150 nr=2 flags=10500426020 bl=ffff9699474f3680
```

The request then retries to fetch the remaining 250 bytes. The retry finishes successfully, but the final commit step is skipped because the buffer list pointer (`bl`) evaluates to `NULL` during completion:

```c
[   64.256903] IOURING-TRACE FINISH: sel_val=400 done_io=150 this_ret=250 iter_count=0 msg_inq=0 req_flags=10500426020 sr_flags=10 bl=0000000000000000 nbufs=4
[   64.256907] IOURING-TRACE AFTER_PUT: this_ret=250 done_io=150 req_flags=10500406020 cflags=1 bl=0000000000000000
```

Based on this trace, the hypothesized root cause is that during the retry, the request retains the previously peeked buffers but loses the necessary state (`REQ_F_BUFFERS_COMMIT` or a valid `bl` pointer) required to trigger `io_kbuf_commit()` when `__io_put_kbuf_ring()` is called during `io_recv_finish()`. Because `bl` is null, the function exits without committing the final buffers, leaving the ring head stale.

## Technical Impact

This is a reliable application-level data corruption bug. Because the overwrite occurs entirely within the bounds of the userspace application's explicitly registered buffer memory, it does not provide a kernel memory-corruption primitive, cross-process data leak, or privilege escalation vector.

## Investigation Timeline and Upstream Status

- August 23, 2026, 04:55: The reproducer, kernel trace, and affected versions were reported via email to the io_uring maintainer, Jens Axboe, and the kernel security list.
- August 23, 2026, 05:14: Jens Axboe responded, confirming that the data corruption behavior does not constitute a security boundary violation. He requested that a fix be submitted as a standard patch to the public io-uring@vger.kernel.org mailing list.
As of this writeup, an upstream patch resolving the missing buffer commit state during `MSG_WAITALL` retries has been requested by the maintainer, but no patch has yet been submitted or merged.
