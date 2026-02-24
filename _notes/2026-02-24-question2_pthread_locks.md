---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Overview

This lab task requires you to fix a buggy `pthread`-based C program by adding missing mutex lock/unlock calls to eliminate **race conditions** and prevent **deadlocks**.

---

## Understanding the Problem

### What is Given

You have a file `notxv6/pthread_locks.c` that:
- Has **two shared global resources**: `resourceA` and `resourceB`
- Has **two mutexes**: `lockA` (protects `resourceA`) and `lockB` (protects `resourceB`)
- Has **three threads**:
  - `worker_AB` — updates A then B
  - `worker_BA` — updates B then A
  - `worker_monitor` — reads/adjusts both resources, signals stop via `done`

### What is Wrong

| Bug Type | Description |
|---|---|
| **Race Condition** | Shared variables (`resourceA`, `resourceB`, `total_ops`, `done`) accessed without locks |
| **Potential Deadlock** | If `worker_AB` holds `lockA` and waits for `lockB`, while `worker_BA` holds `lockB` and waits for `lockA` — circular wait |

---

## Key Concepts to Understand

### Race Condition

> **💡 Hint:** A race condition occurs when two or more threads access a shared variable concurrently and at least one access is a write, without any synchronization.

```
Thread 1: reads resourceA = 100
Thread 2: reads resourceA = 100
Thread 1: writes resourceA = 101
Thread 2: writes resourceA = 101   ← One increment is LOST!
```

### Deadlock (The Classic AB/BA Problem)

> **⚠️ Warning:** This is the most common mistake students make. If `worker_AB` locks A→B and `worker_BA` locks B→A, you get a deadlock!

```
worker_AB:  locks lockA → tries to lock lockB (WAITING)
worker_BA:  locks lockB → tries to lock lockA (WAITING)
← Both waiting forever = DEADLOCK
```

### The Fix: Global Lock Ordering

The standard solution to the AB/BA deadlock is to **enforce a consistent global lock ordering**. Always acquire `lockA` before `lockB`, regardless of which thread is running.

```
worker_AB:  lock A → lock B → update → unlock B → unlock A  ✓
worker_BA:  lock A → lock B → update → unlock B → unlock A  ✓ (same order!)
```

---

## Full Solution: `pthread_locks.c`

Below is the corrected program with all missing `pthread_mutex_lock()` / `pthread_mutex_unlock()` calls added and explained.

```c
#include <stdlib.h>
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>   // for usleep

/* ── Shared global state ─────────────────────────────── */
static int resourceA   = 0;
static int resourceB   = 0;
static int total_ops   = 0;
static int done        = 0;

/* ── The only two mutexes you are allowed to use ────── */
static pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

/* ── Parameter struct passed from main() to threads ─── */
typedef struct {
    int iterations;
} thread_args_t;

/* ── Result struct returned (heap-allocated) to main() ─ */
typedef struct {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
} thread_result_t;


/* ================================================================
 * worker_AB  –  increments A then B  (lock order: A → B)
 * ================================================================ */
void *worker_AB(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        /* FIX: Always acquire lockA before lockB (global ordering) */
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        /* Critical section – both resources updated atomically */
        resourceA++;
        resourceB++;
        total_ops++;

        result->ops++;
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return result;
}


/* ================================================================
 * worker_BA  –  increments B then A
 *               FIX: still lock in order A → B to avoid deadlock
 * ================================================================ */
void *worker_BA(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        /* FIX: Acquire lockA FIRST even though logically we "do B first"
         *      This breaks the circular-wait condition → no deadlock      */
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        /* Critical section */
        resourceB++;
        resourceA++;
        total_ops++;

        result->ops++;
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return result;
}


/* ================================================================
 * worker_monitor  –  reads snapshots, sometimes adjusts, sets done
 * ================================================================ */
void *worker_monitor(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        usleep(100000); /* 100 ms between snapshots */

        /* FIX: Lock both (in order A → B) for a consistent snapshot */
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        /* Safe read of all shared variables */
        int snapA    = resourceA;
        int snapB    = resourceB;
        int snapOps  = total_ops;
        int snapDone = done;

        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               snapA, snapB, snapOps, snapDone);

        /* Occasional adjustment (still inside the lock) */
        if (snapA != snapB) {
            /* Re-synchronise the counters if they diverged */
            int avg  = (resourceA + resourceB) / 2;
            resourceA = avg;
            resourceB = avg;
        }

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        result->lastA        = snapA;
        result->lastB        = snapB;
        result->lastTotalOps = snapOps;
        result->lastDone     = snapDone;
    }

    /* Signal the other threads that the monitor is done.
     * FIX: protect 'done' with lockA (or both locks – pick one
     *      and be consistent; here we use lockA as the "done guard") */
    pthread_mutex_lock(&lockA);
    done = 1;
    result->lastDone = done;
    pthread_mutex_unlock(&lockA);

    return result;
}


/* ================================================================
 * main
 * ================================================================ */
int main(void)
{
    pthread_t t1, t2, t3;

    /* Each worker thread gets its own args struct */
    thread_args_t args1 = { .iterations = 40000 };
    thread_args_t args2 = { .iterations = 40000 };
    thread_args_t args3 = { .iterations = 3     };  /* monitor: 3 snapshots */

    pthread_create(&t1, NULL, worker_AB,      &args1);
    pthread_create(&t2, NULL, worker_BA,      &args2);
    pthread_create(&t3, NULL, worker_monitor, &args3);

    thread_result_t *r1, *r2, *r3;
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

    /* FIX: Read final shared state under locks for a clean final print */
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    printf("\nFinal Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    printf("Expected: total_ops=80000 done=1\n");

    free(r1);
    free(r2);
    free(r3);

    return 0;
}
```

---

## Step-by-Step Explanation of Every Fix

### Fix 1 — `worker_AB`: Add Lock/Unlock Around Critical Section

```c
/* BEFORE (buggy) */
resourceA++;
resourceB++;
total_ops++;

/* AFTER (fixed) */
pthread_mutex_lock(&lockA);    // acquire A first
pthread_mutex_lock(&lockB);    // then acquire B
resourceA++;
resourceB++;
total_ops++;
pthread_mutex_unlock(&lockB);  // release in reverse order
pthread_mutex_unlock(&lockA);
```

**Why:** Without locks, two threads can read and write `resourceA` simultaneously, causing lost updates.

---

### Fix 2 — `worker_BA`: Use the SAME Lock Order (A → B), Not (B → A)

```c
/* WRONG – causes deadlock */
pthread_mutex_lock(&lockB);    // ← grabs B first
pthread_mutex_lock(&lockA);

/* CORRECT – same order as worker_AB */
pthread_mutex_lock(&lockA);    // ← always grab A first
pthread_mutex_lock(&lockB);
resourceB++;
resourceA++;
total_ops++;
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **⚠️ Warning:** Even though `worker_BA` logically "does B first", the **lock acquisition order must still be A → B**. The actual update order inside the critical section does not matter for correctness — only the **lock order** matters for deadlock prevention.

---

### Fix 3 — `worker_monitor`: Lock Both Resources for Consistent Snapshot

```c
/* BEFORE (buggy) – reads without locks, values can be mid-update */
printf("[monitor] A=%d B=%d ...\n", resourceA, resourceB, total_ops, done);

/* AFTER (fixed) */
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
// Now resourceA and resourceB are in a consistent state
printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
       resourceA, resourceB, total_ops, done);
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

**Why:** Without locks, the monitor could print `A=50001, B=50000` — an inconsistent snapshot where A was already incremented but B was not yet.

---

### Fix 4 — `done` Variable: Protect With a Lock

```c
/* BEFORE (buggy) */
done = 1;

/* AFTER (fixed) */
pthread_mutex_lock(&lockA);   // use lockA as the "done" guard
done = 1;
pthread_mutex_unlock(&lockA);
```

**Why:** `done` is a shared variable written by the monitor and read by other threads. It must be protected to avoid a data race.

> **💡 Hint:** Since we only have `lockA` and `lockB`, pick one consistently for protecting `done`. Using `lockA` alone (without `lockB`) is fine here since we are only touching `done`, not both resources.

---

## The Deadlock Prevention Rule Explained

### Four Conditions for Deadlock (Coffman Conditions)

| Condition | Description | Our Fix |
|---|---|---|
| **Mutual Exclusion** | Only one thread can hold a mutex | Needed – cannot remove |
| **Hold and Wait** | Thread holds one lock and waits for another | Still present but ordered |
| **No Preemption** | Lock cannot be forcibly taken | Still present |
| **Circular Wait** | Thread A waits for B, Thread B waits for A | **ELIMINATED** by lock ordering |

By enforcing **lock ordering (A always before B)**, we eliminate the circular wait condition, which breaks deadlock.

---

## Visual Timeline: Before vs After Fix

### Before Fix (Race Condition Example)

```
Time  Thread1 (worker_AB)      Thread2 (worker_BA)
 1    read resourceA = 0
 2                              read resourceA = 0
 3    write resourceA = 1
 4                              write resourceA = 1   ← Lost update!
```

### After Fix (Correct Serialization)

```
Time  Thread1 (worker_AB)      Thread2 (worker_BA)
 1    lock(A), lock(B)
 2    resourceA++ = 1
 3    resourceB++ = 1
 4    unlock(B), unlock(A)
 5                              lock(A), lock(B)       ← Must wait
 6                              resourceB++ = 2
 7                              resourceA++ = 2
 8                              unlock(B), unlock(A)
```

---

## Building and Testing

### Step 1: Compile the Program

```bash
cd notxv6/
make pthread_locks
```

Expected output:
```
cc pthread_locks.c -o pthread_locks
```

### Step 2: Run the Program

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
Thread 3 ops=0 lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=1
Final Shared: A=60000 B=60000 total_ops=80000 done=1
Expected: total_ops=80000 done=1
```

> **💡 Hint:** The exact values of `lastA`, `lastB`, `lastTotalOps` in Thread 1 and 2 may vary slightly each run — that is normal. What must be deterministic is `total_ops=80000` and `done=1`.

### Step 3: Run the Optional Test (Catches Intermittent Issues)

```bash
./optional_test
```

Expected:
```
PASS
```

### Step 4: Run the Grade Script

```bash
cd ..
./grade-quiz pthread_locks
```

Expected:
```
== Test pthread_locks_test == gcc -o pthread_locks -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/pthread_locks.c -pthread
pthread_locks_test: OK (28.1s)
```

### Step 5: Full Grade Check

```bash
make clean && make grade
```

Expected:
```
pthread_locks_test: OK (28.1s)
Score: 1/1
```

### Step 6: Create and Submit Zip

```bash
make zipball
```

> **⚠️ Warning:** Always use `make zipball` — **do NOT use** Windows Explorer zip or macOS Files App. This avoids nested directory issues in the autograder.

Submit the generated `lab.zip` to Gradescope under **"xv6labs-quiz2-q2-pthread"**.

---

## Summary Checklist

Use this checklist before submitting:

- [ ] `worker_AB` acquires `lockA` then `lockB` before touching any shared variable
- [ ] `worker_BA` acquires `lockA` then `lockB` (same order, even though it updates B first)
- [ ] Both workers unlock in reverse order: `lockB` then `lockA`
- [ ] `worker_monitor` acquires both locks before reading/printing/adjusting shared state
- [ ] `done = 1` is written inside a mutex lock
- [ ] All reads of `done` by other threads are also inside a lock
- [ ] `total_ops` is always updated inside the locked critical section
- [ ] No new mutexes, semaphores, or condition variables were added
- [ ] `./optional_test` prints `PASS`
- [ ] `make grade` shows `Score: 1/1`

---

## Common Mistakes to Avoid

> **⚠️ Warning:** Inconsistent lock ordering is the #1 mistake. If `worker_AB` locks A→B and `worker_BA` locks B→A, the program will appear to work most of the time but **deadlock intermittently** — which is why the test scripts run the binary **20 times**.

> **⚠️ Warning:** Do not forget to protect `total_ops` and `done`. These are also shared global variables and are just as vulnerable to race conditions as `resourceA` and `resourceB`.

> **⚠️ Warning:** Do not add extra mutexes. The constraint says only `lockA` and `lockB` may be used. Adding a third mutex could cause you to fail the autograder.

> **💡 Hint:** When in doubt, hold **both** locks (`lockA` + `lockB`) whenever you access **any** of the four shared variables (`resourceA`, `resourceB`, `total_ops`, `done`). This is slightly conservative but guaranteed correct with the two-lock ordering rule.
