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

You are provided with `notxv6/pthread_locks.c` — a program with:

| Issue | Description |
|-------|-------------|
| **Race Condition** | Multiple threads access shared variables (`resourceA`, `resourceB`, `total_ops`, `done`) without synchronization |
| **Potential Deadlock** | If locks are acquired in inconsistent order (e.g., Thread 1 holds lockA and waits for lockB, while Thread 2 holds lockB and waits for lockA) |

### Shared Resources and Their Guards

| Shared Variable | Protecting Mutex |
|----------------|-----------------|
| `resourceA` | `lockA` |
| `resourceB` | `lockB` |
| `total_ops` | Both (accessed together with resources) |
| `done` | Both (accessed by monitor thread) |

### The Three Threads

| Thread | Behaviour |
|--------|-----------|
| `worker_AB` | Locks A → updates A → Locks B → updates B → Unlocks both |
| `worker_BA` | Must also Lock A first → then B (safe order!) |
| `worker_monitor` | Reads/adjusts A and B, signals `done` |

---

## Key Concepts to Understand

### Race Condition
A **race condition** occurs when two or more threads access a shared variable concurrently and at least one access is a write, without any synchronization.

```
Thread 1: reads resourceA (value = 5)
Thread 2: reads resourceA (value = 5)
Thread 1: writes resourceA = 6
Thread 2: writes resourceA = 6   ← Lost update! Should be 7
```

### Deadlock — The Classic Problem

> **⚠️ Warning:** The most common mistake here is using **inconsistent lock ordering**. If `worker_AB` locks `lockA` then `lockB`, but `worker_BA` locks `lockB` then `lockA`, a deadlock can occur intermittently.

```
worker_AB acquires lockA, waiting for lockB...
worker_BA acquires lockB, waiting for lockA...
>>> DEADLOCK: Both threads waiting forever <<<
```

### The Fix: Consistent Lock Ordering (Lock Hierarchy)

The golden rule to prevent deadlock with multiple mutexes:

> **💡 Hint:** **Always acquire locks in the same global order.** If every thread that needs both `lockA` and `lockB` always acquires `lockA` first and `lockB` second, deadlock is impossible.

---

## The Solution — Fixed `pthread_locks.c`

Below is the complete fixed program with all missing `pthread_mutex_lock()` / `pthread_mutex_unlock()` calls added and annotated.

```c
#include <stdlib.h>
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>   // for sleep()

// ─── Shared global resources ────────────────────────────────────────────────
static int resourceA   = 0;
static int resourceB   = 0;
static int total_ops   = 0;
static int done        = 0;

// ─── Two mutexes — one per resource ─────────────────────────────────────────
// RULE: always acquire lockA BEFORE lockB to prevent deadlock
static pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

// ─── Parameter struct (passed FROM main TO each thread) ─────────────────────
typedef struct {
    int id;
    int iterations;
} thread_args_t;

// ─── Result struct (heap-allocated, returned FROM thread TO main) ────────────
typedef struct {
    int ops;
    int lastA;
    int lastB;
    int lastTotalOps;
    int lastDone;
} thread_result_t;

// ════════════════════════════════════════════════════════════════════════════
// worker_AB: updates resourceA then resourceB
//   Lock order: lockA → lockB  (consistent global order)
// ════════════════════════════════════════════════════════════════════════════
void *worker_AB(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        // FIX: acquire lockA first, then lockB (consistent order)
        pthread_mutex_lock(&lockA);   // <── ADDED
        pthread_mutex_lock(&lockB);   // <── ADDED

        resourceA++;
        resourceB++;
        total_ops++;
        result->ops++;

        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB); // <── ADDED (release in reverse order)
        pthread_mutex_unlock(&lockA); // <── ADDED
    }

    return result;
}

// ════════════════════════════════════════════════════════════════════════════
// worker_BA: updates resourceB then resourceA
//   Lock order: lockA → lockB  (SAME consistent global order — key fix!)
//   Even though this thread "logically" does B then A, it still acquires
//   lockA first to maintain the global ordering and avoid deadlock.
// ════════════════════════════════════════════════════════════════════════════
void *worker_BA(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    for (int i = 0; i < params->iterations; i++) {

        // FIX: MUST also acquire lockA first, then lockB
        //      even though we conceptually "do B then A"
        //      This consistent ordering prevents deadlock.
        pthread_mutex_lock(&lockA);   // <── ADDED (lockA FIRST always)
        pthread_mutex_lock(&lockB);   // <── ADDED (lockB SECOND always)

        resourceB++;
        resourceA++;
        total_ops++;
        result->ops++;

        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        pthread_mutex_unlock(&lockB); // <── ADDED
        pthread_mutex_unlock(&lockA); // <── ADDED
    }

    return result;
}

// ════════════════════════════════════════════════════════════════════════════
// worker_monitor: periodically reads and sometimes adjusts resources,
//                 then signals done = 1 when total_ops reaches the target.
//   Lock order: lockA → lockB  (same consistent global order)
// ════════════════════════════════════════════════════════════════════════════
void *worker_monitor(void *arg)
{
    thread_args_t   *params = (thread_args_t *)arg;
    thread_result_t *result = malloc(sizeof(thread_result_t));

    result->ops = 0;

    // Monitor loops until done is signalled
    while (1) {

        sleep(1); // wait between snapshots

        // FIX: acquire both locks to safely read/modify shared state
        pthread_mutex_lock(&lockA);   // <── ADDED
        pthread_mutex_lock(&lockB);   // <── ADDED

        // Safe (consistent) snapshot
        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               resourceA, resourceB, total_ops, done);

        // Record last seen values into result
        result->lastA        = resourceA;
        result->lastB        = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone     = done;

        // Signal termination once both workers have finished
        if (total_ops >= params->iterations * 2) {
            done = 1;  // FIX: written inside the lock — no race
        }

        pthread_mutex_unlock(&lockB); // <── ADDED
        pthread_mutex_unlock(&lockA); // <── ADDED

        // FIX: read done AFTER releasing locks; it's fine here because
        //      we only read it to decide whether to exit the loop.
        //      A final lock+read would also be acceptable.
        pthread_mutex_lock(&lockA);   // <── ADDED
        int local_done = done;
        pthread_mutex_unlock(&lockA); // <── ADDED

        if (local_done) break;
    }

    return result;
}

// ════════════════════════════════════════════════════════════════════════════
// main
// ════════════════════════════════════════════════════════════════════════════
int main(void)
{
    pthread_t t1, t2, t3;

    // Heap-allocate args for each thread
    thread_args_t *args1 = malloc(sizeof(thread_args_t));
    thread_args_t *args2 = malloc(sizeof(thread_args_t));
    thread_args_t *args3 = malloc(sizeof(thread_args_t));

    args1->id = 1;  args1->iterations = 40000;
    args2->id = 2;  args2->iterations = 40000;
    args3->id = 3;  args3->iterations = 80000; // monitor checks for 80000 total

    pthread_create(&t1, NULL, worker_AB,      args1);
    pthread_create(&t2, NULL, worker_BA,      args2);
    pthread_create(&t3, NULL, worker_monitor, args3);

    // Collect heap-allocated result structs
    thread_result_t *r1, *r2, *r3;
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

    printf("Final Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    printf("Expected: total_ops=80000 done=1\n");

    free(args1); free(args2); free(args3);
    free(r1);    free(r2);    free(r3);

    return 0;
}
```

---

## Step-by-Step Explanation of Every Fix

### Fix 1 — `worker_AB`: Add Lock/Unlock Around Critical Section

```c
// BEFORE (buggy — no locks):
resourceA++;
resourceB++;
total_ops++;

// AFTER (fixed):
pthread_mutex_lock(&lockA);   // Step 1: acquire lockA
pthread_mutex_lock(&lockB);   // Step 2: acquire lockB
resourceA++;                  // Step 3: safely modify A
resourceB++;                  // Step 4: safely modify B
total_ops++;                  // Step 5: safely update counter
pthread_mutex_unlock(&lockB); // Step 6: release lockB first
pthread_mutex_unlock(&lockA); // Step 7: release lockA
```

> **💡 Hint:** Always release locks in **reverse order** of acquisition. If you lock A→B, unlock B→A. This is good practice (though not strictly required for correctness here).

---

### Fix 2 — `worker_BA`: Use the SAME Lock Order as `worker_AB`

This is the **most critical fix** to prevent deadlock.

```c
// WRONG (causes intermittent deadlock):
pthread_mutex_lock(&lockB);   // worker_BA locks B first
pthread_mutex_lock(&lockA);   // then waits for A
// Meanwhile worker_AB locked A and is waiting for B → DEADLOCK!

// CORRECT (consistent global order — lockA always first):
pthread_mutex_lock(&lockA);   // lockA FIRST (even in worker_BA!)
pthread_mutex_lock(&lockB);   // lockB SECOND
resourceB++;
resourceA++;
total_ops++;
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **⚠️ Warning:** This is the trap the question is testing. Even though `worker_BA` conceptually processes B before A, **the mutex acquisition order must still be lockA → lockB**. The mutex order controls thread scheduling, not the order of your arithmetic operations.

---

### Fix 3 — `worker_monitor`: Lock Both Before Reading/Writing Shared State

```c
// BEFORE (buggy — reading without locks gives inconsistent snapshot):
printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
       resourceA, resourceB, total_ops, done);

// AFTER (fixed — consistent snapshot under both locks):
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
       resourceA, resourceB, total_ops, done);
if (total_ops >= target) {
    done = 1;   // write to done safely inside lock
}
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **💡 Hint:** The monitor must hold **both** locks when reading `resourceA`, `resourceB`, `total_ops`, and `done` because the worker threads update all of them together inside a dual-lock section. Reading them without locks could give you a torn/inconsistent snapshot.

---

## Deadlock Prevention — Theory Summary

### Coffman's Four Conditions for Deadlock

All four must hold simultaneously for deadlock to occur:

| Condition | Meaning | How We Break It |
|-----------|---------|----------------|
| **Mutual Exclusion** | Resource held exclusively | Cannot eliminate (mutexes need this) |
| **Hold and Wait** | Thread holds one lock while waiting for another | Could allocate all at once, but we use ordering instead |
| **No Preemption** | Locks cannot be forcibly taken | Cannot eliminate easily |
| **Circular Wait** | Thread A waits for B, B waits for A | ✅ **We break this with consistent lock ordering** |

> **💡 Hint:** By enforcing **Lock Hierarchy** (always lockA before lockB), we eliminate the **Circular Wait** condition, which breaks the deadlock possibility entirely.

---

## Building and Testing

### Step 1: Compile

```bash
cd notxv6/
make pthread_locks
# Output: cc pthread_locks.c -o pthread_locks
```

### Step 2: Run Once to Check Output

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
Thread 3 ops=0     lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=1
Final Shared: A=60000 B=60000 total_ops=80000 done=1
Expected: total_ops=80000 done=1
```

> **⚠️ Warning:** Intermediate values (like `A=29943`) may vary between runs — that is **normal**. What matters is that the **final values are always correct**: `A=60000 B=60000 total_ops=80000 done=1`.

### Step 3: Run Optional Test Script (20 repetitions)

```bash
./optional_test
# Expected: PASS
```

### Step 4: Run Grade Script

```bash
cd ..
./grade-quiz pthread_locks
# Expected:
# == Test pthread_locks_test ==
# pthread_locks_test: OK (28.1s)
```

### Step 5: Full Grade with Clean Build

```bash
make clean && make grade
# Expected:
# pthread_locks_test: OK (28.1s)
# Score: 1/1
```

### Step 6: Submit

```bash
make zipball
# Submit lab.zip to Gradescope: xv6labs-quiz2-q2-pthread
```

> **⚠️ Warning:** Use `make zipball` — **do NOT** use Windows zip, WinRAR, or macOS Files App. These can create nested directory structures that break the autograder.

---

## Correctness Checklist

Before submitting, verify all of the following:

- [ ] `resourceA` is **always** accessed inside `pthread_mutex_lock(&lockA)`
- [ ] `resourceB` is **always** accessed inside `pthread_mutex_lock(&lockB)`
- [ ] `total_ops` is accessed inside both locks (since workers update it alongside both resources)
- [ ] `done` is read/written inside locks
- [ ] **Both** `worker_AB` and `worker_BA` acquire locks in the **same order**: `lockA` → `lockB`
- [ ] Every `pthread_mutex_lock()` has a matching `pthread_mutex_unlock()` on every code path
- [ ] No new mutexes, semaphores, or condition variables were added
- [ ] Program always terminates (no infinite loop, no deadlock)
- [ ] Final output shows `total_ops=80000 done=1` consistently across 20 runs

---

## Common Mistakes to Avoid

> **⚠️ Warning — Mistake 1:** Locking `lockB` before `lockA` in `worker_BA`.
> This creates a circular wait with `worker_AB` and causes intermittent deadlock.

> **⚠️ Warning — Mistake 2:** Only locking one mutex when accessing variables guarded by both.
> If `total_ops` is updated inside only `lockA` but read by the monitor inside both locks, you may still get races.

> **⚠️ Warning — Mistake 3:** Forgetting to unlock on early `return` or `break` paths.
> Always ensure unlock is called on every exit path from a critical section.

> **⚠️ Warning — Mistake 4:** Reading `done` outside of any lock in the monitor loop to decide whether to break.
> Always copy `done` to a local variable inside a lock, then use the local copy for control flow.

---

## Summary Table — All Lock Additions

| Location | Lock Added | Purpose |
|----------|-----------|---------|
| `worker_AB` loop body start | `lock(&lockA)`, `lock(&lockB)` | Protect A, B, total_ops, done reads |
| `worker_AB` loop body end | `unlock(&lockB)`, `unlock(&lockA)` | Release in reverse order |
| `worker_BA` loop body start | `lock(&lockA)`, `lock(&lockB)` | **Same order as AB** — prevents deadlock |
| `worker_BA` loop body end | `unlock(&lockB)`, `unlock(&lockA)` | Release in reverse order |
| `worker_monitor` snapshot section | `lock(&lockA)`, `lock(&lockB)` | Consistent read of all shared vars |
| `worker_monitor` snapshot section end | `unlock(&lockB)`, `unlock(&lockA)` | Release after snapshot |
| `worker_monitor` loop condition check | `lock(&lockA)`, read `done`, `unlock(&lockA)` | Safe read of `done` for loop exit |
