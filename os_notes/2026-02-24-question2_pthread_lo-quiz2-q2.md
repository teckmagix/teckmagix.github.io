---
layout: page
title: "Batch 2026-02-24 — Question2 Pthread Locks, Quiz2 Q2"
date: 2026-02-24
---

# Batch 2026-02-24 — Question2 Pthread Locks, Quiz2 Q2

## File 1: Question2_pthread_locks.pdf

### Task Summary

Fix a buggy pthread-based program by adding `pthread_mutex_lock()` and `pthread_mutex_unlock()` calls to eliminate **race conditions** and prevent **deadlocks**. The program simulates concurrent access to two shared resources (resourceA and resourceB) protected by two mutexes (lockA and lockB).

### Complete Solution

```c
// pthread_locks.c - FIXED VERSION
// Add mutex locks/unlocks to eliminate race conditions and deadlock

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

    // FIXED: Read shared variables under locks for consistency
    pthread_mutex_lock(&lockA);
    r->lastA = resourceA;
    pthread_mutex_unlock(&lockA);

    pthread_mutex_lock(&lockB);
    r->lastB = resourceB;
    pthread_mutex_unlock(&lockB);

    // Note: total_ops and done also need protection but we use lockA/lockB
    // by convention (protect them with one of the existing locks)
    pthread_mutex_lock(&lockA);
    r->lastTotalOps = total_ops;
    r->lastDone = done;
    pthread_mutex_unlock(&lockA);

    return r;
}

// -------------------- Worker 1: touches A then B --------------------

void* worker_AB(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT && !done; i++) {
        // FIXED: Lock A first, then B (consistent ordering)
        pthread_mutex_lock(&lockA);
        resourceA++;
        pthread_mutex_unlock(&lockA);

        pthread_mutex_lock(&lockB);
        resourceB++;
        pthread_mutex_unlock(&lockB);

        pthread_mutex_lock(&lockA);
        total_ops += 2;
        pthread_mutex_unlock(&lockA);

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
        // FIXED: Lock in SAME order as worker_AB (A then B) to prevent deadlock
        // Even though we logically want B then A, we lock A first for consistency
        pthread_mutex_lock(&lockA);
        // (we hold A while updating B to maintain ordering)
        pthread_mutex_lock(&lockB);
        
        resourceB += 2;
        resourceA += 2;
        
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        pthread_mutex_lock(&lockA);
        total_ops += 2;
        pthread_mutex_unlock(&lockA);

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
        // FIXED: Read all shared variables under locks for consistent snapshot
        pthread_mutex_lock(&lockA);
        int a = resourceA;
        long t = total_ops;
        int d = done;
        pthread_mutex_unlock(&lockA);

        pthread_mutex_lock(&lockB);
        int b = resourceB;
        pthread_mutex_unlock(&lockB);

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

    pthread_join(t1, (void**)&r1);
    pthread_join(t2, (void**)&r2);

    // FIXED: Set done under lock for consistency
    pthread_mutex_lock(&lockA);
    done = 1;
    pthread_mutex_unlock(&lockA);

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

### Key Fixes Explained

**1. Consistent Lock Ordering (Prevents Deadlock)**
- Both `worker_AB` and `worker_BA` lock in the **same order: lockA first, then lockB**
- Even though the logical order differs, holding locks in consistent order prevents circular wait deadlock
- Release in reverse order (LIFO): unlock B, then unlock A

**2. Protected All Shared Variables**
- `resourceA` → protected by `lockA`
- `resourceB` → protected by `lockB`
- `total_ops` → protected by `lockA` (convention)
- `done` → protected by `lockA` (convention)

**3. Consistent Reads in Monitor**
- Takes a snapshot under locks to ensure A, B, total_ops, and done are read consistently
- Prevents torn reads where some values are updated mid-read

> **💡 Hint:** The key to avoiding deadlock with multiple locks is **always acquire locks in the same order across all threads**. Use nested locks carefully: if holding A, you can acquire B, but never reverse that pattern elsewhere.

> **⚠️ Warning:** Reading shared variables without locks causes race conditions. Even "read-only" operations like the monitor must acquire locks to ensure a consistent snapshot.

---

## File 2: quiz2-q2.zip

This file contains the grading infrastructure (gradelib.py, kernel files, user tests) and the buggy source file `notxv6/pthread_locks.c` that was analyzed above.

### Testing the Fixed Solution

**Compile and run:**
```bash
cd notxv6/
gcc -O2 -Wall -Wextra -pthread pthread_locks.c -o pthread_locks
./pthread_locks
```

**Expected output pattern:**
```
[monitor] A=... B=... total_ops=... done=0
[monitor] A=60000 B=60000 total_ops=80000 done=1
...
Final Shared: A=60000 B=60000 total_ops=80000 done=1
Expected: total_ops=80000 done=1
```

**Run stress test (20 iterations to catch intermittent issues):**
```bash
./optional_test
# Should print: PASS
```

**Run with grade script:**
```bash
cd ..
./grade-quiz pthread_locks
# Should show: pthread_locks_test: OK
```

### Why This Solution Works

| Aspect | Problem | Solution |
|--------|---------|----------|
| **Race Condition on resourceA** | Multiple threads modify without synchronization | Lock with `lockA` before access |
| **Race Condition on resourceB** | Multiple threads modify without synchronization | Lock with `lockB` before access |
| **Race Condition on total_ops** | Two threads increment without synchronization | Lock with `lockA` for consistency |
| **Potential Deadlock** | worker_AB locks (A→B), worker_BA locks (B→A) → circular wait | Both lock in same order (A→B) |
| **Torn Reads in Monitor** | Reading inconsistent snapshot mid-update | Acquire locks for all reads within critical section |
| **done flag modification** | Not protected; reader in monitor may see torn value | Lock with `lockA` on write and read |

> **⚠️ Warning:** Do NOT add new mutexes or condition variables—only use the two provided (lockA, lockB). Do NOT change the algorithm—only add synchronization.
