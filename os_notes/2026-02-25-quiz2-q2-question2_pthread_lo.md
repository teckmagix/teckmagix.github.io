---
layout: page
title: "Batch 2026-02-25 — Quiz2 Q2, Question2 Pthread Locks"
date: 2026-02-25
---

# Batch 2026-02-25 — Quiz2 Q2, Question2 Pthread Locks

## File 1: pthread_locks.c (notxv6/pthread_locks.c)

```c
// pthread_locks.c
//
// Compile: gcc -O2 -Wall -Wextra -pthread pthread_locks.c -o student
// Run: ./pthread_locks
//
// - Deterministic end state
//     done == 1
//     total_ops == 4 * LOOP_COUNT
//
// To identify and ADD pthread_mutex_lock()/pthread_mutex_unlock() calls ONLY.
// Do not change the algorithm.

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
    // Fixed micro delay to encourage scheduling interleavings
    usleep(1);
}

static thread_result_t* make_result(int id, long ops)
{
    thread_result_t* r = malloc(sizeof(thread_result_t));
    if (!r) { perror("malloc"); exit(1); }

    r->thread_id = id;
    r->ops_done = ops;

    r->lastA = resourceA;
    r->lastB = resourceB;
    r->lastTotalOps = total_ops;
    r->lastDone = done;

    return r;
}


// -------------------- Worker 1: touches A then B --------------------

void* worker_AB(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT && !done; i++) {

        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

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

void* worker_BA(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT && !done; i++) {

        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);

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


// -------------------- Worker 3: monitor (READ-ONLY) --------------------

void* worker_monitor(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    while (!done) {
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

    done = 1;

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

    // Exit 0 only if deterministic expectations are met
    return (total_ops == expected_total_ops && done == 1) ? 0 : 1;
}
```

> **💡 Hint:** The key fix is to use **consistent lock ordering** — always acquire `lockA` before `lockB` in ALL threads (including monitor). This prevents deadlock even though `worker_BA` logically touches B then A; the lock acquisition order must still be A→B everywhere.

> **⚠️ Warning:** The original `worker_BA` name suggests acquiring `lockB` first then `lockA`, but doing so would create a **deadlock** when `worker_AB` holds `lockA` and waits for `lockB` while `worker_BA` holds `lockB` and waits for `lockA`. Always lock in the same order (lockA → lockB) regardless of the logical update order.

---

## File 2: Question2_pthread_locks.pdf

### Key Concepts Study Guide

**What was broken (race conditions):**
- `resourceA`, `resourceB`, `total_ops`, `done` were all read/written by multiple threads without any locks → undefined behavior / non-deterministic results.

**The Fix — Three rules applied:**

| Rule | Detail |
|------|--------|
| **Consistent lock order** | Always acquire `lockA` → `lockB` in every thread |
| **Lock before read/write** | All accesses to shared vars inside lock/unlock pair |
| **Unlock in reverse order** | Release `lockB` first, then `lockA` |

**Changes made to each worker:**

- `worker_AB`: Added `lock(A)` → `lock(B)` before the three increments, `unlock(B)` → `unlock(A)` after.
- `worker_BA`: Same lock order (`lockA` then `lockB`) even though the *data* updates B before A — the **lock acquisition order** is what matters for deadlock prevention, not the update order.
- `worker_monitor`: Added `lock(A)` → `lock(B)` before reading all four shared variables, `unlock(B)` → `unlock(A)` after the snapshot. This ensures a consistent atomic snapshot.

**Expected final state:**
```
total_ops == 4 * LOOP_COUNT  (== 80000)
done == 1
```

> **💡 Hint:** `total_ops` gets +2 per iteration from each of the 2 worker threads, and each runs `LOOP_COUNT` (20000) iterations → `2 × 2 × 20000 = 80000`.

> **⚠️ Warning:** Do NOT acquire `lockB` before `lockA` in any thread — even if you're only accessing `resourceB`. Breaking consistent ordering causes intermittent deadlocks that are hard to reproduce but will be caught by the grader running the binary 20 times.
