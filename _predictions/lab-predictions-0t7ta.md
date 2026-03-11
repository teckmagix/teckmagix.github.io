---
layout: page
title: "Lab Predictions - lab-predictions-0t7ta"
lab: lab-predictions-0t7ta
description: "Lab predictions - lab-predictions-0t7ta"
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
  if(tid < 0 || tid >= MAX_THREAD)
    return -1;
  if(all_thread[tid].state != SLEEPING)
    return -1;
  all_thread[tid].state = RUNNABLE;
  return 0;
}

void thread_schedule(void)
{
  struct thread *tr, *nxt_thr;
  int dormant_count = 0;
  nxt_thr = 0;
  tr = current_thread + 1;
  for(int idx = 0; idx < MAX_THREAD; idx++){
    if(tr >= all_thread + MAX_THREAD)
      tr = all_thread;
    if(tr->state == RUNNABLE) {
      nxt_thr = tr;
      break;
    }
    if(tr->state == SLEEPING)
      dormant_count = 1;
    tr = tr + 1;
  }

  if (nxt_thr == 0) {
    if (dormant_count == 1){
      printf("only thread sleepings, wake them all\n");
      int initial_wake = 0;
      for(int idx = 0; idx < MAX_THREAD; idx++){ 
        if(all_thread[idx].state == SLEEPING){
          if (initial_wake == 0){
            nxt_thr = &all_thread[idx];
            nxt_thr->state = RUNNABLE;
            initial_wake = 1; 
          } else {
            all_thread[idx].state = RUNNABLE;
          }
        }
      }
    }
    else{
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  }

  if (current_thread != nxt_thr) {
    nxt_thr->state = RUNNING;
    tr = current_thread;
    current_thread = nxt_thr;
    thread_switch(&tr->context, &current_thread->context);
  } else
    nxt_thr = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *tr;

  for (tr = all_thread; tr < all_thread + MAX_THREAD; tr++) {
    if (tr->state == FREE) break;
  }
  tr->state = RUNNABLE;
  tr->context.sp = (uint64)&tr->stack[STACK_SIZE-1];
  tr->context.ra = (uint64)(*func);
}

void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}
```
{% endraw %}
