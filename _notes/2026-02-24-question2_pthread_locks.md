---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Overview

This lab task requires fixing a buggy `pthread`-based C program that suffers from **race conditions** and potential **deadlocks** due to missing mutex lock/unlock calls. You must add synchronization using only the two provided mutexes: `lockA` and `lockB`.

---

## Understanding the Problem

### Shared Resources and Their Protectors

| Shared Variable | Protected By |
|----------------|-------------|
| `resourceA` | `lockA` |
| `resourceB` | `lockB` |
| `total_ops` | Either `lockA` or `lockB` (consistently) |
| `done` | Either `lockA` or `lockB` (consistently) |

### The Three Threads

| Thread | Behaviour |
|--------|-----------|
| `worker_AB` | Locks A → updates A, then locks B → updates B |
| `worker_BA` | Locks B → updates B, then locks A → updates A |
| `worker_monitor` | Reads A and B periodically, adjusts them, signals `done` |

> **⚠️ Warning:** `worker_AB` locks A then B, while `worker_BA` locks B then A — this **inconsistent lock ordering** is the classic cause of deadlock. You must enforce a **consistent global lock order** (always lock A before B) in ALL threads to prevent deadlock.

---

## Key Concepts You Must Know

### What Is a Race Condition?

A race condition occurs when two or more threads access a shared variable **without synchronization**, and at least one access is a write. The final value depends on the unpredictable scheduling order.

```c
// BUGGY - race condition:
resourceA++;        // read-modify-write is NOT atomic
total_ops++;        // another thread could interleave here
```

### What Is a Deadlock?

Deadlock occurs when:
- Thread 1 holds `lockA` and waits for `lockB`
- Thread 2 holds `lockB` and waits for `lockA`
- Both threads wait forever → program hangs

```
Thread worker_AB:  holds lockA → waiting for lockB
Thread worker_BA:  holds lockB → waiting for lockA
                   ↑ DEADLOCK!
```

### The Fix: Consistent Lock Ordering

**Always acquire locks in the same global order: lockA first, then lockB.**

```c
// SAFE - consistent order in ALL threads:
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
// ... access resourceA and resourceB ...
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **💡 Hint:** Even `worker_BA` (which logically updates B then A) must **acquire locks** in the order A → B to avoid deadlock. It can still *update* B first after acquiring both locks.

---

## Step-by-Step Solution

### Step 1: Understand the Original Buggy Structure

The original code looks roughly like this (without locks):

```c
// worker_AB - BUGGY (no locks)
void *worker_AB(void *arg) {
    struct thread_args *a = (struct thread_args *)arg;
    struct thread_result *res = malloc(sizeof(*res));
    int ops = 0;

    while (!done) {
        resourceA += a->increment;   // RACE CONDITION
        resourceB += a->increment;   // RACE CONDITION
        total_ops++;                 // RACE CONDITION
        ops++;
    }

    res->ops = ops;
    res->lastA = resourceA;
    res->lastB = resourceB;
    res->lastTotalOps = total_ops;
    res->lastDone = done;
    return res;
}
```

```c
// worker_BA - BUGGY (no locks)
void *worker_BA(void *arg) {
    struct thread_args *a = (struct thread_args *)arg;
    struct thread_result *res = malloc(sizeof(*res));
    int ops = 0;

    while (!done) {
        resourceB += a->increment;   // RACE CONDITION
        resourceA += a->increment;   // RACE CONDITION
        total_ops++;                 // RACE CONDITION
        ops++;
    }

    res->ops = ops;
    // ...
    return res;
}
```

```c
// worker_monitor - BUGGY (no locks)
void *worker_monitor(void *arg) {
    struct thread_result *res = malloc(sizeof(*res));

    while (!done) {                          // RACE CONDITION on done
        sleep(1);
        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               resourceA, resourceB, total_ops, done);  // RACE CONDITION

        if (total_ops >= 80000) {            // RACE CONDITION
            done = 1;                        // RACE CONDITION
        }
    }
    // ...
    return res;
}
```

---

### Step 2: Apply the Fix — Add All Mutex Lock/Unlock Calls

The corrected `pthread_locks.c` with all locks added:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

// ── Shared global state ──────────────────────────────────────────────────────
int resourceA  = 0;
int resourceB  = 0;
int total_ops  = 0;
int done       = 0;

// ── The two (and only two) mutexes you are allowed to use ────────────────────
pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

// ── Structs for passing arguments and returning results ──────────────────────
struct thread_args {
    int increment;
};

struct thread_result {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
};

// ── worker_AB: updates A then B ──────────────────────────────────────────────
// Lock order: ALWAYS lockA → lockB  (prevents deadlock)
void *worker_AB(void *arg)
{
    struct thread_args  *a   = (struct thread_args *)arg;
    struct thread_result *res = malloc(sizeof(*res));
    int ops = 0;

    int local_done = 0;

    // Read 'done' safely before entering loop
    pthread_mutex_lock(&lockA);
    local_done = done;
    pthread_mutex_unlock(&lockA);

    while (!local_done) {
        // Acquire BOTH locks in consistent order: A first, then B
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceA += a->increment;   // protected by lockA
        resourceB += a->increment;   // protected by lockB
        total_ops++;                 // protected (under both locks is fine)
        ops++;
        local_done = done;           // read done safely while holding lockA

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    // Final snapshot — take both locks for consistent read
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    res->ops          = ops;
    res->lastA        = resourceA;
    res->lastB        = resourceB;
    res->lastTotalOps = total_ops;
    res->lastDone     = done;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    return res;
}

// ── worker_BA: logically updates B then A, but locks A → B for safety ────────
// Lock order: ALWAYS lockA → lockB  (prevents deadlock)
void *worker_BA(void *arg)
{
    struct thread_args  *a   = (struct thread_args *)arg;
    struct thread_result *res = malloc(sizeof(*res));
    int ops = 0;

    int local_done = 0;

    pthread_mutex_lock(&lockA);
    local_done = done;
    pthread_mutex_unlock(&lockA);

    while (!local_done) {
        // IMPORTANT: even though this worker "logically" does B then A,
        // we MUST lock in A → B order to be consistent and avoid deadlock.
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        resourceB += a->increment;   // update B first (logical order kept)
        resourceA += a->increment;   // then A
        total_ops++;
        ops++;
        local_done = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    res->ops          = ops;
    res->lastA        = resourceA;
    res->lastB        = resourceB;
    res->lastTotalOps = total_ops;
    res->lastDone     = done;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    return res;
}

// ── worker_monitor: reads resources, adjusts them, signals done ──────────────
void *worker_monitor(void *arg)
{
    (void)arg;   // monitor may not need thread_args
    struct thread_result *res = malloc(sizeof(*res));

    int local_done = 0;

    pthread_mutex_lock(&lockA);
    local_done = done;
    pthread_mutex_unlock(&lockA);

    while (!local_done) {
        sleep(1);

        // Take BOTH locks (A → B order) for a consistent snapshot
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               resourceA, resourceB, total_ops, done);

        // Signal done when enough operations have been counted
        if (total_ops >= 80000) {
            done = 1;
        }

        local_done = done;

        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
    }

    // Final snapshot
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    res->ops          = 0;           // monitor does not count worker ops
    res->lastA        = resourceA;
    res->lastB        = resourceB;
    res->lastTotalOps = total_ops;
    res->lastDone     = done;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    return res;
}

// ── main ─────────────────────────────────────────────────────────────────────
int main(void)
{
    pthread_t t1, t2, t3;

    struct thread_args args1 = { .increment = 1 };
    struct thread_args args2 = { .increment = 1 };

    pthread_create(&t1, NULL, worker_AB,      &args1);
    pthread_create(&t2, NULL, worker_BA,      &args2);
    pthread_create(&t3, NULL, worker_monitor, NULL);

    struct thread_result *r1, *r2, *r3;
    pthread_join(t1, (void **)&r1);
    pthread_join(t2, (void **)&r2);
    pthread_join(t3, (void **)&r3);

    printf("--- Results (returned to main) ---\n");
    printf("Thread 1 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r1->ops, r1->lastA, r1->lastB, r1->lastTotalOps, r1->lastDone);
    printf("Thread 2 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r2->ops, r2->lastA, r2->lastB, r2->lastTotalOps, r2->lastDone);
    printf("Thread 3 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           r3->ops, r3->lastA, r3->lastB, r3->lastTotalOps, r3->lastDone);

    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    printf("Final Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    printf("Expected: total_ops=80000 done=1\n");
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    free(r1); free(r2); free(r3);
    return 0;
}
```

---

## Explanation of Every Fix

### Fix 1: Protecting `resourceA` and `resourceB`

```c
// BEFORE (buggy):
resourceA += a->increment;
resourceB += a->increment;

// AFTER (fixed):
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
resourceA += a->increment;   // lockA guards resourceA
resourceB += a->increment;   // lockB guards resourceB
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **💡 Hint:** `+=` is a **read-modify-write** operation. It compiles to multiple CPU instructions and is **never atomic** without explicit synchronization.

### Fix 2: Protecting `total_ops`

`total_ops` is written by both worker threads and read by the monitor. It must be protected. Since we already hold both `lockA` and `lockB` when we update the resources, we protect `total_ops` under the same lock region.

```c
// Inside the locked region (safe):
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
resourceA += a->increment;
resourceB += a->increment;
total_ops++;               // ← safe: inside locked region
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

### Fix 3: Protecting `done`

`done` is written by `worker_monitor` and read by all threads as a loop termination condition.

```c
// Monitor writes done safely:
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
if (total_ops >= 80000) {
    done = 1;              // ← safe: inside locked region
}
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);

// Workers read done safely using a local copy:
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
// ... do work ...
local_done = done;         // ← safe snapshot of done
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
// loop condition uses local_done (not the raw shared variable)
```

### Fix 4: Consistent Lock Ordering (Prevents Deadlock)

This is the **most critical fix**:

```
Global lock order rule: ALWAYS acquire lockA before lockB
```

| Thread | Must acquire locks in order |
|--------|-----------------------------|
| `worker_AB` | `lockA` → `lockB` ✅ |
| `worker_BA` | `lockA` → `lockB` ✅ (even though it updates B first!) |
| `worker_monitor` | `lockA` → `lockB` ✅ |

> **⚠️ Warning:** The trap here is thinking `worker_BA` should lock B first because it "updates B then A". **The lock acquisition order is independent of the logical update order.** You can update in any order once you hold both locks.

---

## Deadlock Prevention: Visual Explanation

### Without Consistent Ordering (DEADLOCK possible)

```
Time →

worker_AB:   lock(A) ──────────────────── waiting for B ← BLOCKED
worker_BA:   lock(B) ──────────────────── waiting for A ← BLOCKED
                                          ↑ DEADLOCK
```

### With Consistent Ordering (SAFE)

```
Time →

worker_AB:   lock(A) → lock(B) → work → unlock(B) → unlock(A)
worker_BA:   waiting ──────────→ lock(A) → lock(B) → work → unlock(B) → unlock(A)
```

---

## Building and Testing

### Step 1: Compile

```bash
cd notxv6/
make pthread_locks
# Expected output:
# cc pthread_locks.c -o pthread_locks
```

### Step 2: Run Once

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

### Step 3: Run Optional Test (20 repetitions)

```bash
./optional_test
# Expected: PASS
```

### Step 4: Grade

```bash
cd ..
./grade-quiz pthread_locks
# Expected:
# pthread_locks_test: OK (28.1s)
```

### Step 5: Full Grade Check

```bash
make clean && make grade
# Expected:
# pthread_locks_test: OK (28.1s)
# Score: 1/1
```

### Step 6: Submit

```bash
make zipball
# Submit the generated lab.zip to Gradescope
```

> **⚠️ Warning:** Always use `make zipball` to create your submission ZIP. Do **not** use Windows Explorer zip or macOS Files App — these may create nested directory structures that break the autograder.

---

## Summary Checklist

Use this checklist before submitting:

- [ ] Every read/write of `resourceA` is inside `pthread_mutex_lock(&lockA)` ... `pthread_mutex_unlock(&lockA)`
- [ ] Every read/write of `resourceB` is inside `pthread_mutex_lock(&lockB)` ... `pthread_mutex_unlock(&lockB)`
- [ ] Every read/write of `total_ops` is inside a locked region
- [ ] Every read/write of `done` is inside a locked region
- [ ] **All threads acquire locks in the same order: `lockA` first, then `lockB`**
- [ ] **All threads release locks in the reverse order: `lockB` first, then `lockA`**
- [ ] No new mutexes, semaphores, or condition variables were added
- [ ] Program terminates correctly and prints `total_ops=80000 done=1`
- [ ] `./optional_test` passes (run 20 times, no race or deadlock)
- [ ] `make grade` shows `OK`

---

## Common Mistakes to Avoid

> **⚠️ Warning:** Inconsistent lock ordering is the #1 source of intermittent deadlocks. Your program may run fine most of the time but deadlock occasionally — this is why the test script runs the binary **20 times**.

> **⚠️ Warning:** Do not use the raw `done` variable as a loop condition without holding the lock. Copy it to a local variable inside the locked region and use that.

> **⚠️ Warning:** Do not forget to unlock in the **reverse** order of locking. If you locked A then B, unlock B then A.

> **⚠️ Warning:** Do not add any new synchronization primitives (no new mutexes, no semaphores, no condition variables) — this violates the task constraints and will be marked wrong.
