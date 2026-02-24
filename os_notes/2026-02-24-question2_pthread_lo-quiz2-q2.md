---
layout: page
title: "Batch 2026-02-24 — Question2 Pthread Locks, Quiz2 Q2"
date: 2026-02-24
---

# Batch 2026-02-24 — Question2 Pthread Locks, Quiz2 Q2

## File 1: Question2_pthread_locks.pdf

### Task Summary

Fix a buggy pthread-based C program by adding **mutex lock/unlock calls** to eliminate:
- **Race conditions** on shared global variables (`resourceA`, `resourceB`, `total_ops`, `done`)
- **Deadlocks** from inconsistent lock ordering

**Constraints:**
- Only use the two provided mutexes: `lockA` (for resourceA) and `lockB` (for resourceB)
- No new mutexes, semaphores, or condition variables
- Do not change the algorithm

---

### Solution: Complete Fixed Code

```c
// notxv6/pthread_locks.c (FIXED VERSION)

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

    for (int i = 0; i < LOOP_COUNT && !done; i++) {
        pthread_mutex_lock(&lockA);
        resourceA++;
        pthread_mutex_lock(&lockB);
        resourceB++;
        total_ops += 2;
        ops += 2;
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);

        delay();
    }

    return make_result(p->thread_id, ops);
}

// -------------------- Worker 2: touches B then A --------------------

void* worker_BA(void* arg)
{
    thread_args_t* p = (thread_args_t*)arg;
    long ops = 0;

    for (int i = 0; i < LOOP_COUNT && !done; i++) {
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);
        resourceB += 2;
        pthread_mutex_unlock(&lockB);
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

    pthread_join(t1, (void**)&r1);
    pthread_join(t2, (void**)&r2);

    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    done = 1;
    pthread_mutex_unlock(&lockB);
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

---

### Key Fixes Explained

| Issue | Solution |
|-------|----------|
| **Race on resourceA/B** | Lock `lockA` before reading/writing `resourceA`; lock `lockB` before reading/writing `resourceB` |
| **Race on total_ops** | Increment inside both lock sections (protected by the resource locks) |
| **Race on done** | Read and write `done` inside locked section; write `done=1` in main with locks held |
| **Deadlock prevention** | **Consistent lock order: always acquire lockA before lockB; always release in reverse order (B then A)** |
| **Monitor consistency** | Acquire both locks for safe snapshot reads in `worker_monitor` |

> **💡 Hint:** The critical pattern is:
> ```c
> pthread_mutex_lock(&lockA);
> pthread_mutex_lock(&lockB);
> // critical section
> pthread_mutex_unlock(&lockB);
> pthread_mutex_unlock(&lockA);
> ```

> **⚠️ Warning:** Reversing lock order between threads causes deadlock. Always use **A→B** order everywhere.

---

### Testing

**Compilation & Execution:**
```bash
cd notxv6/
make pthread_locks
./pthread_locks
# Expected: deterministic end state with total_ops=80000 and done=1
```

**With stress test:**
```bash
./optional_test  # Runs binary 20 times to catch intermittent issues
```

**With grading script:**
```bash
./grade-quiz pthread_locks
make grade
```

---

## File 2: quiz2-q2.zip

This archive contains the complete lab framework including:

- **gradelib.py** — Test harness infrastructure
- **Kernel files** (bio.c, console.c, fs.c, etc.) — xv6 operating system components
- **User programs** (cat.c, echo.c, grep.c, etc.) — Test utilities
- **notxv6/pthread_locks.c** — The buggy program to fix (provided as template)

The archive is used during submission for automated grading via `make grade` and `make zipball`.
