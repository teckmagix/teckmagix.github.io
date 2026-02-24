---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Lab Extension Task: pthread Deadlock & Race Condition

### Overview

This task requires fixing a buggy pthread-based C program that has race conditions and potential deadlock issues. You must add missing mutex lock/unlock calls to synchronize access to shared resources without introducing deadlocks.

---

## Task Description

### Program Structure

The program simulates concurrent access to two shared global resources:

| Component | Purpose |
|-----------|---------|
| **resourceA** | Shared resource protected by `lockA` |
| **resourceB** | Shared resource protected by `lockB` |
| **worker_AB** | Thread that updates A then B |
| **worker_BA** | Thread that updates B then A |
| **worker_monitor** | Thread that periodically reads both resources and signals termination via `done` flag |

### Key Issues to Fix

1. **Race Conditions** — Shared variables accessed without synchronization:
   - `resourceA`
   - `resourceB`
   - `total_ops`
   - `done`

2. **Deadlock Risk** — Incorrect lock ordering can cause deadlock between `worker_AB` and `worker_BA`

3. **Inconsistent Reads** — Monitor thread must read resources atomically

> **⚠️ Warning:** Adding locks in different orders in different threads (e.g., `worker_AB` locks A then B, but `worker_BA` locks B then A) **will cause deadlock**. You must enforce a consistent locking discipline across all threads.

---

## Task Requirements

### Constraints

- ✅ You **may only use** the two provided mutexes: `lockA` and `lockB`
- ❌ Do **not** add new mutexes, semaphores, or condition variables
- ✅ Must fix all data races on `resourceA`, `resourceB`, `total_ops`, `done`
- ✅ Must never deadlock and always terminate
- ✅ Monitor's snapshots must be taken safely (consistent reads)

### Correctness Requirements

Your fixed program must satisfy:

1. **No Data Races** — All shared variables protected by appropriate locks
2. **No Deadlock** — Program always terminates without hanging
3. **Consistency** — Monitor snapshots read resources atomically

> **💡 Hint:** Enforce a **consistent global lock ordering**: always acquire `lockA` before `lockB`, and always release in reverse order (unlock `lockB` before `lockA`). This prevents circular wait conditions.

---

## Solution Approach

### Locking Discipline Strategy

To prevent deadlock, follow this principle:

**Always acquire locks in the same order and release in reverse order:**

```
Acquire order: lockA → lockB
Release order: lockB → lockA
```

### Protected Sections

| Thread | Critical Section | Locks Needed |
|--------|------------------|--------------|
| **worker_AB** | Read/Update A, then Read/Update B | `lockA`, then `lockB` |
| **worker_BA** | Read/Update B, then Read/Update A | Must still use `lockA` then `lockB` (NOT B then A!) |
| **worker_monitor** | Read A, Read B, possibly update `done` | Both `lockA` and `lockB` for consistent snapshot |
| **Any thread** | Reading/Writing `total_ops`, `done` | Protected by one of the locks or both |

### Pseudo-code Pattern

```c
// Pattern for worker_AB (updates A then B)
pthread_mutex_lock(&lockA);
  // ... update resourceA ...
  pthread_mutex_lock(&lockB);
    // ... update resourceB ...
  pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);

// Pattern for worker_BA (MUST still respect lock order!)
pthread_mutex_lock(&lockA);  // NOT lockB first!
  pthread_mutex_lock(&lockB);
    // ... update resourceB ...
  pthread_mutex_unlock(&lockB);
  // ... update resourceA ...
pthread_mutex_unlock(&lockA);

// Pattern for monitor (consistent read)
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);
  int a = resourceA;
  int b = resourceB;
  // ... snapshot taken ...
pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

> **💡 Hint:** Even though `worker_BA` logically wants to update B first, it must still acquire locks in order A→B. This ensures consistency and prevents deadlock.

---

## Testing

### Build and Run

```bash
cd notxv6/
make pthread_locks
./pthread_locks
```

### Expected Output

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

### Test with Optional Script

```bash
./optional_test
```

This runs the binary 20 times to catch intermittent race conditions and deadlocks.

### Test with Grade Script

```bash
cd ..
./grade-quiz pthread_locks
```

Or with full make:

```bash
make clean && make grade
```

> **⚠️ Warning:** The grade script may not catch all possible issues. You must perform additional testing to ensure robustness. Run the program multiple times and under various conditions.

---

## Submission Checklist

- [ ] Fixed all race conditions with proper mutex locks/unlocks
- [ ] Enforced consistent lock ordering (no deadlock)
- [ ] Program runs without hanging
- [ ] `make grade` passes
- [ ] Tested with `optional_test` script successfully
- [ ] Created zip file with `make zipball` (not Windows zip)
- [ ] Submitted `lab.zip` to Gradescope
- [ ] Performed additional manual testing beyond automated tests

> **💡 Hint:** Submit early to the autograder for verification before the deadline. The autograder will regrade with different test cases after the due date.

---

## Key Takeaways

| Concept | Key Point |
|---------|-----------|
| **Lock Ordering** | Always acquire in same order; prevents circular wait |
| **Deadlock Prevention** | Reverse release order; lock hierarchy must be total |
| **Race Condition** | Protected every shared variable access with appropriate lock |
| **Atomicity** | Monitor's snapshot must read both A and B under same lock set |
| **Testing** | Run multiple times; race conditions are intermittent by nature |
