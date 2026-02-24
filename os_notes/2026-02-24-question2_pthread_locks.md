---
layout: page
title: "Question2 Pthread Locks"
date: 2026-02-24
---

## Lab Extension Task: pthread Deadlock & Race Condition Fix (4 marks)

### Prerequisites

Download the "quiz2-q2.zip" file from:
- **Location:** LMS/xSiTe folder → "Lab Quizzes (Prerequisites) / Quiz 2 / Question 2"

---

## Task Overview

### Program Context

You are given a buggy pthread-based C program (`notxv6/pthread_locks.c`) that simulates access to shared global resources with synchronization issues.

### Program Architecture

#### Shared Resources
- **resourceA** - protected by `lockA` mutex
- **resourceB** - protected by `lockB` mutex

#### Thread Structure
The program contains three worker threads:

| Thread | Function | Operation |
|--------|----------|-----------|
| `worker_AB` | Updates A then B in sequence | Increments resourceA, then resourceB |
| `worker_BA` | Updates B then A in sequence | Increments resourceB, then resourceA |
| `worker_monitor` | Periodic reader and controller | Reads both resources, adjusts them, signals completion via `done` flag |

#### Data Passing Mechanism
- Threads receive parameters from `main()` via **struct pointer**
- Threads return results via **heap-allocated result struct**

### Current Issues

The program suffers from:
1. **Race conditions** - Shared variables accessed without synchronization:
   - `resourceA`
   - `resourceB`
   - `total_ops`
   - `done`

2. **Potential deadlocks** - If locks are added in incorrect/inconsistent order across threads, intermittent deadlocks can occur

---

## Task Requirements

### Objective
Add all missing `pthread_mutex_lock()` and `pthread_mutex_unlock()` calls to fix synchronization issues.

### Constraints

> **⚠️ Warning:** You have limited resources available!

- **Only** two provided mutexes available:
  - `lockA` (for resource A)
  - `lockB` (for resource B)
- Do **NOT** add new mutexes, semaphores, or condition variables
- All synchronization must use only these two locks

### Correctness Requirements

Your fixed program must satisfy:

1. **No data races** on:
   - `resourceA`
   - `resourceB`
   - `total_ops`
   - `done`

2. **No deadlocks**
   - Program must always terminate successfully
   - No circular lock dependencies

3. **Safe monitoring**
   - Monitor's printed snapshots must be taken safely (consistent read)
   - No partial reads across separate unlock/lock cycles

> **💡 Hint:** To avoid deadlock when multiple locks are needed, establish a consistent locking order across all threads. For example, always acquire `lockA` before `lockB`.

---

## Task Testing

### Environment
- **Platform:** Unix/Linux (NOT xv6)
- **Working directory:** `notxv6/`

### Test Method 1: Manual Execution

```bash
cd notxv6/
make pthread_locks
./pthread_locks
```

#### Expected Output

```
[monitor] A=29943 B=29943 total_ops=40000 done=0
[monitor] A=60000 B=60000 total_ops=80000 done=0
[monitor] A=60000 B=60000 total_ops=80000 done=1
--- Results (returned to main) ---
Thread 1 ops=40000 lastA=59248 lastB=59248 lastTotalOps=79248
lastDone=0
Thread 2 ops=40000 lastA=60000 lastB=60000 lastTotalOps=80000
lastDone=0
Thread 3 ops=0 lastA=60000 lastB=60000 lastTotalOps=80000 lastDone=1
Final Shared: A=60000 B=60000 total_ops=80000 done=1
Expected: total_ops=80000 done=1
```

### Test Method 2: Optional Shell Script

```bash
./optional_test
```

- Runs the binary **20 times** to catch intermittent race conditions
- Takes a few seconds to complete
- Expected output: `PASS`

### Test Method 3: Grading Script

```bash
cd ..
./grade-quiz pthread_locks
```

**Expected output:**
```
== Test pthread_locks_test == 
gcc -o pthread_locks -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/pthread_locks.c -pthread
pthread_locks_test: OK (28.1s)
```

### Test Method 4: Full Make Grade

```bash
make clean && make grade
```

**Expected output:**
```
== Test pthread_locks_test == 
make[1]: Entering directory '/mnt/c/ICT1012/xv6labs-w8-quiz2/q2_instructor'
gcc -o pthread_locks -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/pthread_locks.c -pthread
pthread_locks_test: OK (28.1s)
Score: 1/1
```

> **⚠️ Warning:** The grade script does NOT cover all edge cases! You are responsible for additional testing beyond the automated tests.

---

## Key Implementation Insights

### Locking Strategy

To prevent deadlock while protecting multiple resources:

1. **Establish a consistent lock order**
   - Example: Always acquire `lockA` before `lockB`
   - Reverse order when releasing: Release `lockB` before `lockA`

2. **Identify critical sections**
   - Any code that reads or modifies `resourceA` must hold `lockA`
   - Any code that reads or modifies `resourceB` must hold `lockB`
   - Any code that accesses both must acquire both locks in consistent order

3. **Protect compound operations**
   - If checking `done` and then modifying resources, must hold locks for entire sequence
   - If reading for monitoring, snapshot all values under lock

### Example Pattern

```c
// Safe multi-resource access
pthread_mutex_lock(&lockA);
pthread_mutex_lock(&lockB);

// Modify resourceA and resourceB
resourceA++;
resourceB++;

pthread_mutex_unlock(&lockB);
pthread_mutex_unlock(&lockA);
```

---

## Quiz Submission

### Step 1: Prepare Your Submission

After fixing the code and passing all tests:

```bash
make zipball
```

This creates `lab.zip` in the project root.

> **⚠️ Warning:** Use `make zipball` command, NOT Windows Zip or iOS Files App, to avoid creating multiple hierarchical directories.

### Step 2: Submit to Gradescope

1. Navigate to Gradescope
2. Find assignment: **"xsv6labs-quiz2-q2-pthread"**
3. Upload your `lab.zip` file
4. Autograder will run `make grade` for verification

### Step 3: Important Notes

- **Autograding is verification only** - Your score will NOT be included in the total score
- **Regrade after deadline** - Different test cases will be used after the quiz deadline
- **Submit early** - Avoid last-minute technical issues by submitting as early as possible
- **Your testing responsibility** - Test cases may not cover all edge cases; ensure your code is robust

---

## Common Pitfalls to Avoid

| Issue | Solution |
|-------|----------|
| Acquiring locks in different orders across threads | Establish one consistent lock order globally |
| Forgetting to unlock after lock | Use matching lock/unlock pairs; consider using helper macros |
| Holding locks longer than necessary | Minimize critical section size |
| Partial reads without full lock coverage | Hold all necessary locks when reading multiple related variables |
| Modifying shared variables outside locks | Identify ALL shared state and protect with locks |

---

## Summary Checklist

- [ ] Downloaded `quiz2-q2.zip` from LMS
- [ ] Identified all shared variables: `resourceA`, `resourceB`, `total_ops`, `done`
- [ ] Established consistent lock acquisition order
- [ ] Added `pthread_mutex_lock()` before accessing shared variables
- [ ] Added `pthread_mutex_unlock()` after accessing shared variables
- [ ] Tested with `./pthread_locks` - no crashes or hangs
- [ ] Tested with `./optional_test` - all 20 runs pass
- [ ] Tested with `./grade-quiz pthread_locks` - OK status
- [ ] Tested with `make grade` - Score 1/1
- [ ] Created `lab.zip` with `make zipball`
- [ ] Submitted `lab.zip` to Gradescope
