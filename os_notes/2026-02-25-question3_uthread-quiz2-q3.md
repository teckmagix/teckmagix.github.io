---
layout: page
title: "Batch 2026-02-25 — Question3 Uthread, Quiz2 Q3"
date: 2026-02-25
---

# Batch 2026-02-25 — Question3 Uthread, Quiz2 Q3

## File 1: Question3_uthread.pdf

> **💡 Hint:** You only need to modify `user/uthread.c`. The key file to edit is the one with the empty shells for `thread_sleep()`, `thread_wakeup()`, and the placeholders in `thread_schedule()`.

## File 2: quiz2-q3.zip — `user/uthread.c` (Complete Solution)

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "user/uthread.h"

struct thread all_thread[MAX_THREAD];
struct thread *current_thread;

void 
thread_init(void)
{
  // main() is thread 0, which will make the first invocation to
  // thread_schedule(). It needs a stack so that the first thread_switch() can
  // save thread 0's state.
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}

void 
thread_sleep()
{
  current_thread->state = SLEEPING;
  thread_schedule();
}

int 
thread_wakeup(int thread_id)
{
  // Validate thread_id: must be >= 0 and < MAX_THREAD
  if (thread_id < 0 || thread_id >= MAX_THREAD)
    return -1;
  // Only wake up if the thread is actually sleeping
  if (all_thread[thread_id].state == SLEEPING) {
    all_thread[thread_id].state = RUNNABLE;
    return 0;
  }
  return -1;
}

void thread_schedule(void)
{
  struct thread *t, *next_thread;
  int sleeping_thread = 0;
  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    // add code to identify that there are sleeping threads
    if(t->state == SLEEPING)
      sleeping_thread = 1;
    t = t + 1;
  }

  if (next_thread == 0) {
    // No runnable thread. If there is a sleeping thread, wake it up. Otherwise, panic.
    if (sleeping_thread == 1){
      printf("only thread sleepings, wake them all\n");
      // we use first_sleep to make sure only the first sleeping thread is set to RUNNING,
      // and the rest are set to RUNNABLE, so that they can be scheduled in the future.
      int first_sleep = 0;
      for(int i = 0; i < MAX_THREAD; i++){ 
        if(all_thread[i].state == SLEEPING){
          if (first_sleep == 0){
            next_thread = &all_thread[i];
            first_sleep = 1; 
          }
          // wake up all sleeping threads (set to RUNNABLE), except the first one
          // which will be set to RUNNING by the code below
          all_thread[i].state = RUNNABLE;
        }
      }
    }
    else{
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    thread_switch(&t->context, &current_thread->context);
  } else
    next_thread = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  t->context.sp = (uint64)&t->stack[STACK_SIZE-1];
  t->context.ra = (uint64)(*func);
}

void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}
```

**Explanation of changes:**

**`thread_sleep()`**: Sets the current thread's state to `SLEEPING`, then calls `thread_schedule()` to yield the CPU to another thread.

**`thread_wakeup(int thread_id)`**: Validates that `thread_id` is within range `[0, MAX_THREAD)` and that the target thread is in `SLEEPING` state; if so, sets it to `RUNNABLE` and returns `0`, otherwise returns `-1`.

**`thread_schedule()` — sleeping thread detection**: Added `if(t->state == SLEEPING) sleeping_thread = 1;` inside the loop to track if any sleeping threads exist.

**`thread_schedule()` — wake all sleeping threads**: The `if(1)` placeholder is replaced with `if(all_thread[i].state == SLEEPING)`. Inside, the first sleeping thread found is saved as `next_thread` (and `first_sleep` set to 1). All sleeping threads (including the first) are set to `RUNNABLE` — the first one will then be promoted to `RUNNING` by the `next_thread->state = RUNNING` line at the bottom of the scheduler.

> **💡 Hint:** Thread IDs correspond to their index in `all_thread[]`. Thread 1 = `all_thread[1]`, thread 2 = `all_thread[2]`, etc. (index 0 is the main thread).

> **⚠️ Warning:** Do NOT call `thread_schedule()` recursively after waking threads in the "wake all sleeping" branch — just set `next_thread` and let the existing switch logic at the bottom handle the context switch.

> **⚠️ Warning:** The `thread_wakeup` function must return `-1` for invalid IDs (negative or `>= MAX_THREAD`) AND for threads that are not in `SLEEPING` state (e.g., FREE, RUNNING, RUNNABLE threads). The caller will continue running after `thread_wakeup` — no context switch happens.
