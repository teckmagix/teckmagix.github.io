---
layout: page
title: "Batch 2026-02-24 — Quiz2 Q2, Question2 Pthread Locks"
date: 2026-02-24
---

# Batch 2026-02-24 — Quiz2 Q2, Question2 Pthread Locks

## File 1: notxv6/pthread_locks.c

### Solution

```c
// pthread_locks.c
//
// Compile: gcc -O2 -Wall -Wextra -pthread pthread_locks.c -o student
// Run: ./ pthread_locks
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

    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    thread_result_t* result = make_result(p->thread_id, ops);
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);
    
    return result;
}

// -------------------- Worker 2: touches B then A (logic order) --------------------

void* worker_BA(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT && !done; i++) {
        pthread_mutex_lock(&lockB);
        pthread_mutex_lock(&lockA);
        
        resourceB += 2;
        resourceA += 2;
        total_ops += 2;
        ops += 2;
        
        pthread_mutex_unlock(&lockA);
        pthread_mutex_unlock(&lockB);

        delay();
    }

    pthread_mutex_lock(&lockB);
    pthread_mutex_lock(&lockA);
    thread_result_t* result = make_result(p->thread_id, ops);
    pthread_mutex_unlock(&lockA);
    pthread_mutex_unlock(&lockB);
    
    return result;
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
        
        if ((t % 20000) == 0) {
            printf("[monitor] A=%d B=%d total_ops=%ld done=%d\n", a, b, t, d);
        }
        
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

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

**Explanation:** The solution adds mutex locks to protect all accesses to shared global variables (`resourceA`, `resourceB`, `total_ops`, `done`). Critical sections are protected by acquiring both locks in a consistent order (A before B) to prevent deadlock. The monitor thread also uses the same lock order when reading values to ensure consistent snapshots.

---

## File 2: Question2_pthread_locks.pdf

### Study Guide: Pthread Locks & Synchronization

#### Core Concepts

**Mutexes (Mutual Exclusion)**
- Protect shared resources from concurrent access
- `pthread_mutex_lock()` acquires the lock (blocks if held)
- `pthread_mutex_unlock()` releases the lock
- Only one thread can hold a mutex at a time

**Race Conditions**
- Occur when multiple threads access shared data without synchronization
- Results are unpredictable and depend on thread scheduling
- Example: `resourceA++` is not atomic (read-modify-write)

**Deadlock**
- Occurs when threads hold locks and wait for other locks in circular dependency
- Thread 1: locks A, waits for B
- Thread 2: locks B, waits for A
- **Prevention:** Always acquire locks in the same consistent order

**Lock Ordering Discipline**
- Acquire locks in a fixed order (e.g., A before B)
- Release locks in reverse order (B before A)
- All threads must follow the same discipline

#### Critical Sections

A critical section is a code block accessing shared data that must execute atomously:

```c
pthread_mutex_lock(&lockA);
// Protected code: read/modify shared data
resourceA++;
pthread_mutex_unlock(&lockA);
```

#### Common Pitfalls

> **⚠️ Warning:** Deadlock Risk
> If one thread locks A then B, but another locks B then A, deadlock occurs. Always use consistent lock ordering across all threads.

> **⚠️ Warning:** Data Races
> Forgetting to lock before accessing shared variables causes unpredictable behavior. The program may pass sometimes and fail others.

#### Best Practices

> **💡 Hint:** Consistent Lock Order
> Define a global lock order (e.g., lockA before lockB) and enforce it everywhere. Document this order in comments.

> **💡 Hint:** Minimize Critical Sections
> Hold locks only while accessing/modifying shared data. Release locks before long operations (delays, I/O).

> **💡 Hint:** Monitor Reads
> Even read-only operations should acquire locks to ensure consistent snapshots (all threads see coherent state).

#### Testing Strategy

```bash
# Single run (may hide race conditions)
./pthread_locks

# Multiple runs (catches intermittent issues)
for i in {1..20}; do ./pthread_locks || break; done

# With timing instrumentation
./optional_test
```

#### Key Takeaway

**Synchronization ensures correctness in multithreaded programs.** Proper mutex usage prevents race conditions and deadlock through consistent lock ordering and minimal critical sections.
