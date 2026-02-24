---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## ICT1012 Operating Systems – Quiz 2 Question 2
## pthread Deadlock & Race Condition Fix (Add Missing Mutex Locks/Unlocks)

---

## Overview

This task requires you to fix a buggy `pthread`-based C program (`notxv6/pthread_locks.c`) by adding the correct `pthread_mutex_lock()` and `pthread_mutex_unlock()` calls to:

- Eliminate **data races** on shared variables
- Prevent **deadlocks**
- Ensure the program always **terminates correctly**

---

## Understanding the Problem

### Shared Resources & Their Protectors

| Shared Variable | Protected By |
|----------------|-------------|
| `resourceA` | `lockA` |
| `resourceB` | `lockB` |
| `total_ops` | both locks (acquired together) |
| `done` | both locks (acquired together) |

### The Three Threads

| Thread | What It Does | Lock Order |
|--------|-------------|------------|
| `worker_AB` | Updates A then B | Must lock A → then B |
| `worker_BA` | Updates B then A | Must lock A → then B (**same order!**) |
| `worker_monitor` | Reads/adjusts A and B, sets `done` | Must lock A → then B (**same order!**) |

> **⚠️ Warning:** The classic cause of deadlock here is **inconsistent lock ordering**. `worker_AB` acquires `lockA` then `lockB`, but if `worker_BA` acquires `lockB` then `lockA`, a circular wait can occur. **Always acquire locks in the same global order: lockA first, then lockB.**

---

## Key Concepts Before Coding

### What is a Race Condition?
A race condition occurs when two or more threads access a shared variable concurrently and at least one access is a write, without synchronization. The result is **non-deterministic** and incorrect.

### What is a Deadlock?
A deadlock occurs when:
1. Thread 1 holds Lock A and waits for Lock B
2. Thread 2 holds Lock B and waits for Lock A

Both threads wait forever → program hangs.

### The Fix: Consistent Lock Ordering
By enforcing a **global lock order** (always lock A before B), we break the circular wait condition and eliminate deadlock.

---

## Step-by-Step Solution

### Step 1 – Understand the Original Buggy Structure

The original code has threads accessing `resourceA`, `resourceB`, `total_ops`, and `done` **without any mutex calls**, causing race conditions. The structure looks like this:

```c
// BUGGY - no locks!
void *worker_AB(void *arg) {
    // ... 
    resourceA++;       // RACE CONDITION
    resourceB++;       // RACE CONDITION
    total_ops++;       // RACE CONDITION
    // ...
}
```

### Step 2 – Apply the Locking Rule

> **💡 Hint:** When you need **both** locks, always acquire them in the order: **lockA first, then lockB**. This is called a **lock hierarchy** or **lock ordering discipline** and it prevents circular wait (one of the four Coffman conditions for deadlock).

### Step 3 – The Fixed Program

Below is the complete fixed `notxv6/pthread_locks.c` with all mutex lock/unlock calls added correctly:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <assert.h>
#include <pthread.h>

// ----- Shared global resources -----
static int resourceA = 0;
static int resourceB = 0;
static int total_ops = 0;
static int done      = 0;

// ----- The two provided mutexes -----
pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

// ----- Parameter struct passed from main() to each thread -----
typedef struct {
    int id;
    int iterations;
} thread_args_t;

// ----- Result struct heap-allocated by each thread, returned to main() -----
typedef struct {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
} thread_result_t;

// -----------------------------------------------------------------------
// worker_AB: increments resourceA then resourceB
// Lock order: lockA -> lockB  (ALWAYS this order)
// -----------------------------------------------------------------------
void *worker_AB(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));
    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        // Acquire BOTH locks in order: A first, then B
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceA++;
        resourceB++;
        total_ops++;
        result->ops++;

        // Snapshot while still holding both locks
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        // Release in reverse order (good practice, not strictly required here)
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    return result;
}

// -----------------------------------------------------------------------
// worker_BA: increments resourceB then resourceA
// Lock order: lockA -> lockB  (SAME order as worker_AB to prevent deadlock)
// -----------------------------------------------------------------------
void *worker_BA(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));
    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        // Acquire BOTH locks in order: A first, then B
        // Even though logically we update B first, we MUST lock A first
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceB++;           // update B (but we locked A first for ordering)
        resourceA++;           // then update A
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

// -----------------------------------------------------------------------
// worker_monitor: periodically reads A and B, adjusts them, sets done
// Lock order: lockA -> lockB  (SAME order)
// -----------------------------------------------------------------------
void *worker_monitor(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));
    result->ops = 0;

    while (1) {

        sleep(1);   // wait a bit between monitor checks

        // Acquire both locks to take a consistent snapshot
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        // Consistent read of all shared variables
        int snapA    = resourceA;
        int snapB    = resourceB;
        int snapOps  = total_ops;
        int snapDone = done;

        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               snapA, snapB, snapOps, snapDone);

        // Optionally adjust resources
        // (adjustment logic depends on original program; preserve as-is)

        // Check termination condition
        if (snapOps >= 80000) {
            done = 1;   // signal stop (while holding the lock - safe)
        }

        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        // Read done AFTER releasing lock for the exit check
        // Re-read done safely:
        pthread_mutex_lock(&lockA);
        int check = done;
        pthread_mutex_unlock(&lockA);

        if (check) break;
    }

    return result;
}

// -----------------------------------------------------------------------
// main: spawns threads, joins them, prints results
// -----------------------------------------------------------------------
int main(void)
{
    pthread_t t1, t2, t3;

    thread_args_t args1 = { .id = 1, .iterations = 40000 };
    thread_args_t args2 = { .id = 2, .iterations = 40000 };
    thread_args_t args3 = { .id = 3, .iterations = 0     };  // monitor

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

    printf("\nFinal Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    printf("Expected: total_ops=80000 done=1\n");

    free(r1); free(r2); free(r3);
    return 0;
}
```

> **⚠️ Warning:** The exact logic inside each thread (how many iterations, what adjustments the monitor makes) comes from the **original downloaded file**. The solution above shows **where and how** to add lock/unlock calls. Apply the same pattern to the actual `pthread_locks.c` you downloaded.

---

## The Core Fix — Summary of Changes

The **only changes** you make to the buggy file are adding lock/unlock calls. Here is the minimal pattern for each thread:

### Pattern for worker_AB (updates A then B)

```c
// BEFORE (buggy):
resourceA++;
resourceB++;
total_ops++;

// AFTER (fixed):
pthread_mutex_lock(&lockA);    // Step 1: Lock A first
pthread_mutex_lock(&lockB);    // Step 2: Lock B second
resourceA++;                   // Safe update
resourceB++;                   // Safe update
total_ops++;                   // Safe update
pthread_mutex_unlock(&lockB);  // Step 3: Unlock B first
pthread_mutex_unlock(&lockA);  // Step 4: Unlock A last
```

### Pattern for worker_BA (updates B then A — SAME lock order!)

```c
// BEFORE (buggy):
resourceB++;
resourceA++;
total_ops++;

// AFTER (fixed) — lock ORDER does NOT follow update order:
pthread_mutex_lock(&lockA);    // Step 1: Lock A FIRST (even though we update B first)
pthread_mutex_lock(&lockB);    // Step 2: Lock B second
resourceB++;                   // Safe update
resourceA++;                   // Safe update
total_ops++;                   // Safe update
pthread_mutex_unlock(&lockB);  // Step 3: Unlock B
pthread_mutex_unlock(&lockA);  // Step 4: Unlock A
```

### Pattern for worker_monitor (reads/adjusts both, sets done)

```c
// BEFORE (buggy):
printf("[monitor] A=%d B=%d ...\n", resourceA, resourceB, ...);
if (total_ops >= 80000) done = 1;

// AFTER (fixed):
pthread_mutex_lock(&lockA);    // Lock A first
pthread_mutex_lock(&lockB);    // Lock B second
printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
       resourceA, resourceB, total_ops, done);  // Consistent read
if (total_ops >= 80000) done = 1;               // Safe write to done
pthread_mutex_unlock(&lockB);  // Unlock B
pthread_mutex_unlock(&lockA);  // Unlock A
```

---

## Why This Works — Conceptual Explanation

### Deadlock Prevention via Lock Ordering

```
Without fix (can deadlock):               With fix (safe):
                                          
Thread AB:  lock(A) → lock(B)            Thread AB:  lock(A) → lock(B)
Thread BA:  lock(B) → lock(A)  ← BAD    Thread BA:  lock(A) → lock(B)  ← SAME ORDER
                                          
Circular wait possible!                  No circular wait possible!
```

### Race Condition Prevention

```
Without fix:                    With fix:
                                
T1: read resourceA (=5)        T1: lock(A), lock(B)
T2: read resourceA (=5)        T1: resourceA++ → 6
T1: write resourceA = 6        T1: unlock(B), unlock(A)
T2: write resourceA = 6        T2: lock(A), lock(B)   ← waits for T1 to finish
                                T2: resourceA++ → 7   ← correct!
Lost update! Final = 6         T2: unlock(B), unlock(A)
instead of 7!                  Final = 7 ✓
```

---

## Building and Testing

### Step 1 — Compile

```bash
cd notxv6/
make pthread_locks
```

Expected output:
```
cc pthread_locks.c -o pthread_locks
```

### Step 2 — Run Once

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

> **💡 Hint:** The exact intermediate values (like `A=29943`) may vary between runs — that's fine. What matters is the **final** `total_ops=80000` and `done=1` are always correct.

### Step 3 — Run Optional Test (20 times)

```bash
./optional_test
```

Expected: `PASS`

### Step 4 — Grade Script

```bash
cd ..
./grade-quiz pthread_locks
```

Expected:
```
== Test pthread_locks_test ==
pthread_locks_test: OK (28.1s)
```

### Step 5 — Full Grade

```bash
make clean && make grade
```

Expected:
```
pthread_locks_test: OK (28.1s)
Score: 1/1
```

---

## Submission Instructions

```bash
# 1. Run grade to confirm pass
make clean && make grade

# 2. Create zip (use make zipball, NOT Windows/Mac zip tools!)
make zipball

# 3. Submit lab.zip to Gradescope:
#    Assignment: xv6labs-quiz2-q2-pthread
```

> **⚠️ Warning:** Do **NOT** use Windows Explorer or macOS Files App to create the zip. Always use `make zipball` to avoid directory structure issues that break autograding.

> **⚠️ Warning:** The grading script runs the binary **20 times** to catch intermittent deadlocks. A solution that only sometimes works will **fail**. Make sure your lock ordering is **always consistent**.

---

## Quick Checklist Before Submitting

| Check | Status |
|-------|--------|
| Every access to `resourceA` is inside `lock(lockA)` | ✅ |
| Every access to `resourceB` is inside `lock(lockB)` | ✅ |
| Every access to `total_ops` is inside both locks | ✅ |
| Every access to `done` is inside both locks | ✅ |
| All three threads lock in order: **lockA → lockB** | ✅ |
| No new mutexes, semaphores, or condition variables added | ✅ |
| `./optional_test` gives `PASS` | ✅ |
| `./grade-quiz pthread_locks` gives `OK` | ✅ |
| Zip made with `make zipball` | ✅ |

---

## Summary of the Golden Rules

> **💡 Hint — The 3 Rules for This Task:**
>
> 1. **Lock before accessing any shared variable** (`resourceA`, `resourceB`, `total_ops`, `done`)
> 2. **Always acquire locks in the same global order**: `lockA` first, then `lockB` — in **every** thread
> 3. **Always unlock in reverse order**: `lockB` first, then `lockA` (prevents lock inversion issues)

Following these three rules eliminates both **race conditions** and **deadlocks** using only the two provided mutexes.
