---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Overview

This lab task requires you to fix a buggy `pthread`-based C program by adding the missing `pthread_mutex_lock()` and `pthread_mutex_unlock()` calls to eliminate **race conditions** and prevent **deadlocks**.

---

## Understanding the Problem

### What the Program Does

The program has **three threads** sharing two global resources:

| Thread | Behaviour |
|---|---|
| `worker_AB` | Locks A → updates A → Locks B → updates B (A then B order) |
| `worker_BA` | Locks B → updates B → Locks A → updates A (B then A order) |
| `worker_monitor` | Periodically reads/adjusts both resources, sets `done` flag |

### Shared Global Variables

| Variable | Protected By |
|---|---|
| `resourceA` | `lockA` |
| `resourceB` | `lockB` |
| `total_ops` | Both locks (accessed when holding both) |
| `done` | Both locks (accessed when holding both) |

### The Two Bugs Without Synchronisation

1. **Race Condition** — Multiple threads read/write `resourceA`, `resourceB`, `total_ops`, and `done` concurrently without any locks, causing undefined/corrupted values.
2. **Potential Deadlock** — If locks are acquired in **inconsistent order** (Thread 1 holds `lockA` waiting for `lockB`, Thread 2 holds `lockB` waiting for `lockA`), a circular wait deadlock occurs.

---

## Key Concepts

### Race Condition
> **⚠️ Warning:** A race condition occurs when two or more threads access a shared variable simultaneously and at least one access is a write, producing unpredictable results.

### Deadlock (Dining Philosophers style)
> **⚠️ Warning:** Deadlock happens when Thread 1 holds Lock A waiting for Lock B, while Thread 2 holds Lock B waiting for Lock A — neither can proceed.

### The Fix: Global Lock Ordering (Lock Hierarchy)

The universally accepted solution to prevent deadlock with multiple locks is to **always acquire locks in the same global order**.

**Rule:** Always lock `lockA` **before** `lockB`, regardless of which thread you are in.

This eliminates circular wait — one of the four necessary conditions for deadlock.

> **💡 Hint:** Even `worker_BA` (which logically updates B then A) must **still acquire lockA first**, then lockB, to maintain consistent lock ordering. It can update B first inside the critical section — the key is the *acquisition order*, not the update order.

---

## Step-by-Step Fix

### Step 1 — Identify Every Access to Shared Variables

Go through the code and mark every line that reads or writes:
- `resourceA`
- `resourceB`
- `total_ops`
- `done`

Each such access **must be inside a locked region**.

### Step 2 — Apply the Lock Ordering Rule

**Always: lock `lockA` first, then `lockB`**

```
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
// ... critical section accessing resourceA, resourceB, total_ops, done ...
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

Unlock in **reverse order** (LIFO) — this is best practice though not strictly required by pthreads.

### Step 3 — Fix Each Thread Function

---

## The Fixed Code: `notxv6/pthread_locks.c`

Below is the complete corrected program with all mutex lock/unlock calls added and annotated:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

// ─── Shared global resources ─────────────────────────────────────────────────
static int resourceA   = 0;
static int resourceB   = 0;
static int total_ops   = 0;
static int done        = 0;

// ─── Two provided mutexes — DO NOT add new ones ───────────────────────────────
static pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

// ─── Parameter struct passed from main() to each thread ──────────────────────
typedef struct {
    int thread_id;
    int iterations;
} thread_args_t;

// ─── Result struct heap-allocated by each thread, returned to main() ─────────
typedef struct {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
} thread_result_t;

// ─────────────────────────────────────────────────────────────────────────────
// worker_AB: updates resourceA then resourceB
// LOCK ORDER: always lockA → lockB  (safe, consistent with global order)
// ─────────────────────────────────────────────────────────────────────────────
void *worker_AB(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {
        // Acquire BOTH locks in global order: lockA first, then lockB
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        // ── Critical section ─────────────────────────
        resourceA++;
        resourceB++;
        total_ops++;
        result->ops++;
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;
        // ── End critical section ──────────────────────

        // Release in reverse order
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return result;
}

// ─────────────────────────────────────────────────────────────────────────────
// worker_BA: logically updates resourceB then resourceA
// LOCK ORDER: still lockA → lockB  ← KEY FIX to prevent deadlock
//             (update order inside critical section is B then A — that's fine)
// ─────────────────────────────────────────────────────────────────────────────
void *worker_BA(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {
        // Acquire BOTH locks in global order: lockA first, then lockB
        // (even though we update B before A — lock ORDER must be consistent)
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        // ── Critical section ─────────────────────────
        // Update B first (logical order preserved inside critical section)
        resourceB++;
        resourceA++;
        total_ops++;
        result->ops++;
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;
        // ── End critical section ──────────────────────

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return result;
}

// ─────────────────────────────────────────────────────────────────────────────
// worker_monitor: periodically reads and adjusts resources, sets done flag
// LOCK ORDER: lockA → lockB  (consistent with global order)
// ─────────────────────────────────────────────────────────────────────────────
void *worker_monitor(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    // Monitor loops until done is set
    while (1) {
        usleep(100000);  // sleep 100 ms between checks

        // Acquire BOTH locks in global order for a consistent snapshot
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        // ── Critical section ─────────────────────────
        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               resourceA, resourceB, total_ops, done);

        // Save snapshot into result
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        int current_done = done;   // read while holding lock
        // ── End critical section ──────────────────────

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        if (current_done) {
            break;   // exit loop only after releasing locks
        }
    }

    return result;
}

// ─────────────────────────────────────────────────────────────────────────────
// main: creates threads, waits for them, signals done, prints results
// ─────────────────────────────────────────────────────────────────────────────
int main(void)
{
    pthread_t t1, t2, t3;

    // Set up arguments for each thread
    thread_args_t args1 = { .thread_id = 1, .iterations = 40000 };
    thread_args_t args2 = { .thread_id = 2, .iterations = 40000 };
    thread_args_t args3 = { .thread_id = 3, .iterations = 0     };

    // Create all three threads
    pthread_create(&t1, NULL, worker_AB,      &args1);
    pthread_create(&t2, NULL, worker_BA,      &args2);
    pthread_create(&t3, NULL, worker_monitor, &args3);

    // Wait for worker_AB and worker_BA to finish
    thread_result_t *r1, *r2, *r3;
    pthread_join(t1, (void **)&r1);
    pthread_join(t2, (void **)&r2);

    // Signal monitor to stop (must be done under locks)
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    done = 1;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    // Wait for monitor to finish
    pthread_join(t3, (void **)&r3);

    // Print results
    printf("--- Results (returned to main) ---\n");
    printf("Thread 1 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r1->ops, r1->lastA, r1->lastB, r1->lastTotalOps, r1->lastDone);
    printf("Thread 2 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r2->ops, r2->lastA, r2->lastB, r2->lastTotalOps, r2->lastDone);
    printf("Thread 3 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r3->ops, r3->lastA, r3->lastB, r3->lastTotalOps, r3->lastDone);

    printf("Final Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    printf("Expected: total_ops=80000 done=1\n");

    // Free heap-allocated results
    free(r1);
    free(r2);
    free(r3);

    return 0;
}
```

---

## Why This Fix Works

### Deadlock Prevention — Lock Ordering

```
worker_AB:      lockA → lockB  ✅
worker_BA:      lockA → lockB  ✅  (NOT lockB → lockA!)
worker_monitor: lockA → lockB  ✅
main (done=1):  lockA → lockB  ✅
```

All threads acquire locks in the **same global order**: `lockA` first, `lockB` second.

This breaks the **circular wait** condition — a thread can only wait for `lockB` if it already holds `lockA`. No thread ever holds `lockB` and waits for `lockA`, so a deadlock cycle is impossible.

### Race Condition Prevention

| Shared Variable | Protected By |
|---|---|
| `resourceA` | `lockA` (held before any access) |
| `resourceB` | `lockB` (held before any access) |
| `total_ops` | Both locks held simultaneously |
| `done` | Both locks held simultaneously |

Every read and write of shared variables occurs **inside** a `lock()` / `unlock()` pair.

---

## Common Mistakes to Avoid

> **⚠️ Warning:** Do NOT acquire locks in different orders in different threads. This is the #1 cause of deadlock.
> ```c
> // BAD — worker_BA acquires in reverse order → potential deadlock
> pthread_mutex_lock(&lockB);  // ← WRONG
> pthread_mutex_lock(&lockA);
> ```

> **⚠️ Warning:** Do NOT read `done` outside a lock — it is a shared variable written by `main()` from a different thread.
> ```c
> // BAD — reading done without holding the lock
> while (!done) { ... }   // ← race condition on 'done'
> ```

> **⚠️ Warning:** Do NOT forget to `free()` the heap-allocated result structs returned by threads — this causes memory leaks.

> **⚠️ Warning:** Do NOT add new mutexes, semaphores, or condition variables — the task constraints prohibit this.

---

## Build and Test Instructions

### Compile and Run

```bash
cd notxv6/
make pthread_locks
./pthread_locks
```

### Expected Output

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

> **💡 Hint:** The exact intermediate values (e.g., `A=29943`) may differ each run — that is normal. What must be consistent is the **final** `total_ops=80000` and `done=1`.

### Run the Optional Test (20 iterations)

```bash
cd notxv6/
./optional_test
# Expected: PASS
```

### Run the Grade Script

```bash
cd ..
./grade-quiz pthread_locks
# Expected: pthread_locks_test: OK
```

### Run Full Make Grade

```bash
make clean && make grade
# Expected: Score: 1/1
```

---

## Submission Steps

```bash
# 1. Verify everything passes
make clean && make grade

# 2. Create the zip file using make (NOT Windows/macOS zip tools)
make zipball

# 3. Submit lab.zip to Gradescope assignment:
#    "xv6labs-quiz2-q2-pthread"
```

> **⚠️ Warning:** Always use `make zipball` to create your submission zip. Using Windows Explorer or macOS Files App zip will create nested directory structures that break the autograder.

---

## Summary Cheat Sheet

| Concept | Solution Applied |
|---|---|
| Race condition on `resourceA` | Wrap all accesses with `lockA` |
| Race condition on `resourceB` | Wrap all accesses with `lockB` |
| Race condition on `total_ops`, `done` | Wrap all accesses with both locks |
| Deadlock between `worker_AB` and `worker_BA` | Enforce global lock order: always `lockA → lockB` |
| Consistent monitor snapshot | Acquire both locks before reading any shared variable |
| Setting `done` from `main()` | Acquire both locks before writing `done = 1` |
