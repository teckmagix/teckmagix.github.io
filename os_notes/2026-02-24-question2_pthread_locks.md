---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Task: Fix pthread Deadlock & Race Condition

## Objective
Add missing `pthread_mutex_lock()` and `pthread_mutex_unlock()` calls to synchronize access to shared resources and prevent race conditions and deadlocks.

## Key Requirements

| Requirement | Details |
|-------------|---------|
| **Shared Resources** | resourceA (lockA), resourceB (lockB), total_ops, done |
| **Threads** | worker_AB (A→B), worker_BA (B→A), worker_monitor (read & signal) |
| **Mutexes Available** | Only lockA and lockB |
| **Must Achieve** | No data races, no deadlocks, always terminates |

## Solution Structure

### Critical Locking Rules
1. **Consistent lock ordering** — Always acquire locks in the same order (e.g., lockA before lockB) to prevent deadlock
2. **Protect all shared variable access** — resourceA, resourceB, total_ops, done
3. **Monitor reads safely** — Lock both resources when reading for consistent snapshots

### Annotated Solution Pattern

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

pthread_mutex_t lockA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lockB = PTHREAD_MUTEX_INITIALIZER;

int resourceA = 0;
int resourceB = 0;
int total_ops = 0;
int done = 0;

struct thread_result {
    int ops;
    int lastA, lastB, lastTotalOps, lastDone;
};

struct thread_args {
    int target_ops;
};

// worker_AB: updates A then B (lock order: A, B)
void* worker_AB(void *arg) {
    struct thread_args *args = (struct thread_args *)arg;
    struct thread_result *result = malloc(sizeof(struct thread_result));
    int ops = 0;
    
    while (1) {
        pthread_mutex_lock(&lockA);        // LOCK A FIRST
        pthread_mutex_lock(&lockB);        // THEN LOCK B
        
        if (done) {
            result->lastA = resourceA;
            result->lastB = resourceB;
            result->lastTotalOps = total_ops;
            result->lastDone = done;
            pthread_mutex_unlock(&lockB);
            pthread_mutex_unlock(&lockA);
            break;
        }
        
        resourceA++;
        resourceB++;
        total_ops++;
        ops++;
        
        pthread_mutex_unlock(&lockB);      // UNLOCK B FIRST
        pthread_mutex_unlock(&lockA);      // THEN UNLOCK A
        
        if (ops >= args->target_ops) {
            pthread_mutex_lock(&lockA);
            pthread_mutex_lock(&lockB);
            result->lastA = resourceA;
            result->lastB = resourceB;
            result->lastTotalOps = total_ops;
            result->lastDone = done;
            pthread_mutex_unlock(&lockB);
            pthread_mutex_unlock(&lockA);
            break;
        }
    }
    
    result->ops = ops;
    free(args);
    return result;
}

// worker_BA: updates B then A (lock order: STILL A, B to match AB)
void* worker_BA(void *arg) {
    struct thread_args *args = (struct thread_args *)arg;
    struct thread_result *result = malloc(sizeof(struct thread_result));
    int ops = 0;
    
    while (1) {
        pthread_mutex_lock(&lockA);        // LOCK A FIRST (consistent order)
        pthread_mutex_lock(&lockB);        // THEN LOCK B
        
        if (done) {
            result->lastA = resourceA;
            result->lastB = resourceB;
            result->lastTotalOps = total_ops;
            result->lastDone = done;
            pthread_mutex_unlock(&lockB);
            pthread_mutex_unlock(&lockA);
            break;
        }
        
        resourceB++;
        resourceA++;
        total_ops++;
        ops++;
        
        pthread_mutex_unlock(&lockB);      // UNLOCK B FIRST
        pthread_mutex_unlock(&lockA);      // THEN UNLOCK A
        
        if (ops >= args->target_ops) {
            pthread_mutex_lock(&lockA);
            pthread_mutex_lock(&lockB);
            result->lastA = resourceA;
            result->lastB = resourceB;
            result->lastTotalOps = total_ops;
            result->lastDone = done;
            pthread_mutex_unlock(&lockB);
            pthread_mutex_unlock(&lockA);
            break;
        }
    }
    
    result->ops = ops;
    free(args);
    return result;
}

// worker_monitor: reads both, signals done (lock order: A, B)
void* worker_monitor(void *arg) {
    struct thread_result *result = malloc(sizeof(struct thread_result));
    int ops = 0;
    
    while (1) {
        sleep(1);
        
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);
        
        printf("[monitor] A=%d B=%d total_ops=%d done=%d\n",
               resourceA, resourceB, total_ops, done);
        
        // Conditionally set done
        if (total_ops >= 80000) {
            done = 1;
        }
        
        result->lastA = resourceA;
        result->lastB = resourceB;
        result->lastTotalOps = total_ops;
        result->lastDone = done;
        
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
        
        if (done) {
            break;
        }
    }
    
    result->ops = ops;
    return result;
}

int main() {
    pthread_t t1, t2, t3;
    struct thread_args *args1 = malloc(sizeof(struct thread_args));
    struct thread_args *args2 = malloc(sizeof(struct thread_args));
    struct thread_result *res1, *res2, *res3;
    
    args1->target_ops = 40000;
    args2->target_ops = 40000;
    
    pthread_create(&t1, NULL, worker_AB, args1);
    pthread_create(&t2, NULL, worker_BA, args2);
    pthread_create(&t3, NULL, worker_monitor, NULL);
    
    pthread_join(t1, (void**)&res1);
    pthread_join(t2, (void**)&res2);
    pthread_join(t3, (void**)&res3);
    
    printf("--- Results (returned to main) ---\n");
    printf("Thread 1 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           res1->ops, res1->lastA, res1->lastB, res1->lastTotalOps, res1->lastDone);
    printf("Thread 2 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           res2->ops, res2->lastA, res2->lastB, res2->lastTotalOps, res2->lastDone);
    printf("Thread 3 ops=%d lastA=%d lastB=%d lastTotalOps=%d lastDone=%d\n",
           res3->ops, res3->lastA, res3->lastB, res3->lastTotalOps, res3->lastDone);
    
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    printf("Final Shared: A=%d B=%d total_ops=%d done=%d\n",
           resourceA, resourceB, total_ops, done);
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);
    
    free(res1); free(res2); free(res3);
    return 0;
}
```

This solution protects all shared data access and uses consistent lock ordering (A before B) to eliminate deadlocks.

## Testing Commands

```bash
# Compile and run
$ cd notxv6/
$ make pthread_locks
$ ./pthread_locks

# Run optional test (20 iterations)
$ ./optional_test

# Full grading test
$ cd ..
$ ./grade-quiz pthread_locks

# Make grade
$ make grade
```

> **💡 Hint:** The key to avoiding deadlock is **lock ordering discipline** — always acquire lockA before lockB in all threads, and always release in reverse order (B before A).

> **⚠️ Warning:** Deadlocks happen when threads acquire locks in different orders. Even if worker_AB updates A-then-B and worker_BA updates B-then-A, they must both lock in the same order: lockA first, then lockB.
