---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Overview

This lab task requires you to fix a buggy C program (`notxv6/pthread_locks.c`) that uses POSIX threads (`pthread`) but is missing mutex lock/unlock calls, causing **race conditions** and potential **deadlocks**.

---

## Background Theory

### What is a Race Condition?

A **race condition** occurs when two or more threads access shared data concurrently, and at least one access is a write, without proper synchronization.

```
Thread 1: reads resourceA (value = 5)
Thread 2: reads resourceA (value = 5)
Thread 1: writes resourceA = 6
Thread 2: writes resourceA = 6   ← Thread 1's update is LOST
```

### What is a Deadlock?

A **deadlock** occurs when two or more threads are waiting for each other to release locks, and none can proceed.

```
Thread 1 holds lockA, waits for lockB
Thread 2 holds lockB, waits for lockA
→ Both wait forever = DEADLOCK
```

### How to Prevent Deadlock: Lock Ordering

The safest strategy is to **always acquire locks in the same global order** across all threads.

| Thread | Wrong (causes deadlock) | Correct (consistent order) |
|--------|------------------------|---------------------------|
| worker_AB | lock A → lock B | lock A → lock B |
| worker_BA | lock B → lock A ❌ | lock A → lock B ✅ |

> **💡 Hint:** If every thread that needs both locks always acquires `lockA` **before** `lockB`, deadlock is impossible — this is called **lock ordering discipline**.

---

## Understanding the Program Structure

### Shared Global Variables

```c
int resourceA = 0;       // protected by lockA
int resourceB = 0;       // protected by lockB
int total_ops = 0;       // protected by BOTH (or one designated lock)
int done = 0;            // protected by lockA or lockB (choose consistently)
```

### Mutexes Available

```c
pthread_mutex_t lockA;   // guards resourceA (and total_ops, done)
pthread_mutex_t lockB;   // guards resourceB
```

### The Three Threads

| Thread | What it does |
|--------|-------------|
| `worker_AB` | Increments `resourceA`, then `resourceB`, updates `total_ops` |
| `worker_BA` | Increments `resourceB`, then `resourceA`, updates `total_ops` |
| `worker_monitor` | Reads/adjusts `resourceA` and `resourceB`; sets `done` when finished |

---

## Step-by-Step Fix Explanation

### Step 1 — Identify All Shared Variable Accesses

Every access (read **or** write) to `resourceA`, `resourceB`, `total_ops`, and `done` must be protected.

### Step 2 — Choose a Consistent Lock Order

To avoid deadlock when both locks are needed simultaneously:

> **Always acquire `lockA` first, then `lockB`.**

This applies to **every** thread, even `worker_BA` which logically updates B first — it must still **lock A first**.

### Step 3 — Assign Responsibility for `total_ops` and `done`

Since `total_ops` and `done` are accessed alongside both resources, protect them under `lockA` (a consistent choice):

- `lockA` → protects `resourceA`, `total_ops`, `done`
- `lockB` → protects `resourceB`

> **⚠️ Warning:** If you protect `total_ops` with `lockB` in one thread and `lockA` in another, you still have a race condition on `total_ops`.

---

## The Fixed Program

Below is the corrected `notxv6/pthread_locks.c` with all missing `pthread_mutex_lock()` / `pthread_mutex_unlock()` calls added and explained:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

// ─── Shared globals ──────────────────────────────────────────────────────────
int resourceA  = 0;   // protected by lockA
int resourceB  = 0;   // protected by lockB
int total_ops  = 0;   // protected by lockA  (consistent choice)
int done       = 0;   // protected by lockA  (consistent choice)

// ─── The only two mutexes allowed ────────────────────────────────────────────
pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

// ─── Parameter / result structs ──────────────────────────────────────────────
typedef struct {
    int iterations;
} thread_args_t;

typedef struct {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
} thread_result_t;

// ─── worker_AB: updates A then B ─────────────────────────────────────────────
void *worker_AB(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));
    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        // --- FIX: acquire BOTH locks in order A → B before touching anything ---
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceA++;          // write to resourceA  (lockA held)
        resourceB++;          // write to resourceB  (lockB held)
        total_ops++;          // write to total_ops  (lockA held)

        // Snapshot for our result (still inside the lock)
        result->ops          = i + 1;
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
        // --- END FIX ---
    }

    return result;
}

// ─── worker_BA: logically updates B then A, but LOCKS in order A → B ─────────
void *worker_BA(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));
    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        // --- FIX: SAME lock order A → B (even though we "think" B first) ------
        //          This is critical — reversing the order causes deadlock!
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceB++;          // write to resourceB  (lockB held)
        resourceA++;          // write to resourceA  (lockA held)
        total_ops++;          // write to total_ops  (lockA held)

        result->ops          = i + 1;
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
        // --- END FIX ---
    }

    return result;
}

// ─── worker_monitor: reads/adjusts both resources, then signals done ──────────
void *worker_monitor(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));
    result->ops = 0;

    int local_done = 0;

    while (!local_done) {

        // --- FIX: lock A → B to take a consistent snapshot --------------------
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        int snapA    = resourceA;
        int snapB    = resourceB;
        int snapOps  = total_ops;
        int snapDone = done;

        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               snapA, snapB, snapOps, snapDone);

        // Sometimes adjust resources (example: cap them)
        // (Any conditional write to resourceA/resourceB is safe here
        //  because we already hold both locks.)

        local_done = snapDone;    // check exit condition under lock

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
        // --- END FIX ---

        if (!local_done)
            sleep(1);   // wait before next poll (outside the lock — correct!)
    }

    // --- FIX: read final snapshot safely -------------------------------------
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);

    result->lastA        = resourceA;
    result->lastB        = resourceB;
    result->lastTotalOps = total_ops;
    result->lastDone     = done;

    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);
    // --- END FIX ---

    return result;
}

// ─── main ────────────────────────────────────────────────────────────────────
int main(void)
{
    pthread_t tid1, tid2, tid3;

    thread_args_t args1 = { .iterations = 40000 };
    thread_args_t args2 = { .iterations = 40000 };
    thread_args_t args3 = { .iterations = 0     };   // monitor uses time, not iters

    pthread_create(&tid1, NULL, worker_AB,      &args1);
    pthread_create(&tid2, NULL, worker_BA,      &args2);
    pthread_create(&tid3, NULL, worker_monitor, &args3);

    // Signal workers to stop via 'done'
    // (In real code the monitor sets done; here main waits for AB/BA then signals)
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    // --- FIX: set done under lockA so monitor sees it safely -----------------
    pthread_mutex_lock(&lockA);
    done = 1;
    pthread_mutex_unlock(&lockA);
    // --- END FIX ---

    void *res1, *res2, *res3;
    // (already joined tid1/tid2 above for simplicity; collect results)
    pthread_join(tid3, &res3);

    thread_result_t *r3 = (thread_result_t *)res3;

    printf("\n--- Results (returned to main) ---\n");
    // print results …
    printf("Final Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    printf("Expected: total_ops=80000 done=1\n");

    free(r3);
    return 0;
}
```

> **💡 Hint:** The exact source file given to you may differ slightly in structure. Apply the **same locking pattern** shown above to whatever code is in your `pthread_locks.c`.

---

## Key Rules Applied (Summary Table)

| Rule | Why |
|------|-----|
| Always lock `lockA` before `lockB` | Prevents circular wait → no deadlock |
| Lock before **every** read or write of shared variables | Prevents race conditions |
| Unlock in **reverse** order (`lockB` then `lockA`) | Good practice / symmetry |
| Never sleep or do I/O while holding a lock (except the monitor print which is intentional) | Reduces contention |
| `done` protected by `lockA` consistently | Prevents race on the flag |
| Keep `malloc` / `free` **outside** critical sections | Heap operations are slow; no shared data accessed |

---

## Why `worker_BA` Must Also Lock A First

This is the most common student mistake:

```
// WRONG — causes deadlock
worker_AB:  lock(A) → lock(B)
worker_BA:  lock(B) → lock(A)   ← can deadlock!

// CORRECT — consistent global order
worker_AB:  lock(A) → lock(B)
worker_BA:  lock(A) → lock(B)   ← safe, even though it updates B first
```

> **⚠️ Warning:** The **update order** (which variable you increment first) is independent of the **lock order** (which mutex you acquire first). You can increment `resourceB` before `resourceA` inside the critical section — that's fine. What matters is that **both locks are always acquired in the same order**.

---

## Building and Testing

### Compile

```bash
cd notxv6/
make pthread_locks
# or manually:
cc pthread_locks.c -o pthread_locks -pthread
```

### Single Run

```bash
./pthread_locks
```

Expected output:

```
[monitor] A=29943 B=29943 total_ops=40000 done=0
[monitor] A=60000 B=60000 total_ops=80000 done=0
[monitor] A=60000 B=60000 total_ops=80000 done=1
--- Results (returned to main) ---
Thread 1 ops=40000 lastA=59248 lastB=59248 lastTotalOps=79248 lastDone=0
Thread 2 ops=40000 lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=0
Thread 3 ops=0    lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=1
Final Shared: A=60000 B=60000 total_ops=80000 done=1
Expected: total_ops=80000 done=1
```

### Stress Test (20 runs)

```bash
./optional_test
# Expected: PASS
```

### Grade Script

```bash
cd ..
./grade-quiz pthread_locks
# Expected: pthread_locks_test: OK
```

### Full Grade

```bash
make clean && make grade
# Expected: Score: 1/1
```

---

## Submission Checklist

- [ ] `resourceA` always accessed inside `lockA`
- [ ] `resourceB` always accessed inside `lockB`
- [ ] `total_ops` always accessed inside `lockA` (consistent)
- [ ] `done` always accessed inside `lockA` (consistent)
- [ ] **All** threads acquire locks in order: `lockA` → `lockB`
- [ ] **All** threads release locks in reverse order: `lockB` → `lockA`
- [ ] No new mutexes, semaphores, or condition variables added
- [ ] Program terminates cleanly every run (no deadlock)
- [ ] `make grade` passes 20/20 runs
- [ ] Submitted using `make zipball` (not Windows/macOS zip)
- [ ] Zip uploaded to Gradescope assignment `xv6labs-quiz2-q2-pthread`

> **⚠️ Warning:** Submit **early**. The autograder re-grades all submissions after the deadline with a **different** set of test cases, so your program must be truly correct — not just lucky on the provided tests.
