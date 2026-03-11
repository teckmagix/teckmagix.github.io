---
layout: page
title: "Lab Predictions - lab-predictions-04h95"
lab: lab-predictions-04h95
description: "Lab predictions - lab-predictions-04h95"
---

{% raw %}
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
thread_wakeup(int tid)
{
  if (tid < 0 || tid >= MAX_THREAD)
    return -1;
  if (all_thread[tid].state != SLEEPING)
    return -1;
  all_thread[tid].state = RUNNABLE;
  return 0;
}

void thread_schedule(void)
{
  struct thread *ptr, *nxt_thd;
  int has_sleeping = 0;

  nxt_thd = 0;
  ptr = current_thread + 1;
  for (int idx = 0; idx < MAX_THREAD; idx++) {
    if (ptr >= all_thread + MAX_THREAD)
      ptr = all_thread;
    if (ptr->state == RUNNABLE) {
      nxt_thd = ptr;
      break;
    }
    if (ptr->state == SLEEPING)
      has_sleeping = 1;
    ptr = ptr + 1;
  }

  if (nxt_thd == 0) {
    if (has_sleeping == 1) {
      printf("only thread sleepings, wake them all\n");
      int first_found = 0;
      for (int idx = 0; idx < MAX_THREAD; idx++) {
        if (all_thread[idx].state == SLEEPING) {
          if (first_found == 0) {
            nxt_thd = &all_thread[idx];
            nxt_thd->state = RUNNABLE;
            first_found = 1;
          } else {
            all_thread[idx].state = RUNNABLE;
          }
        }
      }
    } else {
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  }

  if (current_thread != nxt_thd) {
    nxt_thd->state = RUNNING;
    ptr = current_thread;
    current_thread = nxt_thd;
    thread_switch(&ptr->context, &current_thread->context);
  } else
    nxt_thd = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *thr;

  for (thr = all_thread; thr < all_thread + MAX_THREAD; thr++) {
    if (thr->state == FREE) break;
  }
  thr->state = RUNNABLE;
  thr->context.sp = (uint64)&thr->stack[STACK_SIZE - 1];
  thr->context.ra = (uint64)(*func);
}

void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}
```
{% endraw %}
