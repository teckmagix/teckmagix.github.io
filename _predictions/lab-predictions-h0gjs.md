---
layout: page
title: "Lab Predictions - lab-predictions-h0gjs"
lab: lab-predictions-h0gjs
description: "Lab predictions - lab-predictions-h0gjs"
---

{% raw %}
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "user/uthread.h"

struct thread tbl_threads[MAX_THREAD];
struct thread *cur_thread;

void 
thread_init(void)
{
  // main() is thread 0, which will make the first invocation to
  // thread_schedule(). It needs a stack so that the first thread_switch() can
  // save thread 0's state.
  cur_thread = &tbl_threads[0];
  cur_thread->state = RUNNING;
}

void 
thread_sleep()
{
  cur_thread->state = SLEEPING;
  thread_schedule();
}

int 
thread_wakeup(int tid)
{
  if (tid < 0 || tid >= MAX_THREAD)
    return -1;
  if (tbl_threads[tid].state != SLEEPING)
    return -1;
  tbl_threads[tid].state = RUNNABLE;
  return 0;
}

void thread_schedule(void)
{
  struct thread *tp, *nxt_thread;
  int has_sleeping = 0;
  /* Find another runnable thread. */
  nxt_thread = 0;
  tp = cur_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(tp >= tbl_threads + MAX_THREAD)
      tp = tbl_threads;
    if(tp->state == RUNNABLE) {
      nxt_thread = tp;
      break;
    }
    // identify that there are sleeping threads
    if(tp->state == SLEEPING){
      has_sleeping = 1;
    }
    tp = tp + 1;
  }

  if (nxt_thread == 0) {
    // No runnable thread. If there is a sleeping thread, wake it up. Otherwise, panic.
    if (has_sleeping == 1){
      printf("only thread sleepings, wake them all\n");
      // we use first_slp to make sure only the first sleeping thread is set to RUNNING,
      // and the rest are set to RUNNABLE, so that they can be scheduled in the future.
      int first_slp = 0;
      for(int i = 0; i < MAX_THREAD; i++){ 
        if(tbl_threads[i].state == SLEEPING){
          if (first_slp == 0){
            nxt_thread = &tbl_threads[i];
            tbl_threads[i].state = RUNNABLE;
            first_slp = 1; 
          } else {
            tbl_threads[i].state = RUNNABLE;
          }
        }
      }
    }
    else{
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  }

  if (cur_thread != nxt_thread) {         /* switch threads?  */
    nxt_thread->state = RUNNING;
    tp = cur_thread;
    cur_thread = nxt_thread;
    thread_switch(&tp->context, &cur_thread->context);
  } else
    nxt_thread = 0;
}

void 
thread_create(void (*func_ptr)())
{
  struct thread *tp;

  for (tp = tbl_threads; tp < tbl_threads + MAX_THREAD; tp++) {
    if (tp->state == FREE) break;
  }
  tp->state = RUNNABLE;
  tp->context.sp = (uint64)&tp->stack[STACK_SIZE-1];
  tp->context.ra = (uint64)(*func_ptr);
}

void 
thread_yield(void)
{
  cur_thread->state = RUNNABLE;
  thread_schedule();
}
```
{% endraw %}
