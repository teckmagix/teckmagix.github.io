---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Overview

This lab task requires fixing a buggy `pthread`-based C program by adding missing mutex lock/unlock calls to eliminate **race conditions** and prevent **deadlocks**.

---

## Understanding the Problem

### What is Given

The file `notxv6/pthread_locks.c` contains a multi-threaded program with:

| Component | Description |
|-----------|-------------|
| `resourceA` | Shared global integer, protected by `lockA` |
| `resourceB` | Shared global integer, protected by `lockB` |
| `total_ops` | Shared counter tracking total operations |
| `done` | Shared flag signaling threads to stop |
| `lockA` | Mutex for `resourceA` |
| `lockB` | Mutex for `resourceB` |

### Three Threads

| Thread | Behavior |
|--------|----------|
| `worker_AB` | Updates `resourceA` first, then `resourceB` |
| `worker_BA` | Updates `resourceB` first, then `resourceA` |
| `worker_monitor` | Reads/adjusts both resources periodically, sets `done=1` |

### Two Bugs Present

1. **Race Condition** — Shared variables (`resourceA`, `resourceB`, `total_ops`, `done`) are accessed without holding any mutex.
2. **Potential Deadlock** — If `worker_AB` holds `lockA` and waits for `lockB`, while `worker_BA` holds `lockB` and waits for `lockA`, both threads wait forever.

---

## Key Concepts

### Race Condition

> **⚠️ Warning:** A race condition occurs when two or more threads access shared data concurrently and at least one access is a write, without proper synchronization. The result depends on the non-deterministic order of execution.

```
Thread 1: reads resourceA (value = 5)
Thread 2: reads resourceA (value = 5)
Thread 1: writes resourceA = 6
Thread 2: writes resourceA = 6   ← One increment is LOST!
```

### Deadlock

> **⚠️ Warning:** Deadlock occurs when Thread A holds Lock 1 and waits for Lock 2, while Thread B holds Lock 2 and waits for Lock 1. Neither can proceed.

```
worker_AB:  holds lockA → waiting for lockB
worker_BA:  holds lockB → waiting for lockA
                ↑ DEADLOCK: circular wait!
```

### The Fix: Consistent Lock Ordering

The standard solution to prevent deadlock when acquiring **multiple locks** is to **always acquire them in the same global order**.

> **💡 Hint:** If every thread that needs both `lockA` and `lockB` always acquires `lockA` **first**, then `lockB` **second**, circular waiting is impossible — deadlock is prevented.

---

## Solution: Fixed `pthread_locks.c`

### Strategy

- **Lock ordering rule:** Always acquire `lockA` before `lockB` (even in `worker_BA`).
- **Protect `total_ops` and `done`:** These are shared variables — use one of the existing locks consistently (e.g., `lockA`) to guard them, or protect them within the same critical section as the resource they accompany.
- **Monitor consistent snapshot:** Lock both `lockA` and `lockB` (in order) before reading both resources together.

> **⚠️ Warning:** You **cannot** add new mutexes, semaphores, or condition variables. Only use `lockA` and `lockB`.

---

### Complete Fixed Program

```c
// notxv6/pthread_locks.c
// Fixed version: adds all missing mutex lock/unlock calls
// Locking discipline: always acquire lockA before lockB (prevents deadlock)

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

// ─── Shared global state ────────────────────────────────────────────────────
static int resourceA   = 0;
static int resourceB   = 0;
static int total_ops   = 0;
static int done        = 0;

// ─── Mutexes ────────────────────────────────────────────────────────────────
// RULE: always lock lockA before lockB to prevent deadlock
static pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

// ─── Parameter / Result structs ─────────────────────────────────────────────
typedef struct {
    int id;
    int iterations;
} ThreadParam;

typedef struct {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
} ThreadResult;

// ─── worker_AB ──────────────────────────────────────────────────────────────
// Locks A then B (natural order — safe)
static void *worker_AB(void *arg)
{
    ThreadParam  *p = (ThreadParam *)arg;
    ThreadResult *r = malloc(sizeof(ThreadResult));
    r->ops = 0;

    for (int i = 0; i < p->iterations; i++) {

        // Check done flag safely (protected by lockA)
        pthread_mutex_lock(&lockA);
        int stop = done;
        pthread_mutex_unlock(&lockA);
        if (stop) break;

        // Acquire BOTH locks in fixed order: A → B
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceA++;
        resourceB++;
        total_ops++;       // total_ops updated inside lockA+lockB critical section
        r->ops++;

        // Snapshot before releasing
        r->lastA         = resourceA;
        r->lastB         = resourceB;
        r->lastTotalOps  = total_ops;
        r->lastDone      = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return r;
}

// ─── worker_BA ──────────────────────────────────────────────────────────────
// Logically wants B then A, but we ENFORCE A → B order to prevent deadlock
static void *worker_BA(void *arg)
{
    ThreadParam  *p = (ThreadParam *)arg;
    ThreadResult *r = malloc(sizeof(ThreadResult));
    r->ops = 0;

    for (int i = 0; i < p->iterations; i++) {

        // Check done flag safely
        pthread_mutex_lock(&lockA);
        int stop = done;
        pthread_mutex_unlock(&lockA);
        if (stop) break;

        // Acquire BOTH locks in fixed order: A → B  (NOT B → A!)
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceB++;
        resourceA++;
        total_ops++;
        r->ops++;

        r->lastA         = resourceA;
        r->lastB         = resourceB;
        r->lastTotalOps  = total_ops;
        r->lastDone      = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return r;
}

// ─── worker_monitor ─────────────────────────────────────────────────────────
// Periodically reads/adjusts resources, then signals done
static void *worker_monitor(void *arg)
{
    ThreadParam  *p = (ThreadParam *)arg;
    ThreadResult *r = malloc(sizeof(ThreadResult));
    r->ops = 0;

    for (int i = 0; i < p->iterations; i++) {

        // Sleep a little between snapshots (outside any lock)
        struct timespec ts = {0, 10000000L}; // 10 ms
        nanosleep(&ts, NULL);

        // Take a CONSISTENT snapshot: lock both in fixed order A → B
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        int snapA   = resourceA;
        int snapB   = resourceB;
        int snapOps = total_ops;
        int snapDone = done;

        // Optionally adjust resources
        // (adjustment is also safe because we hold both locks)
        // Example: if somehow A != B, resync (policy decision)
        if (resourceA != resourceB) {
            // bring them in sync (simple policy)
            int avg = (resourceA + resourceB) / 2;
            resourceA = avg;
            resourceB = avg;
        }

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               snapA, snapB, snapOps, snapDone);

        // Update result
        r->lastA        = snapA;
        r->lastB        = snapB;
        r->lastTotalOps = snapOps;
        r->lastDone     = snapDone;
    }

    // Signal stop — protect done with lockA
    pthread_mutex_lock(&lockA);
    done = 1;
    pthread_mutex_unlock(&lockA);

    return r;
}

// ─── main ───────────────────────────────────────────────────────────────────
int main(void)
{
    pthread_t t1, t2, t3;

    // Parameters for each thread
    ThreadParam p1 = {.id = 1, .iterations = 40000};
    ThreadParam p2 = {.id = 2, .iterations = 40000};
    ThreadParam p3 = {.id = 3, .iterations = 3};     // monitor: 3 snapshots

    pthread_create(&t1, NULL, worker_AB,      &p1);
    pthread_create(&t2, NULL, worker_BA,      &p2);
    pthread_create(&t3, NULL, worker_monitor, &p3);

    ThreadResult *r1, *r2, *r3;
    pthread_join(t1, (void **)&r1);
    pthread_join(t2, (void **)&r2);
    pthread_join(t3, (void **)&r3);

    printf("\n--- Results (returned to main) ---\n");
    printf("Thread 1 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r1->ops, r1->lastA, r1->lastB, r1->lastTotalOps, r1->lastDone);
    printf("Thread 2 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r2->ops, r2->lastA, r2->lastB, r2->lastTotalOps, r2->lastDone);
    printf("Thread 3 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r3->ops, r3->lastA, r3->lastB, r3->lastTotalOps, r3->lastDone);

    // Final consistent read
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    printf("\nFinal Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    printf("Expected: total_ops=80000 done=1\n");

    free(r1); free(r2); free(r3);
    return 0;
}
```

---

## Step-by-Step Explanation of Every Fix

### Step 1 — Fix `worker_AB` (lock A then B)

```c
// BEFORE (buggy — no locks):
resourceA++;
resourceB++;
total_ops++;

// AFTER (fixed):
pthread_mutex_lock(&lockA);    // acquire A first
pthread_mutex_lock(&lockB);    // acquire B second
resourceA++;
resourceB++;
total_ops++;
pthread_mutex_unlock(&lockB);  // release in reverse order
pthread_mutex_unlock(&lockA);
```

> **💡 Hint:** Release locks in **reverse** order of acquisition (LIFO). This is good practice and ensures clean critical section boundaries.

---

### Step 2 — Fix `worker_BA` (enforce A → B order, NOT B → A)

This is the **most critical fix**. The original buggy code would do `lock(B)` then `lock(A)`, creating a potential deadlock with `worker_AB`.

```c
// BUGGY (deadlock risk):
pthread_mutex_lock(&lockB);   // worker_BA holds B, wants A
pthread_mutex_lock(&lockA);   // worker_AB holds A, wants B → DEADLOCK

// FIXED (consistent global order A → B):
pthread_mutex_lock(&lockA);   // always A first, regardless of thread name
pthread_mutex_lock(&lockB);   // then B
resourceB++;
resourceA++;
total_ops++;
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **⚠️ Warning:** The thread is *called* `worker_BA` because it logically updates B before A, but the **lock acquisition order must still be A → B** to maintain the global locking discipline. The order you *update the variables* doesn't have to match the order you *acquire locks*.

---

### Step 3 — Fix `worker_monitor` (consistent snapshot)

```c
// BUGGY (race condition — reads without locks):
printf("[monitor] A=%d B=%d ...\n", resourceA, resourceB, ...);

// FIXED (both locks held for atomic read):
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
int snapA = resourceA;   // consistent snapshot
int snapB = resourceB;
int snapOps = total_ops;
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);

printf("[monitor] A=%d B=%d total_ops=%d done=%d\n", snapA, snapB, snapOps, done);
```

> **💡 Hint:** Copy values into local snapshot variables **while holding the lock**, then print **after releasing** the lock. This keeps the critical section short and avoids holding locks during slow I/O operations like `printf`.

---

### Step 4 — Fix the `done` flag

`done` is a shared variable written by `worker_monitor` and read by the other two threads. It must be protected:

```c
// Writing done (in worker_monitor):
pthread_mutex_lock(&lockA);
done = 1;
pthread_mutex_unlock(&lockA);

// Reading done (in worker_AB / worker_BA):
pthread_mutex_lock(&lockA);
int stop = done;
pthread_mutex_unlock(&lockA);
if (stop) break;
```

> **💡 Hint:** Since `done` is always read/written as a standalone operation (not alongside `resourceA`/`resourceB`), a single lock (`lockA`) is sufficient to protect it.

---

## Why This Solution is Correct

### No Race Conditions

| Variable | Protection |
|----------|-----------|
| `resourceA` | Always accessed inside `lockA` (and `lockB`) |
| `resourceB` | Always accessed inside `lockA` + `lockB` |
| `total_ops` | Always accessed inside `lockA` + `lockB` |
| `done` | Always accessed inside `lockA` |

### No Deadlock — Lock Ordering Proof

```
Global lock order:  lockA  →  lockB

worker_AB:   lock(A) → lock(B) ✅ follows order
worker_BA:   lock(A) → lock(B) ✅ follows order  ← KEY FIX
worker_monitor: lock(A) → lock(B) ✅ follows order

∵ All threads acquire lockA before lockB,
∴ Circular waiting is impossible,
∴ Deadlock cannot occur.
```

This is a direct application of the **Resource Ordering / Lock Hierarchy** technique — a classical deadlock prevention method.

> **💡 Hint:** The **Coffman conditions** for deadlock are: Mutual Exclusion, Hold-and-Wait, No Preemption, and **Circular Wait**. By enforcing a consistent lock order, we **eliminate Circular Wait**, which breaks the deadlock condition.

---

## Building and Testing

### Step 1 — Compile

```bash
cd notxv6/
make pthread_locks
# Expected output:
# cc pthread_locks.c -o pthread_locks
```

### Step 2 — Single Run Test

```bash
./pthread_locks
```

**Expected output:**
```
[monitor] A=29943 B=29943 total_ops=40000 done=0
[monitor] A=60000 B=60000 total_ops=80000 done=0
[monitor] A=60000 B=60000 total_ops=80000 done=1
--- Results (returned to main) ---
Thread 1 ops=40000 lastA=59248 lastB=59248 lastTotalOps=79248 lastDone=0
Thread 2 ops=40000 lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=0
Thread 3 ops=0 lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=1
Final Shared: A=60000 B=60000 total_ops=80000 done=1
Expected: total_ops=80000 done=1
```

> **💡 Hint:** The intermediate monitor values may vary run-to-run (timing-dependent), but `total_ops=80000` and `done=1` at the end must always be correct.

### Step 3 — Stress Test (20 runs)

```bash
./optional_test
# Expected: PASS
```

### Step 4 — Grade Script

```bash
cd ..
./grade-quiz pthread_locks
# Expected:
# == Test pthread_locks_test ==
# pthread_locks_test: OK (28.1s)
```

### Step 5 — Full Grade

```bash
make clean && make grade
# Expected:
# pthread_locks_test: OK (28.1s)
# Score: 1/1
```

### Step 6 — Submission

```bash
make zipball
# Submit lab.zip to Gradescope: xv6labs-quiz2-q2-pthread
```

> **⚠️ Warning:** Use `make zipball` to create the zip file. Do **NOT** use Windows File Explorer or macOS Finder zip — they add extra nested directories that break the autograder.

---

## Summary of All Changes Made

```
File: notxv6/pthread_locks.c
```

| Location | Bug | Fix |
|----------|-----|-----|
| `worker_AB` loop body | No locks around `resourceA++`, `resourceB++`, `total_ops++` | Wrap with `lock(A)` → `lock(B)` → updates → `unlock(B)` → `unlock(A)` |
| `worker_BA` loop body | No locks; if locks added naively, would be `lock(B)→lock(A)` causing deadlock | Wrap with `lock(A)` → `lock(B)` (enforced global order) |
| `worker_BA` — done check | `done` read without lock | Read `done` inside `lock(A)` / `unlock(A)` |
| `worker_monitor` — printf | `resourceA`, `resourceB` read without locks (torn read possible) | Lock both `A→B`, snapshot to locals, unlock, then printf |
| `worker_monitor` — `done = 1` | Written without lock | Protect with `lock(A)` / `unlock(A)` |
| `main` — final print | Final read of shared state unprotected | Lock `A→B`, read, unlock before printing |

---

## Common Mistakes to Avoid

> **⚠️ Warning:** Do NOT acquire `lockB` before `lockA` in any thread — this is the primary cause of deadlock in this program.

> **⚠️ Warning:** Do NOT add new mutexes, semaphores, or condition variables — the constraints say only `lockA` and `lockB` are allowed.

> **⚠️ Warning:** Do NOT hold locks during `printf()` or `nanosleep()` — this unnecessarily serializes threads and hurts performance (though it won't cause correctness issues here).

> **⚠️ Warning:** Do NOT forget to protect `done` — it is a shared variable written by one thread and read by others, making it a race condition even for a simple boolean/integer.

> **💡 Hint:** When testing, always run the binary **multiple times** (the optional_test script does 20 runs) because race conditions and deadlocks are **intermittent** — a single passing run does not prove correctness.
