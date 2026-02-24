---
layout: page
title: "Batch 2026-02-25 — Quiz2 Q2, Question2 Pthread Locks"
date: 2026-02-25
---

# Batch 2026-02-25 — Quiz2 Q2, Question2 Pthread Locks

## File 1: notxv6/pthread_locks.c

### Solution

```c
// pthread_locks.c
//
// Compile: gcc -O2 -Wall -Wextra -pthread pthread_locks.c -o student
// Run: ./ pthread_locks
//
// FIXED VERSION with mutex locks/unlocks added

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

    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    r->lastA = resourceA;
    r->lastB = resourceB;
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);
    
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
        resourceA++;
        pthread_mutex_unlock(&lockA);
        
        pthread_mutex_lock(&lockB);
        resourceB++;
        pthread_mutex_unlock(&lockB);
        
        total_ops += 2;
        ops += 2;

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
        pthread_mutex_lock(&lockB);
        resourceB += 2;
        pthread_mutex_unlock(&lockB);
        
        pthread_mutex_lock(&lockA);
        resourceA += 2;
        pthread_mutex_unlock(&lockA);
        
        total_ops += 2;
        ops += 2;

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
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
        
        long t = total_ops;
        int d = done;

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

**Explanation:** The solution adds mutex lock/unlock calls to protect all shared resource accesses:
- `worker_AB` locks A, increments it, unlocks; then locks B, increments it, unlocks
- `worker_BA` locks B first, then A (opposite order) to avoid deadlock potential
- `worker_monitor` acquires both locks in consistent order (A then B) for safe reads
- `make_result` also acquires locks in order to capture consistent state snapshots
- All reads of `resourceA` and `resourceB` are protected by their respective locks

## File 2: Question2_pthread_locks.pdf

### Quiz Requirements & Testing Guide

#### Task Overview
Fix a buggy pthread program with race conditions by adding **only** `pthread_mutex_lock()` and `pthread_mutex_unlock()` calls using the two provided mutexes: `lockA` and `lockB`.

#### Key Points

> **💡 Hint:** Maintain a **consistent lock ordering** to prevent deadlock: always acquire locks in the same order (e.g., A before B).

> **⚠️ Warning:** If you acquire locks in different orders in different threads (A→B in one, B→A in another), intermittent deadlocks may occur.

#### Critical Sections to Protect

1. **`resourceA` and `resourceB`** — must be protected by `lockA` and `lockB` respectively
2. **`total_ops`** — shared counter updated by multiple threads
3. **`done`** — flag read and written by monitor and main
4. **Monitor snapshots** — must acquire locks in consistent order to read A and B atomically

#### Race Conditions in Original Code

- **resourceA/B access without locks** → multiple threads read/write simultaneously
- **total_ops updates unprotected** → lost writes possible
- **Monitor inconsistent reads** → A and B may be from different update cycles
- **done flag unprotected** → race between monitor write and worker reads

#### Deadlock Prevention Strategy

```
Thread 1 (worker_AB): lock(A) → lock(B) → unlock(B) → unlock(A)
Thread 2 (worker_BA): lock(B) → lock(A) → unlock(A) → unlock(B)
                        ↑ Different order!
```

**Solution:** Use **lock ordering discipline** (always A before B globally):
```
Thread 1: lock(A) → unlock(A) → lock(B) → unlock(B)  ✓
Thread 2: lock(B) → lock(A) → unlock(A) → unlock(B)  ✗ Must be: A first!
```

Modify `worker_BA` to lock in order A→B (acquire B, modify; acquire A, modify).

#### Compilation & Testing

```bash
cd notxv6/
make pthread_locks
./pthread_locks
```

**Expected output:**
- Monitor prints snapshots with consistent A==B values
- Final: `total_ops=80000 done=1`
- Exit code: 0

**Run optional test (catches intermittent issues):**
```bash
./optional_test  # Runs 20 iterations
```

**Grade verification:**
```bash
cd ..
./grade-quiz pthread_locks
```

> **⚠️ Warning:** The grader runs the binary 20 times to detect intermittent deadlocks. Incorrect lock ordering will fail ~20% of runs.

#### Common Mistakes

| Mistake | Problem |
|---------|---------|
| Forgetting lock/unlock on `total_ops` | Lost updates, wrong final count |
| Not protecting monitor reads | Inconsistent snapshots (A≠B) |
| Inconsistent lock order | Intermittent deadlock (A→B vs B→A) |
| Holding locks during `delay()` | Unnecessary contention, timeouts |
| Using locks in wrong scope | Locks released too early/late |
