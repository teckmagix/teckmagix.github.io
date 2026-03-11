---
layout: page
title: "Lab Predictions - lab-predictions-747tr"
lab: lab-predictions-747tr
description: "Lab predictions - lab-predictions-747tr"
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
  struct thread *th, *next_th;
  int has_sleeping = 0;
  next_th = 0;
  th = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(th >= all_thread + MAX_THREAD)
      th = all_thread;
    if(th->state == RUNNABLE) {
      next_th = th;
      break;
    }
    if(th->state == SLEEPING)
      has_sleeping = 1;
    th = th + 1;
  }

  if (next_th == 0) {
    if (has_sleeping == 1){
      printf("only thread sleepings, wake them all\n");
      int first_found = 0;
      for(int i = 0; i < MAX_THREAD; i++){ 
        if(all_thread[i].state == SLEEPING){
          if (first_found == 0){
            next_th = &all_thread[i];
            all_thread[i].state = RUNNABLE;
            first_found = 1; 
          } else {
            all_thread[i].state = RUNNABLE;
          }
        }
      }
    }
    else{
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  }

  if (current_thread != next_th) {
    next_th->state = RUNNING;
    th = current_thread;
    current_thread = next_th;
    thread_switch(&th->context, &current_thread->context);
  } else
    next_th = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *tp;

  for (tp = all_thread; tp < all_thread + MAX_THREAD; tp++) {
    if (tp->state == FREE) break;
  }
  tp->state = RUNNABLE;
  tp->context.sp = (uint64)&tp->stack[STACK_SIZE-1];
  tp->context.ra = (uint64)(*func);
}

void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}
```
{% endraw %}
