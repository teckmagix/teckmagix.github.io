---
layout: page	
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Lab Extension Task: pthread Deadlock & Race Condition Fix

## Task Overview

Fix a buggy pthread program by adding missing mutex lock/unlock calls to prevent race conditions and deadlocks when accessing shared resources `resourceA` and `resourceB`.

### Key Points
- **Three threads**: `worker_AB` (updates A→B), `worker_BA` (updates B→A), `worker_monitor` (reads and adjusts)
- **Shared variables**: `resourceA`, `resourceB`, `total_ops`, `done`
- **Available mutexes**: `lockA`, `lockB` only
- **Critical constraint**: Must maintain consistent lock ordering to prevent deadlock

## Solution Approach

### Critical Rule: Lock Ordering
Always acquire locks in the same order across all threads to prevent deadlock:
- **Lock order**: Always lock `lockA` before `lockB`
- **Unlock order**: Always unlock in reverse (unlock `lockB` before `lockA`)

```c
// CORRECT pattern (prevents deadlock)
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
// ... access resourceA and resourceB ...
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);

// WRONG pattern (can cause deadlock)
pthread_mutex_lock(&lockB);
pthread_mutex_lock(&lockA);
// ... access resourceA and resourceB ...
```

## Implementation Guide

### Step 1: Protect Single Resource Access

```c
// When accessing only resourceA:
pthread_mutex_lock(&lockA);
resourceA = new_value;  // or read it
pthread_mutex_unlock(&lockA);

// When accessing only resourceB:
pthread_mutex_lock(&lockB);
resourceB = new_value;  // or read it
pthread_mutex_unlock(&lockB);
```

### Step 2: Protect Multi-Resource Access

```c
// When accessing BOTH resourceA and resourceB:
pthread_mutex_lock(&lockA);      // Lock first
pthread_mutex_lock(&lockB);      // Lock second
// Now safely access both resourceA and resourceB
resourceA = some_value;
resourceB = some_value;
pthread_mutex_unlock(&lockB);    // Unlock in reverse order
pthread_mutex_unlock(&lockA);
```

### Step 3: Protect Global Variables

```c
// Protect total_ops (shared counter):
pthread_mutex_lock(&lockA);  // Use any lock for consistency
total_ops++;
pthread_mutex_unlock(&lockA);

// Protect done flag:
pthread_mutex_lock(&lockA);  // Use same lock as above
done = 1;
pthread_mutex_unlock(&lockA);
```

## Code Pattern for Each Thread

### worker_AB Thread
```c
void* worker_AB(void* arg) {
    // ... loop ...
    pthread_mutex_lock(&lockA);
    pthread_mutex_lock(&lockB);
    
    resourceA++;  // Update A
    resourceB++;  // Then B
    
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);
    
    pthread_mutex_lock(&lockA);
    total_ops++;
    pthread_mutex_unlock(&lockA);
    // ... loop end ...
}
```

### worker_BA Thread
```c
void* worker_BA(void* arg) {
    // ... loop ...
    pthread_mutex_lock(&lockA);  // IMPORTANT: Still lock A first
    pthread_mutex_lock(&lockB);  // Then B (same order as worker_AB)
    
    resourceB++;  // Update B first logically, but locks acquired in order A→B
    resourceA++;  // Then A
    
    pthread_mutex_unlock(&lockB);
    pthread_mutex_unlock(&lockA);
    
    pthread_mutex_lock(&lockA);
    total_ops++;
    pthread_mutex_unlock(&lockA);
    // ... loop end ...
}
```

### worker_monitor Thread
```c
void* worker_monitor(void* arg) {
    while (1) {
        pthread_mutex_lock(&lockA);
        pthread_mutex_lock(&lockB);
        
        // Safely read both
        int a = resourceA;
        int b = resourceB;
        
        // Possibly adjust (must hold locks)
        if (a > THRESHOLD) resourceA = THRESHOLD;
        if (b > THRESHOLD) resourceB = THRESHOLD;
        
        pthread_mutex_unlock(&lockB);
        pthread_mutex_unlock(&lockA);
        
        pthread_mutex_lock(&lockA);
        if (/* stop condition */) done = 1;
        pthread_mutex_unlock(&lockA);
        
        sleep(1);  // Outside critical section
    }
}
```

## Testing Commands

```bash
# Build the program
$ cd notxv6/
$ make pthread_locks

# Run once
$ ./pthread_locks

# Run optional test script (20 iterations)
$ ./optional_test

# Run grading script
$ cd ..
$ ./grade-quiz pthread_locks

# Full make grade test
$ make clean && make grade
```

## Checklist Before Submission

- [ ] All accesses to `resourceA` are protected by `lockA`
- [ ] All accesses to `resourceB` are protected by `lockB`
- [ ] All accesses to `total_ops` use consistent locking
- [ ] All accesses to `done` use consistent locking
- [ ] Lock order is always `lockA` → `lockB` (never reversed)
- [ ] All `pthread_mutex_lock()` have matching `pthread_mutex_unlock()`
- [ ] Program passes `./optional_test` (20 runs without race conditions)
- [ ] Program passes `./grade-quiz pthread_locks`
- [ ] Run `make grade` successfully
- [ ] Create zip with `make zipball` (not Windows zip)
- [ ] Submit `lab.zip` to Gradescope

> **⚠️ Warning:** Do NOT lock the same mutex twice in sequence without unlocking—this causes self-deadlock. Do NOT acquire locks in different orders in different threads—this causes mutual deadlock.

> **💡 Hint:** If the program still intermittently fails after adding locks, check that you're using the same lock order everywhere. Even one thread breaking the order can cause deadlock.
