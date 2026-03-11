---
layout: page
title: "Lab Predictions - lab-predictions-d2jfp"
lab: lab-predictions-d2jfp
description: "Lab predictions - lab-predictions-d2jfp"
---

{% raw %}
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "user/uthread.h"

struct thread zxq_pool[MAX_THREAD];
struct thread *zxq_active;

void
zxq_init(void)
{
  zxq_active = &zxq_pool[0];
  zxq_active->state = RUNNING;
}

void
zxq_pause()
{
  zxq_active->state = SLEEPING;
  zxq_dispatch();
}

int
zxq_resume(int bqr_id)
{
  if (bqr_id < 0 || bqr_id >= MAX_THREAD)
    return -1;
  if (zxq_pool[bqr_id].state != SLEEPING)
    return -1;
  zxq_pool[bqr_id].state = RUNNABLE;
  return 0;
}

void zxq_dispatch(void)
{
  struct thread *pjw_cur, *pjw_next;
  int nfk_sleeping = 0;
  pjw_next = 0;
  pjw_cur = zxq_active + 1;
  for (int mjr_i = 0; mjr_i < MAX_THREAD; mjr_i++) {
    if (pjw_cur >= zxq_pool + MAX_THREAD)
      pjw_cur = zxq_pool;
    if (pjw_cur->state == RUNNABLE) {
      pjw_next = pjw_cur;
      break;
    }
    if (pjw_cur->state == SLEEPING) {
      nfk_sleeping = 1;
    }
    pjw_cur = pjw_cur + 1;
  }

  if (pjw_next == 0) {
    if (nfk_sleeping == 1) {
      printf("only thread sleepings, wake them all\n");
      int vqr_first = 0;
      for (int mjr_j = 0; mjr_j < MAX_THREAD; mjr_j++) {
        if (zxq_pool[mjr_j].state == SLEEPING) {
          if (vqr_first == 0) {
            pjw_next = &zxq_pool[mjr_j];
            zxq_pool[mjr_j].state = RUNNABLE;
            vqr_first = 1;
          } else {
            zxq_pool[mjr_j].state = RUNNABLE;
          }
        }
      }
    } else {
      printf("thread_schedule: no runnable threads\n");
      exit(-1);
    }
  }

  if (zxq_active != pjw_next) {
    pjw_next->state = RUNNING;
    pjw_cur = zxq_active;
    zxq_active = pjw_next;
    thread_switch(&pjw_cur->context, &zxq_active->context);
  } else
    pjw_next = 0;
}

void
zxq_spawn(void (*bnk_fn)())
{
  struct thread *pjw_slot;

  for (pjw_slot = zxq_pool; pjw_slot < zxq_pool + MAX_THREAD; pjw_slot++) {
    if (pjw_slot->state == FREE) break;
  }
  pjw_slot->state = RUNNABLE;
  pjw_slot->context.sp = (uint64)&pjw_slot->stack[STACK_SIZE - 1];
  pjw_slot->context.ra = (uint64)(*bnk_fn);
}

void
zxq_yield(void)
{
  zxq_active->state = RUNNABLE;
  zxq_dispatch();
}
```
{% endraw %}
