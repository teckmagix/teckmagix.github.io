---
layout: page
title: "Batch 2026-02-24 — Question2 Pthread Locks, Quiz2 Q2"
date: 2026-02-24
---

# Batch 2026-02-24 — Question2 Pthread Locks, Quiz2 Q2

## File 1: Question2_pthread_locks.pdf

### Task: Add Missing Mutex Lock/Unlock Calls to `notxv6/pthread_locks.c`

The key insight is to use a **consistent global lock ordering** (always acquire `lockA` before `lockB`) to prevent deadlock, and protect all shared variables (`resourceA`, `resourceB`, `total_ops`, `done`) with the appropriate mutex.

> **💡 Hint:** Always acquire locks in the same order (lockA → lockB) across ALL threads to prevent deadlock. The monitor needs BOTH locks held simultaneously for a consistent snapshot.

> **⚠️ Warning:** If `worker_BA` acquires `lockB` before `lockA` while `worker_AB` acquires `lockA` before `lockB`, you get a classic deadlock. Fix by enforcing the same order in both workers.

```c
// pthread_locks.c — FIXED VERSION
// Compile: gcc -O2 -Wall -Wextra -pthread pthread_locks.c -o pthread_locks

#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// -------------------- Shared Global Resources --------------------

static int resourceA = 0;
static int resourceB = 0;

static long total_ops = 0;
static int done = 0;

// Provided mutexes (must use ONLY these)
static pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
static pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

#define LOOP_COUNT 20000

// -------------------- Thread Argument / Result Types --------------------

typedef struct {
    int thread_id;
} thread_args_t;

typedef struct {
    int thread_id;
    long ops_done;
    int lastA;
    int lastB;
    long lastTotalOps;
    int lastDone;
} thread_result_t;

static void delay(void)
{
    usleep(1);
}

static thread_result_t* make_result(int id, long ops)
{
    thread_result_t* r = malloc(sizeof(thread_result_t));
    if (!r) { perror("malloc"); exit(1); }

    r->thread_id = id;
    r->ops_done = ops;

    // Take a consistent snapshot: acquire both locks in order
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    r->lastA = resourceA;
    r->lastB = resourceB;
    r->lastTotalOps = total_ops;
    r->lastDone = done;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    return r;
}

// -------------------- Worker 1: touches A then B --------------------

void* worker_AB(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT; i++) {
        // Check done WITHOUT lock first (safe early exit check)
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);
        if (done) {
            pthread_mutex_unlock(&lockB);
            pthread_mutex_unlock(&lockA);
            break;
        }
        resourceA++;
        resourceB++;
        total_ops += 2;
        ops += 2;
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        delay();
    }

    return make_result(p->thread_id, ops);
}

// -------------------- Worker 2: touches B then A (logic order) --------------------
// NOTE: We still acquire lockA FIRST then lockB to avoid deadlock!

void* worker_BA(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT; i++) {
        // Acquire in SAME order as worker_AB: lockA first, then lockB
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);
        if (done) {
            pthread_mutex_unlock(&lockB);
            pthread_mutex_unlock(&lockA);
            break;
        }
        resourceB += 2;
        resourceA += 2;
        total_ops += 2;
        ops += 2;
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        delay();
    }

    return make_result(p->thread_id, ops);
}

// -------------------- Worker 3: monitor --------------------

void* worker_monitor(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    while (1) {
        // Take consistent snapshot
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);
        int a = resourceA;
        int b = resourceB;
        long t = total_ops;
        int d = done;
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        if ((t % 20000) == 0) {
            printf("[monitor] A=%d B=%d total_ops=%ld done=%d\n", a, b, t, d);
        }

        if (d) break;

        delay();
    }

    return make_result(p->thread_id, ops);
}

// -------------------- Main --------------------

int main(void)
{
    pthread_t t1, t2, t3;

    thread_args_t a1 = { 1 };
    thread_args_t a2 = { 2 };
    thread_args_t a3 = { 3 };

    pthread_create(&t1, NULL, worker_AB, &a1);
    pthread_create(&t2, NULL, worker_BA, &a2);
    pthread_create(&t3, NULL, worker_monitor, &a3);

    thread_result_t *r1 = NULL, *r2 = NULL, *r3 = NULL;

    // Join workers first
    pthread_join(t1, (void**)&r1);
    pthread_join(t2, (void**)&r2);

    // Signal done under lock
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    done = 1;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);

    // Now join monitor
    pthread_join(t3, (void**)&r3);

    const long expected_total_ops = 4L * LOOP_COUNT;

    printf("\n--- Results (returned to main) ---\n");
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           r1->thread_id, r1->ops_done, r1->lastA, r1->lastB, r1->lastTotalOps, r1->lastDone);
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           r2->thread_id, r2->ops_done, r2->lastA, r2->lastB, r2->lastTotalOps, r2->lastDone);
    printf("Thread %d ops=%ld lastA=%d lastB=%d lastTotalOps=%ld lastDone=%d\n",
           r3->thread_id, r3->ops_done, r3->lastA, r3->lastB, r3->lastTotalOps, r3->lastDone);

    printf("\nFinal Shared: A=%d B=%d total_ops=%ld done=%d\n",
           resourceA, resourceB, total_ops, done);
    printf("Expected: total_ops=%ld done=1\n", expected_total_ops);

    free(r1);
    free(r2);
    free(r3);

    return (total_ops == expected_total_ops && done == 1) ? 0 : 1;
}
```

**Explanation of all changes made:**

| Location | Change | Why |
|---|---|---|
| `worker_AB` loop body | Wrap entire read-modify-write with `lock(lockA) → lock(lockB)` ... `unlock(lockB) → unlock(lockA)` | Prevents data race on `resourceA`, `resourceB`, `total_ops`, `done` |
| `worker_BA` loop body | Same lock order: `lockA` first, then `lockB` (even though it updates B before A logically) | **Critical**: same acquisition order prevents deadlock |
| `worker_monitor` loop | Lock both before reading snapshot, unlock after | Guarantees consistent read of all shared vars |
| `make_result()` | Lock both before reading snapshot | Consistent final snapshot in result struct |
| `main()` — setting `done = 1` | Wrap `done = 1` with lockA+lockB | Prevents race on `done` |

---

## File 2: quiz2-q2.zip

### Key Files Reference Guide

> **💡 Hint:** The zip contains the full xv6 lab skeleton. The only file you need to modify for Quiz 2 Q2 is `notxv6/pthread_locks.c`. All other files (kernel/, user/, gradelib.py) are read-only infrastructure.

#### How to Build and Test

```bash
# Step 1: Go into the notxv6 directory and compile
cd notxv6/
make pthread_locks
# Output: cc pthread_locks.c -o pthread_locks

# Step 2: Run once manually
./pthread_locks
# Expected output (values may vary slightly):
# [monitor] A=29943 B=29943 total_ops=40000 done=0
# [monitor] A=60000 B=60000 total_ops=80000 done=0
# [monitor] A=60000 B=60000 total_ops=80000 done=1
# --- Results (returned to main) ---
# Thread 1 ops=40000 ...
# Thread 2 ops=40000 ...
# Thread 3 ops=0 ...
# Final Shared: A=60000 B=60000 total_ops=80000 done=1
# Expected: total_ops=80000 done=1

# Step 3: Run the optional stress test (20 iterations)
./optional_test
# Expected: PASS

# Step 4: Run the grader from the parent directory
cd ..
./grade-quiz pthread_locks
# Expected: pthread_locks_test: OK (28.1s)

# Step 5: Full grade
make clean && make grade
# Expected: Score: 1/1

# Step 6: Create submission zip
make zipball
# Submit lab.zip to Gradescope: xv6labs-quiz2-q2-pthread
```

> **⚠️ Warning:** Use `make zipball` to create your submission — do NOT use Windows zip or macOS Files app, as they create nested directory structures that break the autograder.

#### Concurrency Concepts Summary (for the theory behind the fix)

| Concept | Definition | Example in this lab |
|---|---|---|
| **Race Condition** | Two threads access shared data concurrently, at least one writes, without synchronization | `resourceA++` in both `worker_AB` and `worker_BA` simultaneously |
| **Mutex Lock** | Mutual exclusion primitive — only one thread holds it at a time | `pthread_mutex_lock(&lockA)` |
| **Deadlock** | Two threads each hold a lock the other needs, both wait forever | Thread1 holds lockA wants lockB; Thread2 holds lockB wants lockA |
| **Lock Ordering** | Acquiring multiple locks always in the same global order prevents deadlock | Always acquire lockA before lockB in every thread |
| **Consistent Snapshot** | Reading multiple related variables while holding all relevant locks | Hold lockA+lockB while reading resourceA, resourceB, total_ops, done together |
| **Critical Section** | Code that accesses shared resources — must be protected | The `resourceA++; resourceB++; total_ops += 2;` block |

#### Why the Original Code Was Buggy

```
ORIGINAL worker_AB:          ORIGINAL worker_BA:
  resourceA++    ← RACE        resourceB += 2   ← RACE
  resourceB++    ← RACE        resourceA += 2   ← RACE
  total_ops += 2 ← RACE        total_ops += 2   ← RACE

POTENTIAL DEADLOCK (if locks added naively):
  worker_AB: lock(A) → wants lock(B)
  worker_BA: lock(B) → wants lock(A)
  → DEADLOCK!
```

#### Fix Summary (3 Rules)

1. **Always lock before touching any shared variable** (`resourceA`, `resourceB`, `total_ops`, `done`)
2. **Always acquire in the same order**: `lockA` then `lockB` — even in `worker_BA` which logically updates B first
3. **Hold both locks for any consistent multi-variable read** (monitor snapshots, `make_result`)
