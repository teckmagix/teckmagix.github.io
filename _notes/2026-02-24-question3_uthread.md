---
layout: page
title: "Question3 Uthread"
date: 2026-02-24
---

## Overview

This lab task extends the xv6 user-thread library to support **thread sleeping and waking up**. You will implement `thread_sleep()` and `thread_wakeup(int thread_id)` in `uthread.c`, and modify `thread_schedule()` to handle the case where all threads are sleeping.

---

## Understanding the Existing Code Structure

Before implementing, you need to understand the existing user-thread library structure.

### Key Files

| File | Purpose |
|------|---------|
| `uthread.c` | Main user-thread library implementation |
| `uthread.h` | Header file with thread struct and state definitions |
| `uthread_switch.s` | Assembly context-switch routine |
| `uthread_ex0.c` – `uthread_ex3.c` | Test programs using the library |

### Thread States (already in uthread.h)

The existing implementation has two states. We need to add a third:

```c
// Existing states
#define FREE       0
#define RUNNING    1
#define RUNNABLE   2

// NEW state you need to add (if not already present)
#define SLEEPING   3
```

> **⚠️ Warning:** Check `uthread.h` carefully. The `SLEEPING` state constant may already be defined in the provided zip. If not, add it yourself.

### Typical Thread Struct (uthread.h)

```c
struct thread {
  int        sp;                   /* saved stack pointer */
  char       stack[STACK_SIZE];    /* thread stack */
  int        state;                /* thread state: FREE, RUNNING, RUNNABLE, SLEEPING */
};
```

---

## Task 1 — Implementing `thread_sleep()`

### What it must do

- Set the **current running thread's state** to `SLEEPING`
- Call the **scheduler** so another thread gets CPU time
- The sleeping thread will not be picked again until it is woken up

### Step-by-Step Implementation

```c
void
thread_sleep(void)
{
  // Step 1: Set the current thread's state to SLEEPING
  // 'current_thread' is a pointer to the currently running thread
  current_thread->state = SLEEPING;

  // Step 2: Yield the CPU — call the scheduler to run another thread
  // thread_schedule() will pick the next RUNNABLE thread
  thread_schedule();
}
```

> **💡 Hint:** `current_thread` is a global pointer already maintained by the library. After setting the state to `SLEEPING`, calling `thread_schedule()` ensures the sleeping thread is skipped when the scheduler loops through threads looking for `RUNNABLE` ones.

---

## Task 2 — Implementing `thread_wakeup(int thread_id)`

### What it must do

- Validate that `thread_id` is a valid index
- Check that the target thread is **currently SLEEPING**
- If both conditions are true → set its state to `RUNNABLE`
- The **caller continues execution** (no context switch happens)
- Return `0` on success, `-1` on failure

### Step-by-Step Implementation

```c
int
thread_wakeup(int thread_id)
{
  // Step 1: Validate thread_id bounds
  // MAX_THREAD is the maximum number of threads defined in uthread.h
  if (thread_id < 0 || thread_id >= MAX_THREAD) {
    return -1;  // Invalid thread index
  }

  // Step 2: Check that the target thread is actually SLEEPING
  if (all_thread[thread_id].state != SLEEPING) {
    return -1;  // Thread is not sleeping, cannot wake it
  }

  // Step 3: Wake the thread by making it RUNNABLE
  all_thread[thread_id].state = RUNNABLE;

  // Step 4: Caller continues — NO call to thread_schedule() here
  return 0;  // Success
}
```

> **⚠️ Warning:** Do **not** call `thread_schedule()` inside `thread_wakeup()`. The specification says "the caller will continue its execution, i.e., no other thread is scheduled." A context switch here would be wrong.

> **💡 Hint:** `all_thread[]` is the global array of all thread structs. `MAX_THREAD` defines the size of this array — check `uthread.h` for the exact constant name (it may be `MAX_UTHREADS` or similar).

---

## Task 3 — Modifying `thread_schedule()`

### What it must do (new behaviour)

When the scheduler loops through all threads and finds **no RUNNABLE thread**, it must check if any threads are **SLEEPING**. If so:
1. Wake **all** sleeping threads (set them to `RUNNABLE`)
2. Print the message: `"only thread sleepings, wake them all"`
3. Then run the thread with the **smallest `thread_id`** among the newly woken threads

### Understanding the Existing `thread_schedule()`

The typical existing scheduler looks like this:

```c
void
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;  // Start searching from the thread AFTER current

  for (int i = 0; i < MAX_THREAD; i++) {
    // Wrap around the array
    if (t >= all_thread + MAX_THREAD)
      t = all_thread;

    if (t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    // <<< THIS IS WHERE YOU ADD THE SLEEPING WAKE-UP LOGIC >>>
    printf("thread_schedule: no runnable threads\n");
    exit(0);
  }

  /* Switch to next_thread */
  if (current_thread != next_thread) {
    next_thread->state = RUNNING;
    struct thread *old_thread = current_thread;
    current_thread = next_thread;
    thread_switch((uint64)&old_thread->sp, (uint64)&next_thread->sp);
  }
}
```

### Modified `thread_schedule()` with Sleep Wake-Up Logic

```c
void
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;

  for (int i = 0; i < MAX_THREAD; i++) {
    if (t >= all_thread + MAX_THREAD)
      t = all_thread;

    if (t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    /*
     * PLACEHOLDER 1:
     * No RUNNABLE thread found. Check if any threads are SLEEPING.
     * If yes, wake them all up and pick the one with the smallest thread_id.
     */

    // Step A: Check if any thread is sleeping
    int any_sleeping = 0;
    for (int i = 0; i < MAX_THREAD; i++) {
      if (all_thread[i].state == SLEEPING) {
        any_sleeping = 1;
        break;
      }
    }

    if (any_sleeping) {
      // Step B: Wake ALL sleeping threads
      printf("only thread sleepings, wake them all\n");
      for (int i = 0; i < MAX_THREAD; i++) {
        if (all_thread[i].state == SLEEPING) {
          all_thread[i].state = RUNNABLE;
        }
      }

      // Step C: Pick the thread with the SMALLEST thread_id (lowest index)
      for (int i = 0; i < MAX_THREAD; i++) {
        if (all_thread[i].state == RUNNABLE) {
          next_thread = &all_thread[i];
          break;  // Smallest index = first RUNNABLE found from index 0
        }
      }
    }

    // Step D: If still no thread found after waking, truly exit
    if (next_thread == 0) {
      printf("thread_schedule: no runnable threads\n");
      exit(0);
    }
  }

  /* Switch to next_thread */
  if (current_thread != next_thread) {
    next_thread->state = RUNNING;
    struct thread *old_thread = current_thread;
    current_thread = next_thread;
    thread_switch((uint64)&old_thread->sp, (uint64)&next_thread->sp);
  }
}
```

> **💡 Hint:** The "smallest `thread_id`" means the thread at the **lowest array index** in `all_thread[]`. Scanning from index `0` upward and taking the first `RUNNABLE` one achieves this.

> **⚠️ Warning:** Make sure you print `"only thread sleepings, wake them all"` **before** setting states to `RUNNABLE`, or at least before the context switch. The expected output shows this message appearing at the right moment.

---

## Complete `uthread.c` — Summary of All Changes

Here is a consolidated view of what your final `uthread.c` additions look like:

```c
/* =========================================================
   thread_sleep()
   - Sets current thread to SLEEPING
   - Calls scheduler to yield CPU
   ========================================================= */
void
thread_sleep(void)
{
  current_thread->state = SLEEPING;
  thread_schedule();
}


/* =========================================================
   thread_wakeup(int thread_id)
   - Wakes a SLEEPING thread by thread_id
   - Returns 0 on success, -1 on failure
   - Caller CONTINUES executing (no context switch)
   ========================================================= */
int
thread_wakeup(int thread_id)
{
  if (thread_id < 0 || thread_id >= MAX_THREAD)
    return -1;

  if (all_thread[thread_id].state != SLEEPING)
    return -1;

  all_thread[thread_id].state = RUNNABLE;
  return 0;
}
```

---

## Expected Output Walkthrough

### `uthread_ex0` / `uthread_ex1` — Basic sleep/wakeup

```
thread_a started
thread_b started
thread_c started
thread_a 0
thread_a 1
thread_a 2
thread_a: exit after 3
thread_b 0
thread_b 1
thread_b 2
thread_b: exit after 3
thread_c 0
thread_c 1
thread_c 2
thread_c: exit after 3
thread_schedule: no runnable threads
```

**What's happening:**
- Threads take turns running
- `thread_sleep()` and `thread_wakeup()` are exercised
- All threads eventually complete → scheduler finds no runnable threads → exits

### `uthread_ex2` / `uthread_ex3` — All-sleeping wake-up

```
thread_a started
thread_b started
thread_c started
thread_a 0
thread_a 1
thread_a 2
thread_a: exit after 3
thread_b 0
thread_b 1
thread_b 2
thread_b: exit after 3
thread_c 0
thread_c 1
thread_c 2
only thread sleepings, wake them all    <-- NEW scheduler behaviour
thread_c: exit after 3
thread_schedule: no runnable threads
```

**What's happening:**
- At some point, all remaining threads are sleeping
- The modified scheduler detects this, prints the message, wakes them all
- The thread with the smallest `thread_id` (here `thread_c`) runs first

---

## Common Mistakes to Avoid

| Mistake | Why It's Wrong | Fix |
|--------|---------------|-----|
| Calling `thread_schedule()` in `thread_wakeup()` | Causes an unwanted context switch; caller should continue | Remove the schedule call |
| Not checking bounds in `thread_wakeup()` | May access invalid memory | Always validate `thread_id >= 0 && thread_id < MAX_THREAD` |
| Waking only one thread instead of all | Spec says "wake them ALL" | Loop through entire `all_thread[]` array |
| Picking a random sleeping thread instead of smallest `thread_id` | Spec says smallest `thread_id` runs first | Loop from index `0` upward |
| Forgetting to change state to `RUNNING` before switch | Thread will still appear `RUNNABLE` | `next_thread->state = RUNNING` before `thread_switch()` |

---

## Testing Your Implementation

### Step 1 — Build and Run in QEMU

```bash
make qemu
```

Then inside xv6:
```
$ uthread_ex0
$ uthread_ex1
$ uthread_ex2
$ uthread_ex3
```

### Step 2 — Run the Grade Script

```bash
cd ..
./grade-quiz uthread
```

Expected:
```
== Test uthreads == uthreads: OK (6.5s)
```

### Step 3 — Full Grade

```bash
make clean && make grade
```

Expected:
```
== Test uthread ==
$ make qemu-gdb
uthread: OK (6.7s)
Score: 7/7
```

### Step 4 — Create Submission Zip

```bash
make zipball
```

> **⚠️ Warning:** Always use `make zipball` to create the zip file. Do **not** use Windows Explorer, WinZip, or macOS Finder. These tools create nested directory structures that break the autograder.

Then submit `lab.zip` to Gradescope under **"xv6labs-quiz2-q3-uthread"**.

---

## Key Concepts Summary

| Concept | Description |
|---------|-------------|
| `thread_sleep()` | Sets state → `SLEEPING`, then yields via `thread_schedule()` |
| `thread_wakeup(id)` | Validates id, checks `SLEEPING` state, sets → `RUNNABLE`, returns 0 or -1 |
| Scheduler no-runnable case | If no `RUNNABLE` but some `SLEEPING` → wake all, run smallest id |
| Context switch | Only happens in `thread_schedule()` via `thread_switch()` assembly call |
| `current_thread` | Global pointer to the currently executing thread |
| `all_thread[]` | Global array of all thread structs, indexed by `thread_id` |
